---
title: 学习一个vulkan(14)-画个三角形(绘制)-帧缓冲区
date: 2018-11-13 09:14:50
tags: vulkan
categories: computer graphic
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
<b>帧缓冲区</b>
在过去的几章中我们已经讨论了很多关于帧缓冲的内容，并且我们已经设置了渲染过程并希望输出一个与交换链图像格式一致的帧缓冲区，但是我们还没有实际创建任何帧缓冲器。

在创建渲染过程的期间指定的附件可以通过将它们包装到VkFramebuffer对象中进行绑定。帧缓冲区（framebuffer）对象引用表示为附件的所有的VkImageView对象。在我们的例子中，只有一个颜色附件。但是，我们用于附件的图像取决于交换链在我们检索用于显示时返回的图像。这意味着我们必须为交换链中的所有图像创建一个帧缓冲区，并在绘制时使用与检索到的图像对应的图像缓冲区。

为此，创建另一个std :: vector类成员来保存帧缓冲区：
```cpp
std::vector<VkFramebuffer> swapChainFramebuffers;
```
我们将在initVulkan里创建图形管道之后调用新函数createFramebuffers来为此数组创建对象：
```cpp
void initVulkan() {
    createInstance();
    setupDebugCallback();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
}

...

void createFramebuffers() {

}
```
首先调整容器大小来容纳所有帧缓冲区：
```cpp
void createFramebuffers() {
    swapChainFramebuffers.resize(swapChainImageViews.size());
}
```
然后我们将遍历图像视图并从中创建帧缓冲区：
```cpp
for (size_t i = 0; i < swapChainImageViews.size(); i++) {
    VkImageView attachments[] = {
        swapChainImageViews[i]
    };

    VkFramebufferCreateInfo framebufferInfo = {};
    framebufferInfo.sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO;
    framebufferInfo.renderPass = renderPass;
    framebufferInfo.attachmentCount = 1;
    framebufferInfo.pAttachments = attachments;
    framebufferInfo.width = swapChainExtent.width;
    framebufferInfo.height = swapChainExtent.height;
    framebufferInfo.layers = 1;

    if (vkCreateFramebuffer(device, &framebufferInfo, nullptr, &swapChainFramebuffers[i]) != VK_SUCCESS) {
        throw std::runtime_error("failed to create framebuffer!");
    }
}
```
可以看到，帧缓冲区的创建非常简单。我们首先需要指定帧缓冲区需要与哪个renderPass兼容。您只能将帧缓冲与其兼容的渲染过程一起使用，这也大致意味着它们会使用相同数量和类型的附件。

attachmentCount和pAttachments参数指定了VkImageView对象绑定到的渲染过程中pAttachment数组的相关说明。

width和height参数显而易见，layers指的是图像数组中的图层数，我们的交换链图像是单个图像，因此层数为1。

我们在图像视图和渲染通道渲染完毕之后，删除对应的帧缓冲区:
```cpp
void cleanup() {
    for (auto framebuffer : swapChainFramebuffers) {
        vkDestroyFramebuffer(device, framebuffer, nullptr);
    }

    ...
}
```
我们现在已经达到了里程碑，我们现在有了拥有渲染所需的所有对象。在下一章中，我们将编写第一个实际的绘图命令。









































































