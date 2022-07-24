---
title: 学习一个vulkan(22)-统一缓冲区-描述符布局和缓冲区
date: 2018-11-22 09:18:53
tags: vulkan
categories: computer graphic
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
<b>介绍</b>
我们现在能够将任意属性传递给每个顶点的顶点着色器，但是全局变量呢？我们将从本章开始转向3D图形，这需要一个模型-视图-投影矩阵。我们可以将它包含为顶点数据，但这会浪费内存，并且只要转换发生变化，就需要我们更新顶点缓冲区。这种变换通常会发生在每一帧。

在Vulkan中解决这个问题的正确方法是使用资源描述符（resource descriptors）。描述符是着色器可以自由访问缓冲区和图像等资源的一种方式。我们将设置一个包含转换矩阵的缓冲区，并让顶点着色器通过描述符访问它们。描述符的使用包括三个部分：
* 在管道创建期间指定描述符布局
* 从描述符池分配描述符集
* 在渲染期间绑定描述符集

描述符布局（descriptor layout）指定了管道将访问的资源类型，就像渲染过程指定了要访问的附件类型一样。描述符集（descriptor set）指定将绑定到描述符的实际缓冲区或图像资源，就像帧缓冲区指定要绑定到渲染通道附件的实际图像视图一样。然后，描述符集绑定到绘图命令，就像顶点缓冲区和帧缓冲区一样。

描述符有很多种类，但在本章中我们将使用统一缓冲区对象（UBO）。我们将在后面的章节中介绍其他类型的描述符，但基本过程是相同的。假设我们有一个数据，我们希望顶点着色器拥有一个这样的C语言形式的结构体：
```cpp
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```
然后我们可以将数据复制到VkBuffer并通过顶点着色器中的统一缓冲区对象描述符访问它，如下所示：
```glsl
layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```
我们会在每一帧更新模型，视图和投影矩阵，使前一章的矩形以3D旋转。

<b>顶点着色器</b>
修改顶点着色器以包含上面指定的统一缓冲区对象。这里假设您熟悉MVP转换，如果不是，请参阅[这个资源](http://opengl.datenwolf.net/gltut/html/index.html)。
```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```
请注意，uniform，in和out声明的顺序无关紧要。binding指令类似于属性的location指令，我们将在描述符布局中引用此绑定。将gl_Position行更改为使用变换矩阵计算裁剪坐标的最终位置。与2D三角形不同，裁剪坐标的最后一个分量可能不是1，这将导致在屏幕上转换为最终标准化设备坐标时会做一个除法。在透视投影中的透视除法（perspective division）也用到了这种方法，而且这对于能使更近的物体看起来比远离的物体更大是必要的。

<b>描述符集布局</b>
下一步是在C++端定义UBO，并告知Vulkan在顶点着色器使用该描述符：
```cpp
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```
我们可以使用GLM中的与着色器中结构体完全匹配的数据类型。矩阵中的数据与着色器预期的二进制数据兼容，所以我们可以稍后将一个UniformBufferObject通过memcpy拷贝到VkBuffer中。

我们需要提供有关在着色器中用于管道创建的每个描述符绑定的详细信息，就像我们必须对每个顶点属性及其位置索引所做的那样。我们将设置一个新函数来定义所有这些名为createDescriptorSetLayout的信息，它应该在管道创建之前调用，因为我们之后就会用到它。
```cpp
void initVulkan() {
    ...
    createDescriptorSetLayout();
    createGraphicsPipeline();
    ...
}

...

void createDescriptorSetLayout() {

}
```
每个绑定都需要通过VkDescriptorSetLayoutBinding结构来描述。
```cpp
void createDescriptorSetLayout() {
    VkDescriptorSetLayoutBinding uboLayoutBinding = {};
    uboLayoutBinding.binding = 0;
    uboLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    uboLayoutBinding.descriptorCount = 1;
}
```
前两个参数指定了着色器中使用的绑定和描述符的类型，他是一个统一缓冲区对象。着色器变量可以表示统一缓冲区对象的数组，而descriptorCount则指定数组中的值的数量。这可以用于为骨架动画指定骨架中每个骨骼的变换。我们的MVP转换在一个统一的缓冲区对象中，因此我们使用的descriptorCount为1。
```cpp
uboLayoutBinding.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;
```
我们还需要指定描述符将被引用于哪个着色器阶段。stageFlags字段可以是VkShaderStageFlagBits或VK_SHADER_STAGE_ALL_GRAPHICS的各种组合，在我们的例子中，我们只引用顶点着色器中的描述符。
```cpp
uboLayoutBinding.pImmutableSamplers = nullptr; // Optional
```
pImmutableSamplers字段仅与图像采样相关的描述符相关，我们将在后面介绍。现在可以将其保留为默认值。

所有描述符绑定都会组合到一个VkDescriptorSetLayout对象中。在pipelineLayout上面定义一个新的类成员：
```cpp
VkDescriptorSetLayout descriptorSetLayout;
VkPipelineLayout pipelineLayout;
```
然后我们可以使用vkCreateDescriptorSetLayout创建它。这个函数接受一个简单的结构体VkDescriptorSetLayoutCreateInfo，该结构体持有一个绑定数组：
```cpp
VkDescriptorSetLayoutCreateInfo layoutInfo = {};
layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
layoutInfo.bindingCount = 1;
layoutInfo.pBindings = &uboLayoutBinding;

if (vkCreateDescriptorSetLayout(device, &layoutInfo, nullptr, &descriptorSetLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create descriptor set layout!");
}
```
我们需要在管道创建期间指定描述符集布局，用以告诉Vulkan着色器将使用哪些描述符。描述符集布局在管道布局对象中指定，修改VkPipelineLayoutCreateInfo来引用布局对象：
```cpp
VkPipelineLayoutCreateInfo pipelineLayoutInfo = {};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 1;
pipelineLayoutInfo.pSetLayouts = &descriptorSetLayout;
```
您可能想知道为什么可以在此指定多个描述符集布局，这是因为单个布局已包含所有绑定了。我们将在下一章回到这里，那时我们将研究描述符池和描述符集。

描述符布局应该在程序退出前始终有效：
```cpp
void cleanup() {
    cleanupSwapChain();

    vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);

    ...
}
```

<b>统一缓冲区</b>
在下一章中，我们将指定包含着色器的UBO数据的缓冲区，但我们需要首先创建此缓冲区。我们将每帧的新数据都复制到统一缓冲区，因此拥有一个临时缓冲区并没有任何意义，在这种情况下，它只会增加额外的开销，并可能降低性能而不是改进它。

我们应该有多个缓冲区，因为多个帧可能同时都在飞行状态，我们不希望更新缓冲区以准备下一帧的时候，前一帧仍然在读取。但是，由于我们需要从每个交换链图像的命令缓冲区引用统一缓冲区，因此每个交换链图像也有一个统一的缓冲区是很有意义的。

为此，需要为uniformBuffers和uniformBuffersMemory添加新的类成员：
```cpp
VkBuffer indexBuffer;
VkDeviceMemory indexBufferMemory;

std::vector<VkBuffer> uniformBuffers;
std::vector<VkDeviceMemory> uniformBuffersMemory;
```
同样，创建一个新函数createUniformBuffers，它在createIndexBuffer之后调用并分配缓冲区：
```cpp
void initVulkan() {
    ...
    createVertexBuffer();
    createIndexBuffer();
    createUniformBuffer();
    ...
}

...

void createUniformBuffer() {
    VkDeviceSize bufferSize = sizeof(UniformBufferObject);

    uniformBuffers.resize(swapChainImages.size());
    uniformBuffersMemory.resize(swapChainImages.size());

    for (size_t i = 0; i < swapChainImages.size(); i++) {
        createBuffer(bufferSize, VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, uniformBuffers[i], uniformBuffersMemory[i]);
    }
}
```
我们将编写一个单独的函数来更新统一缓冲区，每帧都有一个新的变换，因此这里不会有vkMapMemory。统一数据将将用于所有绘制调用，因此包含它的缓冲区会在最后才进行销毁：
```cpp
void cleanup() {
    cleanupSwapChain();

    vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);

    for (size_t i = 0; i < swapChainImages.size(); i++) {
        vkDestroyBuffer(device, uniformBuffers[i], nullptr);
        vkFreeMemory(device, uniformBuffersMemory[i], nullptr);
    }

    ...
}
```

<b>更新统一数据</b>
在我们知道要获取哪个交换链图像后，立即创建一个新函数updateUniformBuffer并在drawFrame函数里添加对它的调用：
```cpp
void drawFrame() {
    ...

    uint32_t imageIndex;
    VkResult result = vkAcquireNextImageKHR(device, swapChain, std::numeric_limits<uint64_t>::max(), imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

    ...

    updateUniformBuffer(imageIndex);

    VkSubmitInfo submitInfo = {};
    submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

    ...
}

...

void updateUniformBuffer(uint32_t currentImage) {

}
```
此函数将每帧生成一个新变换，来使几何体旋转。我们需要包含两个新的头文件来实现此功能：
```cpp
#define GLM_FORCE_RADIANS
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>

#include <chrono>
```
glm/gtc/matrix_transform.hpp拥有可用于生成模型转换的函数，如glm::rotate，查看转换的函数，如glm::lookAt，以及投影变换函数，比如glm::perspective。GLM_FORCE_RADIANS的定义是必要的，这可以确保像glm::rotate这样的函数会使用弧度作为参数，用以避免任何可能的混淆。

chrono标准库拥有执行精确计时的功能。我们将使用它来确保几何体每秒旋转90度，而不管帧速率如何。
```cpp
void updateUniformBuffer(uint32_t currentImage) {
    static auto startTime = std::chrono::high_resolution_clock::now();

    auto currentTime = std::chrono::high_resolution_clock::now();
    float time = std::chrono::duration<float, std::chrono::seconds::period>(currentTime - startTime).count();
}
```
updateUniformBuffer函数将以一些逻辑开始以浮点精度来计算自渲染开始以来的经过的秒数。

我们现在将在统一缓冲区对象中定义模型，视图和投影变换。模型的旋转将使用时间变量围绕Z轴进行简单旋转：
```cpp
UniformBufferObject ubo = {};
ubo.model = glm::rotate(glm::mat4(1.0f), time * glm::radians(90.0f), glm::vec3(0.0f, 0.0f, 1.0f));
```
glm::rotate函数将现有变换矩阵，旋转角度和旋转轴作为参数。glm::mat4(1.0f)构造函数返回一个单位矩阵。使用旋转角度time*glm::radians(90.0f)可以实现每秒90度旋转的目的。
```cpp
ubo.view = glm::lookAt(glm::vec3(2.0f, 2.0f, 2.0f), glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(0.0f, 0.0f, 1.0f));
```
对于视图变换，我决定以45度角从上面来观察。glm::lookAt函数将眼睛位置，中心位置和向上轴作为参数。
```cpp
ubo.proj = glm::perspective(glm::radians(45.0f), swapChainExtent.width / (float) swapChainExtent.height, 0.1f, 10.0f);
```
我选择使用具有45度垂直视场角（FOV）的透视投影。其他参数分别是宽高比，近裁剪面和远裁剪面。使用当前交换链的范围来计算宽高比非常重要，以便在调整大小后考虑窗口的新宽度和高度。
```cpp
ubo.proj[1][1] *= -1;
```
GLM最初是为OpenGL设计的，其中裁剪坐标的Y坐标是反转的。最简单的修正方法是在投影矩阵中翻转Y轴缩放因子的符号。如果你不这样做，那么图像将被颠倒渲染。

现在定义了所有转换，因此我们可以将统一缓冲区对象中的数据复制到当前的统一缓冲区。这与我们对顶点缓冲区的操作完全相同，除了没有暂存缓冲区：
```cpp
void* data;
vkMapMemory(device, uniformBuffersMemory[currentImage], 0, sizeof(ubo), 0, &data);
    memcpy(data, &ubo, sizeof(ubo));
vkUnmapMemory(device, uniformBuffersMemory[currentImage]);
```
使用UBO这种方式不是将频繁更改的值传递给着色器的最有效方法。将小数据缓冲区传递给着色器的更有效方法是推式常量（push constants）。我们可以在以后的章节中看到。

在下一章中，我们将讨论描述符集，它实际上将VkBuffers绑定到统一缓冲区描述符，以便着色器可以访问相关的转换数据（转换矩阵等）。



























































































































































