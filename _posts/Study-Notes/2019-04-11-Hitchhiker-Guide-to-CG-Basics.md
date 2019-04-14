---
layout:     post
title:      "CG Basics"
subtitle:   "Hitchhiker's Guide to CG Basics"
date:       2019-04-10 20:35:00
author:     "StephenNG"
header-img: "img/in-post/post-cg-basics/bg.png"
tags:
    - CG
    - OpenGL
---

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>

> Here we go

- [<font color="#0060A0">OpenGL图形管线</font>](#font-color%220060a0%22opengl%E5%9B%BE%E5%BD%A2%E7%AE%A1%E7%BA%BFfont)
	- [NOTE - 不同空间的变换关系](#note---%E4%B8%8D%E5%90%8C%E7%A9%BA%E9%97%B4%E7%9A%84%E5%8F%98%E6%8D%A2%E5%85%B3%E7%B3%BB)
	- [0. 准备](#0-%E5%87%86%E5%A4%87)
	- [1. 顶点处理(Vertex Processing)](#1-%E9%A1%B6%E7%82%B9%E5%A4%84%E7%90%86vertex-processing)
		- [顶点着色器(Vertex Shader)](#%E9%A1%B6%E7%82%B9%E7%9D%80%E8%89%B2%E5%99%A8vertex-shader)
			- [优化](#%E4%BC%98%E5%8C%96)
		- [镶嵌/细分曲面/细分网格(Tessellation)](#%E9%95%B6%E5%B5%8C%E7%BB%86%E5%88%86%E6%9B%B2%E9%9D%A2%E7%BB%86%E5%88%86%E7%BD%91%E6%A0%BCtessellation)
		- [几何着色器(Geometry Shader)](#%E5%87%A0%E4%BD%95%E7%9D%80%E8%89%B2%E5%99%A8geometry-shader)
	- [2. 顶点后期处理(Vertex Post-Processing)](#2-%E9%A1%B6%E7%82%B9%E5%90%8E%E6%9C%9F%E5%A4%84%E7%90%86vertex-post-processing)
		- [Transform Feedback](#transform-feedback)
		- [裁切(Clipping)](#%E8%A3%81%E5%88%87clipping)
		- [透视除法(Perspective Division)](#%E9%80%8F%E8%A7%86%E9%99%A4%E6%B3%95perspective-division)
		- [Viewport Transform](#viewport-transform)
	- [3. 图元装配(Primitive Assembly)](#3-%E5%9B%BE%E5%85%83%E8%A3%85%E9%85%8Dprimitive-assembly)
		- [中止光栅化(Rasterizer Discard)](#%E4%B8%AD%E6%AD%A2%E5%85%89%E6%A0%85%E5%8C%96rasterizer-discard)
		- [面剔除(Face Culling)](#%E9%9D%A2%E5%89%94%E9%99%A4face-culling)
			- [环绕顺序(Winding Order)](#%E7%8E%AF%E7%BB%95%E9%A1%BA%E5%BA%8Fwinding-order)
			- [剔除(Culling)](#%E5%89%94%E9%99%A4culling)
	- [4. 光栅化(Rasterization)](#4-%E5%85%89%E6%A0%85%E5%8C%96rasterization)
		- [采样和过滤(Sampling and Filtering)](#%E9%87%87%E6%A0%B7%E5%92%8C%E8%BF%87%E6%BB%A4sampling-and-filtering)
			- [多级渐远纹理(Mipmapping)](#%E5%A4%9A%E7%BA%A7%E6%B8%90%E8%BF%9C%E7%BA%B9%E7%90%86mipmapping)
		- [隐藏面移除(Hidden Surface Removal)](#%E9%9A%90%E8%97%8F%E9%9D%A2%E7%A7%BB%E9%99%A4hidden-surface-removal)
			- [Z-buffering](#z-buffering)
- [<font color="CornflowerBlue">test</font>](#font-color%22cornflowerblue%22testfont)

# <font color="#0060A0">OpenGL图形管线</font>

在OpenGL中，任何事物都在3D空间中，而屏幕和窗口却是2D数组，所以OpenGL的大部分工作，就是<strong><font color="darkviolet">把3D坐标转换为适应你屏幕的2D像素</font></strong>。把3D坐标转为2D像素是由OpenGL的<strong><font color="royalblue">*图形渲染管线*</font></strong>(Graphics Pipeline，下简称“管线”)管理的，实际上它接受一堆原始的图形数据，经过处理最终映射到屏幕上。~~管线主要分为两部分：1. 把3D坐标转换为2D坐标；2. 把2D坐标转换为实际上的有颜色的2D像素。~~

管线可以划分为几个阶段，每个阶段把前一个阶段的输出作为输入。每一个阶段都高度专门化（它们都有一个特定的函数），并且很容易<strong><font color="darkviolet">并行执行</font></strong>，因此GPU可以得心应手地、快速地处理它们，为我们节约宝贵的CPU时间。不仅如此，<strong><font color="darkviolet">我们还可以私人订制其中的一些阶段</font></strong>，使得对管线的掌控更加细致。

![pipeline](../../../../img/in-post/post-cg-basics/RenderingPipeline.png)
<small class="img-hint">渲染管线。蓝色方框代表可编程的着色器阶段。</small>

NOTE - 不同空间的变换关系
---

model-space --[model-matrix]--> world-space --[view-matrix]--> camera-space --[projection-matrix]--> clip-space --[clipping & perspective division]--> NDC-space --[viewport transfrom]--> window-space --[rasterization]--> screen-space

## 0. 准备

在你进行**渲染操作**的时候，OpenGL渲染管线将被初始化。进行渲染操作前，需要准备两样东西：定义好的顶点数组对象，以及链接好的程序对象。

1. **顶点数组对象**(vertex array object - **VAO**)；

    ![VAO](../../../../img/in-post/post-cg-basics/vao2.png "VAO")

    VAO是一个存储了<font color="darkviolet"><strong>所有所需的顶点状态</strong></font>(state)的<strong><font color="royalblue">*OpenGL对象*</font></strong>，包括<font color="darkviolet"><strong>顶点数据的格式</strong></font>，以及<font color="darkviolet"><strong>提供了顶点数据数组的**缓存对象**</strong></font>(buffer objects)(VBO、EBO等众多种类[see wiki](https://www.khronos.org/opengl/wiki/Buffer_Object#General_use))。

    > <strong><font color="CornflowerBlue">OpenGL Object</font></strong>是一个OpenGL的结构体(construct)，它存储了一些状态(state)。当一个OpenGL对象被<font color="MediumPurple"><strong>上下文绑定</strong></font>(bound to the context)时，它记录的状态就和上下文的状态有了映射关系：如果改变上下文状态，OpenGL对象里的状态也会相应改变；相反，如果有函数要使用上下文状态，那它实际上使用的就是OpenGL对象里记录的状态。记住，OpenGL对象就是状态的容器(containers)。

    看概念似乎有点抽象，如果以VAO为例，它存储了顶点数据的格式、VBO、EBO等东西，所以如果要更改VAO的数据格式，或者更改VAO所引用的VBO、EBO，就需要先调用

    ```C++ 
    glBindVertexArray(some_VAO);
    ```

    来把some_VAO绑定到上下文中，然后再进行数据格式的指定，或者修改some_VAO所引用的VBO和EBO：

    ```C++
    // 指定0号属性的格式(index, size, type, normalized, stride, pointer)
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);

    glBindBuffer(GL_ARRAY_BUFFER, some_VBO);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, some_EBO);
    ```

    值得注意的是，这里VAO作为一个OpenGL对象，<font color="darkviolet"><strong>储存了对其他OpenGL对象（VBO、EBO）的引用</strong></font>。所以在OpenGL对象里，它被分为<strong><font color="royalblue">容器对象</font></strong>(container objects)，区分于VBO和EBO的<strong><font color="royalblue">普通对象</font></strong>(regular objects)。

    在上面的代码中，我们把some_VBO和some_EBO绑定到了上下文中，同样地，我们也可以对这两个OpenGL对象进行修改了：

    ```C++
    glBufferData(GL_ARRAY_BUFFER, data_size * sizeof(float), data, GL_STATIC_DRAW);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, indices_size * sizeof(unsigned int), indices, GL_STATIC_DRAW);
    ```

    作为OpenGL对象，VAO<font color="darkviolet"><strong>并不是复制、存储了被引用的buffer的*内容*</strong></font>：如果buffer里的数据被更改，VAO的使用者也会看到这些变化。

    ![VAO](../../../../img/in-post/post-cg-basics/vao.png "VAO")

    上述代码中我们指定了0号属性的格式；在一个VAO中我们一共可以指定*GL_MAX_VERTEX_ATTRIBS*个属性(attributes)。每个属性可以开放或者关闭它们的<strong><font color="royalblue">数组访问</font></strong>(array access)：
    
    ```C++
    glEnableVertexAttribArray(0);       // 开放0号属性的数组访问
    glDisableVertexAttribArray(0);      // 关闭……
    ```

    属性号码对应着在<strong><font color="royalblue">vertex shader</font></strong>([see below](#Vertex-Shader))里面数据传入的位置：

    ```C++
    // vertex shader
    layout (location = 0) in vec3 aPos;       // 0号属性
    layout (location = 1) in vec3 aNormal;    // 1号属性
    ……
    ```

    如果数组访问被关闭，那么当vertex shader需要读取该属性的某个数据的时候，会得到一个常数值，这个常数值由一个特殊的<strong><font color="darkviolet">上下文状态</font></strong>(context state)决定，为什么说它是特殊的？因为这个状态<strong><font color="darkviolet">不属于VAO</font></strong>。


2. 链接好的程序对象(linked program object)；

    一个程序对象是用处理好的OpenGL Shading Language（GLSL是OpenGL直接支持的语言，也可以使用一些其他的语言，但需要安装extensions）组成的可执行代码。通过创建程序对象、给程序对象附上编译好的shader、链接程序对象，获得一个渲染所需的程序对象。它为管线中可编程的阶段提供了shaders。
    
    简而言之，准备阶段主要需要处理一些OpenGL对象，比如VAO和VBO；VAO定义了每个顶点有什么属性，以及属性如何存储；VBO则存储了真正的顶点数据。
    
    我们使用类似VBO的缓冲对象的好处是，它们会在GPU内存（显存）中储存这些顶点数据，然后在需要的时候，<strong><font color="darkviolet">一次性发送一大批数据给显卡</font></strong>，而不是每个顶点发送一次。从CPU发数据到显卡相对较慢，所以我们希望一次发送尽可能多的数据。
    
    当顶点数据和程序对象都准备好了，就可以下达渲染指令，使管线启动了！

## 1. 顶点处理(Vertex Processing)

### 顶点着色器(Vertex Shader)

Vertex shader对每个顶点的属性输入进行基本的处理，然后逐个顶点输出。<strong><font color="darkviolet">它是用户必须提供的</font></strong>。

处理方式<strong><font color="darkviolet">由用户自定义的程序来决定</font></strong>。通常有一个操作是基本要完成的：把MVP(model-matrix, view-matrix, projection-matrix)传入，然后对顶点model-space的3D坐标进行矩阵变换，得到clip-space的坐标，然后把它放到一个特殊的输出里（见下）。

输出同样可以由用户定义，vertex shader可以有多个输出，不过有一个特殊的输出代表了顶点的最终位置；如果没有后续的顶点处理阶段，那么这个输出里应该放入顶点的clip-space坐标；这个输出就是***gl_Position***。

#### 优化

注意到vertex shader是对每个顶点进行处理的，所以<strong><font color="darkviolet">理论上它被调用的次数即渲染的顶点数</font></strong>。不过实际上，OpenGL会自己进行优化。

如果两个顶点的属性数据完全相同，那么vertex shader自然会有完全相同的输出，如果OpenGL能检测到这一次的输入数据和之前的某一次完全一样，那么它就可以重用之前的计算结果。

不过，OpenGL不可能在每次计算前真正地去比较这次输入和之前的输入是不是相同（这太慢了）。实际上，这种优化能实现的前提条件是：用户使用了<strong><font color="darkviolet">下标索引的渲染函数</font></strong>(indexed rendering functions)，或者使用了<strong><font color="darkviolet">实例化渲染</font></strong>(instanced rendering)。

> 下标索引的渲染：在上下文绑定了VAO之后，通过绑定一个缓冲对象到GL_ELEMENT_ARRAY_BUFFER上，然后调用类似于gl\*Draw\*Elements的命令，就会使用该缓冲对象的下标索引来进行渲染。

> 实例化渲染：使用glVertexAttribDivisor(index, divisor)([glDocs](http://docs.gl/gl4/glVertexAttribDivisor))来指定第index号属性的重用方式，然后调用glDrawArrayInstanced(*, *, count, instancecount)([glDocs](http://docs.gl/gl4/glDrawArraysInstanced))函数进行渲染。
> 
> instancecount为实例化渲染的图元的数量；如果某个index的devisor为0，则第index号属性的数据，每到下一个顶点就根据步长(stride)前进一次（即，和平常一样）；如果devisor为非0，那么第index号属性的数据每经过divisor个实例（即，经过devisor * count个顶点），才根据步长前进一次。

OpenGL会有一个储存vertex shader计算结果的cache，如果相同的索引下标或者实例再次出现，而之前的结果仍在cache中，就可以直接使用。

### 镶嵌/细分曲面/细分网格(Tessellation)

在这个阶段，网格可以被进一步分割；通过一头一尾两个用户定制的shader——TCS(tessellation control shader)和TES(tessellation evaluation shader)以及中间的一个固定功能的tessellator来实现。

![曲面细分](../../../../img/in-post/post-cg-basics/QuadFull.png)

<span id="LOD"></span>
当相机非常靠近一个物体的时候，我们可以通过网格细分使物体表面仍然能保持圆滑（如果我们希望它是圆滑的话）。理想情况下，我们希望<strong><font color="darkviolet">无论物体是远是近，都能有一致的三角形对像素密度</font></strong>——细分曲面能够满足这一需求，表面可以根据和相机的距离进行镶嵌，使每个三角形的尺寸都小于一个像素。

> 游戏开发者经常尝试使用一系列不同版本的三角形网格链去逼近这个三角形对像素密度，每一版本称为一个**层次细节**(level of detail - **LOD**)，LOD 0代表最高程度的镶嵌，在相机非常接近物体的时候使用，接下来是LOD 1，LOD 2……

> 另一种用于游戏中的技术是**动态镶嵌**(dynamic tessellation)，常用于可扩展的网格上，例如水面和地形。在此技术中，网格通常以高度场(height field)来表示。另一种技术是**渐进网格**，在物体远离相机的过程中，通过把某些棱收缩为点，能自动生成半连续的LOD链。（《游戏引擎架构 Game Engine Architecture》，Jason Gregory，电子工业出版社，，第10章，第10.1.1.2节，镶嵌）

这一阶段是可选的，在OpenGL4.0之后才有。

### 几何着色器(Geometry Shader)

几何着色器把每个输入的图元进行处理之后，输出0个或者更多的图元。这一阶段是可选的。

它可以移除图元、细分图元、直接输出原图元、或者进行一些额外的顶点处理，它甚至可以把转变图元的类型，把输入的点变成三角形、或者把输入的线变成点。如果按照法向量的方向对每一个图元进行一定距离的平移，那么可以制造出类似爆炸的效果（虽然不太逼真）：

![GS](../../../../img/in-post/post-cg-basics/geometry_shader_explosion.png)






## 2. 顶点后期处理(Vertex Post-Processing)

### Transform Feedback

这是一个可以捕获<strong><font color="royalblue">顶点处理阶段</font></strong>之后的图元，<strong><font color="darkviolet">把这些数据记录进缓存对象，以供之后的渲染使用</font></strong>的过程。

![tf](../../../../img/in-post/post-cg-basics/tf2.jpg)

TF在粒子系统中的应用最常见。在TF出现之前，更新粒子状态之后，需要把VBO中的数据全部更新一遍，然后重新传给显存，性能消耗较大。

而TF的出现，使得<strong><font color="darkviolet">这些更新可以交给shader来完成</font></strong>；它可以通过在缓存对象中记录之前一次的输出结果，进行位置更新后作为下一次渲染的输入传入给顶点处理阶段。

一点需要注意的是，因为缓存对象不能同时作为输出和输入，所以需要两个缓存对象交替作为输入和输出。

另外，若使用GL_POINT表达粒子，对于TF而言，几何着色器的输出也应该是GL_POINT，但是Rasterization的输入需要是三角形（这样在屏幕上才能看到成型的粒子），这两者是不可调和的矛盾。

解决方法是<strong><font color="royalblue">2-Pass</font></strong>：第一次pass的输入输出都是points，使它在Rasterization之前终止；第二次pass在几何着色器里把points膨胀为quads，然后再交付给管线继续完成渲染。

![tf_in_particles](../../../../img/in-post/post-cg-basics/tf.jpg)
<small class="img-hint">使用transform feedback在shader中实现粒子更新以及碰撞检测的图示。</small>

[中文具体介绍 Transform Feedback](http://www.zwqxin.com/archives/opengl/talk-about-transform-feedback.html) 
/ [英文具体介绍 Transform Feedback](https://open.gl/feedback)
/ [中文粒子教程](http://wiki.jikexueyuan.com/project/modern-opengl-tutorial/tutorial28.html)

### 裁切(Clipping)

之前的阶段产生的<strong><font color="darkviolet">图元会被裁切到<font color="royalblue">观察体</font>(viewing volume)</font></strong>中。一个顶点的观察体(clip-space)由以下决定：

$$
\begin{cases}
-w_c \le x_c \le w_c \\
-w_c \le y_c \le w_c \\
-w_c \le z_c \le w_c \\
\end{cases}
$$

其中$$w_c$$是顶点的<strong><font color="royalblue">齐次坐标</font></strong>。

接下来讲述的事情发生在裁切之前。从camera-space中出来，$$w_c$$的值经过投影矩阵进入clip-space中，它的值有可能发生变化，这取决于投影矩阵(projection matrix)是正射(orthographic)还是透视(perspective)。

<strong><font color="darkviolet">如果投影矩阵是正射的，那么$$w_c$$不会发生变化</font></strong>：如果原值是1.0，which is the most often case，那么现在还是保持1.0；此时可以把观察体想象为一个长方体。

<strong><font color="darkviolet">如果投影矩阵是透视的，那么$$w_c$$会发生改变</font></strong>，它的值和顶点与相机的距离呈线性正相关：距离越远，$$w_c$$越大；这意味着，距离相机越远，顶点被保留下来的可能性越大（因为看得越远，视线范围越大）！此时想象中的观察体变成了一个四棱锥的一部分。

![Frustum](../../../../img/in-post/post-cg-basics/ViewFrustum.png)

~~接下来讲述的事情发生在裁切之前。投影矩阵相当于指定了一个范围的坐标，即创建了一个观察体，这个观察体的形状是平截头体(frustum)，它可能是一个长方体，也可能是一个四棱锥的一部分，取决于投影矩阵是正射(orthographic)还是透视(perspective)。~~

~~投影矩阵会把camera-space的坐标投影到clip-space中，那些在camera-space中处于观察体之内的顶点，经过投影之后会处于<strong><font color="royalblue">标准化设备坐标</font></strong>(normalized device coordinates - NDC)的范围之内，即(-1.0, 1.0)；而在camera-space中处于观察体范围之外的顶点，投影后不会在(-1.0, 1.0)的范围之内（x，y，z任意一个坐标不在其中都属于此情况），那么它（如果不作特别设定）会被裁切掉。~~

回到裁切阶段：<strong><font color="darkviolet">z方向的裁切可以被用户禁用</font></strong>，如果禁用，则近平面到相机之间的图元以及远平面以外的图元将不会被裁切。

另外，用户还可以通过gl_ClipDistance来自定义需要裁切的距离(详见[OpenGL wiki](https://www.khronos.org/opengl/wiki/Vertex_Post-Processing#User-defined_clipping))。

如果顶点需要裁切，会根据图元的类型进行对应的裁切：如果是点，则直接丢弃；如果是线和三角形，并且有一部分在观察体之内，则进行裁切，经过裁切后的顶点会在观察体的边缘上，另外，三角形经过裁切可能变成多个三角形。

<strong><font color="darkorange">此时顶点位置仍然在clip-space之中，经过后面的perspective divide和viewport transform，顶点位置将进入window-space。</font></strong>

### 透视除法(Perspective Division)

透视除法把clip-space坐标转变为<strong><font color="royalblue">标准化设备坐标(normalized device coordinates - NDC)</font></strong>

$$
\begin{pmatrix} 
x_{ndc} \\ 
y_{ndc} \\ 
z_{ndc} 
\end{pmatrix} 
= \begin{pmatrix} 
\frac{x_c}{w_c} \\ 
\frac{y_c}{w_c} \\ 
\frac{z_c}{w_c} 
\end{pmatrix}
$$

还记得上面说过，如果投影矩阵是透视的，那么距离相机越远，$$w_c$$就越大；所以在透视除法里，距离相机越远，顶点坐标被除后就越小——这正与我们对于透视的直觉相符合！

（一篇关于投影矩阵计算的[文章](http://www.songho.ca/opengl/gl_projectionmatrix.html)/红宝书关于齐次坐标和变换矩阵的[介绍](http://glprogramming.com/red/appendixf.html)）

现在，顶点坐标已经<strong><font color="darkorange">从4D的clip-space转换到3D的NDC之内了</font></strong>——透视除法就是这么简单而巧妙。

### Viewport Transform

用户通过***glViewport***、***glDepthRange***等函数指定viewport范围，<strong><font color="darkorange">viewport transform通过方程把3D的NDC坐标转换到3D的window space。</font></strong>




## 3. 图元装配(Primitive Assembly)

图元装配是<strong><font color="darkviolet">从前一阶段收集一系列的输出顶点数据，并将其组合成一系列简单的图元（lines，points，or triangles）的过程</font></strong>。用户需要渲染的图元类型决定了此过程的工作方式。

比如，如果输入是包含12个顶点的triangle strip，那么输出就是10个triangles。

如果曲面细分(tessellation)和几何着色器被启用，那么在这些阶段进行之前还会有一定程度的图元装配发生，因为它们需要的输入是一些单独的图元，而非连续的一系列顶点。

### 中止光栅化(Rasterizer Discard)

通过激活***GL_RASTERIZER_DISCARD***可以丢弃掉所有图元，<strong><font color="darkviolet">阻止它们被光栅化</font></strong>。

可以用来测量之前阶段的性能，也可以<strong><font color="darkviolet">避免Transform Feedback过程中产生的图元被渲染</font></strong>。

### 面剔除(Face Culling)

#### 环绕顺序(Winding Order)

当用户下达渲染指令，渲染管线会根据用户制定好的顺序处理顶点。虽然几何着色器GS可以改变顶点的顺序，但在GS的单一调用中处理的顶点，和其他的顶点仍然保持着相对顺序的不变。

当顶点经过图元装配阶段从而被分配为基本的图元时，组成图元的顶点之间的相互顺序会被记录。如果图元是三角形，那么<strong><font color="darkviolet">可以根据三个顶点按顺序的旋转方向来判断它面向我们的是正面还是背面</font></strong>。

![winding-order](../../../../img/in-post/post-cg-basics/Winding_order.png)

用户可以自定义正面的顶点旋转方向（**这是一个全局的状态**）：

```C++
glFrontFace(GL_CCW);		// 设置counter-clockwise是正面(默认设置)
glFrontFace(GL_CW);			// 设置clockwise是正面
```

<strong><font color="royalblue">片段着色器</font></strong>(fragment shader)会得到一个内建的输入值来告知当前片段是否由三角形的正面产生的（**对于其他形状，总是正面**）。

#### 剔除(Culling)

定义三角形哪一面是正面，可以用于剔除那些背面面对相机的三角形（或者正面面对相机，可以自行更改）。

假设有一个立方体（由12个三角形组成），那么其中至少有6个三角形是和其他的三角形们面对的方向是不同的。除非立方体是透明的，否则<strong><font color="darkviolet">至少有6个三角形始终会被其他的三角形所遮挡</font></strong>；在某些情况下，可能除了一个面（即两个三角形）之外，其他面都朝向远离相机的方向。

面剔除可以在进行<strong><font color="royalblue">光栅化</font></strong>和<strong><font color="royalblue">片段着色器</font></strong>这些<strong><font color="darkviolet">昂贵的操作之前，丢弃掉那些在封闭曲面(closed surfaces)中，不能被相机看到的三角形</font></strong>。

面剔除同样可以用户定制：

```C++
glEnable(GL_CULL_FACE);		// 启用面剔除（默认不启用）
glCullFace(GL_BACK);		// 设置背面朝向相机的三角形被剔除，默认为背面剔除（其他：GL_FRONT / GL_FRONT_AND_BACK）
```

注意启用了***GL_FRONT_AND_BACK***的面剔除即剔除所有三角形，但<strong><font color="darkviolet">这和中止光栅化不同</font></strong>——前者只剔除三角形（因为只有三角形有面），后者则阻止**所有**图元进入光栅化阶段。

## 4. 光栅化(Rasterization)

光栅化是<strong><font color="darkviolet">把3D的window-space坐标转换到2D的screen-space坐标（即像素坐标）的过程</font></strong>。

跟其他的渲染技术（比如光线跟踪(ray tracing)）相比，光栅化是非常快速的。它通常由渲染管线里面固定的功能硬件实现，因为没有必要进行自定义。

### 采样和过滤(Sampling and Filtering)

渲染，是一个<strong><font color="darkviolet">试图用有限的像素颜色把图像空间中一些连续的方程(continuous function)描绘出来</font></strong>的过程。那些小于一个像素的细节、尖峰、低谷，即一些高频的信号，是无法表现出来的。如果不使用采样和过滤，就可能出现难看的<strong><font color="royalblue">失真</font></strong>(aliasing)现象。

渲染像素时，图形硬件会计算像素中心落入纹理空间的位置，来对纹理贴图<strong><font color="royalblue">采样</font></strong>。通常，图形硬件需要采样出多于一个纹理像素，并把采样结果混合以得出实际的采样纹理像素颜色，此过程称为<strong><font color="royalblue">纹理过滤</font></strong>。

如果我们使用纹理，那么我们希望<strong><font color="darkviolet">纹理像素和屏幕像素的分辨率之比（称为<strong><font color="royalblue">纹素密度</font></strong>）接近于1</font></strong>；如果远低于1，每个纹理像素会显著地比屏幕像素大，纹理像素的边缘会明显被觉察到，影响视觉品质；如果远大于1，许多纹理像素会影响单个屏幕像素，会产生**莫列波纹**，并且如果还用非常高的纹素密度去渲染，将会浪费内存。

<table>
    <tr>
        <td ><center><img src="../../../../img/in-post/post-cg-basics/Moire_pattern_of_bricks.jpg" width="300" >合理的采样结果 </center></td>
        <td ><center><img src="../../../../img/in-post/post-cg-basics/Moire_pattern_of_bricks_small.jpg" width="300" >不合理的采样出现的莫列波纹</center></td>
    </tr>
</table>

理想情况下，我们希望<strong><font color="darkviolet">在相机和物体的距离发生变化时，纹素密度可以保持在1附近</font></strong>。有时候，一个屏幕像素可能要从上百个纹理像素中采样混合，如果每次都对原纹理图像进行采样混合操作，会产生可观的性能和空间的开销，同时加重GPU和CPU的负担；**多级渐远纹理**可以帮助我们更轻松地实现维持纹素密度稳定的这个目标。

#### 多级渐远纹理(Mipmapping)

<details><summary>展开/折叠详细</summary>

<p>Mipmap通过把一个纹理图像不断downsize成原先的1/4（长和宽各变为1/2），获得一个mipmap set，每个mip level的像素由前一级mip level经过纹理过滤得到。如果原纹理图像分辨率是256*256时，它的mipmap set包含了8张图像，分辨率依次为：128*128，64*64，32*32，16*16，8*8，4*4，2*2，1*1。</p>

<center><img src="../../../../img/in-post/post-cg-basics/mipmap4.png">mipmap纹理过滤图示</center>

<p>当使用mipmap时，图形硬件会根据相机和三角形的距离，选择合适的渐远纹理级数(mip level)，以尝试维持纹素密度接近于1。一种方法是采样一个高于理想分辨率的mip level，例如，若纹理占据屏幕40*40的位置，那么可能会选择64*64的mip level；或者，可以同时采样一个高于、一个低于理想分辨率的mip level，然后对两者的采样结果进行线性插值(这种方法即<strong><font color="royalblue">三线性过滤</font></strong>，trilinear filtering)。</p>

<p>多级渐远纹理的存储与原纹理图像相比，需要（至少）额外的33%的空间。另外，mipmap还可以用在之前提到的<a href="#LOD">层次细节(LOD)</a>上。</p>

<center><img src="../../../../img/in-post/post-cg-basics/MipMap_Example_STS101.jpg">一个mipmap图像存储的例子，原图像在左侧</center>

</details>

（一篇很棒的介绍纹理和采样的文章：[Textures and Sampling](https://cglearn.codelight.eu/pub/computer-graphics/textures-and-sampling)）

纹理过滤的方法：

1. 最近邻
2. 双线性
3. 三线性
4. 各向异性

### 隐藏面移除(Hidden Surface Removal)

隐藏面移除(HSR)，也称hidden surface determination。这个过程里，那些不会被用户看到的表面（比如被不透明物体遮挡的表面）将被移除出渲染管线，避免浪费渲染资源。目前有许多可供完成HSM的技术。

#### Z-buffering

目前的标准实现。





# <font color="CornflowerBlue">test</font>
CornflowerBlue
LightSkyBlue
http://www.runoob.com/tags/html-colorname.html




https://en.wikipedia.org/wiki/Rasterisation
https://en.wikipedia.org/wiki/Hidden-surface_determination
https://en.wikipedia.org/wiki/Bresenham%27s_line_algorithm#Algorithm_for_integer_arithmetic
https://en.wikipedia.org/wiki/Mipmap
