---
title: 学习一个vulkan(16)-画个三角形(绘制)-渲染和显示
date: 2018-11-15 14:42:59
tags: vulkan
categories: computer graphic
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
<b>创建</b>
在这个章节里，我们会将所有的东西聚合到一起。现在编写drawFrame函数，该函数在主循环中调用用于将三角形显示到屏幕上。创建函数并在mainLoop里调用它：
```cpp
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
        drawFrame();
    }
}

...

void drawFrame() {

}
```

<b>同步</b>
drawFrame函数将执行以下操作：
* 从交换链中获取图像
* 在帧缓冲区中将该图像作为附件执行命令缓冲区中的命令
* 将图像返回到交换链以进行演示

这些事件中的每一个都需要使用单个函数调用来启动，但它们是异步执行的。函数的调用将会在操作实际完成之前就返回了，并且执行顺序也是未定义的。这是不幸的，因为我们希望每个操作都取决于前一个操作。

有两种同步交换链消息的方法：栅栏和信号量。它们都是可用于协调操作的对象，具有一个操作信号和另一个操作等待栅栏或信号量从无信号状态变为信号状态，如果莫某个操作没有信号，则会一直等待着栅栏或者信号量来令他转换到有信号的状态。

两种方式的区别在于可以使用vkWaitForFences之类的函数从程序中访问栅栏的状态，但是信号量却不行。栅栏主要用于将应用程序本身与渲染操作同步，而信号量用于同步命令队列内或跨命令队列的操作。我们想要同步绘制命令和显示队列操作，这使得使用信号量比较合适。

<b>信号量</b>
我们需要一个信号量来表示已经获取了图像并准备好进行渲染，另一个信号量表示渲染已经完成并且可以进行显示。创建两个类成员来存储这些信号量对象：
```cpp
VkSemaphore imageAvailableSemaphore;
VkSemaphore renderFinishedSemaphore;
```
要创建信号量，还需要为添加最后的一个create函数：createSemaphores：
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
    createSemaphores();
}

...

void createSemaphores() {

}
```
创建信号量需要填充VkSemaphoreCreateInfo结构，但是在当前版本的API中，除了sType之外，实际上没有任何必需的字段：
```cpp
void createSemaphores() {
    VkSemaphoreCreateInfo semaphoreInfo = {};
    semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;
}
```
Vulkan API的未来版本或有什么扩展可能会为flags和pNext参数添加一些功能，就像它的其他结构一样。创建信号量vkCreateSemaphor和其他对象的创建模式差不多：
```cpp
if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &imageAvailableSemaphore) != VK_SUCCESS ||
    vkCreateSemaphore(device, &semaphoreInfo, nullptr, &renderFinishedSemaphore) != VK_SUCCESS) {

    throw std::runtime_error("failed to create semaphores!");
}
```
当程序结束时，所有命令都已完成且不再需要同步时，应清除信号量：
```cpp
void cleanup() {
    vkDestroySemaphore(device, renderFinishedSemaphore, nullptr);
    vkDestroySemaphore(device, imageAvailableSemaphore, nullptr);
```

<b>从交换链中获取图像</b>
如前所述，我们需要在drawFrame函数中做的第一件事就是从交换链中获取一个图像。回想一下，交换链是一个扩展功能，因此我们必须使用具有vk*KHR命名的函数：
```cpp
void drawFrame() {
    uint32_t imageIndex;
    vkAcquireNextImageKHR(device, swapChain, std::numeric_limits<uint64_t>::max(), imageAvailableSemaphore, VK_NULL_HANDLE, &imageIndex);
}
```
vkAcquireNextImageKHR的前两个参数是我们希望从中获取图像的逻辑设备和交换链。第三个参数指定处理图像可用的时间限制（以纳秒为单位），这里使用64位无符号整数的最大值可禁用时间的限制。

接下来的两个参数指定当显示引擎处理完图像后要发出信号的同步对象。这就是开始绘制的时候了，可以指定使用信号量，栅栏或都使用，在此处我们使用imageAvailableSemaphore。

最后一个参数指定一个变量，用于输出已变为可用的交换链图像的索引。索引引用了swapChainImages数组中的VkImage图像，我们将使用该索引来选择正确的命令缓冲区。

<b>提交命令缓冲区</b>
队列提交和同步是通过配置VkSubmitInfo结构中的参数完成的：
```cpp
VkSubmitInfo submitInfo = {};
submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

VkSemaphore waitSemaphores[] = {imageAvailableSemaphore};
VkPipelineStageFlags waitStages[] = {VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT};
submitInfo.waitSemaphoreCount = 1;
submitInfo.pWaitSemaphores = waitSemaphores;
submitInfo.pWaitDstStageMask = waitStages;
```
前三个参数指定在执行开始之前等待的信号量以及要等待的管道位于的阶段。我们希望一直等待为图像写入颜色直到它可用，所以我们指定了写入颜色附件的图形管道的阶段。这意味着从理论上讲，现在的实现已经可以开始执行我们的顶点着色器了，而图像还不可用。waitStages数组对应的pWaitSemaphores中会具有相同索引的信号量。
```cpp
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffers[imageIndex];
```
接下来的两个参数指定实际提交来执行的命令缓冲区。和之前说的一样，我们提交到的命令缓冲区会将我们刚刚获取的交换链图像绑定为颜色附件。
```cpp
VkSemaphore signalSemaphores[] = {renderFinishedSemaphore};
submitInfo.signalSemaphoreCount = 1;
submitInfo.pSignalSemaphores = signalSemaphores;
```
signalSemaphoreCount和pSignalSemaphores参数指定在命令缓冲区完成执行后向哪些信号量发出信号。在我们的例子中，根据需要我们使用了renderFinishedSemaphore。
```cpp
if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit draw command buffer!");
}
```
我们现在可以使用vkQueueSubmit将命令缓冲区提交到图形队列。当工作负载大得多时，该函数将VkSubmitInfo结构数组作为参数来改善效率。最后一个参数是一个可选的fence，当命令缓冲区完成执行时，它将发出信号。我们只使用信号量进行同步，因此这里只传递一个VK_NULL_HANDLE。

<b>子通道依赖项</b>
请记住，渲染过程中的子通道会自动处理图像布局的变化。这些转换由子通道依赖项（subpass dependencies）控制，子通道依赖项指定了子通道之间的内存和执行依赖关系。我们现在只有一个子通道，但是在此子通道之前和之后的操作也算作隐式的“子通道”。

有两个内置依赖项，用于处理渲染过程开始时和渲染过程结束时的转换，但前者发生的时间并不是刚好正确的。它假设转换发生是在管道的开始，但是那时我们还没有获得图像。有两种方法可以解决这个问题。可以将imageAvailableSemaphore的waitStages更改为VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT，确保在图像有效之前渲染过程不会开始，或者我们可以使渲染过程等待VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT阶段。我决定在这里使用第二个选项，因为它也是查看子通道的依赖关系以及它们如何工作的一个很好的方式。

子通道依赖项在VkSubpassDependency结构中指​​定。转到createRenderPass函数并添加如下结构：
```cpp
VkSubpassDependency dependency = {};
dependency.srcSubpass = VK_SUBPASS_EXTERNAL;
dependency.dstSubpass = 0;
```
前两个字段指定了依赖关系和从属子通道的索引。VK_SUBPASS_EXTERNAL指的是渲染通道之前或之后的隐式子通道，具体取决于它是在srcSubpass还是dstSubpass中指定的。索引0指的是我们的子通道，它是第一个也是唯一一个。dstSubpass必须始终高于srcSubpass以防止依赖关系图中产生循环。
```cpp
dependency.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.srcAccessMask = 0;
```
接下来的两个字段指定要等待的操作以及这些操作发生的阶段。在我们访问之前，我们需要等待交换链完成从图像中进行读取，这可以通过等待颜色附件输出的阶段来实现。
```cpp
dependency.dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_READ_BIT | VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
```
应该等待的操作是在颜色附加阶段，这里涉及到了读取和写入颜色附件。这些设置将阻止转换的发生，直到我们实际需要（并允许），也就是我们想要开始为其写入颜色的时候。
```cpp
renderPassInfo.dependencyCount = 1;
renderPassInfo.pDependencies = &dependency;
```
VkRenderPassCreateInfo结构有两个字段来指定依赖项数组。

<b>显示</b>
绘制的最后一步是将结果提交回交换链，以使其最终显示在屏幕上。显示可以通过drawFrame函数末尾的VkPresentInfoKHR结构进行配置。
```cpp
VkPresentInfoKHR presentInfo = {};
presentInfo.sType = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR;

presentInfo.waitSemaphoreCount = 1;
presentInfo.pWaitSemaphores = signalSemaphores;
```
前两个参数指定在显示发生之前要等待的信号量，和VkSubmitInfo一样。
```cpp
VkSwapchainKHR swapChains[] = {swapChain};
presentInfo.swapchainCount = 1;
presentInfo.pSwapchains = swapChains;
presentInfo.pImageIndices = &imageIndex;
```
接下来的两个参数指定用于呈现图像的交换链以及每个交换链的图像索引，大多数情况下只有一个。
```cpp
presentInfo.pResults = nullptr; // Optional
```
最后参数是可选的，叫做pResults。它允许您指定一个VkResult数组，以便在显示成功时检查每个单独的交换链。如果您只使用单个交换链，则没有必要设置，因为可以简单地使用当前函数的返回值就能知道了。
```cpp
vkQueuePresentKHR(presentQueue, &presentInfo);
```
vkQueuePresentKHR函数提交将图像呈现给交换链的请求。我们将在下一章中为vkAcquireNextImageKHR和vkQueuePresentKHR添加错误处理，因为他们的失败也并不一定意味着程序应该终止，这与我们迄今为止看到的功能不同。

如果到目前为止你已经正确地完成了所有操作，那么在运行程序时，您现在应该看到类似于以下的内容：
![](学习一个vulkan-16-画个三角形-绘制-渲染和演示/1.png)

好极了！
不幸的是，在启用验证图层后，可以看到程序会在关闭它时立即崩溃。从debugCallback打印到终端的消息可以看到原因：
![](学习一个vulkan-16-画个三角形-绘制-渲染和演示/2.png)

请记住，drawFrame中的所有操作都是异步的。这意味着当我们退出mainLoop中的循环时，绘图和演示操作可能仍在继续，
在这时候清理资源是并不是一个好的想法。

要解决这个问题，我们应该在退出mainLoop并销毁窗口之前等待逻辑设备完成操作：
```cpp
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
        drawFrame();
    }

    vkDeviceWaitIdle(device);
}
```
您还可以等待特定命令队列中的操作以vkQueueWaitIdle结束。这些功能可以用作执行同步的非常基本的方法，现在关闭窗口时程序可以毫无问题地退出。

<b>飞行帧（Frames in flight）</b>
如果运行启用了验证层的应用程序，则会监视应用程序的内存使用情况，您可能会注意到它正在缓慢增长。这样做的原因是应用程序正在drawFrame函数中快速提交工作，但却并没有检查它是否完成。如果CPU提交的工作速度超过了GPU可以跟上的速度，那么队列将慢慢被工作填满。更糟糕的是，我们还正在重复使用imageAvailableSemaphore和renderFinishedSemaphore同时进行多个帧的操作。

解决此问题的简单方法是在提交后等待工作完成，例如使用vkQueueWaitIdle：
```cpp
void drawFrame() {
    ...

    vkQueuePresentKHR(presentQueue, &presentInfo);

    vkQueueWaitIdle(presentQueue);
}
```
但是，这种方式并不能很好的使用GPU，这样做的话整个图形管道现在一次仅用于一帧。只有当前帧已经处理完毕而且是空闲的，才会用于下一帧。我们现在将扩展我们的应用程序，以允许多个帧在飞行同时仍然对堆积的工作量做出限制。

首先在程序顶部添加一个常量，该常量定义应同时处理多少帧：
```cpp
const int MAX_FRAMES_IN_FLIGHT = 2;
```
每个框架都应该有一组自己的信号量：
```cpp
std::vector<VkSemaphore> imageAvailableSemaphores;
std::vector<VkSemaphore> renderFinishedSemaphores;
```
需要更改createSemaphores来创建以下的东西：
```cpp
void createSemaphores() {
    imageAvailableSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
    renderFinishedSemaphores.resize(MAX_FRAMES_IN_FLIGHT);

    VkSemaphoreCreateInfo semaphoreInfo = {};
    semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &imageAvailableSemaphores[i]) != VK_SUCCESS ||
            vkCreateSemaphore(device, &semaphoreInfo, nullptr, &renderFinishedSemaphores[i]) != VK_SUCCESS) {

            throw std::runtime_error("failed to create semaphores for a frame!");
        }
}
```
同样，它们也应该全部清理掉：
```cpp
void cleanup() {
    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
        vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
    }

    ...
}
```
要每次使用正确的信号量，我们需要跟踪当前帧。因此需要使用帧索引：
```cpp
size_t currentFrame = 0;
```
现在可以修改drawFrame函数来使用正确的对象：
```cpp
void drawFrame() {
    vkAcquireNextImageKHR(device, swapChain, std::numeric_limits<uint64_t>::max(), imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

    ...

    VkSemaphore waitSemaphores[] = {imageAvailableSemaphores[currentFrame]};

    ...

    VkSemaphore signalSemaphores[] = {renderFinishedSemaphores[currentFrame]};

    ...
}
```
当然，也不要忘记每次都要进入下一帧：
```cpp
void drawFrame() {
    ...

    currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHT;
}
```
通过使用modulo（％）运算符，我们可以确保帧索引每隔MAX_FRAMES_IN_FLIGHT就会循环（和循环队列差不多）。

虽然我们没有设置所需的对象来同时处理多个帧，我们实际上还并没有能够阻止提交的数目超过MAX_FRAMES_IN_FLIGHT。目前只有GPU-GPU同步，还没有CPU-GPU同步来继续跟踪工作的进展情况，也就是说在帧＃0仍然在飞行状态时，我们也可能在使用他。

为了执行CPU-GPU同步，Vulkan提供了第二种类型的同步原语，称为栅栏（fences）。栅栏类似于信号量，因为它们可以发出信号并等待，但这次我们在我们自己的代码中来等待它们。首先为每个帧创建一个栅栏：
```cpp
std::vector<VkSemaphore> imageAvailableSemaphores;
std::vector<VkSemaphore> renderFinishedSemaphores;
std::vector<VkFence> inFlightFences;
size_t currentFrame = 0;
```
我们决定将栅栏和信号量在一起创建，将createSemaphores重命名为createSyncObjects：
```cpp
void createSyncObjects() {
    imageAvailableSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
    renderFinishedSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
    inFlightFences.resize(MAX_FRAMES_IN_FLIGHT);

    VkSemaphoreCreateInfo semaphoreInfo = {};
    semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;

    VkFenceCreateInfo fenceInfo = {};
    fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &imageAvailableSemaphores[i]) != VK_SUCCESS ||
            vkCreateSemaphore(device, &semaphoreInfo, nullptr, &renderFinishedSemaphores[i]) != VK_SUCCESS ||
            vkCreateFence(device, &fenceInfo, nullptr, &inFlightFences[i]) != VK_SUCCESS) {

            throw std::runtime_error("failed to create synchronization objects for a frame!");
        }
    }
}
```
栅栏的创建（VkFence）与信号量的创建非常相似，当然最也还要确保清理干净：
```cpp
void cleanup() {
    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
        vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
        vkDestroyFence(device, inFlightFences[i], nullptr);
    }

    ...
}
```
现在来修改drawFrame以使用fences进行同步。调用vkQueueSubmit包含了一个可选参数，用于传递在命令缓冲区完成执行时应发出信号的栅栏。我们可以使用它来表示帧已经完成：
```cpp
void drawFrame() {
    ...

    if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
        throw std::runtime_error("failed to submit draw command buffer!");
    }
    ...
}
```
现在唯一剩下的就是修改drawFrame的开头用来等待帧完成：
```cpp
void drawFrame() {
    vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, std::numeric_limits<uint64_t>::max());
    vkResetFences(device, 1, &inFlightFences[currentFrame]);

    ...
}
```
vkWaitForFences函数用一系列栅栏作为参数，并在返回之前等待其中的任何一个或全部发出信号。VK_TRUE表示我们要等待所有栅栏，但是在我们就只有一个，所以在我们的例子中它显然无关紧要。就像vkAcquireNextImageKHR一样，这个函数也有一个时间限制参数。与信号量不同的是，我们需要手动调用vkResetFences来重置栅栏使他恢复到没有信号的状态。

如果你现在运行程序，你会发现一些奇怪的东西。应用程序似乎不再渲染任何东西。启用验证层后，您将看到以下错误信息：
![](学习一个vulkan-16-画个三角形-绘制-渲染和演示/3.png)

这意味着我们正在等待尚未提交的栅栏。这里的问题是，默认情况下，栅栏是在没有信号状态下创建的，所以由于我们之前没有使用过栅栏，vkWaitForFences会永远等待。为了解决这个问题，我们可以更改栅栏的创建以在信号状态下初始化它，就好像我们已经渲染了一个完成的初始帧一样：
```cpp
void createSyncObjects() {
    ...

    VkFenceCreateInfo fenceInfo = {};
    fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
    fenceInfo.flags = VK_FENCE_CREATE_SIGNALED_BIT;

    ...
}
```
现在程序可以正常工作没有内存泄漏了。我们现在已经实现了所有需要的同步，以确保排队的工作帧不超过两帧。请注意，对于代码的其他部分（例如最终的清理）来说，如果能够依赖于更粗略的同步（如vkDeviceWaitIdle）会更好。您可以根据性能的要求决定使用哪种方法。

要通过示例了解有关同步的更多信息，请查看Khronos的[相关概述](https://github.com/KhronosGroup/Vulkan-Docs/wiki/Synchronization-Examples#swapchain-image-acquire-and-present)。

<b>总结</b>
写了差不多900行代码后，终于进入了在屏幕上弹出一些东西的阶段。引导Vulkan程序肯定是需要做很多工作的，但好处是通过这些明细的操作步骤，Vulkan为您提供了巨大的控制权。我建议你现在花点时间重新阅读代码并在心中构建模型，了解程序中所有Vulkan对象的用途以及它们之间的关系。我们将在这些知识的基础上进一步扩展该程序的功能。

在下一章中，我们将讨论一个好的Vulkan程序所需的一个小东西。