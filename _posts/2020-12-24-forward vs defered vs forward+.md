---
layout: post
title: 前向渲染、延迟渲染和Forward+
tags:
typora-root-url: ..\
---

项目最近计划要在移动平台上添加大量的动态光，计划使用Cluster Forward Plus，由于项目使用URP，别说CFP，延迟渲染现在还没被支持（据说之后Unity有支持的计划），所以需要我们自己实现。本文是对一篇网络论文的整理，有部分是自己的理解，但大部分还是作者的论述。



### 简介

前端渲染就是通过栅格化场景的多边形对象来工作，迭代场景的灯光列表以确定几何对象如何被照亮。这意味着每个灯光并且每个场景物体都要被处理。当然，我们能通过不渲染被遮罩的或者在摄像机视锥之外的几何对象来优化。如果光的范围是已知的，我们也可以在渲染场景几何之前通过光的体积上执行视锥剔除。如此通过剔除的技术其实优化有限而且难以实践。更常见的做法是简单地限制可以影响场景对象的灯光数量。例如一些图形引擎会对最近的两三个灯光执行逐像素光照，对次近的进行逐顶点光照。传统OpenGL和DirectX固定函数渲染管线的动态光在场景被激活的的数量任何时候都被限制为**8个**。甚至在现代图形硬件中，如果没有明显的帧率问题，前端渲染会被限制到**100个**动态场景光源。



延迟渲染，换句话说，是通过栅格化场景对象（没有光照）为一系列的图像缓存，用于保存执行后面的光照计算的几何信息。保存为2D图像缓存的有：

- 屏幕空间深度
- 表面Normal
- diffuse颜色
- specular颜色和次幂

这些二维图像缓冲区的组合称为几何缓冲区(G-Buffer)。Specular的Power项保存在Specular的Alpha通道，见下图的右上角。

![G-Buffer-900x506](/assets/postasset/2020-12-24-forward vs defered vs forward+/G-Buffer-900x506.jpg)



其他信息如果被光照计算并之后渲染，同样可以保存在图像缓存中。他们的G-buffer纹理HD（1080p）32位时至少占用8.29MB。

G-buffer创建后，几何信息能用来在光照通道里计算光照信息，每个像素被光的几何表现接触，就会通过期望的光照等式来着色。

延迟渲染技术相对前向渲染的明显好处就是，当在HD分辨率（1080p）只渲染不透明物体的时候，它能处理2500动态场景光照并且还没出现掉帧问题。



延迟渲染相对前向渲染有两个**缺点**：

1、延迟渲染只能渲染不透明物体。这是由于多个透明物体有可能会覆盖同一个像素，而G-buffer在一个像素仅仅只能存一个值。在这个光照pass中，深度值，表面normal，diffuse和specular颜色，都会被最近被照亮的屏幕像素采样。因为每一个G-buffer仅仅能被采样到一个值，透明物体是不能被这光照pass支持的。要避免这个问题，透明物体必须用标准的前向的渲染技术，要么限制一下透明几何物体的数量，要么限制一下动态光的数量。而不透明物体一般能处理2000盏灯而不丢帧。



2、延迟渲染只能使用单一光照模型。这是由于在渲染几何体的光照的时候，它只能绑定单一pixel shader。如果你用单一光照shader，这不是什么麻烦事，如果不同的pixel shader实现了不一样的光照模型，延迟渲染管线就是个大问题了。



Forward+（又称tiled forward shading）是结合前向渲染和基于tiled的光剔除，减少光的数量，这是着色阶段必须被关注的。Forward+主要由两个阶段组成：

1、光照剔除

2、前向渲染



![Forward--900x253](/assets/postasset/2020-12-24-forward vs defered vs forward+/Forward--900x253.jpg)

（上图为Forward+光照。默认光照（左边），光的热力图（右边）。热力图的颜色表明在一个tile中被多少个光影响。黑色的tiles不包含任何光，蓝色tiles包含1-10盏光。绿色tiles包含20-30盏光。）



Forward+渲染技术的第一个pass在屏幕空间使用统一的tiles格子，将光划分为每一个tile的列表。



第二个pass使用一个标准前向pass，在场景中进行着色，但不是循环每一个场景的动态光照，最近的像素的屏幕空间位置是被用于查找在前一个pass计算过的格子的光照列表。光照 剔除相较标准的前向渲染，提供一个极大的性能提升，因为它大大降低了那些必须被迭代来正确光照到对应像素。不透明和透明的几何体能被处理为一个类似的方式，没有重大的性能损耗和处理多个材质和光照模型，并且光照模型是自然地被Forward+支持。



因为Forwad+技术里包含标准前向渲染管线，Forward+能被整合为现存的使用前向构建的图形引擎中。Forward+不能使用 G-buffers和不受延迟渲染的限制。不透明物体和透明物体都能被Forward+渲染。使用现代图形硬件中，一个由5000-6000动态光组成的场景能在HD分辨率（1080p）中实时渲染。



在本文的剩余部分中，我会描述三点具体技术实现：

1、前向渲染

2、延迟渲染

3、Forward+（Tiles Forward Renderer）



同时我会显示基于不同情景的性能图表，以确定哪种条件下一种技术是优于另外的技术。



## 定义

下面将概况性地定义各个渲染的流程大纲：



前向渲染：

1、不透明pass

2、透明pass

不透明pass是以摄像机为目标，从近到远渲染，以降低overdraw。透明pass则是从远到近渲染，用于Alpha混合。



延迟渲染：

1、几何pass

2、光照pass

3、不透明pass

几何pass类似于前向的不透明pass，都是渲染不透明物体，只是它不进行光照计算，但它会几何信息写进G-buffer。而光照pass中，场景中代表光照几何物体和材质信息也会写进G-buffer。



Forward+：

1、光照剔除pass

2、不透明pass

3、透明pass

光照pass是对动态光在屏幕空间tiles中进行排序，光照索引列表则是对覆盖在屏幕tiles光照索引值。这个pass中会创建两组光照索引表：

1、不透明索引表

2、透明索引表

他们分别是用来渲染不透明和透明的光照。

而不透明pass和透明pass跟标准前向渲染是一样的，除了它只渲染剔除完毕的光。



本论文只支持：点光源、聚光灯、方向光，暂不支持区域光。点光源几何表现为球体，聚光灯为圆锥，方向光为全屏四边形。



## 前向渲染



前向渲染是大部分引擎的默认渲染管线（ue4默认是延迟）。这里简述一下通用的前向渲染的过程，不同引擎可能实现细节不一样。

前向渲染简单来说就是把场景所有光都遍历一遍进行渲染计算，所以它能使用的动态光是十分有限的，像UnityURP只支持8个额外光照，因为渲染压力实在是大。而一般引擎会离线光照来实现多光源的效果，譬如lightmapping和light probes，然而他们无法做动态光因为灯光信息在运行的时候已经被丢弃找不到了。

下面会描述前向渲染的实现细节，让它能跟延迟渲染和Forward+进行性能等数据对比。而且很多技术细节它都能跟延迟渲染和Forward+一样，比如vertex shader，或者是最终光照计算方法。



Vertex Shader：

顶点结构：

CommonInclude.hlsl

```c
struct AppData
{
    float3 position : POSITION;
    float3 tangent  : TANGENT;
    float3 binormal : BINORMAL;
    float3 normal   : NORMAL;
    float2 texCoord : TEXCOORD0;
};
```



position是物体空间坐标，texCoord是纹理坐标，normal为法线，tangent，binormal则是用在normal map，要么是美术在建模时创建，要么导入的时候用工具创建。



然后还有对顶点变换的MVP矩阵和用于计算物体在view space的MV矩阵，这里用cbuffer来储存：

CommonInclude.hlsl

```c
cbuffer PerObject : register( b0 )
{
    float4x4 ModelViewProjection;
    float4x4 ModelView;
}
```



然后是vertex shader的输出：

CommonInclude.hlsl

```c
struct VertexShaderOutput
{
    float3 positionVS   : TEXCOORD0;    // View space position.
    float2 texCoord     : TEXCOORD1;    // Texture coordinate
    float3 tangentVS    : TANGENT;      // View space tangent.
    float3 binormalVS   : BINORMAL;     // View space binormal.
    float3 normalVS     : NORMAL;       // View space normal.
    float4 position     : SV_POSITION;  // Clip space position.
};
```

VS后缀的意思是view space，在view space空间的物体，ps使用view space而不是世界空间，是由于延迟渲染和Forward+实现方便。

对于前向渲染，SV_POSITION是在clip space的位置，对于延迟渲染和Forward+则是在屏幕空间的位置。



ForwardRendering.hlsl

```c
VertexShaderOutput VS_main( AppData IN )
{
    VertexShaderOutput OUT;
 
    OUT.position = mul( ModelViewProjection, float4( IN.position, 1.0f ) );
 
    OUT.positionVS = mul( ModelView, float4( IN.position, 1.0f ) ).xyz;
    OUT.tangentVS = mul( ( float3x3 )ModelView, IN.tangent );
    OUT.binormalVS = mul( ( float3x3 )ModelView, IN.binormal );
    OUT.normalVS = mul( ( float3x3 )ModelView, IN.normal );
 
    OUT.texCoord = IN.texCoord;
 
    return OUT;
}
```



这里说一个关于矩阵的事。在DirectX10之后（之前都是右乘），DirectX在C++层面上，矩阵是用向量右乘的（row_major），而HLSL中矩阵则是左乘的（column_major），而OpenGL则都是左乘矩阵，左乘矩阵的标准，向量都是列向量，右乘矩阵则是行向量。个人更喜欢column_major，因为这样矩阵可读性比较高点，矩阵每一行代表一个分量（xyz）的变换结果。可参照[Docs](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-per-component-math)。



Pixel shader：

像素着色器的光照模型则是标准的 [Blinn-Phong](https://en.wikipedia.org/wiki/Blinn–Phong_shading_model)光照模型，这里不再赘述。



材质：

CommonInclude.hlsl

```c
struct Material
{
    float4  GlobalAmbient;
    //-------------------------- ( 16 bytes )
    float4  AmbientColor;
    //-------------------------- ( 16 bytes )
    float4  EmissiveColor;
    //-------------------------- ( 16 bytes )
    float4  DiffuseColor;
    //-------------------------- ( 16 bytes )
    float4  SpecularColor;
    //-------------------------- ( 16 bytes )
    // Reflective value.
    float4  Reflectance;
    //-------------------------- ( 16 bytes )
    float   Opacity;
    float   SpecularPower;
    // For transparent materials, IOR > 0.
    float   IndexOfRefraction;
    bool    HasAmbientTexture;
    //-------------------------- ( 16 bytes )
    bool    HasEmissiveTexture;
    bool    HasDiffuseTexture;
    bool    HasSpecularTexture;
    bool    HasSpecularPowerTexture;
    //-------------------------- ( 16 bytes )
    bool    HasNormalTexture;
    bool    HasBumpTexture;
    bool    HasOpacityTexture;
    float   BumpIntensity;
    //-------------------------- ( 16 bytes )
    float   SpecularScale;
    float   AlphaThreshold;
    float2  Padding;
    //--------------------------- ( 16 bytes )
};  //--------------------------- ( 16 * 10 = 160 bytes )
```

GlobalAmbient是全局环境光，一般来说它应该定义为全局变量。

Ambient, Emissive, Diffuse, Specular就是 [Blinn-Phong](https://en.wikipedia.org/wiki/Blinn–Phong_shading_model)的四个项的光照颜色。

Reflectance是反射项颜色，这里没有对其具体实现。

Opacity为不透明度。

SpecularPower为Phong模型的幂次项。

IndexOfRefraction是折射索引，也是没有实现。

HasTexture定义是否使用纹理。

BumpIntensity在HasBumpTexture为True的时候，使用Bump Mapping而不是Normal Map时的效果强度。

SpecularScale是用来放大Specular纹理幂次的值。

AlphaThreshold是用来在pixel shader时discard低于这个alpha值的。使用它能不使用AlphaBlend的情况下渲染没有半透明的透明物体。

Padding是一个8字节占位符，虽然HLSL本身会自动暗中处理这事情让结构体16字节的倍数，不过这里这么写只是为了看起来直观。



CommonInclude.hlsl

```c
cbuffer Material : register( b2 )
{
    Material Mat;
};
```

这些cbuffer会被文中所有的pixel shader使用。



纹理：

1. Ambient
2. Emissive
3. Diffuse
4. Specular
5. SpecularPower
6. Normals
7. Bump
8. Opacity

8张纹理，不是所有物体每张都在用。



CommonInclude.hlsl

```c
Texture2D AmbientTexture        : register( t0 );
Texture2D EmissiveTexture       : register( t1 );
Texture2D DiffuseTexture        : register( t2 );
Texture2D SpecularTexture       : register( t3 );
Texture2D SpecularPowerTexture  : register( t4 );
Texture2D NormalTexture         : register( t5 );
Texture2D BumpTexture           : register( t6 );
Texture2D OpacityTexture        : register( t7 );
```



灯光：

点光源、聚光灯、方向光用的都是同一个结构，使用Type变量区分。



CommonInclude.hlsl

```c
struct Light
{
    /**
    * Position for point and spot lights (World space).
    */
    float4   PositionWS;
    //--------------------------------------------------------------( 16 bytes )
    /**
    * Direction for spot and directional lights (World space).
    */
    float4   DirectionWS;
    //--------------------------------------------------------------( 16 bytes )
    /**
    * Position for point and spot lights (View space).
    */
    float4   PositionVS;
    //--------------------------------------------------------------( 16 bytes )
    /**
    * Direction for spot and directional lights (View space).
    */
    float4   DirectionVS;
    //--------------------------------------------------------------( 16 bytes )
    /**
    * Color of the light. Diffuse and specular colors are not seperated.
    */
    float4   Color;
    //--------------------------------------------------------------( 16 bytes )
    /**
    * The half angle of the spotlight cone.
    */
    float    SpotlightAngle;
    /**
    * The range of the light.
    */
    float    Range;
 
    /**
     * The intensity of the light.
     */
    float    Intensity;
 
    /**
    * Disable or enable the light.
    */
    bool    Enabled;
    //--------------------------------------------------------------( 16 bytes )
 
    /**
     * Is the light selected in the editor?
     */
    bool    Selected;
 
    /**
    * The type of the light.
    */
    uint    Type;
    float2  Padding;
    //--------------------------------------------------------------( 16 bytes )
    //--------------------------------------------------------------( 16 * 7 = 112 bytes )
};
```

位置和方向都分别保存了WS（世界空间）和VS（视图空间）。这是为了使用方便，10000盏灯光会额外占用1.12MB的内存，当然这也是可以优化掉的。

SpotlightAngle 是聚光灯垂直线与灯光边缘的夹角：

![Spotlight](/assets/postasset/2020-12-24-forward vs defered vs forward+/Spotlight.png)

Range变量定义光照的影响距离，单位是米。点光源表示为半径，聚光灯表示为圆锥体的长，方向光则无用。

Intensity是光照强度，默认是1。

Enabled则是灯光的开关。

Type是灯光类型，定义如下：

CommonInclude.hlsl

```c
#define POINT_LIGHT 0
#define SPOT_LIGHT 1
#define DIRECTIONAL_LIGHT 2
```



光照数组我们使用StructuredBuffer来储存。因为cbuffer被限制为64KB，这样光照数量会被限制为570lights，这样constant内存就爆了。而StructuredBuffer是用texture内存的，一般是被GPU纹理数量限制（桌面GPU肯定都是GB级的），而且texture内存也很快。



CommonInclude.hlsl

```c
StructuredBuffer<Light> Lights : register( t8 );
```



下面继续Pixel Shader：

```c
[earlydepthstencil]
float4 PS_main( VertexShaderOutput IN ) : SV_TARGET
{
    // Everything is in view space.
    float4 eyePos = { 0, 0, 0, 1 };
    Material mat = Mat;
```

[earlydepthstencil] 属性标识为GPU会使用early depth和stencil剔除，会在pixel shader之前进行深度/模板测试。很多GPU本身就有这种early测试，不过还是保留这个属性以防万一。



因为所有光照计算实在是视图空间中的，所以就不需要传递摄像机的坐标位置了，光照计算会比较方便。



ForwardRendering.hlsl

```c
float4 diffuse = mat.DiffuseColor;
if ( mat.HasDiffuseTexture )
{
    float4 diffuseTex = DiffuseTexture.Sample( LinearRepeatSampler, IN.texCoord );
    if ( any( diffuse.rgb ) )
    {
        diffuse *= diffuseTex;
    }
    else
    {
        diffuse = diffuseTex;
    }
}
```



此处处理diffuse，其的意思是如果diffuse是(0, 0, 0)，直接赋值diffuse纹理颜色，否则相乘。

下面Opacity、Ambient、Emissive、Specular等部分都是Blinn-Phong模型的实现，代码也有注释，这里就不在赘述。



法线：

如果既不用normal map，也不用bump map，将直接使用模型的normal。

ForwardRendering.hlsl

```c
// Normal mapping
if ( mat.HasNormalTexture )
{
    // For scenes with normal mapping, I don't have to invert the binormal.
    float3x3 TBN = float3x3( normalize( IN.tangentVS ),
                             normalize( IN.binormalVS ),
                             normalize( IN.normalVS ) );
 
    N = DoNormalMapping( TBN, NormalTexture, LinearRepeatSampler, IN.texCoord );
}
// Bump mapping
else if ( mat.HasBumpTexture )
{
    // For most scenes using bump mapping, I have to invert the binormal.
    float3x3 TBN = float3x3( normalize( IN.tangentVS ),
                             normalize( -IN.binormalVS ), 
                             normalize( IN.normalVS ) );
 
    N = DoBumpMapping( TBN, BumpTexture, LinearRepeatSampler, IN.texCoord, mat.BumpIntensity );
}
// Just use the normal from the model.
else
{
    N = normalize( float4( IN.normalVS, 0 ) );
}
```



Normal Mapping

CommonInclude.hlsl

```c
float3 ExpandNormal( float3 n )
{
    return n * 2.0f - 1.0f;
}
 
float4 DoNormalMapping( float3x3 TBN, Texture2D tex, sampler s, float2 uv )
{
    float3 normal = tex.Sample( s, uv ).xyz;
    normal = ExpandNormal( normal );
 
    // Transform normal from tangent space to view space.
    normal = mul( normal, TBN );
    return normalize( float4( normal, 0 ) );
}
```



Bump Mapping

不同于Normal Mapping，使用的是高度图。法线通过计算纹理中高度的斜率。算法类似Normal，都是基于切线空间，详细细节可参考相关Bump Mapping技术文章。

CommonInclude.hlsl

```c
float4 DoBumpMapping( float3x3 TBN, Texture2D tex, sampler s, float2 uv, float bumpScale )
{
    // Sample the heightmap at the current texture coordinate.
    float height = tex.Sample( s, uv ).r * bumpScale;
    // Sample the heightmap in the U texture coordinate direction.
    float heightU = tex.Sample( s, uv, int2( 1, 0 ) ).r * bumpScale;
    // Sample the heightmap in the V texture coordinate direction.
    float heightV = tex.Sample( s, uv, int2( 0, 1 ) ).r * bumpScale;
 
    float3 p = { 0, 0, height };
    float3 pU = { 1, 0, heightU };
    float3 pV = { 0, 1, heightV };
 
    // normal = tangent x bitangent
    float3 normal = cross( normalize(pU - p), normalize(pV - p) );
 
    // Transform normal from tangent space to view space.
    normal = mul( normal, TBN );
 
    return float4( normal, 0 );
}
```



光照：

整个光照算法都在DoLighting函数中实现，有如下参数：

lights：灯光数组，StructuredBuffer

mat：材质属性

eyePos：摄像机位置，因为是在视图空间，始终为(0,0,0)。

P：着色位置，视图空间

N：着色Normal，视图空间



ForwardRendering.hlsl

```c
// This lighting result is returned by the 
// lighting functions for each light type.
struct LightingResult
{
    float4 Diffuse;
    float4 Specular;
};
 
LightingResult DoLighting( StructuredBuffer<Light> lights, Material mat, float4 eyePos, float4 P, float4 N )
{
    float4 V = normalize( eyePos - P );
 
    LightingResult totalResult = (LightingResult)0;
 
    for ( int i = 0; i < NUM_LIGHTS; ++i )
    {
        LightingResult result = (LightingResult)0;
 
        // Skip lights that are not enabled.
        if ( !lights[i].Enabled ) continue;
        // Skip point and spot lights that are out of range of the point being shaded.
        if ( lights[i].Type != DIRECTIONAL_LIGHT &&
             length( lights[i].PositionVS - P ) > lights[i].Range ) continue;
 
        switch ( lights[i].Type )
        {
        case DIRECTIONAL_LIGHT:
        {
            result = DoDirectionalLight( lights[i], mat, V, P, N );
        }
        break;
        case POINT_LIGHT:
        {
            result = DoPointLight( lights[i], mat, V, P, N );
        }
        break;
        case SPOT_LIGHT:
        {
            result = DoSpotLight( lights[i], mat, V, P, N );
        }
        break;
        }
        totalResult.Diffuse += result.Diffuse;
        totalResult.Specular += result.Specular;
    }
 
    return totalResult;
}
```



Diffuse：

CommonInclude.hlsl

```c
float4 DoDiffuse( Light light, float4 L, float4 N )
{
    float NdotL = max( dot( N, L ), 0 );
    return light.Color * NdotL;
}
```



Specular：

CommonInclude.hlsl

```c
float4 DoSpecular( Light light, Material material, float4 V, float4 L, float4 N )
{
    float4 R = normalize( reflect( -L, N ) );
    float RdotV = max( dot( R, V ), 0 );
 
    return light.Color * pow( RdotV, material.SpecularPower );
}
```



衰减：

光照的衰减传统光照模型是：三个衰减因子的和乘以光照距离的倒数。

1. 常数衰减
2. 线性衰减
3. 二次衰减

但是这些方法里光照的值是无法衰减到0的，对于Forward+和延迟渲染必须有个有限范围的方法来计算衰减。其中有个方法是在光照范围内的，对光照强度从1开始进行线性混合，不在范围的则为0。但这方法不大真实，因为真是的衰减应该是一个二次函数的倒数。不过smoothstep可以替代线性混合来解决这问题。

![Smoothstep-Function2](/assets/postasset/2020-12-24-forward vs defered vs forward+/Smoothstep-Function2.png)

CommonInclude.hlsl

```c
// Compute the attenuation based on the range of the light.
float DoAttenuation( Light light, float d )
{
    return 1.0f - smoothstep( light.Range * 0.75f, light.Range, d );
}
```

上面的做法是当与灯光距离低于3/4的时候，smoothstep返回为0。



点光源：

ForwardRendering.hlsl

```c
LightingResult DoPointLight( Light light, Material mat, float4 V, float4 P, float4 N )
{
    LightingResult result;
 
    float4 L = light.PositionVS - P;
    float distance = length( L );
    L = L / distance;
 
    float attenuation = DoAttenuation( light, distance );
 
    result.Diffuse = DoDiffuse( light, L, N ) * 
                      attenuation * light.Intensity;
    result.Specular = DoSpecular( light, mat, V, L, N ) * 
                       attenuation * light.Intensity;
 
    return result;
}
```



聚光灯：

此聚光灯的做法是：内角无衰减，内角到外角开始进行smoothstep插值。外角指的是聚光灯的角度，内角则是外角一半的位置，当然smoothstep的参数是点到灯光距离与灯光方向的点积，而且刚好cos也是与角度负相关。

CommonInclude.hlsl

```c
float DoSpotCone( Light light, float4 L )
{
    // If the cosine angle of the light's direction 
    // vector and the vector from the light source to the point being 
    // shaded is less than minCos, then the spotlight contribution will be 0.
    float minCos = cos( radians( light.SpotlightAngle ) );
    // If the cosine angle of the light's direction vector
    // and the vector from the light source to the point being shaded
    // is greater than maxCos, then the spotlight contribution will be 1.
    float maxCos = lerp( minCos, 1, 0.5f );
    float cosAngle = dot( light.DirectionVS, -L );
    // Blend between the minimum and maximum cosine angles.
    return smoothstep( minCos, maxCos, cosAngle );
}
```

![Spotlight-Min-Max-Cone-Angles2](/assets/postasset/2020-12-24-forward vs defered vs forward+/Spotlight-Min-Max-Cone-Angles2.png)



ForwardRendering.hlsl

```c
LightingResult DoSpotLight( Light light, Material mat, float4 V, float4 P, float4 N )
{
    LightingResult result;
 
    float4 L = light.PositionVS - P;
    float distance = length( L );
    L = L / distance;
 
    float attenuation = DoAttenuation( light, distance );
    float spotIntensity = DoSpotCone( light, L );
 
    result.Diffuse = DoDiffuse( light, L, N ) * 
                       attenuation * spotIntensity * light.Intensity;
    result.Specular = DoSpecular( light, mat, V, L, N ) * 
                       attenuation * spotIntensity * light.Intensity;
 
    return result;
}
```



方向光：

方向光最简单了，没有衰减距离不需要衰减函数。

ForwardRendering.hlsl

```c
LightingResult DoDirectionalLight( Light light, Material mat, float4 V, float4 P, float4 N )
{
    LightingResult result;
 
    float4 L = normalize( -light.DirectionVS );
 
    result.Diffuse = DoDiffuse( light, L, N ) * light.Intensity;
    result.Specular = DoSpecular( light, mat, V, L, N ) * light.Intensity;
 
    return result;
}
```



最后我们对它进行各项最终相加处理：

ForwardRendering.hlsl

```c
    float4 P = float4( IN.positionVS, 1 );
 
    LightingResult lit = DoLighting( Lights, mat, eyePos, P, N );
 
    diffuse *= float4( lit.Diffuse.rgb, 1.0f ); // Discard the alpha value from the lighting calculations.
 
    float4 specular = 0;
    if ( mat.SpecularPower > 1.0f ) // If specular power is too low, don't use it.
    {
        specular = mat.SpecularColor;
        if ( mat.HasSpecularTexture )
        {
            float4 specularTex = SpecularTexture.Sample( LinearRepeatSampler, IN.texCoord );
            if ( any( specular.rgb ) )
            {
                specular *= specularTex;
            }
            else
            {
                specular = specularTex;
            }
        }
        specular *= lit.Specular;
    }
 
    return float4( ( ambient + emissive + diffuse + specular ).rgb, 
                     alpha * mat.Opacity );
 
}
```

