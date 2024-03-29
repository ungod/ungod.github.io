---
layout: post
title: 三角形的基础、插值和重心坐标
categories: [游戏, 渲染]
math: true
tags:
---

最近需要做一个三角形的AO插值，细致研究了一下用重心坐标的插值方法。

## 简述

假设有一个三角形

![](https://raw.githubusercontent.com/ungod/ungod.github.io/master/_postasset/triangle_basis/1.png)

此每个顶点都有自己关联的属性。举个例子，假设每个三角形有一个关联的颜色:

 - 顶点1(V1)是红色
 - 顶点2(V2)是绿色
 - 顶点3(V3)是紫色

![](https://raw.githubusercontent.com/ungod/ungod.github.io/master/_postasset/triangle_basis/2.png)

对于任意点`P`，于三角形内，如何定义该点颜色？

![](https://raw.githubusercontent.com/ungod/ungod.github.io/master/_postasset/triangle_basis/3.png)

我们使用颜色只是作为例子。实际上我们会插值任意相关顶点属性，比如UV纹理坐标，Z-Buffer深度，normal值。
如果你玩过[SimAnt](https://en.wikipedia.org/wiki/SimAnt)，你或许记得成员的行为控制是用一个三角形来平衡三个动作的比例：forage, dig, 和nurse。这是一个在三角形插值的其他例子。

![](https://raw.githubusercontent.com/ungod/ungod.github.io/master/_postasset/triangle_basis/4.png)


## 懒方法——近邻法

我们简单地认为`P`是最接近于`V1`，因此`P`是红色。如果三角形每个点都这么做，我们得到如下三角形：

![](https://raw.githubusercontent.com/ungod/ungod.github.io/master/_postasset/triangle_basis/5.png)

这种方法称为近邻法，有时候称为维诺图([Voronoi diagram](https://en.wikipedia.org/wiki/Voronoi_diagram))。这怎么说都不像我们要找的东西。


## 一个自然的方案


在给出最终方案的时候，我先试一个并不怎么正确的方案。我发现，当有个方案有点复杂，最好先证明为什么简单的方法行不通。

因此三角形的一个点的值应该是来自三个顶点的。我们知道，它应该是离顶点越近，受到的颜色就越多，离越远，受到的颜色就越少。前面的例子我们看到查询点`P`，相当靠近`V1`和`V2`，离`V3`就很远。因此按理来说，它的颜色应该是更多红色、绿色，更少紫色。

![](https://raw.githubusercontent.com/ungod/ungod.github.io/master/_postasset/triangle_basis/6.png)

我们能算出`P`点到没过顶点的距离，通过勾股定理[Pythagorean theorem](https://en.wikipedia.org/wiki/Pythagorean_theorem)。

$$\begin{array}{l}
Distance_{v1} = \sqrt{(X_{v1} - P_{x})^2 + (Y_{v1} - P_{y})^2}\\
Distance_{v2} = \sqrt{(X_{v2} - P_{x})^2 + (Y_{v2} - P_{y})^2}\\
Distance_{v3} = \sqrt{(X_{v3} - P_{x})^2 + (Y_{v3} - P_{y})^2}
\end{array}$$

距离值是越远，值越大，越近，值就越小。因此我们反转这个值，让它越近，权值越大。换句话说，我们反比插值于每个顶点。

因此我们定义我们的权值为：

$$\begin{array}{l}
W_{v1} = \frac{1}{Distance_{v1}}\\
W_{v2} = \frac{1}{Distance_{v2}}\\
W_{v3} = \frac{1}{Distance_{v3}}\\
\end{array}$$

现在我们定义其颜色为平均权重：

$$Color_{p} = \frac{W_{v1} Color_{v1} + W_{v2} Color_{v2} + W_{v3} Color_{v3}}
{W_{v1} + W_{v2} + W_{v3}}$$

这里有一个相当不错的性质，一个两倍远的顶点，对权值贡献只有一半。这项技术在很多其他领域都是很有用的。比如说机器学习，很常见地他会使用反转来标准化一个距离度量。

总的来说，我们在找一个点`P`的插值，通过混合正比于距离短的顶点颜色。

使用这个方法来对整个三角形颜色进行插值，产生了：

![](https://raw.githubusercontent.com/ungod/ungod.github.io/master/_postasset/triangle_basis/7.png)

看起来还不错吧。此解决方案是简单容易实现，且相当的直观。它可能在大部分程序中都能执行。但是，如果我们吧三角形换一下，我们能看到这个方法的主要问题。

![](https://raw.githubusercontent.com/ungod/ungod.github.io/master/_postasset/triangle_basis/8.png)

上面在这个例子中点`P`，直接落在`V1`和`V3`的直线上。在大部分插值问题，我们确实像`P`的值，由`V1`和`V3`单独贡献，我们一般是完全忽略`V2`。在上面这种自然方案中，就会变得混乱不堪。因为`P`是最接近于`V2`，`P`最后取自于`V2`更加多！

如果你觉得这不是啥大问题，想一下如果你如何在这个三角形中插值数据。比如，如果你插值Z-Buffer坐标，你只想要`V1`到`V3`直线上的点，如果它却是V2的点，那这不可能是一条直线了！

你可以再看看SimAnt。如果我们设置一个准确的光标介于forage和dig的，我们知道这时候nurse应该是0.因此SimAnt有个更好的由于自然方法来处理这事情。

![](https://raw.githubusercontent.com/ungod/ungod.github.io/master/_postasset/triangle_basis/9.png)

## 重心坐标

这个问题真正的解决方案是重心坐标[Barycentric Coordinates](https://en.wikipedia.org/wiki/Barycentric_coordinate_system)。这概念最初可能难以理解，但是并不复杂，稍微思考一下就明白了。

重心坐标的诀窍是，找到能平衡以下等式系统的，`V1` `V2` `V3`的权值：

$$\begin{array}{l}
P_{x} = W_{v1} X_{v1} + W_{v2} X_{v2} + W_{v3} X_{v3}\\
P_{y} = W_{v1} Y_{v1} + W_{v2} Y_{v2} + W_{v3} Y_{v3}\\
W_{v1} + W_{v2} + W_{v3} = 1
\end{array}$$

花点时间想想上面等式的含义。我们追求的是自洽。如果我们想对颜色或者Z-Buffer插值，我们追求的权值应该对X和Y在相同的行为中也是生效的。换句话说，给出一点`P`，我们发现权值能告诉我们P的X坐标是组成自`V1`，`V2`和`V3`，P的Y坐标也是这样。

通过一系列重排，我们得出W1，W2和W3的解为以下等式：

$$\begin{array}{l}
W_{v1}=\frac{(Y_{v2}-Y_{v3})(P_{x}-X_{v3})+(X_{v3}-X_{v2})(P_{y}-Y_{v3})}{(Y_{v2}-Y_{v3})(X_{v1}-X_{v3})+(X_{v3}-X_{v2})(Y_{v1}-Y_{v3})}\\
W_{v2}=\frac{(Y_{v3}-Y_{v1})(P_{x}-X_{v3})+(X_{v1}-X_{v3})(P_{y}-Y_{v3})}{(Y_{v2}-Y_{v3})(X_{v1}-X_{v3})+(X_{v3}-X_{v2})(Y_{v1}-Y_{v3})}\\
W_{v3}=1 - W_{v1} - W_{v2}
\end{array}$$

注意到`W1``W2`的分母都是一样的，`W3`则是通过所有权值的和是1这个事实来计算。

另外一件事关于重心坐标的，是如果`P`点是在三角形外，至少`W1``W2``W3`其中之一是负数。这样我们就可以轻松地判断点是否在三角形内。

实际上，一般三角形绘制算法是观察三角形的包围盒内每个像素。对于每个像素，计算其重心坐标（当然他们都需要某种方法去插值深度缓冲，纹理坐标等）。如果其中一个权值是赋值，该像素会被跳过。这算法的一个好处是图形卡能简单地平行化包围盒的每个像素。这使得画三角形非常地快。

下面是上面这个问题的最终颜色插值结果，如果我们使用重心坐标系统的话。

![](https://raw.githubusercontent.com/ungod/ungod.github.io/master/_postasset/triangle_basis/10.png)

## 重心方程的细节

重心方程通过三个标量能用于表示在三角形内的任意位置。点的位置一般包含在三角形内，或者三个边中，或者三个顶点本身。计算其位置我们可以用以下等式1：

$$P=uA+vB+wC$$

A、B和C是三角形的顶点，u、v和w是在重心坐标中是三个实数（标量），u + v + w = 1（重心坐标是归一化的坐标系）。已知两个值，则可得第三个值：w = 1 - u - v，(u + v <= 1)。
上面等式强调的是P点在平面中是通过顶点A、B和C来构建的。如果0 <= u, v, w >= 1，点在三角形里面，如果小于0或者大于1，则在三角形外面。如果任意值为0，则点P落在其三角边上。

你可以简化为两个坐标（比如说u和v）来表示P点在二维空间中，以A为原点，AB和AC作为边（就像表示2D点在正交二位坐标系统，定义其x轴和y轴。相对我们例子不同的地方在于AB和AC是不需要正交和原点是A）。为此，你可以表示定义一个在三角形内的点为P = A + u*AB + v*AC (u >= 0，v >= 0，u + v <= 1)。此等式能解释为“从A点出发，从AB方向移动一下，再从AC方向移动一下，得到点P”。此时你能写成：
P = A + u*(B - A) + v(C - A) 
= A + uB - uA + vC - vA 
= (1 - u - v)*A + u*B + v*C

此等式近似于等式1，除了如果w = 1 - u - v，可得出：
P = wA + uB + vC
替换成：
P = uA + vB + wC

两个等式是完全近似的但是如果没有上面这段说明，很难解释为什么要用(1 - u - v)*A + uB + vC而不是uA + vB + wC。

重心坐标其实是一个面积坐标。尽管使用不是很普遍，其u、v、w是跟点P划分成的三个子三角形的面积成正比的。此三角形表示为ABP、BCP、CAP。

为此能引导出以下公式，用于计算重坐标：

$$\begin{array}{l}
u = {\dfrac{TriangleCAP_{Area}}{TriangleABC_{Area}}}\\
v = {\dfrac{TriangleABP_{Area}}{TriangleABC_{Area}}}\\
w ={\dfrac{TriangleBCP_{Area}}{TriangleABC_{Area}}}\\
\end{array}$$

假设u是沿AB边移动（u=1则P=B），v沿AC边移动（v=1则P=C）。这就是为什么使用CAP面积来计算u的值和使用ABP的面值来计算v值。这是一个CG编程社区的人们跟随来下的惯例（当我们学习如何使用重心坐标来插值顶点数据）。

![](https://raw.githubusercontent.com/ungod/ungod.github.io/master/_postasset/triangle_basis/11.png)

现在计算三角形的面值是次要的。如果你复制三角形和使用最长的边镜像它，你会得到一个平行四边形。计算平行四边形的面积，简单地计算base边，他的side边，并将两者相乘并乘以sin(θ)，θ是AB和AC的夹角。创建此平行四边形我们需要两个三角形，因此一个三角形的面积是平行四边形的一半。

至此，可以简单地计算u和v（w是被计算为u和v之外）。

{% raw %}
$$Triangle_{area} = {{||(B-A)|| * ||(C - A)|| sin(\theta)} \over {2}}$$
{% endraw %}

要让其简化，我们利用一个事实：子三角形的面积ABP，BCP和CAP是两边的叉乘的值成正比。这里是叉乘的特性：叉乘的模能代表平行四边形的面积。因此我们不用明确地计算前面的公式（其包含比较耗的sin(θ)）。我们能简化为：

$$\begin{array}{l}
Parallelogram_{area} = ||(B-A) \times (C - A)||\\
Triangle_{area}=\dfrac{Parallelogram_{area}}{2}
\end{array}$$

数学术语中双竖杠（|| ||）表示为长度。换句话说，我们需要计算长度结果为(C-B)x(P-B)的叉乘。
根据P=uA+uB+wC，可以计算包括颜色值在内的，任意顶点的插值数据：

{% raw %}
```c
// vertex position
Vec3f triVertex[3] = {{-3,-3,5}, {0,3,5}, {3,-3,5}}; 
// vertex data
Vec3f triColor[3] = {{1,0,0}, {0,1,0}, {0,0,1}}; 
if (rayTriangleIntersect(...)) { 
    // compute pixel color
    // col = w*col0 + u*col1 + v*col2 where w = 1-u-v
    Vec3f PhitColor = u * triColor[0] + v * triColor[1] + (1 - u - v) * triColor[2]; 
} 
```
{% endraw %}

为此这算法已经聊了很久了，我们对AB和AP叉乘，CA和CP叉乘来计算u和v值。但是如果你再次看看上述代码你会发觉我们在三角形内部包含测试中已经计算了其叉乘。他们被用来计算P点是在边0（CAxCP）和边2（CAxCP）的右边还是左边。自然地我们首先可以通过复用来优化这一点。

另外我们发现：
$$\begin{array}{l}
v & = &{\dfrac{triangleABP_{area}}{triangleABC_{area}}}\\
& =  & {\dfrac{parellogramABP_{area} / 2}{parellogramABC_{area} / 2}}={\dfrac{parellogramABP_{area}}{parellogramABC_{area}}}={\dfrac{||AB \times AP||}{||AB \times AC||}}
\end{array}$$

是不需要计算三角面积的，因为三角形ABP的面积和三角形ABC的面积比跟平行四边形ABP（两倍于三角形面积）与平行四边形ABC的面积比是一样的。因此我们能避免除以2，代码能变成：

```c
bool rayTriangleIntersect( 
    const Vec3f &orig, const Vec3f &dir, 
    const Vec3f &v0, const Vec3f &v1, const Vec3f &v2, 
    float &t, float &u, float &v) 
{ 
    // compute plane's normal
    Vec3f v0v1 = v1 - v0; 
    Vec3f v0v2 = v2 - v0; 
    // no need to normalize
    Vec3f N = v0v1.crossProduct(v0v2); // N 
    float area2 = N.length(); 
 
    // Step 1: finding P
 
    // check if ray and plane are parallel ?
    float NdotRayDirection = N.dotProduct(dir); 
    if (fabs(NdotRayDirection) < kEpsilon) // almost 0 
        return false; // they are parallel so they don't intersect ! 
 
    // compute d parameter using equation 2
    float d = N.dotProduct(v0); 
 
    // compute t (equation 3)
    t = (N.dotProduct(orig) + d) / NdotRayDirection; 
    // check if the triangle is in behind the ray
    if (t < 0) return false; // the triangle is behind 
 
    // compute the intersection point using equation 1
    Vec3f P = orig + t * dir; 
 
    // Step 2: inside-outside test
    Vec3f C; // vector perpendicular to triangle's plane 
 
    // edge 0
    Vec3f edge0 = v1 - v0; 
    Vec3f vp0 = P - v0; 
    C = edge0.crossProduct(vp0); 
    if (N.dotProduct(C) < 0) return false; // P is on the right side 
 
    // edge 1
    Vec3f edge1 = v2 - v1; 
    Vec3f vp1 = P - v1; 
    C = edge1.crossProduct(vp1); 
    u = C.length() / area2; 
    if (N.dotProduct(C) < 0)  return false; // P is on the right side 
 
    // edge 2
    Vec3f edge2 = v0 - v2; 
    Vec3f vp2 = P - v2; 
    C = edge2.crossProduct(vp2); 
    v = C.length() / area2; 
    if (N.dotProduct(C) < 0) return false; // P is on the right side; 
 
    return true; // this ray hits the triangle 
} 
```

最后我们要证明下面的等式3：

$$\begin{array}{l}v = {\dfrac{triangleABP_{area}} {triangleABC_{area}}} = {\dfrac{(AB \times AP) \cdot  N}{(AB \times AC) \cdot N}}
\end{array}$$

首先我们记得N = AB x AC。因此我们可以重写一下：

$$\begin{array}{l}
v = {\dfrac{triangleABP_{area}}{triangleABC_{area}}} = {\dfrac{(AB \times AP) \cdot  N}{N \cdot N}}
\end{array}$$

现在我们需要证明这个等式。首先需要变量的点积：

$$||A|| \cdot ||B|| = \cos(\theta)||A||*||B||$$


θ 是矢量A和B的夹角，\|\|A\|\|和\|\|B\|\|是长度。我们同时需要知道向量AB共线时（AB指向同一方向），夹角为0，因此cos(θ) = 1，N dot N的值则是cos(0)\|\|N\|\|\|\|N\|\| = \|\|N\|\|\|\|N\|\|，N的长度的平方。看看等式3的分子，ABP和ABC是共面的。叉乘的结果ABxAP、ABxAC则是共线的。可以重写分子为：

$$(AB \times AP) \cdot N= (AB \times AP) \cdot (AB \times AC)$$

将其写成：

$$AB \times AP=A$$
$$AB \times AC=N=B$$

得到：

$$(AB \times AP) \cdot N = \cos(0)||A|| ||B||=||A|| ||B||$$

A和B是共线的的，因为构建他们的矢量是共面的。最后我们替换其分子和分母：

{% raw %}
$$v = {{triangleABP_{area}} \over {triangleABC_{area}}} = {\dfrac{||AB \times AP|| \cdot N}{N \cdot N}} = {\dfrac{||A|| ||B||} {||B|| || B||}}$$
{% endraw %}

我们消项以下得：

$$v = {\dfrac{triangleABP_{area}}{triangleABC_{area}}} = \dfrac{||A||}{|| B||}$$

又因为：

$$\dfrac{||A||}{||B||}={\dfrac{||AB \times AP||}{||AB \times AC||}}$$

证明成功。这个等式用于首先用于计算v。等式3也是计算v。我们为了计算是否在三角形内，已经计算了
(ABxAP) dot N，因此我们可以重用此值。

下面是最终结果代码：

```c
bool rayTriangleIntersect( 
    const Vec3f &orig, const Vec3f &dir, 
    const Vec3f &v0, const Vec3f &v1, const Vec3f &v2, 
    float &t, float &u, float &v) 
{ 
    // compute plane's normal
    Vec3f v0v1 = v1 - v0; 
    Vec3f v0v2 = v2 - v0; 
    // no need to normalize
    Vec3f N = v0v1.crossProduct(v0v2); // N 
    float denom = N.dotProduct(N); 
 
    // Step 1: finding P
 
    // check if ray and plane are parallel ?
    float NdotRayDirection = N.dotProduct(dir); 
    if (fabs(NdotRayDirection) < kEpsilon) // almost 0 
        return false; // they are parallel so they don't intersect ! 
 
    // compute d parameter using equation 2
    float d = N.dotProduct(v0); 
 
    // compute t (equation 3)
    t = (N.dotProduct(orig) + d) / NdotRayDirection; 
    // check if the triangle is in behind the ray
    if (t < 0) return false; // the triangle is behind 
 
    // compute the intersection point using equation 1
    Vec3f P = orig + t * dir; 
 
    // Step 2: inside-outside test
    Vec3f C; // vector perpendicular to triangle's plane 
 
    // edge 0
    Vec3f edge0 = v1 - v0; 
    Vec3f vp0 = P - v0; 
    C = edge0.crossProduct(vp0); 
    if (N.dotProduct(C) < 0) return false; // P is on the right side 
 
    // edge 1
    Vec3f edge1 = v2 - v1; 
    Vec3f vp1 = P - v1; 
    C = edge1.crossProduct(vp1); 
    if ((u = N.dotProduct(C)) < 0)  return false; // P is on the right side 
 
    // edge 2
    Vec3f edge2 = v0 - v2; 
    Vec3f vp2 = P - v2; 
    C = edge2.crossProduct(vp2); 
    if ((v = N.dotProduct(C)) < 0) return false; // P is on the right side; 
 
    u /= denom; 
    v /= denom; 
 
    return true; // this ray hits the triangle 
} 
```

参考：

[1] https://codeplea.com/triangular-interpolation

[2] https://www.scratchapixel.com/lessons/3d-basic-rendering/ray-tracing-rendering-a-triangle/barycentric-coordinates