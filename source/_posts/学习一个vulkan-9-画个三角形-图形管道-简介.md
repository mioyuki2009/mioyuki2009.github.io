---
title: 学习一个vulkan(9)-画个三角形(图形管道)-简介
date: 2018-11-07 09:23:19
tags: vulkan
categories: learn
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
<b>简介</b>
在接下来的几章中，我们将设置一个图形管道，用来配置绘制的第一个三角形。
图形管道是一系列操作，它们将网格的顶点和纹理来渲染目标中的像素点。一个简化的流程如下所示：
![](学习一个vulkan-9-画个三角形-图形管道-简介/1.png)
The input assembler从指定的缓冲区收集原始顶点数据，也可以使用索引缓冲区重复使用某些元素，而不必复制顶点数据本身。

The vertex shader是针对每个顶点的着色器，而且可以完成将顶点位置从模型空间转换到屏幕空间的转换，同时也能沿着管道传输顶点数据。

The tessellation shaders 允许根据特定规则来细分几何体以提高网格质量。这通常用于使砖墙和楼梯等表面在附近时看起来不那么平坦。

The geometry shader 可以对每一个基元（三角形，点，线）进行操作，而且能够破坏和创建基元，这一点和曲面细分着色器（tessellation shader）很像，但是会更加灵活。但是，它在今天的应用程序中并没有得到太多应用，因为除了英特尔的集成GPU之外，他在大多数显卡的表现都不是很好。

The rasterization 将图元离散化为候选像素（fragments），这是用来填充到帧缓冲区的像素。任何落在屏幕之外的候选像素都将被丢弃，顶点着色器输出的属性将在候选像素之间进行插值，如上图所示。由于深度测试，通常在这里也丢弃其他原始候选像素之后的一段。

The fragment shader 将作用于存活的候选像素，决定它们的颜色，深度以及将写入那一个帧缓冲区。它可以使用来自顶点着色器的插值数据来完成此操作，其中包括纹理坐标和法线照明等。

The color blending 用于将帧缓冲区中映射为相同像素的不同候选像素混合到一起。候选像素可以简单地相互覆盖，相加或基于透明度进行混合。

上图中绿色的阶段称为fixed-function。这些阶段允许您使用参数来调整其中的操作，但它们的工作方式是预定义的。

黄色的阶段称为programmable，这意味着可以使用自己的代码来操作显卡，这样就可以完全应用您所想要的操作了。The fragment shader就是这样，可以自由实现从纹理和照明到光线追踪器的任何事情。这些程序可以同时在多个GPU内核上运行，；来处理多个对象，比如并行的顶点和候选像素。

如果您之前使用过比较旧的API，如OpenGL和Direct3D，您可能会习惯于能够通过glBlendFunc和OMSetBlendState等函数随意更改任何管道设置。Vulkan中的图形管道几乎完全不可变，因此如果要更改着色器，绑定不同的帧缓冲区或更改混合函数，则必须从头开始重新创建管道，这个的缺点是必须创建许多管道，这些管道代表要在渲染操作中使用的所有状态组合，但是，因为在管道中进行的所有操作都是已预先知道了，所以驱动程序也可以更好地优化它。

根据不同的工作要求，某些可编程的阶段是可选的。例如，如果只是绘制简单几何体，则可以禁用曲面细分和几何着色器阶段。如果只对深度值感兴趣，则可以禁用片段着色器阶段，这对[阴影贴图](https://en.wikipedia.org/wiki/Shadow_mapping)生成很有用。

在下一章中，我们将首先创建将绘制三角形的两个可编程阶段：顶点着色器（the vertex shader）和片段着色器（fragment shader）。 混合模式，视口，光栅化等固定功能配置将在此后的章节中设置。在Vulkan中设置图形管道的最后一部分将涉及输入和输出帧缓冲区的规范。

在initVulkan中紧接着createImageViews的后面创建一个createGraphicsPipeline函数。我们将在之后的章节中来实现此功能：
```cpp
void initVulkan() {
    createInstance();
    setupDebugCallback();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
    createGraphicsPipeline();
}

...

void createGraphicsPipeline() {

}
```