---
title: 学习一个vulkan(1)-画个三角形(构建)-物理设备和队列簇
date: 2018-11-01 09:49:28
tags: vulkan
categories: learn
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
<b>选择物理设备</b>
在通过VkInstance初始化Vulkan库之后，我们需要在系统中查找并选择支持我们所需功能的显卡，事实上，我们可以选择任意数量的显卡并同时使用它们，但在本教程中，将使用第一个适合需求的显卡。
添加一个函数pickPhysicalDevice并在initVulkan函数中添加对它的调用：
```cpp
void initVulkan() {
    createInstance();
    setupDebugCallback();
    pickPhysicalDevice();
}

void pickPhysicalDevice() {

}
```
我们最终选择的显卡设备将存储在VkPhysicalDevice类型的新成员中。当VkInstance被销毁时，该对象将被隐式销毁，因此不需要在cleanup函数中做任何新的操作。
```cpp
VkPhysicalDevice physicalDevice = VK_NULL_HANDLE;
```
查询显卡和查询扩展类似，首先需要查找设备的数目。
```cpp
uint32_t deviceCount = 0;
vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);
```
如果没有设备（0个）支持Vulkan的话，就没有必要进行之后的工作了：
```cpp
if (deviceCount == 0) {
    throw std::runtime_error("failed to find GPUs with Vulkan support!");
}
```
否则,分配一个数组来保存所有VkPhysicalDevice句柄：
```cpp
std::vector<VkPhysicalDevice> devices(deviceCount);
vkEnumeratePhysicalDevices(instance, &deviceCount, devices.data());
```
现在需要检查每一个是否适合我们想要执行的操作，因为并非所有的显卡都是相同的。
为此，需要一个新功能：
```cpp
bool isDeviceSuitable(VkPhysicalDevice device) {
    return true;
}
```
这里将检查是否有物理设备符合我们的需要的功能：
```cpp
for (const auto& device : devices) {
    if (isDeviceSuitable(device)) {
        physicalDevice = device;
        break;
    }
}

if (physicalDevice == VK_NULL_HANDLE) {
    throw std::runtime_error("failed to find a suitable GPU!");
}
```
下一节将介绍我们将在isDeviceSuitable函数中检查的第一个要求。
随着之后使用更多Vulkan功能后，也会在这里添加更多检查。

<b>基础设备适用性检查</b>
为了评估设备的适用性，我们可以通过查询一些细节来开始，可以使用vkGetPhysicalDeviceProperties查询基本设备属性，如名称，类型和支持的Vulkan版本：
```cpp
VkPhysicalDeviceProperties deviceProperties;
vkGetPhysicalDeviceProperties(device, &deviceProperties);
```
可以使用vkGetPhysicalDeviceFeatures查询对纹理压缩，64位浮点和多视口渲染（对VR有用）等可选功能的支持：
```cpp
VkPhysicalDeviceFeatures deviceFeatures;
vkGetPhysicalDeviceFeatures(device, &deviceFeatures);
```
当然也有更多的细节可以查询到，这些将在之后讨论设备内存和队列的时候谈到。

举个例子，假设我们认为我们的应用程序仅适用于支持几何着色器的独立显卡，那么isDeviceSuitable函数将如下所示：
```cpp
bool isDeviceSuitable(VkPhysicalDevice device) {
    VkPhysicalDeviceProperties deviceProperties;
    VkPhysicalDeviceFeatures deviceFeatures;
    vkGetPhysicalDeviceProperties(device, &deviceProperties);
    vkGetPhysicalDeviceFeatures(device, &deviceFeatures);

    return deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU &&
           deviceFeatures.geometryShader;
}
```
除了仅仅挑选第一个适合的设备之外，也可以对每一个设备打一个分数然后挑选最高的一个，这样做的话，可以给独立显卡赋予一个更高的分数，如果没有的话，就会挑选到集成显卡上了。
实现起来大概是这样：
```cpp
#include <map>

...

void pickPhysicalDevice() {
    ...

    // Use an ordered map to automatically sort candidates by increasing score
    std::multimap<int, VkPhysicalDevice> candidates;

    for (const auto& device : devices) {
        int score = rateDeviceSuitability(device);
        candidates.insert(std::make_pair(score, device));
    }

    // Check if the best candidate is suitable at all
    if (candidates.rbegin()->first > 0) {
        physicalDevice = candidates.rbegin()->second;
    } else {
        throw std::runtime_error("failed to find a suitable GPU!");
    }
}

int rateDeviceSuitability(VkPhysicalDevice device) {
    ...

    int score = 0;

    // Discrete GPUs have a significant performance advantage
    if (deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU) {
        score += 1000;
    }

    // Maximum possible size of textures affects graphics quality
    score += deviceProperties.limits.maxImageDimension2D;

    // Application can't function without geometry shaders
    if (!deviceFeatures.geometryShader) {
        return 0;
    }

    return score;
}
```
这些其实并不需要全部实现，只是为了更加了解如何设计设备的选择功能。当然，也可以列出相应的名称让用户来选择。

因为这里才刚刚起步，Vulkan支持是当前唯一的需求，所以现在任何GPU都可行：
```cpp
bool isDeviceSuitable(VkPhysicalDevice device) {
    return true;
}
```
在下一节中，我们将讨论要检查的第一个真正需要的功能。

<b>队列簇（Queue families）</b>
之前简要介绍过Vulkan中的几乎所有操作，从绘图到上传纹理，都需要将相关的命令提交到队列。不同的队列簇都会有各种不同类型的队列，而且一个队列中只支持一个命令中的一个子集。
举个例子，一个队列簇要不就只能处理计算命令，要不就只能处理内存传输相关命令。

我们需要检查设备支持哪些队列簇，以及哪一个支持我们要使用的命令。因此，为此，我们将添加一个新函数findQueueFamilies，用于查找我们需要的所有队列簇。现在只需要查找支持图形命令的队列，在之后可能会扩展此功能以便在查找更多内容。

此函数将返回满足某些所需属性的队列系列的索引。最好的方法是使用一个结构，可以使用std :: optional来跟踪是否找到了索引：
```cpp
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;

    bool isComplete() {
        return graphicsFamily.has_value();
    }
};
```
这里需要#include <optional>，现在可以实现findQueueFamilies了：
```cpp
QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;

    ...

    return indices;
}
```
队列的查找还是和之前一样，这里使用vkGetPhysicalDeviceQueueFamilyProperties：
```cpp
uint32_t queueFamilyCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);

std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());
```
VkQueueFamilyProperties结构包含有关队列簇的一些详细信息，包括支持的操作类型以及可以基于该系列创建的队列数目。
我们需要找到至少一个支持VK_QUEUE_GRAPHICS_BIT的队列簇：
```cpp
int i = 0;
for (const auto& queueFamily : queueFamilies) {
    if (queueFamily.queueCount > 0 && queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT) {
        indices.graphicsFamily = i;
    }

    if (indices.isComplete()) {
        break;
    }

    i++;
}
```
这个函数实现完毕了，我们可以将它用于isDeviceSuitable函数中，来确保设备可以处理我们想要使用的命令：
```cpp
bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    return indices.isComplete();
}
```
找到了合适的物理设备之后，下一步就是创建一个对应的逻辑设备了。















