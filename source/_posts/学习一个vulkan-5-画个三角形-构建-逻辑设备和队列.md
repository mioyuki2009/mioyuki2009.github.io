---
title: 学习一个vulkan(5)-画个三角形(构建)-逻辑设备和队列
date: 2018-11-02 09:19:22
tags: vulkan
categories: learn
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
<b>介绍</b>
选择好了要使用的物理设备后，现在需要设置一个逻辑设备和他对接。逻辑设备的创建过程和实例的创建类似，同样需要描述想要使用的功能。
这里还需要指定现在要创建的队列，不过在之前​​已经查询过了哪些队列系列可用。如果有不同的需求，也可以从同一物理设备创建多个逻辑设备。

首先添加一个新的类成员来存储逻辑设备句柄。
```cpp
VkDevice device;
```
接下来，在initVulkan添加createLogicalDevice函数：
```cpp
void initVulkan() {
    createInstance();
    setupDebugCallback();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createLogicalDevice() {

}
```

<b>指定要创建的队列</b>
创建逻辑设备同样需要在结构中设定一堆细节，第一个需要知道的就是VkDeviceQueueCreateInfo了。这个结构描述了单个队列簇所需的队列数目。
目前只需要具有图形功能的相关队列：
```cpp
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

VkDeviceQueueCreateInfo queueCreateInfo = {};
queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
queueCreateInfo.queueFamilyIndex = indices.graphicsFamily.value();
queueCreateInfo.queueCount = 1;
```
当前可用的驱动程序只允许在每个队列簇中创建少量队列，虽然现在也并不不需要多个队列。这是因为可以在多个线程上创建所有的命令缓冲，然后使用一个低开销的调用在主线程上一次性提交。

Vulkan允许为队列分配优先级，使用介于0.0和1.0之间的浮点数来影响命令缓冲区执行的调度顺序。
即使只有一个队列，也必需要设置的：
```cpp
float queuePriority = 1.0f;
queueCreateInfo.pQueuePriorities = &queuePriority;
```

<b>指定使用的设备功能</b>
第二个需要设定的是我们将要使用的设备的功能，这个就是之前已经使用vkPhysicalDevice查询过的支持的功能，比如几何着色器。
所以这里不需要另外指定什么，简单定义一下就行，其他的就自动变成VK_FALSE了，同样，之后添加更多Vulkan功能时会回到这里进行更改。
```cpp
VkPhysicalDeviceFeatures deviceFeatures = {};
```

<b>创建逻辑设备</b>
有了前两个结构之后，现在可以用来填充VkDeviceCreateInfo结构了：
```cpp
VkDeviceCreateInfo createInfo = {};
createInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;
```
首先添加指向队列创建信息和设备功能结构的指针：
```cpp
createInfo.pQueueCreateInfos = &queueCreateInfo;
createInfo.queueCreateInfoCount = 1;

createInfo.pEnabledFeatures = &deviceFeatures;
```
其余信息与VkInstanceCreateInfo结构相似，需要指定扩展和验证层，不同之处在于这次指定的是针对于设备的。

举个例子，有一个针对设备的扩展是VK_KHR_swapchain，它允许将该设备的渲染图像呈现给窗口。系统中可能存在只支持计算而缺乏这个功能的Vulkan设备。之后将在交换链的章节讨论这个扩展。

如验证层的章节中所说的一样，这里会为设备启用与实例相同的验证层。暂时不需要任何特定于设备的扩展：
```cpp
createInfo.enabledExtensionCount = 0;

if (enableValidationLayers) {
    createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
    createInfo.ppEnabledLayerNames = validationLayers.data();
} else {
    createInfo.enabledLayerCount = 0;
}
```
现在可以通过vkCreateDevice来实例化逻辑设备了：
```cpp
if (vkCreateDevice(physicalDevice, &createInfo, nullptr, &device) != VK_SUCCESS) {
    throw std::runtime_error("failed to create logical device!");
}
```
参数分别是和他接口的物理设备，刚才填充的队列和使用信息，一个可选的分配回调指针和指向存储逻辑设备句柄的指针。
与创建实例的函数类似，这个函数也会由于启用了不存在的扩展或指定了不支持的功能的返回错误信息。

最后同样需要在cleanup函数中添加清理方法vkDestroyDevice：
```cpp
void cleanup() {
    vkDestroyDevice(device, nullptr);
    ...
}
```
逻辑设备不直接与实例交互，所以参数里没有它。

<b>检索队列句柄</b>
队列将会和逻辑设备一同被创建，但是现在还没有与之相对应的接口句柄，首先添加一个类成员来存储图形队列的句柄：
```cpp
VkQueue graphicsQueue;
```
当设备被销毁时，会隐式清除设备队列，所以并不需要在cleanup中添加什么

可以使用vkGetDeviceQueue函数来检索每个队列系列的队列句柄，参数分别是逻辑设备，队列簇，队列索引和指向存储队列句柄的指针。
因为我们只是从这个簇中创建了一个队列，所以用索引0就行了：
```cpp
vkGetDeviceQueue(device, indices.graphicsFamily.value(), 0, &graphicsQueue);
```
有了逻辑设备和队列句柄，现在终于可以开始使用显卡了，在接下来的几章中，我们将设置资源以向窗口系统进行显示。





