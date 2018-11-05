---
title: 学习一个vulkan(1)-画个三角形(构建)-基础代码
date: 2018-10-29 16:17:34
tags: vulkan
categories: learn
thumbnail: /images/cat.jpg
---
学习一个vlkan，基本就是照着教程敲了。
参考[这个](https://vulkan-tutorial.com/Introduction)。
<!-- more -->
<b>大致结构</b>
第一步的目标就是画一个简单三角形。首先确保已经正确安装了Vulkan和GLFW。
首先构造一个大概的框架，大概这样：
```cpp
#include <vulkan/vulkan.h>

#include <iostream>
#include <stdexcept>
#include <functional>
#include <cstdlib>

class HelloTriangleApplication {
public:
    void run() {
        initVulkan();
        mainLoop();
        cleanup();
    }

private:
    void initVulkan() {

    }

    void mainLoop() {

    }

    void cleanup() {

    }
};

int main() {
    HelloTriangleApplication app;

    try {
        app.run();
    } catch (const std::exception& e) {
        std::cerr << e.what() << std::endl;
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```
用到头文件的相关功能大概就是：
Vulkan：这个就是学习的东西，当然是要用到的
stdexcept和iostream用来打印相关信息和错误
functional：用到了其中的lambda表达式
cstdlib：用到了其中的EXIT_SUCCESS和EXIT_FAILURE宏
这里将Vulkan的对象封装到这个类里，然后设置一个函数来初始化他们，就是initVulkan了
这些都完成之后就可以进入主循环来渲染帧了
可以在mainLoop中一直循环直到窗口关闭
当窗口关闭之后，还需要在cleanup里清除相关用过的资源
一旦在运行过程中产生任何异常都会抛出std::runtime_error的异常，之后传回main函数打印到控制台上
为了捕获相对详尽的异常，这里需要使用std::exception
后面有个例子就是用这个发现了有个需要的扩展不支持的错误
每一次实现什么新功能后基本都会在initVulkan初始化新的对象，以及在cleanup里进行释放

<b>资源管理</b>
就像每次使用一个内存块都需要malloc和free一样，每一个Vulkan对象在用完之后也需要被销毁
现代C++中，可以通过<memory>中的各种方法来进行自动内存管理，但是在本教程中会明确的说明中的各种方法来进行自动内存管理
毕竟Vulkan的宗旨就是明确自己的每一个操作来避免错误，因此最好能明确对象的生命周期来更好的了解相关API
在本教程之外，可以通过重载std :: shared_ptr来实现自动的内存管理，使用[RAII](https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)是实现大型Vulkan推荐的方式，但是对于学习来说，最好还是要知道背后发生的详细信息
Vulkan对象可以使用vkCreateXXX这种函数直接创建，也可以通过其他具有vkAllocateXXX功能的对象进行分配，确保对象不再在任何地方使用后，您需要使用对应的vkDestroyXXX和vkFreeXXX销毁它。
这些函数的参数通常因不同类型的对象而异，但是它们共享一个参数：pAllocator。这是一个可选参数，允许为自定义内存分配器指定回调。
我们将在教程中忽略此参数，并始终将nullptr作为参数传递。

<b>整合GLFW</b>
在不使用窗口，只是单单进行屏幕外渲染时，Vulkan是非常好的，但能显示当然是更好了，所以使用下文替换掉#include <vulkan/vulkan.h>：
```cpp
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
```
这样GLFW将包含自己的定义并自动加载Vulkan标头。
添加一个initWindow函数，并在run函数其他调用函数之前执行他。
我们将使用该函数初始化GLFW并创建一个窗口。
```cpp
void run() {
    initWindow();
    initVulkan();
    mainLoop();
    cleanup();
}

private:
    void initWindow() {

    }
```
initWindow中的第一个调用应该是glfwInit()，这个函数是用来初始化GLFW库的，因为GLFW最初是为创建OpenGL上下文而设计的，所以我们需要告诉它不要在后续调用中创建OpenGL上下文：
```cpp
glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
```
因为处理调整大小的窗口需要特别小心，之后才会使用到，现在需要设置一下来禁用它：
```cpp
glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
```
现在的就是创建真正的窗口了。添加GLFWwindow *window;存储为私有类的对象，并初始化窗口：
```cpp
window = glfwCreateWindow(800, 600, "Vulkan", nullptr, nullptr);
```
前三个参数指定窗口的宽度，高度和标题。第四个参数选择指定监视器来打开窗口，最后一个参数与OpenGL相关，设为nullptr就行。
使用常量而不是使用简单的数字来定义宽度和高度会比较好，因为之后会多次使用这些值。
在HelloTriangleApplication的类定义的上面添加以下行：
```cpp
const int WIDTH = 800;
const int HEIGHT = 600;
```
并用下文来替换之前窗口创建的方法：
```cpp
window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
```
现在initWindow函数应该长这样子：
```cpp
void initWindow() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);

    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
}
```
为了使应用程序一直运行直到发生错误或窗口关闭，我们需要向mainLoop函数添加一个事件循环，如下所示：
```cpp
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }
}
```
这段代码很简单的。
它一直循环并检查事件，例如按下关闭的时候，直到用户关闭窗口才会停止循环。
这也是我们稍后用来渲染单个帧的循环。
窗口关闭后，我们需要通过销毁资源并终止GLFW来清理资源。
我们的第一个清理代码如下所示：
```cpp
void cleanup() {
    glfwDestroyWindow(window);

    glfwTerminate();
}
```
现在运行该程序时，可以看到一个标题为Vulkan的窗口，在关闭窗口之前都会一直存在。
现在已经构造好了Vulkan应用程序的大致框架，下一步就是创建第一个Vulkan对象了