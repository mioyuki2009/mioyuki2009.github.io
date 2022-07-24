---
title: 学习一个vulkan(20)-顶点缓冲区-临时缓冲区
date: 2018-11-20 09:05:14
tags: vulkan
categories: computer graphic
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
<b>介绍</b>
我们现在的顶点缓冲区工作正常，但允许我们从CPU访问它的内存类型可能不是显卡本身能够读取的最佳内存类型。最佳内存具有VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT的标志，并且CPU通常无法在专用显卡上访问。在本章中，我们将创建两个顶点缓冲区。一个是临时缓冲区（staging buffer），CPU可以将数据从顶点数组上传到其中，另一个是用于设备本地内存中的最终顶点缓冲区。然后，我们将使用缓冲区复制命令将数据从临时缓冲区移动到实际的顶点缓冲区。

<b>传输队列</b>
缓冲区复制命令需要一个支持传输操作的队列簇，使用VK_QUEUE_TRANSFER_BIT指示。好消息是任何具有VK_QUEUE_GRAPHICS_BIT或VK_QUEUE_COMPUTE_BIT功能的队列簇都已经隐式支持VK_QUEUE_TRANSFER_BIT操作了。在这些情况下，并不需要在queueFlags中明确的列出它。

如果您喜欢挑战，那么仍然可以尝试使用专门用于传输操作的不同队列簇。它将要求您对您的程序进行以下修改：
* 修改QueueFamilyIndi​​ces和findQueueFamilies以显式查找具有VK_QUEUE_TRANSFER标志位位但不是VK_QUEUE_GRAPHICS_BIT的队列簇。
* 修改createLogicalDevice以请求传输队列的句柄
* 为传输队列簇上提交的命令缓冲区创建第二个命令池
* 将资源的sharingMode更改为VK_SHARING_MODE_CONCURRENT并指定图形和传输队列簇
* 任何传输命令，如vkCmdCopyBuffer（我们将在本章中使用）都需要提交到到传输队列而不是图形队列

工作量有一点点，但它可以教你如何在队列簇之间共享资源。

<b>创建抽象缓冲区</b>
因为我们将在本章中创建多个缓冲区，所以将缓冲区创建移动到辅助函数会更好一点。创建一个新的函数createBuffer并将createVertexBuffer中的代码（除外映射相关的）移动到这里。
```cpp
void createBuffer(VkDeviceSize size, VkBufferUsageFlags usage, VkMemoryPropertyFlags properties, VkBuffer& buffer, VkDeviceMemory& bufferMemory) {
    VkBufferCreateInfo bufferInfo = {};
    bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
    bufferInfo.size = size;
    bufferInfo.usage = usage;
    bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateBuffer(device, &bufferInfo, nullptr, &buffer) != VK_SUCCESS) {
        throw std::runtime_error("failed to create buffer!");
    }

    VkMemoryRequirements memRequirements;
    vkGetBufferMemoryRequirements(device, buffer, &memRequirements);

    VkMemoryAllocateInfo allocInfo = {};
    allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
    allocInfo.allocationSize = memRequirements.size;
    allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, properties);

    if (vkAllocateMemory(device, &allocInfo, nullptr, &bufferMemory) != VK_SUCCESS) {
        throw std::runtime_error("failed to allocate buffer memory!");
    }

    vkBindBufferMemory(device, buffer, bufferMemory, 0);
}
```
确保为缓冲区大小，内存属性和用法添加参数，以便我们可以使用此函数创建许多不同类型的缓冲区。最后两个参数是输出变量用来写入句柄。

从createVertexBuffer中删除缓冲区创建和内存分配代码，现在只需调用createBuffer就行了：
```cpp
void createVertexBuffer() {
    VkDeviceSize bufferSize = sizeof(vertices[0]) * vertices.size();
    createBuffer(bufferSize, VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, vertexBuffer, vertexBufferMemory);

    void* data;
    vkMapMemory(device, vertexBufferMemory, 0, bufferSize, 0, &data);
        memcpy(data, vertices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, vertexBufferMemory);
}
```
运行程序以确保顶点缓冲区仍能正常工作。

<b>使用临时缓冲区</b>
我们现在要将createVertexBuffer更改为仅使用主机可见缓冲区作为临时缓冲区，并使用设备本地缓冲区作为实际顶点缓冲区。
```cpp
void createVertexBuffer() {
    VkDeviceSize bufferSize = sizeof(vertices[0]) * vertices.size();

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, bufferSize, 0, &data);
        memcpy(data, vertices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, stagingBufferMemory);

    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, vertexBuffer, vertexBufferMemory);
}
```
我们现在使用带有stagingBufferMemory的新stagingBuffer来映射和复制顶点数据。在本章中，我们将使用两个新的缓冲区使用标志符：
* VK_BUFFER_USAGE_TRANSFER_SRC_BIT：缓冲区可用作存储器传输操作中的源头。
* VK_BUFFER_USAGE_TRANSFER_DST_BIT：缓冲区可用作存储器传输操作中的目标。

vertexBuffer现在是从设备本地的内存类型中分配的，这通常意味着我们无法使用vkMapMemory。但是，我们可以将数据从stagingBuffer复制到vertexBuffer。我们需要指定stagingBuffer的传输源标志位，还要为顶点缓冲区vertexBuffer设置传输目标标志位以及顶点缓冲区使用标志。

我们现在要写一个copyBuffer函数将相关内容从一个缓冲区复制到另一个缓冲区：
```cpp
void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size) {

}
```
内存传输操作也是通过命令缓冲区执行的，就像绘制命令一样。因此我们必须首先分配一个临时命令缓冲区。您可能希望为短期缓冲区来创建单独的命令池，因为该实现可能能够应用内存分配优化。在这种情况下，您应该在命令池生成期间使用VK_COMMAND_POOL_CREATE_TRANSIENT_BIT标志。
```cpp
void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size) {
    VkCommandBufferAllocateInfo allocInfo = {};
    allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
    allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    allocInfo.commandPool = commandPool;
    allocInfo.commandBufferCount = 1;

    VkCommandBuffer commandBuffer;
    vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer);
}
```
并立即开始命令缓冲区的记录：
```cpp
VkCommandBufferBeginInfo beginInfo = {};
beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
beginInfo.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;

vkBeginCommandBuffer(commandBuffer, &beginInfo);
```
这里不需要我们用于绘图命令缓冲区的VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT标志，因为我们只使用命令缓冲区一次并等待从函数返回直到复制操作完成执行。使用VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT来告诉驱动程序我们的意图是一个好习惯。
```cpp
VkBufferCopy copyRegion = {};
copyRegion.srcOffset = 0; // Optional
copyRegion.dstOffset = 0; // Optional
copyRegion.size = size;
vkCmdCopyBuffer(commandBuffer, srcBuffer, dstBuffer, 1, &copyRegion);
```
使用vkCmdCopyBuffer命令来传输缓冲区的内容。它将源缓冲区和目标缓冲区以及要复制的区域数组作为参数。区域在VkBufferCopy结构中定义，由源缓冲区、目标缓冲区的偏移量和大小组成。与vkMapMemory命令不同，此处无法指定VK_WHOLE_SIZE。
```cpp
vkEndCommandBuffer(commandBuffer);
```
此命令缓冲区仅包含复制命令，因此我们可以在此之后立即停止录制。现在执行命令缓冲区以完成传输：
```cpp
VkSubmitInfo submitInfo = {};
submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffer;

vkQueueSubmit(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE);
vkQueueWaitIdle(graphicsQueue);
```
与绘制命令不同的是，这个时候我们不需要等待任何事件，我们只想立即在缓冲区上执行传输命令。还有两种方法可以等待传输命令完成。我们可以使用栅栏fence并使用vkWaitForFences来等待，或者只是使用vkQueueWaitIdle来等待传输队列变为空闲。栅栏允许您同时安排多个传输并等待所有传输完成，而不是一次执行一个，这就给了驱动程序更多的优化空间。
```cpp
vkFreeCommandBuffers(device, commandPool, 1, &commandBuffer);
```
不要忘记清理用于传输操作的命令缓冲区。

我们现在可以从createVertexBuffer函数调用copyBuffer来将顶点数据移动到设备本地缓冲区：
```cpp
createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, vertexBuffer, vertexBufferMemory);

copyBuffer(stagingBuffer, vertexBuffer, bufferSize);
```
将数据从临时缓冲区复制到设备缓冲区后，我们应该清理它：
```cpp
    ...

    copyBuffer(stagingBuffer, vertexBuffer, bufferSize);

    vkDestroyBuffer(device, stagingBuffer, nullptr);
    vkFreeMemory(device, stagingBufferMemory, nullptr);
}
```
运行程序以验证是否再次看到三角形。这种改进目前可能不可见，但其顶点数据现在正从高性能内存加载。当我们要开始渲染更复杂的几何体时，这会变得很重要。

<b>结论</b>
应该注意的是，在实际的应用程序中，您不应该为每个缓冲区都调用vkAllocateMemory。最大并发内存分配数受maxMemoryAllocationCount物理设备限制的限制，即使在像NVIDIA GTX 1080这样的高端硬件上也可能低至4096。为大量对象同时分配内存的正确方法是创建一个自定义分配器，通过使用我们在许多函数中看到的偏移参数，在多个不同对象之间拆分成单个进行分配。

您可以自己实现这样的分配器，也可以使用GPUOpen计划提供的[VulkanMemoryAllocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator)库。但是，对于本教程，可以为每个资源使用单独的分配，因为我们现在不会接近任何的限制。

















































































































































































