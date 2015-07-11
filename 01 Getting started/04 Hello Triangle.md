本文作者JoeyDeVries，由[Django](http://bullteacher.com/5-hello-triangle.html)翻译自[http://learnopengl.com](http://www.learnopengl.com/#!Getting-started/Hello-Triangle)

# Hello Triangle

在OpenGL中任何事物都在3D空间中，但是屏幕和窗口是一个2D像素阵列，所以OpenGL的大部分工作都是关于如何把3D坐标转变为适应你的屏幕的2D像素。3D坐标转为2D坐标的处理过程是由OpenGL的**图形输送管道**(pipeline，大多译为管线，实际上指的是一堆原始图形数据途经一个输送管道，期间经过各种变化处理最终出现在屏幕的过程)管理的。图形输送管道可以被划分为两个主要部分：第一个部分把你的3D坐标转换为2D坐标，第二部分是把2D坐标转变为实际的有颜色的像素。这个教程里，我们会简单地讨论一下图形输送管道，以及如何使用它创建一些像素，这对我们来说是有好处的。

<div style="border:solid #AFDFAF;border-radius:10px;background-color:#D8F5D8;margin:10px 10px 10px 0px;padding:10px">
2D坐标和像素也是不同的，2D坐标是在2D空间中的一个点的非常精确的表达，2D像素是这个点的近似，它受到你的屏幕/窗口解析度的限制。
</div>

图形输送管道接收一组3D坐标，然后把它们转变为你屏幕上的有色2D像素。图形输送管道可以被划分为几个阶段，每个阶段需要把前一个阶段的输出作为输入。所有这些阶段都是高度专门化的（它们有一个特定的函数），它们能简单地并行执行。由于它们的并行执行的特征，当今大多数显卡都有成千上万的小处理核心，在GPU上为每一个（输送管道的）阶段运行各自的小程序，从而在图形输送管道中快速处理你的数据。这些小程序叫做**着色器**(Shader)。

有些着色器允许开发者自己配置，我们可以用自己写的着色器替换默认的。这样我们就可以更细致地控制输送管道的特定的部分了，因为它们运行在GPU上，它们也会节约宝贵的CPU时间。着色器是用**OpenGL着色器语言**(OpenGL Shading Language, GLSL)写成的，我们在下节会花更多时间研究它。

在下面，你会看到一个图形输送管道的每个阶段的抽象表达。要注意蓝色部分代表的是我们可以自定义的着色器。

![](http://bullteacher.com/wp-content/uploads/2015/05/pipeline.gif)

如你所见，图形输入管道包含很多部分，每个都是将你的顶点数据转变为最后渲染出来的像素这个大过程中的一个特定阶段。我们会概括性地解释输送管道的每个部分，从而使你对输送管道的工作方式有个大概了解。

我们以数组的形式传递3个3D坐标作为图形输送管道的输入，它用来表示一个三角形，这个数组叫做顶点数据（Vertex Data）；这里顶点数据是一些顶点的集合。一个**顶点**是一个3D坐标的集合（也就是x、y、z数据）。而顶点数据是用**顶点属性**（vertex attributes）表示的，它可以包含任何我们希望用的数据，但是简单起见，我们还是假定每个顶点只由一个3D位置①和几个颜色值组成的吧。

①译注:当我们谈论一个“位置”的时候，它代表在一个“空间”中所处地点的这个特殊属性；同时“空间”代表着任何一种坐标系，比如x、y、z三维坐标系，x、y二维坐标系，或者一条直线上的x和y的线性关系，只不过二维坐标系是一个扁扁的平面空间，而一条直线是一个很瘦的长长的空间。

<div style="border:solid #AFDFAF;border-radius:10px;background-color:#D8F5D8;margin:10px 10px 10px 0px;padding:10px">
为了让OpenGL知道我们的坐标和颜色值构成的到底是什么，OpenGL需要你去提示你希望这些数据所表示的是什么类型。我们是希望把这些数据渲染成一系列的点？一系列的三角形？还是仅仅是一个长长的线？做出的这些提示叫做**基本图形**(Primitives)，任何一个绘制命令的调用都必须把基本图形类型传递给OpenGL。这是其中的几个：`GL_POINTS`、`GL_TRIANGLES`、`GL_LINE_STRIP`。
</div>

输送管道的第一个部分是**顶点着色器**(Vertex Shader)，它把一个单独的顶点作为输入。顶点着色器主要的目的是把3D坐标转为另一种3D坐标（后面会解释），同时顶点着色器允许我们对顶点属性进行一些基本处理。

**基本图形组装**（Primitive Assembly）阶段把顶点着色器的表示为基本图形的所有顶点作为输入（如果选择的是`GL_POINTS`，那么就是一个单独顶点），把所有点组装为特定的基本图形的形状；本节例子是一个三角形。

基本图形组装阶段的输出会传递给**几何着色器**（Geometry Shader）。几何着色器把基本图形形式的一系列顶点的集合作为输入，它可以通过产生新顶点构造出新的（或是其他的）基本图形来生成其他形状。例子中，它生成了另一个三角形。

**细分着色器**（Tessellation Shaders）拥有把给定基本图形**细分**为更多小基本图形的能力。这样我们就能在物体更接近玩家的时候通过创建更多的三角形的方式创建出更加平滑的视觉效果。

细分着色器的输出会进入**像素化**（Rasterization也译为光栅化）阶段，这里它会把基本图形映射为屏幕上相应的像素，生成供片段着色器（Fragment Shader）使用的片段(Fragment)②。在片段着色器运行之前，会执行**裁切**（clipping）。裁切会丢弃超出你的视图以外的那些像素，来提升执行效率。

②译注：Fragment通常译为片段，但从根本上来说它就是带有一些额外信息的像素，由于带有额外信息，OpenGL就没有给它取名叫“像素”，所以“片段”的中文译法比较不准确听起来就像一个fragment可能包含多个像素一样，实际上一个fragment只包含一个像素，现在你只要简单记住它是像素就行了。后面你会知道除了像素信息以外它还包含了什么。

<div style="border:solid #AFDFAF;border-radius:10px;background-color:#D8F5D8;margin:10px 10px 10px 0px;padding:10px">
OpenGL中的一个fragment是OpenGL渲染一个独立像素所需的所有数据。
</div>

**片段着色器**的主要目的是计算一个像素的最终颜色，这也是OpenGL高级效果产生的地方。通常，片段着色器包含用来计算像素最终颜色的3D场景的一些数据（比如光照、阴影、光的颜色等等）。

在所有相应颜色值确定以后，最终它会传到另一个阶段，我们叫做**alpha测试**和**混合**（Blending）阶段。这个阶段检测像素的相应的深度（和Stencil）值（后面会讲），使用这些，来检查这个像素是否在另一个物体的前面或后面，如此做到相应取舍。这个阶段也会查看**alpha值**（alpha值是一个物体的透明度值）和物体之间的**混合**（Blend）。所以即使在片段着色器中计算出来了一个像素所输出的颜色，最后的像素颜色在渲染多个三角形的时候也可能完全不同。

正如你所见的那样，图形输送管道非常复杂，它包含很多要配置的部分。然而，对于大多数场合，我们必须做的只是顶点和片段着色器。几何着色器和细分着色器是可选的，通常使用默认的着色器就行了。

在现代OpenGL中，我们**必须**定义至少一个顶点着色器和一个片段着色器（因为GPU中没有默认的顶点/片段着色器）。出于这个原因，开始学习现代OpenGL的时候非常困难，因为在你能够渲染自己的第一个三角形之前需要一大堆知识。这章结束就是你可以最终渲染出你的三角形的时候，你也会了解到很多图形编程知识。

## 顶点输入

开始绘制一些东西之前，我们必须给OpenGL输入一些顶点数据。OpenGL是一个3D图形库，所以我们在OpenGL中指定的所有坐标都是在3D坐标里（x、y和z）。OpenGL不是简单的把你所有的3D坐标变换为你屏幕上的2D像素；OpenGL只是在当它们的3个轴（x、y和z）在特定的-1.0到1.0的范围内时才处理3D坐标。所有在这个范围内的坐标叫做**标准化设备坐标**（Normalized Device Coordinates）会最终显示在你的屏幕上（所有出了这个范围的都不会显示）。

由于我们希望渲染一个三角形，我们指定所有的这三个顶点都有一个3D位置。我们把它们以`GLfloat`数组的方式定义为标准化设备坐标（也就是在OpenGL的可见区域）中。

```c++
GLfloat vertices[] = {
    -0.5f, -0.5f, 0.0f,
     0.5f, -0.5f, 0.0f,
     0.0f,  0.5f, 0.0f
};
```

由于OpenGL是在3D空间中工作的，我们渲染一个2D三角形，它的每个顶点都要有一个z坐标0.0。在这样的方式中，三角形的每一处的深度（depth）都一样，从而使它看上去就像2D的③。

③译注：通常深度可以理解为z坐标，它代表一个像素在空间中和你的距离，如果离你远就可能被别的像素遮挡，你就看不到它了，它会被丢弃，以节省资源。

<div style="border:solid #AFDFAF;border-radius:10px;background-color:#D8F5D8;margin:10px 10px 10px 0px;padding:10px">
**标准化设备坐标**

一旦你的顶点坐标已经在顶点着色器中处理过，它们就应该是**标准化设备坐标**了，标准化设备坐标是一个x、y和z值在-1.0到1.0的一小段空间。任何落在范围外的坐标都会被丢弃/裁剪，不会显示在你的屏幕上。下面你会看到我们定义的在标准化设备坐标中的三角形（忽略z轴）：

![](http://www.learnopengl.com/img/getting-started/ndc.png)

与通常的屏幕坐标不同，y轴正方向上的点和（0, 0）坐标是这个图像的中心，而不是左上角。最后你希望所有（变换过的）坐标都在这个坐标空间中，否则它们就不可见了。

你的标准化设备坐标接着会变换为**屏幕空间坐标**（screen-space coordinates），这是使用你通过`glViewport`函数提供的数据，进行**视口变换**（viewport transform）完成的。最后的屏幕空间坐标被变换为像素输入到片段着色器。
</div>

有了这样的顶点数据，我们会把它作为输入发送给图形输送管道的第一个处理阶段：顶点着色器。它会在GPU上创建储存空间用于储存我们的顶点数据，还要配置OpenGL如何解释这些内存，并且指定如何发送给显卡。顶点着色器接着会处理我们告诉它要处理内存中的顶点的数量。

我们通过**顶点缓冲对象**（Vertex Buffer Objects, VBO）管理这个内存，它会在GPU内存储存大批顶点。使用这些缓冲对象的好处是我们可以一次性的发送一大批数据到显卡上，而不是每个顶点发送一次。从CPU把数据发送到显卡相对较慢，所以无论何处我们都要尝试尽量一次性发送尽可能多的数据。当数据到了显卡内存中时，顶点着色器几乎立即就能获得顶点，这非常快。

顶点缓冲对象（VBO）是我们在[OpenGL教程](http://www.learnopengl.com/#!Getting-Started/OpenGL)中第一个出现的OpenGL对象。就像OpenGL中的其他对象一样，这个缓冲有一个独一无二的ID，所以我们可以使用`glGenBuffers`函数生成一个缓冲ID：

```c++
GLuint VBO;
glGenBuffers(1, &VBO);  
```

OpenGL有很多缓冲对象类型，`GL_ARRAY_BUFFER`是其中一个顶点缓冲对象的缓冲类型。OpenGL允许我们同时绑定多个缓冲，只要它们是不同的缓冲类型。我们可以使用`glBindBuffer`函数把新创建的缓冲绑定到`GL_ARRAY_BUFFER`上：

```c++
glBindBuffer(GL_ARRAY_BUFFER, VBO);  
```

从这一刻起，我们使用的任何缓冲函数（在`GL_ARRAY_BUFFER`目标上）都会用来配置当前绑定的缓冲（`VBO`）。然后我们可以调用`glBufferData`函数，它会把之前定义的顶点数据复制到缓冲的内存中：

```c++
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
```

`glBufferData`是一个用来把用户定义数据复制到当前绑定缓冲的函数。它的第一个参数是我们希望把数据复制到上面的缓冲类型：顶点缓冲对象当前绑定到`GL_ARRAY_BUFFER`目标上。第二个参数指定我们希望传递给缓冲的数据的大小（字节）；用一个简单的`sizeof`计算出顶点数据就行。第三个参数是我们希望发送的真实数据。

第四个参数指定了我们希望显卡如何管理给定的数据。有三种形式：

- GL_STATIC_DRAW：数据不会或几乎不会改变。
- GL_DYNAMIC_DRAW：数据会被改变很多。
- GL_STREAM_DRAW：数据每次绘制时都会改变。

三角形的位置数据不会改变，每次渲染调用时都保持原样，所以它使用的类型最好是`GL_STATIC_DRAW`。如果，比如，一个缓冲中的数据将频繁被改变，那么使用的类型就是`GL_DYNAMIC_DRAW`或`GL_STREAM_DRAW`。这样就能确保图形卡把数据放在高速写入的内存部分。

现在我们把顶点数据储存在显卡的内存中，用VBO顶点缓冲对象管理。下面我们会创建一个顶点和片段着色器，来处理这些数据，所以我们开始创建它们吧。

## 顶点着色器