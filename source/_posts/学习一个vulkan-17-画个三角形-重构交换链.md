---
title: 学习一个vulkan(17)-画个三角形-重构交换链
date: 2018-11-16 09:45:47
tags: vulkan
categories: computer graphic
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
<b>介绍</b>
我们现在的应用程序可以成功绘制一个三角形了，但对于某些情况它还不能正确的处理。其中一个可能的情况就是窗口表面可能会发生变化，使得交换链不再与之兼容。导致这种情况发生的原因之一是窗口大小的变化，我们必须捕获这些事件并重新创建交换链。

<b>重新创建交换链</b>
创建一个新的recreateSwapChain函数，该函数调用createSwapChain以及依赖于交换链或窗口大小对象的所有相关的创建函数：
```cpp
void recreateSwapChain() {
    vkDeviceWaitIdle(device);

    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandBuffers();
}
```
我们首先调用vkDeviceWaitIdle，因为就像在上一章中一样，我们不应该触及到可能仍在使用中的资源。显然，我们要做的第一件事就是重新构建交换链本身，还需要重新创建图像视图，因为它们是直接基于交换链图像的，也需要重新创建渲染过程，因为它取决于交换链图像的格式，虽然交换链图像格式很少在窗口调整大小等操作期间发生变化，但仍应进行处理。视口和裁剪矩形的大小是在创建图形管道期间指定的，因此还需要重建管道。也可以通过对视口和裁剪矩形使用动态状态来避免这种情况。最后，帧缓冲区和命令缓冲区也直接依赖于交换链图像，所以也要重建。

为了确保在重新创建它们之前可以清除这些对象的旧版本，我们应该将一些清理代码移动到我们可以从recreateSwapChain中调用的单独的函数中。这里创建cleanupSwapChain：
```cpp
void cleanupSwapChain() {

}

void recreateSwapChain() {
    vkDeviceWaitIdle(device);

    cleanupSwapChain();

    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandBuffers();
}
```
我们从cleanup中将需要被重新创建的对象所对应的清理代码移动到cleanupSwapChain中:
```cpp
void cleanupSwapChain() {
    for (size_t i = 0; i < swapChainFramebuffers.size(); i++) {
        vkDestroyFramebuffer(device, swapChainFramebuffers[i], nullptr);
    }

    vkFreeCommandBuffers(device, commandPool, static_cast<uint32_t>(commandBuffers.size()), commandBuffers.data());

    vkDestroyPipeline(device, graphicsPipeline, nullptr);
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    vkDestroyRenderPass(device, renderPass, nullptr);

    for (size_t i = 0; i < swapChainImageViews.size(); i++) {
        vkDestroyImageView(device, swapChainImageViews[i], nullptr);
    }

    vkDestroySwapchainKHR(device, swapChain, nullptr);
}

void cleanup() {
    cleanupSwapChain();

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
        vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
        vkDestroyFence(device, inFlightFences[i], nullptr);
    }

    vkDestroyCommandPool(device, commandPool, nullptr);

    vkDestroyDevice(device, nullptr);

    if (enableValidationLayers) {
        DestroyDebugReportCallbackEXT(instance, callback, nullptr);
    }

    vkDestroySurfaceKHR(instance, surface, nullptr);
    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```
我们可以从头开始重新创建命令池，但这太浪费了。这里选择使用vkFreeCommandBuffers函数来清理现有的命令缓冲区。这样我们就可以重用现有的池来分配新的命令缓冲区。

要正确处理窗口大小调整，我们还需要查询帧缓冲区的当前大小，以确保交换链图像具有（新的）正确的尺寸。要做到这一点，需要更改chooseSwapExtent函数以考虑实际大小：
```cpp
VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities) {
    if (capabilities.currentExtent.width != std::numeric_limits<uint32_t>::max()) {
        return capabilities.currentExtent;
    } else {
        int width, height;
        glfwGetFramebufferSize(window, &width, &height);

        VkExtent2D actualExtent = {
            static_cast<uint32_t>(width),
            static_cast<uint32_t>(height)
        };

        ...
    }
}
```
这就是重建交换链需要的所有东西了。但是，这种方法的缺点是我们需要在创建新的交换链之前停止所有渲染。在旧交换链的绘制命令执行过程时来创建新的交换链是可能的，您需要将之前的交换链传递给VkSwapchainCreateInfoKHR结构中的oldSwapChain字段，并在完成使用后立即销毁旧的交换链。

<b>次优或过时的交换链</b>
现在我们只需要确定何时需要交换链重新创建并调用我们的新的recreateSwapChain函数。幸运的是，Vulkan在显示期间通常可以告诉我们交换链已经不再适用了，vkAcquireNextImageKHR和vkQueuePresentKHR函数可以返回以下特殊值来指示这个信息：
* VK_ERROR_OUT_OF_DATE_KHR：交换链已与表面不兼容，无法再用于渲染。通常在窗口调整大小后发生。
* VK_SUBOPTIMAL_KHR：交换链仍然可以用于成功呈现到表面，但表面属性不再完全匹配。

```cpp
VkResult result = vkAcquireNextImageKHR(device, swapChain, std::numeric_limits<uint64_t>::max(), imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

if (result == VK_ERROR_OUT_OF_DATE_KHR) {
    recreateSwapChain();
    return;
} else if (result != VK_SUCCESS && result != VK_SUBOPTIMAL_KHR) {
    throw std::runtime_error("failed to acquire swap chain image!");
}
```
如果交换链在尝试获取图像时过时（out of date）了，则不会再呈现给它。因此，我们应该立即重新创建交换链，并在下一个drawFrame的调用中再次尝试。

但是，如果我们在此时中止绘图，那么栅栏将永远无法通过vkQueueSubmit进行提交，并且当我们稍后尝试等待它时它将处于意外（unexpected）状态。我们可以重新创建围栏作为交换链重新创建的一部分，但移动vkResetFences调用显得更简单：
```cpp
vkResetFences(device, 1, &inFlightFences[currentFrame]);

if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit draw command buffer!");
}
```
如果交换链不是最理想的话，您也可以同样处理，但在这种情况下我选择继续进行，因为我们已经获取到一个图像了。VK_SUCCESS和VK_SUBOPTIMAL_KHR都被视为“成功”的返回码。
```cpp
result = vkQueuePresentKHR(presentQueue, &presentInfo);

if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR) {
    recreateSwapChain();
} else if (result != VK_SUCCESS) {
    throw std::runtime_error("failed to present swap chain image!");
}

currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHT;
```
vkQueuePresentKHR函数返回具有相同含义的相同的值。在这种情况下，如果交换链不是最理想的，我们也会重新创建交换链，因为我们希望得到最好的结果。

<b>处理尺寸变化</b>
虽然许多驱动程序和平台在窗口调整大小后都会自动触发VK_ERROR_OUT_OF_DATE_KHR，但这种情况是不能完全保证的。这就是为什么我们会添加一些额外的代码来明确的处理尺寸的改变。首先添加一个新的成员变量，标记尺寸改变已发生：
```cpp
std::vector<VkFence> inFlightFences;
size_t currentFrame = 0;

bool framebufferResized = false;
```
然后修改drawFrame函数检查此标志：
```cpp
if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR || framebufferResized) {
    framebufferResized = false;
    recreateSwapChain();
} else if (result != VK_SUCCESS) {
    ...
}
```
现在要实际检测调整大小，我们可以使用GLFW框架中的glfwSetFramebufferSizeCallback函数来设置回调：
```cpp
void initWindow() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);

    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
    glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
}

static void framebufferResizeCallback(GLFWwindow* window, int width, int height) {

}
```
我们创建一个静态函数作为回调的原因是因为GLFW不知道如何正确的使用指向HelloTriangleApplication实例的指针来调用相应的成员函数。

但是，我们确实在回调中获得了对GLFWwindow的引用，并且还有另一个GLFW函数允许您在其中存储任意指针：glfwSetWindowUserPointer：
```cpp
window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
glfwSetWindowUserPointer(window, this);
glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
```
现在可以使用glfwGetWindowUserPointer从回调中检索这个值，来正确设置标志：
```cpp
static void framebufferResizeCallback(GLFWwindow* window, int width, int height) {
    auto app = reinterpret_cast<HelloTriangleApplication*>(glfwGetWindowUserPointer(window));
    app->framebufferResized = true;
}
```
现在尝试运行程序并调整窗口大小，查看帧缓冲区是否确实能通过窗口大小的改变来正确调整大小。

<b>处理最小化</b>
还有另一种情况，交换链可能会丢失数据，这是一种特殊的窗口大小调整：窗口最小化。这种情况很特殊，因为它会导致帧缓冲区大小为0。在本教程中，我们将通过扩展recreateSwapChain函数，使窗口一直暂停直到他再次回到前台：
```cpp
void recreateSwapChain() {
    int width = 0, height = 0;
    while (width == 0 || height == 0) {
        glfwGetFramebufferSize(window, &width, &height);
        glfwWaitEvents();
    }

    vkDeviceWaitIdle(device);

    ...
}
```
现在已经完成了的第一个表现还不错的Vulkan程序。在下一章中，我们将抛弃顶点着色器中的硬编码顶点，真正的使用顶点缓冲区。





