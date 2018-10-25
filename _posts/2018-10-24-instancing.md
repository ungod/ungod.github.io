---
layout: post
title: Instancing
tags:
---

### Instancing

假设一个场景有大量的模型绘制，且模型的顶点数据相同而世界坐标的变换不同。假设是有大量青草绿叶的场景：每一个青草绿叶模型都简单得仅仅由少量的三角形组成。你可能打算画相当多的此类模型，然后会导致场景每一帧就会渲染成千上万的草和叶。因为每片叶子仅仅由少量的三角形组成的渲染会快，但是数以千计的渲染调用会使性能大幅降低。

如果我们渲染大量的对象，就像如下代码所述：

```c
for(unsigned int i = 0; i < amount_of_models_to_draw; i++)
{
    DoSomePreparations(); // bind VAO, bind textures, set uniforms etc.
    glDrawArrays(GL_TRIANGLES, 0, amount_of_vertices);
}
```

因为大量drawcall，如此绘制大量模型实例你必然导致突破渲染瓶颈。对比实际的顶点，通过调用glDrawArray或者glDrawElements来告诉GPU渲染你的顶点数据，会快速耗掉性能，因为OpenGL在渲染顶点之前有很多准备工作要做（比如告诉GPU要读那个缓存，它包含顶点属性，同时它也在相对比较慢的CPU到GPU的总线中）。因此尽管渲染你的顶点相当快，给你的GPU传达渲染指令可就没那么快了。

如果我们一次性发送一些数据到GPU并且告诉OpenGL用一个drawcall绘制多个对象，这样就会相当方便。这就是Instancing。

Instancing是这么一个技术：我们一次性绘制大量的对象，以一次渲染调用，这样能降低每一次渲染一个对象的CPU->GPU通信消耗————这必须只渲染一次。要使用Instancing，我们必须修改渲染调用glDrawArrays和glDrawElements各自为glDrawArraysInstanced和glDrawElementsInstanced。这实例版本的典型渲染函数有一个额外的参数叫instance count，他用来设置我们想渲染的实例数量。因此我们发送所有必要数据到GPU只需一次，且告诉GPU他是如何通过一次调用来画所有实例。这样GPU渲染所有实例就不会有与CPU连续数次的通信了。

本身这个函数是有点没用。渲染同一个对象数千次是对于我们来说是没用的因为每次渲染对象确实是完全相同的且position一样————我们始终只能看到一个对象！因此，GLSL在vertex shader嵌入了一个内建变量叫gl_InstanceID。

当通过其中一个Instancing的渲染调用，gl_InstanceID从0开始每渲染一个实例自增。举个例子如果渲染了第43个实例，gl_InstanceID在vertex shader会变为42。每个实例唯一的值意味着我们可索引一个巨大的position数组且每个实例的世界坐标都不一样。

感受一下实例化绘图，我们将要展示一个简单的例子，在归一化坐标中渲染数百个2D四边形，仅用一个渲染调用。我们通过增加一个小位移，索引一个100位移的容器来实现它。最后结果是一个整齐有序的四边形网格填充整个窗口。

![](https://raw.githubusercontent.com/ungod/ungod.github.io/master/_postasset/2018-10-24-instancing/instancing_quads.png)

每一个四边形由两个三角形和最多6个顶点组成。每个顶点包含一个2D归一设备坐标位置(NDC)向量和一个颜色向量组成。下面是这个例子的顶点数据————这个三角形是很小使他能在屏幕显示大量实例。

```c
float quadVertices[] = {
    // positions     // colors
    -0.05f,  0.05f,  1.0f, 0.0f, 0.0f,
     0.05f, -0.05f,  0.0f, 1.0f, 0.0f,
    -0.05f, -0.05f,  0.0f, 0.0f, 1.0f,

    -0.05f,  0.05f,  1.0f, 0.0f, 0.0f,
     0.05f, -0.05f,  0.0f, 1.0f, 0.0f,   
     0.05f,  0.05f,  0.0f, 1.0f, 1.0f		    		
};  
```

四边形颜色通过前面的vertex shader传入到fragment shader并设置输出。

```c
#version 330 core
out vec4 FragColor;
  
in vec3 fColor;

void main()
{
    FragColor = vec4(fColor, 1.0);
}
```

为此也没啥新东西，除了vertex shader的前面让你感兴趣：

```c

#version 330 core
layout (location = 0) in vec2 aPos;
layout (location = 1) in vec3 aColor;

out vec3 fColor;

uniform vec2 offsets[100];

void main()
{
    vec2 offset = offsets[gl_InstanceID];
    gl_Position = vec4(aPos + offset, 0.0, 1.0);
    fColor = aColor;
}  
```

这里我们定义一个uniform数组叫offsets，包含最多100个偏移向量。在这vertex shader我们通过使用gl_InstanceID索引这个offsets数组，为每一实例取回一个偏移向量。如果我们是要使用实例化绘制100个四边形，我们会得到100个位置不同的四边形，在vertex shader之中。

我们需要设置偏移位置，在进入游戏循环之前，且一个嵌套循环中计算它。

```c
glm::vec2 translations[100];
int index = 0;
float offset = 0.1f;
for(int y = -10; y < 10; y += 2)
{
    for(int x = -10; x < 10; x += 2)
    {
        glm::vec2 translation;
        translation.x = (float)x / 10.0f + offset;
        translation.y = (float)y / 10.0f + offset;
        translations[index++] = translation;
    }
}  
```

这里我们创建了一组100个平移向量，每个平移向量都是在10x10格子的所有位置。除了创建这个平移数组，我们同时需要把他转化为uniform数组：

```c
shader.use();
for(unsigned int i = 0; i < 100; i++)
{
    stringstream ss;
    string index;
    ss << i; 
    index = ss.str(); 
    shader.setVec2(("offsets[" + index + "]").c_str(), translations[i]);
}  
```

在代码片段中，我们在for循环中的计数器i变换为string以动态创建一个位置字符串来查询uniform位置。在offsets uniform数组中每一个都需设置对应的平移向量。

现在所有准备均完成，我们可以开始渲染多边形了。通过glDrawArrayInstanced或者glDrawElementsInstanced来实例化渲染。因为我们没有用index buffer，我们就调用glDrawArrays版本：

```c
glBindVertexArray(quadVAO);
glDrawArraysInstanced(GL_TRIANGLES, 0, 6, 100);
```

glDrawArraysInstanced的参数跟glDrawArrays完全一样，除了最后的参数，它用来设置我们要绘制的实例数量。因为我们希望显示100个四边形在一个10x10格子中，我们设置他为100。运行这段代码，最终会显示熟悉的图片，100个着色的四边形。

### Instanced Arrays(实例化数组)
前面的工作在特定的使用场景中实现得还行，无论何时我们要渲染多于100个实例(其实这相当普通)我们最终会碰到一个可以发送到shader的uniform数据限制(这里指是数量限制，参考[维基](https://www.khronos.org/opengl/wiki/Uniform_(GLSL))的Implementation limits)。其他可替代性方案的称为Instanced Arrays的，是定义一个顶点属性(允许我们保存更多数据)，它仅仅在vertex shader渲染新实例时更新。

使用顶点属性，每次运行vertex shader会引起GLSL找回下一组顶点属性，它属于最近的顶点。然而当定义一个顶点属性为一个实例数组，vertex shader在实力化时仅仅更新顶点属性内容而不是每个顶点。这允许我们顶点使用标准顶点属性作为数据且使用实例化数组为每个实例保存唯一数据。

下面给个实例化数组例子，我们将使用前面的例子并将偏移uniform数组表示为实例化数组。我们必须更新vertex shader，通过增加另外的顶点属性。

```c
#version 330 core
layout (location = 0) in vec2 aPos;
layout (location = 1) in vec3 aColor;
layout (location = 2) in vec2 aOffset;

out vec3 fColor;

void main()
{
    gl_Position = vec4(aPos + aOffset, 0.0, 1.0);
    fColor = aColor;
}  
```

我们不再使用gl_InstanceID，而是直接使用offset属性且不需要先索引一个巨大的uniform数组。

因为一个实例化数组是一个顶点属性，像position和color变量那样，我们同时在vertex buffer object需要保存其内容且配置它的属性指针。我们首先要保存translations数组(前面例子部分)为一个新的缓存对象：

```c
unsigned int instanceVBO;
glGenBuffers(1, &instanceVBO);
glBindBuffer(GL_ARRAY_BUFFER, instanceVBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(glm::vec2) * 100, &translations[0], GL_STATIC_DRAW);
glBindBuffer(GL_ARRAY_BUFFER, 0); 
```

然后我们同时需要设置其顶点指针并启动顶点属性：

```c
glEnableVertexAttribArray(2);
glBindBuffer(GL_ARRAY_BUFFER, instanceVBO);
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 2 * sizeof(float), (void*)0);
glBindBuffer(GL_ARRAY_BUFFER, 0);	
glVertexAttribDivisor(2, 1);  
```

最后一行比较因吹斯汀，我们调用了glVertexAttribDivisor。这个函数告诉OpenGL到下个元素什么时候更新顶点属性内容。第一个参数是顶点属性位置，第二个参数是属性除子。默认属性除子是0，它告诉OpenGL更新顶点属性内容于每次vertex shader迭代中。设置此属性为1我们则告诉OpenGL当我们开始渲染一个新的实例我们更新这顶点属性的内容。设置为2则每两次实例化的时候，以此类推。设置此属性除子为1我们高效地告诉OpenGL顶点属性中位置为2是一个实例化数组。

如果我们再次使用glDrawArraysInstanced渲染四边形，我们获得一下输出：
![](https://raw.githubusercontent.com/ungod/ungod.github.io/master/_postasset/2018-10-24-instancing/instancing_quads.png)

这是跟前面例子完全相同的，但是这时是用实例化数组完成的，它允许我们为实例化绘制传递更大量的数据(只要内存允许)到vertex shader。

来干些更有趣的东西，我们也逐渐缩减每个四边形，从右上到左下，再次使用gl_InstanceID来实现。

```c
void main()
{
    vec2 pos = aPos * (gl_InstanceID / 100.0);
    gl_Position = vec4(pos + aOffset, 0.0, 1.0);
    fColor = aColor;
} 
```

这个结果是第一个四边形实例会被绘制得非常小，以后我们在绘制实例的过程中，gl_InstanceID 越是接近100越使四边形恢复到原大小。他通过gl_InstanceID完美合法地使用实例化数组，如下：

![](https://raw.githubusercontent.com/ungod/ungod.github.io/master/_postasset/2018-10-24-instancing/instancing_quads_arrays.png)

如果你仍然有点不确定实例化渲染是如何工作或者想知道关于他们工作上的一切，我提供了所有[源码](https://raw.githubusercontent.com/ungod/ungod.github.io/master/_postasset/2018-10-24-instancing/example.txt)

这还不够，这个例子没真正的表现出Instancing。没错他确实简单展示了关于实例化的工作概况，但实例化是相当有用的当绘制巨量的相同对象，我们都还没看到。下面我们章节通过漫游太空以展示Instancing的真正实力。