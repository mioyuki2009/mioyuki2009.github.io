---
title: 学习一个vulkan(12)-画个三角形(图形管道)-渲染过程
date: 2018-11-10 09:30:19
tags: vulkan
categories: learn
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
<b>构建</b>
在我们完成管道创建之前，我们需要告诉Vulkan渲染时将使用的帧缓冲附件。我们需要指定将有多少颜色和深度缓冲区，每个缓冲区要使用多少个样本以及在整个渲染操作中如何处理它们的内容。所有的这些信息都包含在一个渲染过程（render pass）对象中，我们将为此创建一个createRenderPass函数。在initVulkan中的createGraphicsPipeline的前面调用：
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
}

...

void createRenderPass() {

}
```

<b>附件描述</b>
在我们的例子中只有一个颜色缓冲附件，由交换链中的一个图像表示：
```cpp
void createRenderPass() {
    VkAttachmentDescription colorAttachment = {};
    colorAttachment.format = swapChainImageFormat;
    colorAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
}
```
颜色附件的format参数应该与交换链图像的格式相匹配，而且我们还没有做任何多重采样方面的事情，所以我们这里使用1个样本。
```cpp
colorAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
colorAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
```
loadOp和storeOp设置在渲染之前和渲染之后如何处理附件中的数据。
对loadOp有如下几个选择：
* VK_ATTACHMENT_LOAD_OP_LOAD：保留附件的现有内容
* VK_ATTACHMENT_LOAD_OP_CLEAR：在开始时以一个常量清理附件内容
* VK_ATTACHMENT_LOAD_OP_DONT_CARE：现有内容未定义，忽略它们

在我们的例子中，我们将使用clear操作在绘制新帧之前将帧缓冲区清除为黑色。
storeOp只有两种选择：
* VK_ATTACHMENT_STORE_OP_STORE：渲染内容将存储在内存中，方便之后读取
* VK_ATTACHMENT_STORE_OP_DONT_CARE：渲染操作后，帧缓冲区的内容变为undefined

我们希望在屏幕上看到渲染后的三角形，所以我们在这里进行store操作。
```cpp
colorAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
colorAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
```
loadOp和storeOp适用于颜色和深度数据，stencilLoadOp/stencilStoreOp适用于模板数据。我们的应用程序不会对模板缓冲区执行任何操作，因此相应的加载和存储结果都无关紧要。
```cpp
colorAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
colorAttachment.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
```
Vulkan中的纹理和帧缓冲区由具有特定像素格式的VkImage对象表示，但是内存中像素的布局会根据您对图像执行的操作而改变。

一些常用的布局:
* VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL：图像用作颜色附件
* VK_IMAGE_LAYOUT_PRESENT_SRC_KHR：图像在交换链中呈现
* VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL：图像作为目标，用于内存复制操作

我们将在纹理章节中更深入地讨论这个主题，现在最重要的是为需要转变的图像指定合适的布局进行操作。

initialLayout指定了渲染通道开始之前图像将具有的布局，finalLayout指定在渲染过程完成时自动转换到的布局。对initialLayout使用VK_IMAGE_LAYOUT_UNDEFINED意味着我们不关心图像之前的布局。这个特殊的值是说图像的内容不能保证会被保留，但这并不重要，因为我们都要清除它的。我们希望在图像渲染完毕后使用交换链呈现，所以finalLayout设置为VK_IMAGE_LAYOUT_PRESENT_SRC_KHR。

<b>子通道和附件参考</b>
单个渲染过程可以包含多个子过程。子通道是一系列的渲染操作，其取决于先前传递中的帧缓冲器的内容，比如说后处理效果的序列通常每一步都依赖之前的操作。如果将这些渲染操作分组到一个渲染通道中，那么Vulkan能够重新排序操作并节省内存带宽，从而获得更好的性能。然而，对于我们的三角形，我们将使用单个子通道就够了。

每个子通道都引用了我们前面的结构中描述的一个或多个附件。这些引用本身就是VkAttachmentReference结构，如下所示：
```cpp
VkAttachmentReference colorAttachmentRef = {};
colorAttachmentRef.attachment = 0;
colorAttachmentRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
```
attachment参数通过附件描述数组中的索引来指定引用哪个附件。我们的数组由单个的VkAttachmentDescription组成，因此其索引为0。layout参数指定在使用此引用的子通道期间我们希望附件具有哪种布局，在启动子通道时，Vulkan会自动将附件转换为此布局。我们打算将附件用作颜色缓冲区，这里VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL布局将为我们提供最好的性能。

子通道需要使用VkSubpassDescription结构来描述：
```cpp
VkSubpassDescription subpass = {};
subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
```
Vulkan也可能在未来支持计算子通道，因此我们必须明确将其指定为图形子通道。接下来，我们指定对颜色附件的引用：
```cpp
subpass.colorAttachmentCount = 1;
subpass.pColorAttachments = &colorAttachmentRef;
```
附件在数组中的索引直接从片段着色器引用，也就是layout(location = 0) out vec4 outColor这个指令了。

子通道还可以引用以下其他类型的附件：
* pInputAttachments：从着色器读取附件
* pResolveAttachments：用于多重采样颜色附件的附件
* pDepthStencilAttachment：用于深度和模板数据的附件
* pPreserveAttachments：此子通道未使用但必须保留数据的附件

<b>渲染过程</b>
现在已经描述了附件和引用它的基本子通道，我们可以创建渲染通道了。创建一个新的类成员变量，将VkRenderPass对象保存在pipelineLayout变量的正上方：
```cpp
VkRenderPass renderPass;
VkPipelineLayout pipelineLayout;
```
然后可以通过使用附件和子通道数组填充VkRenderPassCreateInfo结构来创建渲染过程对象，VkAttachmentReference对象也是通过使用这个数组的索引来引用附件的。
```cpp
VkRenderPassCreateInfo renderPassInfo = {};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
renderPassInfo.attachmentCount = 1;
renderPassInfo.pAttachments = &colorAttachment;
renderPassInfo.subpassCount = 1;
renderPassInfo.pSubpasses = &subpass;

if (vkCreateRenderPass(device, &renderPassInfo, nullptr, &renderPass) != VK_SUCCESS) {
    throw std::runtime_error("failed to create render pass!");
}
```
就像管道布局一样，渲染过程将在整个程序中被引用，因此它也在最后清理：
```cpp
void cleanup() {
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    vkDestroyRenderPass(device, renderPass, nullptr);
    ...
}
```
现在已经做了很多工作了，在下一章里，他们将会一起构成最终的图形管道对象。



















































