---
title: 学习一个vulkan(25)-纹理映射-图像视图和采样器
date: 2018-11-26 09:07:43
tags: vulkan
categories: learn
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
在本章节我们将为图形管线创建另外两个资源来对图像进行采样，第一个资源是我们在使用交换链图像时已经见过的，第二个资源是新的-它涉及着色器如何从图像中读取纹素。

<b>纹理图像视图</b>
我们之前已经看到，使用交换链图像和帧缓冲，图像是通过图像视图而不是直接访问的。我们也需要为纹理图像创建这样的图像视图。

添加一个类成员来保存纹理图像的VkImageView，并创建一个新函数createTextureImageView，我们将在函数内进行创建：
```cpp
VkImageView textureImageView;

...

void initVulkan() {
    ...
    createTextureImage();
    createTextureImageView();
    createVertexBuffer();
    ...
}

...

void createTextureImageView() {

}
```
此函数的代码可以直接使用createImageViews进行更改。唯一需要进行的两项更改是format和image：
```cpp
VkImageViewCreateInfo viewInfo = {};
viewInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
viewInfo.image = textureImage;
viewInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
viewInfo.format = VK_FORMAT_R8G8B8A8_UNORM;
viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
viewInfo.subresourceRange.baseMipLevel = 0;
viewInfo.subresourceRange.levelCount = 1;
viewInfo.subresourceRange.baseArrayLayer = 0;
viewInfo.subresourceRange.layerCount = 1;
```
这里没有了显式的viewInfo.components初始化，因为VK_COMPONENT_SWIZZLE_IDENTITY被定义为0。嘴好通过调用vkCreateImageView完成创建图像视图：
```cpp
if (vkCreateImageView(device, &viewInfo, nullptr, &textureImageView) != VK_SUCCESS) {
    throw std::runtime_error("failed to create texture image view!");
}
```
因为很多逻辑都是从createImageViews复制过来的，所以可以新建一个createImageView函数来封装该部分逻辑：
```cpp
VkImageView createImageView(VkImage image, VkFormat format) {
    VkImageViewCreateInfo viewInfo = {};
    viewInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
    viewInfo.image = image;
    viewInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
    viewInfo.format = format;
    viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    viewInfo.subresourceRange.baseMipLevel = 0;
    viewInfo.subresourceRange.levelCount = 1;
    viewInfo.subresourceRange.baseArrayLayer = 0;
    viewInfo.subresourceRange.layerCount = 1;

    VkImageView imageView;
    if (vkCreateImageView(device, &viewInfo, nullptr, &imageView) != VK_SUCCESS) {
        throw std::runtime_error("failed to create texture image view!");
    }

    return imageView;
}
```
createTextureImageView函数现在可以简化为：
```cpp
void createTextureImageView() {
    textureImageView = createImageView(textureImage, VK_FORMAT_R8G8B8A8_UNORM);
}
```
而createImageViews可以简化为：
```cpp
void createImageViews() {
    swapChainImageViews.resize(swapChainImages.size());

    for (uint32_t i = 0; i < swapChainImages.size(); i++) {
        swapChainImageViews[i] = createImageView(swapChainImages[i], swapChainImageFormat);
    }
}
```
确保在销毁图像本身前清理图像视图：
```cpp
void cleanup() {
    cleanupSwapChain();

    vkDestroyImageView(device, textureImageView, nullptr);

    vkDestroyImage(device, textureImage, nullptr);
    vkFreeMemory(device, textureImageMemory, nullptr);
```

<b>采样</b>
着色器可以直接从图像中读取纹素，但是当它们用作纹理时，并不经常使用这种方式。纹理通常通过采样器访问，采样器将应用过滤和变换来计算得到的最终颜色。

这些过滤器有助于解决过采样等问题。就是一个映射到几何图形的纹理图像，拥有比纹素更多的片元的情况。如果您只是为每个片段中的纹理坐标选取最接近的纹素，那么您将获得与第一个图像类似的结果：
![](学习一个vulkan-25-纹理映射-图像视图和采样器\1.png)

如果您通过线性插值组合了4个最接近的纹素，那么您将获得更平滑的结果，如右侧的结果。当然你的应用程序也可能有需要更符合左边风格的艺术风格要求（Minecraft），但在传统的图形应用中，右边是首选。当从纹理中读取颜色时，采样器对象会自动为您应用此过滤器。

欠采样会有相反的问题，就是在有更多的纹素而不是片元的情况下。比如对于棋盘的纹理进行采样，会导致在有锐角的地方产生失真。
![](学习一个vulkan-25-纹理映射-图像视图和采样器\2.png)

如左图所示，纹理在远处变的模糊。解决方案是使用[各向异性过滤](https://en.wikipedia.org/wiki/Anisotropic_filtering)，这个也可以由采样器自动应用。

除了这些滤波器，采样器还可以处理各种转换。它决定了当您尝试通过其寻址模式读取图像外部的纹素时会发生什么。下图显示了一些可能的转换：
![](学习一个vulkan-25-纹理映射-图像视图和采样器\3.png)

我们现在将创建一个createTextureSampler函数来设置采样器对象。稍后我们将使用该采样器在着色器中读取纹理中的颜色。
```cpp
void initVulkan() {
    ...
    createTextureImage();
    createTextureImageView();
    createTextureSampler();
    ...
}

...

void createTextureSampler() {

}
```
采样器通过VkSamplerCreateInfo结构进行配置，该结构指定需要应用的所有过滤器和转换。
```cpp
VkSamplerCreateInfo samplerInfo = {};
samplerInfo.sType = VK_STRUCTURE_TYPE_SAMPLER_CREATE_INFO;
samplerInfo.magFilter = VK_FILTER_LINEAR;
samplerInfo.minFilter = VK_FILTER_LINEAR;
```
magFilter和minFilter字段指定如何插入放大或缩小纹素。放大会涉及上述的过采样问题，而缩小则涉及欠采样。选项为VK_FILTER_NEAREST或VK_FILTER_LINEAR，分别对应于上面图片中中显示的模式。
```cpp
samplerInfo.addressModeU = VK_SAMPLER_ADDRESS_MODE_REPEAT;
samplerInfo.addressModeV = VK_SAMPLER_ADDRESS_MODE_REPEAT;
samplerInfo.addressModeW = VK_SAMPLER_ADDRESS_MODE_REPEAT;
```
可以使用addressMode字段为每个轴指定寻址模式。可用值列在下方，其中大部分都在上图中进行了演示。需要注意的是轴向在这里称为UVW而不是XYZ。这是纹理空间坐标系中的规定。
* VK_SAMPLER_ADDRESS_MODE_REPEAT：超出图像尺寸时循环填充。
* VK_SAMPLER_ADDRESS_MODE_MIRRORED_REPEAT：像循环填充类似，但是当超出尺寸时，反转坐标使用镜像来填充。
* VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE：当超过图像尺寸的时候，采用边缘最近的颜色进行填充。
* VK_SAMPLER_ADDRESS_MODE_MIRROR_CLAMP_TO_EDGE：与边缘模式类似，但是使用与最近边缘相反的边缘进行填充。
* VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_BORDER：当采样超过图像的尺寸时，返回一个纯色填充。

我们在这里使用哪种模式并不重要，因为在本教程中我们不会在图像之外进行采样。但是，循环模式可能是最常见的模式，因为它可以用于平铺地板和墙壁等纹理。
```cpp
samplerInfo.anisotropyEnable = VK_TRUE;
samplerInfo.maxAnisotropy = 16;
```
这两个字段指定是否应使用各向异性过滤。除非有性能问题，否则没有理由不使用它。maxAnisotropy字段限制了可用于计算最终颜色的纹素样本量，值越低，性能越好，但效果越差。目前没有图形硬件可以使用超过16个采样器，因为超过16之后的差异可以忽略不计。
<b>samplerInfo.borderColor = VK_BORDER_COLOR_INT_OPAQUE_BLACK;</b>
borderColor字段指定采样范围超过图像时候返回的颜色，与之对应的是边缘寻址模式。可以以float或int格式返回黑色，白色或透明，这里不能指定随便什么颜色。
```cpp
samplerInfo.unnormalizedCoordinates = VK_FALSE;
```
unnormalizedCoordinates字段指定要用于处理图像中纹素的坐标系。如果此字段为VK_TRUE，则可以简单地使用范围是[0,texWidth)和[0,texHeight)的坐标系。如果是VK_FALSE，意味着每个轴向纹素都是[0,1)的范围。真实的应用都是使用归一化的坐标系，因为这样可以在同一坐标系中使用不同分辨率的纹理。
```cpp
samplerInfo.compareEnable = VK_FALSE;
samplerInfo.compareOp = VK_COMPARE_OP_ALWAYS;
```
如果启用了比较功能，则首先将纹素与值进行比较，并将该比较的结果用于过滤操作。这通常用于阴影贴图上的[百分比近似滤波（percentage-closer filtering）](https://developer.nvidia.com/gpugems/GPUGems/gpugems_ch11.html)。我们将在以后的章节中讨论这个问题。
```cpp
samplerInfo.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
samplerInfo.mipLodBias = 0.0f;
samplerInfo.minLod = 0.0f;
samplerInfo.maxLod = 0.0f;
```
所有这些字段应用在mipmapping中，mipmapping也会在未来章节中进行讨论。但基本上它就是另一种可以应用的过滤器。

现在完全定义好了采样器的功能。添加一个类成员来保存采样器对象的句柄，并使用vkCreateSampler创建采样器：
```cpp
VkImageView textureImageView;
VkSampler textureSampler;

...

void createTextureSampler() {
    ...

    if (vkCreateSampler(device, &samplerInfo, nullptr, &textureSampler) != VK_SUCCESS) {
        throw std::runtime_error("failed to create texture sampler!");
    }
}
```
请注意，采样器不会在任何地方引用VkImage。采样器是一个独特的对象，它提供了一个从纹理中提取颜色的接口，它可以应用于您想要的任何图像，无论是1D，2D还是3D。这与许多旧的API不同，后者会将纹理图像和过滤器组合成单个的状态。

在程序的最后且不再访问图像的时候，销毁采样器：
```cpp
void cleanup() {
    cleanupSwapChain();

    vkDestroySampler(device, textureSampler, nullptr);
    vkDestroyImageView(device, textureImageView, nullptr);

    ...
}
```

<b>各向异性设备功能</b>
如果您现在运行程序，您将看到如下的验证层消息：
![](学习一个vulkan-25-纹理映射-图像视图和采样器\4.png)

这是因为各向异性过滤实际上是一个可选的设备功能，我们需要更新createLogicalDevice函数进行请求：
```cpp
VkPhysicalDeviceFeatures deviceFeatures = {};
deviceFeatures.samplerAnisotropy = VK_TRUE;
```
即使现代显卡不太可能不支持它，我们还是应该更新isDeviceSuitable来检查它是否可用：
```cpp
bool isDeviceSuitable(VkPhysicalDevice device) {
    ...

    VkPhysicalDeviceFeatures supportedFeatures;
    vkGetPhysicalDeviceFeatures(device, &supportedFeatures);

    return indices.isComplete() && extensionsSupported && swapChainAdequate && supportedFeatures.samplerAnisotropy;
}
```
vkGetPhysicalDeviceFeatures将VkPhysicalDeviceFeatures结构重新定义，指定了哪些特性被支持而不是通过设置boolean值来请求。

除了强制执行各向异性过滤之外，还可以通过条件设置来不使用它：
```cpp
samplerInfo.anisotropyEnable = VK_FALSE;
samplerInfo.maxAnisotropy = 1;
```
在下一章中，我们将图像和采样器对象公开给着色器，以将纹理绘制到正方形上。











































































































































