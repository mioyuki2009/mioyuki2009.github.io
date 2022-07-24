---
title: 学习一个vulkan(18)-顶点缓冲区-顶点输入描述
date: 2018-11-17 09:50:52
tags: vulkan
categories: computer graphic
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
<b>介绍</b>
在接下来的几章中，我们将使用内存中的顶点缓冲区替换顶点着色器中的硬编码顶点数据。我们将从创建CPU可见的缓冲区并使用memcpy直接将顶点数据复制到其中这种最简单的方法开始，之后我们将看到如何使用临时缓冲区将顶点数据复制到高性能内存中。

<b>顶点着色器</b>
首先将顶点着色器更改为不在着色器代码本身中包含顶点数据。顶点着色器使用in关键字从顶点缓冲区获取输入：
```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

out gl_PerVertex {
    vec4 gl_Position;
};

void main() {
    gl_Position = vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```
inPosition和inColor变量称为顶点属性（vertex attributes）。它们是在顶点缓冲区中按顶点指定的相应属性，就和之前我们手动分配数组来指定每个顶点的位置和颜色一样。现在重新编译顶点着色器。

就像fragColor一样，layout(location = x) 告诉了我们以后引用他们时使用的索引。了解其中的某些类型是很重要的，如dvec3 是64位向量，使用了多个槽（slots）。这意味着它之后的索引必须至少高于2：
```glsl
layout(location = 0) in dvec3 inPosition;
layout(location = 2) in vec3 inColor;
```
您可以在[OpenGL wiki](https://www.khronos.org/opengl/wiki/Layout_Qualifier_(GLSL))中找到有关布局限定符的更多信息。

<b>顶点数据</b>
我们将顶点数据的代码从着色器移动到我们程序代码的数组中。首先引入GLM库，它为我们提供了线性代数相关类型，如矢量和矩阵，我们将使用这些类型来指定位置和颜色向量。

创建一个名为Vertex的新结构，其中包含我们将在顶点着色器中使用的两个属性：
```cpp
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;
};
```
GLM为我们提供了与着色器语言中使用的矢量类型完全匹配的C++类型。
```cpp
const std::vector<Vertex> vertices = {
    {{0.0f, -0.5f}, {1.0f, 0.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 1.0f, 0.0f}},
    {{-0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}}
};
```
现在使用Vertex结构指定顶点数据数组。我们使用与以前完全相同的位置和颜色值，但现在它们被组合成一个顶点数组，这称为交错顶点属性（interleaving vertex attributes）。

<b>绑定相关描述</b>
下一步是告诉Vulkan在将数据格式上传到GPU内存后如何将此数据格式传递给顶点着色器。传达此信息需要两种类型的结构。

第一个结构是VkVertexInputBindingDescription，我们将向Vertex结构添加一个成员函数，然后用正确的数据填充它。
```cpp
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;

    static VkVertexInputBindingDescription getBindingDescription() {
        VkVertexInputBindingDescription bindingDescription = {};

        return bindingDescription;
    }
};
```
顶点绑定描述了在整个过程中顶点从内存加载数据的速率。它指定了数据条目之间的字节数以及是在每个顶点之后还是在每个实例之后移动到下一个数据条目。
```cpp
VkVertexInputBindingDescription bindingDescription = {};
bindingDescription.binding = 0;
bindingDescription.stride = sizeof(Vertex);
bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
```
我们所有的顶点数据都被打包在一个数组中，所以我们只需要一个绑定。binding参数指定绑定数组中绑定的索引。stride参数指定从一个条目到下一个条目的字节数，inputRate参数可以具有以下值之一：
* VK_VERTEX_INPUT_RATE_VERTEX：在每个顶点之后移动到下一个数据条目
* VK_VERTEX_INPUT_RATE_INSTANCE：在每个实例之后移动到下一个数据条目

我们不打算使用实例渲染，因此我们这里使用逐顶点数据。

<b>属性描述</b>
描述如何处理顶点输入的第二个结构是VkVertexInputAttributeDescription。我们将为Vertex添加另一个辅助函数来填充这个结构：
```cpp
#include <array>

...

static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions() {
    std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions = {};

    return attributeDescriptions;
}
```
正如函数原型所示，将会有两个这样的结构。属性描述结构描述了如何从源自绑定描述的顶点数据块中提取顶点属性。我们有两个属性，位置和颜色，所以我们需要两个属性描述结构。
```cpp
attributeDescriptions[0].binding = 0;
attributeDescriptions[0].location = 0;
attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
attributeDescriptions[0].offset = offsetof(Vertex, pos);
```
绑定参数告诉Vulkan每个顶点数据的绑定来源。location参数引用顶点着色器中输入的location指令。在我们的顶点着色器中，location为0代表的就是位置（position）了，它是32bit单精度数据。

format参数描述了属性的数据类型。有点令人困惑的是，这个格式是使用与颜色格式相同的枚举来指定的。下列的着色器类型和格式是比较常用的搭配：
* float: VK_FORMAT_R32_SFLOAT
* vec2: VK_FORMAT_R32G32_SFLOAT
* vec3: VK_FORMAT_R32G32B32_SFLOAT
* vec4: VK_FORMAT_R32G32B32A32_SFLOAT

如您所见，您应该使用颜色通道数量与着色器数据类型中的组件数量匹配的格式。虽然也可以使用比着色器中的组件数量更多的通道，但它们将会被静默丢弃。如果通道数低于组件数，则BGA组件将会使用默认值（0,0,1）。颜色的类型（SFLOAT，UINT，SINT）和位宽也应与着色器输入的类型匹配。请参阅以下示例：
* ivec2: VK_FORMAT_R32G32_SINT，一个32位有符号整数的双分量向量
* uvec4: VK_FORMAT_R32G32B32A32_UINT，一个32位无符号整数的4分量向量
* double: VK_FORMAT_R64_SFLOAT，双精度（64位）浮点数

format参数隐式定义了属性数据的字节大小，offset参数指定了每个顶点数据读取的字节宽度偏移量。绑定一次就会加载一个Vertex，position属性(pos)的偏移量在字节数据中为0字节，这是使用offsetof宏自动计算的。
```cpp
attributeDescriptions[1].binding = 0;
attributeDescriptions[1].location = 1;
attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
attributeDescriptions[1].offset = offsetof(Vertex, color);
```
color属性的描述方式也大致相同。

<b>管道顶点输入</b>
我们现在需要通过引用createGraphicsPipeline中的结构来设置图形管道来接受这个格式的顶点数据。修改vertexInputInfo结构以引用这两个描述：
```cpp
auto bindingDescription = Vertex::getBindingDescription();
auto attributeDescriptions = Vertex::getAttributeDescriptions();

vertexInputInfo.vertexBindingDescriptionCount = 1;
vertexInputInfo.vertexAttributeDescriptionCount = static_cast<uint32_t>(attributeDescriptions.size());
vertexInputInfo.pVertexBindingDescriptions = &bindingDescription;
vertexInputInfo.pVertexAttributeDescriptions = attributeDescriptions.data();
```
现在，管道已准备好接受顶点容器格式的顶点数据并将其传递给顶点着色器了。如果现在在启用验证层的情况下运行程序，您将看到没有绑定的顶点缓冲区的提示。下一步是创建顶点缓冲区并将顶点数据移动到顶点缓冲区，以便GPU能够访问它。

























































































































































