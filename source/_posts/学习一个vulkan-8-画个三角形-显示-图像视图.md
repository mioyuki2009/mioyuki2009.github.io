---
title: 学习一个vulkan(8)-画个三角形(显示)-图像视图
date: 2018-11-06 09:11:35
tags: vulkan
categories: learn
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
<b>图像视图</b>、
要在渲染管道中使用任何VkImage，包括交换链中的，我们必须创建一个VkImageView对象。
图像视图实际上是图像的视图。它描述了如何访问图像以及要访问的图像的哪一部分，例如决定是否被作为2D纹理而不使用任何Mip贴图（MipMapping）来处理。

在本章中，我们将编写一个createImageViews函数，为交换链中的每个图像创建一个基本图像视图，以便我们以后可以将它们用作着色目标。

首先添加一个类成员来存储图像视图：
```cpp
std::vector<VkImageView> swapChainImageViews;
```
创建createImageViews函数并在创建交换链后立即调用：
```cpp
void initVulkan() {
    createInstance();
    setupDebugCallback();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
}

void createImageViews() {

}
```
我们需要做的第一件事是调整列表大小以适应我们将要创建的所有图像视图：
```cpp
void createImageViews() {
    swapChainImageViews.resize(swapChainImages.size());

}
```
接下来，设置循环来遍历所有交换链图像：
```cpp
for (size_t i = 0; i < swapChainImages.size(); i++) {

}
```
创建图像视图的参数在VkImageViewCreateInfo结构中指定。
前几个参数很简单：
```cpp
VkImageViewCreateInfo createInfo = {};
createInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
createInfo.image = swapChainImages[i];
```
viewType和format参数指定应以何种方式解析图像数据。viewType参数允许您将图像设置为一维纹理，二维纹理，三维纹理或者立方体贴图：
```cpp
createInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
createInfo.format = swapChainImageFormat;
```
components参数设置对应的颜色通道。例如，您可以将所有通道映射到红色通道以获得单色纹理，还可以将常量值0和1映射到通道。在本例中，我们将使用默认映射。
```cpp
createInfo.components.r = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.g = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.b = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.a = VK_COMPONENT_SWIZZLE_IDENTITY;
```
subresourceRange参数将描述图像用于何种用途以及访问图像的哪个部分。我们的图像将用于着色目的，没有任何mipmapping级别或多图层：
```cpp
createInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
createInfo.subresourceRange.baseMipLevel = 0;
createInfo.subresourceRange.levelCount = 1;
createInfo.subresourceRange.baseArrayLayer = 0;
createInfo.subresourceRange.layerCount = 1;
```
如果您正在处理立体3D应用程序，那么将创建一个包含多个图层的交换链。然后，您可以通过访问不同的图层为每个图像创建多个图像视图，以表示左眼和右眼的视图。

现在，创建图像视图就只需要调用vkCreateImageView了：
```cpp
if (vkCreateImageView(device, &createInfo, nullptr, &swapChainImageViews[i]) != VK_SUCCESS) {
    throw std::runtime_error("failed to create image views!");
}
```
与图像不同，图像视图是由我们显式创建的，因此我们需要添加一个类似的循环以在程序结束时再次销毁它们：
```cpp
void cleanup() {
    for (auto imageView : swapChainImageViews) {
        vkDestroyImageView(device, imageView, nullptr);
    }

    ...
}
```
图像视图已经足够作为将图像用作纹理功能的开端了，但它现在还不能用于渲染目标。还需要一个称为帧缓冲的间接步骤。但首先我们必须设置图形管道。






























