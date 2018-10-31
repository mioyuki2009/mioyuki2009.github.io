---
title: 学习一个vulkan(2)-画个三角形(构建)-实例
date: 2018-10-30 15:08:16
tags: vulkan
categories: learn
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
<b>创建一个实例</b>
最开始需要做的事是通过创建实例（initialize）来初始化Vulkan库。
这个实例是应用程序与Vulkan库之间联系，创建它的时候需要向驱动程序指定一些有关应用程序的详细信息。
首先在initVulkan里添加一个createInstance函数
```cpp
void initVulkan() {
    createInstance();
}
```
同时还要添加一个类成员来保存实例的句柄：
```cpp
private:
VkInstance instance;
```
为了创建这个实例，必须用一些应用程序的信息来填充相关结构，这个过程当然也可以不做，但是明确的提供了特定的信息时候，我们也能对这个特定的程序进行优化，就像某些有特殊功能的图形引擎一样。
这个用来填充的结构叫做VkApplicationInfo：
```cpp
VkApplicationInfo appInfo = {};
appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
appInfo.pApplicationName = "Hello Triangle";
appInfo.applicationVersion = VK_MAKE_VERSION(1, 0, 0);
appInfo.pEngineName = "No Engine";
appInfo.engineVersion = VK_MAKE_VERSION(1, 0, 0);
appInfo.apiVersion = VK_API_VERSION_1_0;
```
Vulkan中的许多结构都需要在sType成员中明确的说明其类型，上面这个也是其中一个，同时他也有一个pNext成员用来指向扩展的信息，这个在以后会说明
这里用默认的就行，会自动设置为nullptr
Vulkan中的很多信息都是通过结构而不是函数参数传递的，我们必须填充结构来为创建实例提供足够的信息。
之后的这个结构是必须要的，它告诉Vulkan驱动程序我们想要使用哪些全局扩展和验证层。全局在这里意味着它们适用于整个程序
```cpp
VkInstanceCreateInfo createInfo = {};
createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
createInfo.pApplicationInfo = &appInfo;
```
这两个参数很简单，一看就知道了。下面的两个会用来明确的指向需要的全局扩展。
Vulkan是一个与平台无关的API，这意味着需要一个扩展接口与窗口系统联系。GLFW有一个方便的内置函数可以直接返回需要的扩展，可以把这个扩展传递给结构：
```cpp
uint32_t glfwExtensionCount = 0;
const char** glfwExtensions;
glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);
createInfo.enabledExtensionCount = glfwExtensionCount;
createInfo.ppEnabledExtensionNames = glfwExtensions;
```
结构的最后两个成员用来确定要启用的全局验证层，之后在讨论个问题，这里先简单的留空就行：
```cpp
createInfo.enabledLayerCount = 0;
```
现在已经完成了构建这个实例需要的东西，现在可以进行vkCreateInstance的调用了：
```cpp
VkResult result = vkCreateInstance(&createInfo, nullptr, &instance);
```
简单看来，构建Vulkan对象的三个参数一般如下：
1、指向有构建信息的指针（createInfo）
2、自定义分配器的回调指针，一般就设为nullptr
3、指向存储新对象句柄的变量的指针（instance）
如果一切顺利，那么实例的句柄就会存储在VkInstance类成员中。基本上所有Vulkan函数都返回VkResult类型的值，即VK_SUCCESS或错误代码。
要检查实例是否已成功创建，我们不需要存储结果，只需检查返回的值：
```cpp
if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
    throw std::runtime_error("failed to create instance!");
}
```
现在可以运行该程序来确认是否无误了。

<b>检查扩展的支持</b>
如果查看vkCreateInstance的[文档](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/vkCreateInstance.html)，
可以看到其中一个可能的错误代码是VK_ERROR_EXTENSION_NOT_PRESENT
如果出现这个错误的话，可以的手动指定需要的扩展，这对于窗口系统界面等基本扩展很有意义
如果我们想检查支持的扩展该怎么办呢？
要在创建实例之前查看支持的扩展列表，可以使用vkEnumerateInstanceExtensionProperties函数。
它需要一个指向扩展数量的变量的指针和一个VkExtensionProperties数组作为参数来存储扩展的详细信息，它的第一个参数是可选的，允许我们通过设定特定的验证层来过滤某些扩展，这里直接不管。
要分配一个数组来保存各个扩展的细节，我们首先需要知道扩展的数目。可以通过将后一个参数留空来得到扩展的数量：
```cpp
uint32_t extensionCount = 0;
vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);
```
现在分配一个数组来保存扩展细节（需要#include <vector>）：
```cpp
std::vector<VkExtensionProperties> extensions(extensionCount);
```
最终得到了相应扩展的细节：
```cpp
vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, extensions.data());
```
每个VkExtensionProperties结构都包含了对应扩展的名称和版本。可以用简单的for打印查看（\t是缩进）：
```cpp
std::cout << "available extensions:" << std::endl;

for (const auto& extension : extensions) {
    std::cout << "\t" << extension.extensionName << std::endl;
}
```
如果想查看有关Vulkan支持的一些详细信息，可以将这个代码添加到createInstance函数中。
作为挑战，也可以尝试创建一个函数，检查glfwGetRequiredInstanceExtensions返回的所有扩展是否都包含在支持的扩展列表中。

<b>清理</b>
VkInstance应该在程序退出之前销毁掉，可以使用vkDestroyInstance函数在clean中销毁：
```cpp
void cleanup() {
    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```
vkDestroyInstance函数的参数很简单，Vulkan中的分配和释放函数有一个可选的allocator回调，这里直接也设为nullptr不管。
以后创建的所有其他Vulkan资源需要在实例销毁之前进行清理。
在进行更复杂的步骤之前，需要通过检查验证层来评估调试的选项。


