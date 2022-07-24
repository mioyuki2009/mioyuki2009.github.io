---
title: 学习一个vulkan(6)-画个三角形(显示)-Window surface
date: 2018-11-03 09:35:05
tags: vulkan
categories: computer graphic
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
由于Vulkan是一个与平台特性无关联的API集合，不能直接与窗口系统进行交互。为了在Vulkan和窗口系统之间建立连接来显示结果，这里需要WSI（窗口系统集成）扩展。
在本章中将讨论其中的第一个，即VK_KHR_surface。它使用VkSurfaceKHR对象，这个对象是一个表示呈现渲染图像surface的抽象类型。
本程序中的surface将由使用GLFW打开的窗口来支持。

VK_KHR_surface扩展是一个实例级扩展，我们实际上在之前已经启用了它，因为它包含在glfwGetRequiredInstanceExtensions返回的列表中了，该列表还包括我们将在接下来的几章中使用的一些其他WSI扩展。

窗口surface需要在实例创建后创建，因为它实际上可以影响物理设备的选择。现在才说这个是因为这个主题太大了，之前说可能引起混乱。
值得注意的是，如果不需要屏幕渲染的话，窗口surface在vulkan中是完全可选的。Vulkan允许这样做，不需要同OpenGL一样必须要创建窗体。

<b>创建窗体surface</b>
在callback成员下面添加一个新的成员变量：
```cpp
VkSurfaceKHR surface;
```
虽然VkSurfaceKHR对象及其用法与平台无关，但创建过程需要依赖具体的窗体系统的细节。例如在Windows中就需要需要HWND和HMODULE句柄。因此针对特定平台会提供相应的扩展，在Windows上为VK_KHR_win32_surface，它自动包含在glfwGetRequiredInstanceExtensions列表中。

我将演示如何使用这个特定于平台的扩展在Windows上的创建方法，但是并不会在这个程序中使用。使用像GLFW这样的库然后继续使用特定于平台的代码是没有意义的。GLFW实际上有glfwCreateWindowSurface来处理平台之间的差异，不过在使用他之前，还是要知道背后的原理。

因为窗口surface 是Vulkan对象，所以它也需要一个填充好的VkWin32SurfaceCreateInfoKHR结构来构建。
它有两个重要参数：hwnd和hinstance，分别是窗口和进程的句柄：
```cpp
VkWin32SurfaceCreateInfoKHR createInfo = {};
createInfo.sType = VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR;
createInfo.hwnd = glfwGetWin32Window(window);
createInfo.hinstance = GetModuleHandle(nullptr);
```
glfwGetWin32Window函数用于从GLFW窗口对象获取原始HWND,GetModuleHandle返回当前进程的HINSTANCE句柄。

之后就可以使用VkWin32SurfaceCreateInfoKHR来创建，同样需要显示加载。
调用很简单，参数分别是实例，surface的创建细节，自定义分配器和要存储的surface句柄的变量：
```cpp
auto CreateWin32SurfaceKHR = (PFN_vkCreateWin32SurfaceKHR) vkGetInstanceProcAddr(instance, "vkCreateWin32SurfaceKHR");

if (!CreateWin32SurfaceKHR || CreateWin32SurfaceKHR(instance, &createInfo, nullptr, &surface) != VK_SUCCESS) {
    throw std::runtime_error("failed to create window surface!");
}
```
这个过程在Linux等其他平台也是类似的，使用X11界面窗体系统，可以通过vkCreateXcbSurfaceKHR函数建立连接。

glfwCreateWindowSurface函数正好可以实现这种操作，并在每个平台下执行不同的实现，我们现在将它集成到我们的程序中，在initVulkan中添加createSurface函数，在setupDebugCallback后面调用：
```cpp
void initVulkan() {
    createInstance();
    setupDebugCallback();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createSurface() {

}
```
GLFW的调用采用简单的参数而不是结构，这使得函数的使用非常简单：
```cpp
void createSurface() {
    if (glfwCreateWindowSurface(instance, window, nullptr, &surface) != VK_SUCCESS) {
        throw std::runtime_error("failed to create window surface!");
    }
}
```
参数分别是VkInstance，GLFW窗口指针，自定义分配器和指向VkSurfaceKHR类型的指针变量。它只是通过相关平台传递调用VkResult。
GLFW不提供清理surface的相关功能，可以通过原始API轻松完成：
```cpp
void cleanup() {
        ...
        vkDestroySurfaceKHR(instance, surface, nullptr);
        vkDestroyInstance(instance, nullptr);
        ...
}
```
确保在销毁实例之前销毁surface 

<b>查询显示（presentation）支持</b>
虽然Vulkan支持窗口系统集成的功能，但这不代表系统中的设备都支持。因此，我们需要扩展isDeviceSuitable以确保设备可以将图像呈现给我们创建的surface。
由于显示是专用队列（queue-specific）的功能，问题就变成了找到一个支持在surface上进行显示功能的队列簇。

支持绘图命令的队列簇和支持显示的队列簇可能不是一个，考虑到可能存在的不同的表示队列，所以需要对QueueFamilyIndices结构进行修改：
```cpp
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;
    std::optional<uint32_t> presentFamily;

    bool isComplete() {
        return graphicsFamily.has_value() && presentFamily.has_value();
    }
};
```
接下来，需要修改find​​QueueFamilies函数来查找具有在窗口surface进行显示功能的队列簇。
相关的查找函数是vkGetPhysicalDeviceSurfaceSupportKHR，参数分别是物理设备，队列簇索引和surface。将这个函数添加到VK_QUEUE_GRAPHICS_BIT所在的循环体中：
```cpp
VkBool32 presentSupport = false;
vkGetPhysicalDeviceSurfaceSupportKHR(device, i, surface, &presentSupport);
```
之后需检查返回的布尔值并存储显示功能队列簇的索引：
```
if (queueFamily.queueCount > 0 && presentSupport) {
    indices.presentFamily = i;
}
```
值得注意的，支持绘图命令的队列簇和支持显示的队列簇可能是相同的队列簇，在程序中这两种情况会有统一的处理方式（当成不同的队列簇）。
不过，可以添加逻辑来优先选择同时支持这两个功能的物理设备以提高性能。

<b>创建显示队列</b>
现在要做的就是修改逻辑设备的创建过程，创建显示队列并获取VkQueue句柄，添加句柄成员变量：
```cpp
VkQueue presentQueue;
```
接下来，我们需要多个VkDeviceQueueCreateInfo结构来创建不同功能的队列。一个优雅的方式是set集合来保存需要的所有队列簇，这样就可以保证唯一性了:
```cpp
#include <set>

...

QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

std::vector<VkDeviceQueueCreateInfo> queueCreateInfos;
std::set<uint32_t> uniqueQueueFamilies = {indices.graphicsFamily.value(), indices.presentFamily.value()};

float queuePriority = 1.0f;
for (uint32_t queueFamily : uniqueQueueFamilies) {
    VkDeviceQueueCreateInfo queueCreateInfo = {};
    queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
    queueCreateInfo.queueFamilyIndex = queueFamily;
    queueCreateInfo.queueCount = 1;
    queueCreateInfo.pQueuePriorities = &queuePriority;
    queueCreateInfos.push_back(queueCreateInfo);
}
```
同时还需要修改VkDeviceCreateInfo指向的队列集合：
```cpp
createInfo.queueCreateInfoCount = static_cast<uint32_t>(queueCreateInfos.size());
createInfo.pQueueCreateInfos = queueCreateInfos.data();
```
如果队列簇相同，那么我们只需要传递一次索引。
最后，添加一个函数来检索队列句柄：
```cpp
vkGetDeviceQueue(device, indices.presentFamily.value(), 0, &presentQueue);
```
如果队列簇相同，则两个句柄很可能具有相同的值。
在下一章中，我们将讨论交换链以及如何使用他来在surface上呈现图像。















