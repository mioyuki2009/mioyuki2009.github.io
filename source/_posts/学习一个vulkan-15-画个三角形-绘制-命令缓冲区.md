---
title: 学习一个vulkan(15)-画个三角形(绘制)-命令缓冲区
date: 2018-11-14 09:26:01
tags: vulkan
categories: computer graphic
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
Vulkan中的命令（如绘图操作和内存传输）并不会直接调用函数来执行，您必须在命令缓冲区对象中记录下要执行的所有操作。这样做的好处是，所有设置绘图命令的工作都可以提前并在多个线程中完成，之后，只需要告诉Vulkan在主循环中执行命令就行了。

<b>命令池</b>
在创建命令缓冲区之前，我们必须创建一个命令池。命令池管理用于存储缓冲区的内存，并从中分配命令缓冲区，添加一个新的类成员来存储VkCommandPool对象：
```cpp
VkCommandPool commandPool;
```
然后创建一个新函数createCommandPool，并在initVulkan中创建framebuffers的函数的后面进行调用：
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
    createCommandPool();
}

...

void createCommandPool() {

}
```
创建命令池只需要两个参数：
```cpp
QueueFamilyIndices queueFamilyIndices = findQueueFamilies(physicalDevice);

VkCommandPoolCreateInfo poolInfo = {};
poolInfo.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
poolInfo.queueFamilyIndex = queueFamilyIndices.graphicsFamily.value();
poolInfo.flags = 0; // Optional
```
命令缓冲区需要通过在其中一个设备队列上提交他们来执行，就像我们之前检索的图形和表示队列一样。每个命令池只能分配给提交到同一类型队列的命令缓冲区。我们将记录绘图命令，所以这里需要选择图形队列簇。

命令池有两种标志：
* VK_COMMAND_POOL_CREATE_TRANSIENT_BIT：提示命令缓冲区非常频繁的重新记录新命令(可能会改变内存分配行为)
* VK_COMMAND_POOL_CREATE_TRANSIENT_BIT：允许命令缓冲区单独重新记录，没有这个标志的话，所有的命令缓冲区都必须一起重置

我们只会在程序的开头记录命令缓冲区，然后在主循环中执行多次，所以这两个标志我们都不会使用。
```cpp
if (vkCreateCommandPool(device, &poolInfo, nullptr, &commandPool) != VK_SUCCESS) {
    throw std::runtime_error("failed to create command pool!");
}
```
使用vkCreateCommandPool函数完成命令池的创建，它没有任何特殊参数。
使用命令在屏幕上绘制内容将会在整个程序运行期都存在，所有需要在最后进行销毁：
```cpp
void cleanup() {
    vkDestroyCommandPool(device, commandPool, nullptr);

    ...
}
```

<b>命令缓冲区分配</b>
我们现在可以开始分配命令缓冲区并在其中记录绘图命令了。因为有一个绘图命令涉及到需要绑定正确的VkFramebuffer，所以现在必须再次为交换链中的每个图像记录一个命令缓冲区。因此这里创建一个VkCommandBuffer对象列表作为类成员。命令缓冲区将在其命令池被销毁时自动释放，因此不需要显式清理。
```cpp
std::vector<VkCommandBuffer> commandBuffers;
```
添加一个createCommandBuffers函数，该函数为每个交换链图像分配和记录命令。
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
    createCommandPool();
    createCommandBuffers();
}

...

void createCommandBuffers() {
    commandBuffers.resize(swapChainFramebuffers.size());
}
```
使用vkAllocateCommandBuffers函数来分配命令缓冲区，它将VkCommandBufferAllocateInfo结构作为参数来指定命令池和要分配的缓冲区数目：
```cpp
VkCommandBufferAllocateInfo allocInfo = {};
allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
allocInfo.commandPool = commandPool;
allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
allocInfo.commandBufferCount = (uint32_t) commandBuffers.size();

if (vkAllocateCommandBuffers(device, &allocInfo, commandBuffers.data()) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate command buffers!");
}
```
level参数指定分配的命令缓冲区是主命令缓冲区还是辅助命令缓冲区。
* VK_COMMAND_BUFFER_LEVEL_PRIMARY：可以提交到队列执行，但不能从其他命令缓冲区调用。
* VK_COMMAND_BUFFER_LEVEL_SECONDARY：无法直接提交，但可以从主命令缓冲区调用。

我们不会在这里使用辅助命令缓冲区功能，但可以预见到从主命令缓冲区重用一些常见操作是很有帮助的。

<b>开始命令缓冲区的记录</b>
我们通过调用vkBeginCommandBuffer并使用VkCommandBufferBeginInfo结构作为参数来开始记录命令缓冲区，该参数指定了有关此特定命令缓冲区用法的一些详细信息。
```cpp
for (size_t i = 0; i < commandBuffers.size(); i++) {
    VkCommandBufferBeginInfo beginInfo = {};
    beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
    beginInfo.flags = VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT;
    beginInfo.pInheritanceInfo = nullptr; // Optional

    if (vkBeginCommandBuffer(commandBuffers[i], &beginInfo) != VK_SUCCESS) {
        throw std::runtime_error("failed to begin recording command buffer!");
    }
}
```
flags参数指定我们将如何使用命令缓冲区。可以使用以下值：
* VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT：执行一次后，将立即重新记录命令缓冲区。
* VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT：这是一个辅助命令缓冲区，仅仅用于单个渲染过程中。
* VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT：在命令缓冲区已挂起的时候可以重新提交。

我们使用了最后一个标志，因为我们可能已经为下一帧安排了绘图命令，而上一帧还没有完成。pInheritanceInfo参数仅与辅助命令缓冲区相关，
它指定从调用主命令缓冲区继承的状态。

如果命令缓冲区已经记录一次，那么对vkBeginCommandBuffer的调用将隐式重置它，直接将命令附加到缓冲区中是不可能的。

<b>开始渲染过程</b>
绘制开始于调用vkCmdBeginRenderPass开启渲染通道，使用VkRenderPassBeginInfo结构中的一些参数配置渲染过程：
```cpp
VkRenderPassBeginInfo renderPassInfo = {};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
renderPassInfo.renderPass = renderPass;
renderPassInfo.framebuffer = swapChainFramebuffers[i];
```
第一个参数是渲染过程本身和要绑定的附件。我们为每个交换链图像创建了一个帧缓冲区，将其指定为颜色附件。
```cpp
renderPassInfo.renderArea.offset = {0, 0};
renderPassInfo.renderArea.extent = swapChainExtent;
```
接下来的两个参数定义渲染区域的大小，渲染区域定义了着色器加载和存储的位置。
```cpp
VkClearValue clearColor = {0.0f, 0.0f, 0.0f, 1.0f};
renderPassInfo.clearValueCount = 1;
renderPassInfo.pClearValues = &clearColor;
```
最后两个参数定义了用于VK_ATTACHMENT_LOAD_OP_CLEAR的清除值，我们将其用作颜色附件的加载操作。这里将清除颜色定义为仅具有100％不透明度的黑色。
```cpp
vkCmdBeginRenderPass(commandBuffers[i], &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE);
```
渲染过程终于可以开始了。记录命令的所有相关功能都可以通过其特有的vkCmd前缀来识别。它们都返回void，因此在我们完成记录之前不会有任何错误处理。

每个命令的第一个参数始终是用于记录命令的命令缓冲区，第二个参数指定我们刚刚提供的渲染过程的详细信息，最后一个参数控制如何提供渲染过程中的绘图命令。它可以是以下两个值其中之一：
* VK_SUBPASS_CONTENTS_INLINE：渲染过程命令将会嵌入在主命令​​缓冲区本身中，并且不会执行辅助命令缓冲区。
* VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS：渲染过程命令将从辅助命令缓冲区执行。

我们没有使用辅助命令缓冲区，因此我们将使用第一个选项。

<b>基本绘图命令</b>
我们现在可以绑定图形管道了：
```cpp
vkCmdBindPipeline(commandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline);
```
第二个参数指定管道对象是图形管道还是计算管道。我们现在告诉Vulkan哪些操作要在图形管道中执行，哪些附件要在片段着色器中使用，所以剩下的就是告诉它我们要绘制三角形了：
```cpp
vkCmdDraw(commandBuffers[i], 3, 1, 0, 0);
```
由于之前我们已经详细的指定了相关信息，现在vkCmdDraw的调用就显得很简单了。除命令缓冲区外，它还具有以下参数：
* vertexCount：即使我们没有顶点缓冲区，这里仍然有3个顶点需要绘制。
* instanceCount：用于实例化渲染，如果不需要的话使用1。
* firstVertex：用作顶点缓冲区的偏移量，gl_VertexIndex定义为其最低值。
* firstInstance：用作实例渲染的偏移量，gl_InstanceIndex定义为其最低值。

<b>结束</b>
渲染过程现在可以结束了：
```cpp
vkCmdEndRenderPass(commandBuffers[i]);
```
现在我们已经完成了命令缓冲区的录制：
```cpp
if (vkEndCommandBuffer(commandBuffers[i]) != VK_SUCCESS) {
    throw std::runtime_error("failed to record command buffer!");
}
```
在下一章中，我们将编写主循环的代码，它将从交换链中获取图像，执行正确的命令缓冲区并将完成的图像返回到交换链。


























