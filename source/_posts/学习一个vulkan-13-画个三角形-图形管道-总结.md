---
title: 学习一个vulkan(13)-画个三角形(图形管道)-总结
date: 2018-11-12 09:18:29
tags: vulkan
categories: learn
thumbnail: /images/cat.jpg
---
继续上一次的
<!-- more -->
<b>总结</b>
现在就是把之前几章创建的所有结构和对象结合起来建立图形管道了。先简单回顾一下：
* 着色器阶段（Shader stages）：着色器模块定义了图形管道的可编程阶段的功能
* 固定功能阶段（Fixed-function state）：所有定义了管道固定功能阶段的结构，如输入组件，光栅化器，视口和颜色混合
* 管道布局（Pipeline layout）：着色器引用的统一值和推送值（the uniform and push values），可以在绘制时更新
* 渲染过程（Render pass）：管道阶段引用的附件及其相关用法

以上这些一同定义了完整的图形管道的功能，因此我们现在可以在createGraphicsPipeline函数的vkDestroyShaderModule之前进行填充，因为这些东西仍然在创建过程中需要使用的：
```cpp
VkGraphicsPipelineCreateInfo pipelineInfo = {};
pipelineInfo.sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO;
pipelineInfo.stageCount = 2;
pipelineInfo.pStages = shaderStages;
```
我们首先引用VkPipelineShaderStageCreateInfo结构数组。
```cpp
pipelineInfo.pVertexInputState = &vertexInputInfo;
pipelineInfo.pInputAssemblyState = &inputAssembly;
pipelineInfo.pViewportState = &viewportState;
pipelineInfo.pRasterizationState = &rasterizer;
pipelineInfo.pMultisampleState = &multisampling;
pipelineInfo.pDepthStencilState = nullptr; // Optional
pipelineInfo.pColorBlendState = &colorBlending;
pipelineInfo.pDynamicState = nullptr; // Optional
```
然后我们引用用来描述固定功能阶段的所有结构。
```cpp
pipelineInfo.layout = pipelineLayout;
```
之后是管道布局，它是一个Vulkan句柄而不是结构指针。
```cpp
pipelineInfo.renderPass = renderPass;
pipelineInfo.subpass = 0;
```
最后，我们引用了渲染过程以及将使用此图形管道的子过程的索引。除开这个特定的实例之外也可以使用这个管道的其他渲染过程，但是它们必须与renderPass兼容。[这里](https://www.khronos.org/registry/vulkan/specs/1.0/html/vkspec.html#renderpass-compatibility)描述了兼容性要求，但我们不会在本教程中用到这个特性。
```cpp
pipelineInfo.basePipelineHandle = VK_NULL_HANDLE; // Optional
pipelineInfo.basePipelineIndex = -1; // Optional
```
实际上还有两个参数：basePipelineHandle和basePipelineIndex。Vulkan允许通过从现有管道派生来创建新的图形管道，派生管道的想法在于，当它们具有与现有管道相同的许多功能时，设置管道的成本更低，而且同一父管道的多个子管道切换速度会更快。您可以使用basePipelineHandle来指定现有管道的句柄，也可以引用通过basePipelineIndex的索引创建的另外的管道。现在只有一个管道，所以我们只需指定一个空句柄和一个无效的索引。只有在VkGraphicsPipelineCreateInfo的flags字段中指定了VK_PIPELINE_CREATE_DERIVATIVE_BIT标志时，才需要使用这些值。

现在准备最后一步，创建一个类成员保存VkPipeline对象:
```cpp
VkPipeline graphicsPipeline;
```
最后创建图形管道：
```cpp
if (vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipelineInfo, nullptr, &graphicsPipeline) != VK_SUCCESS) {
    throw std::runtime_error("failed to create graphics pipeline!");
}
```
vkCreateGraphicsPipelines函数比Vulkan中通常的对象创建函数多几个参数，这旨在获取多个VkGraphicsPipelineCreateInfo对象，并在一次调用中创建多个VkPipeline对象。

我们引用VK_NULL_HANDLE的第二个参数，作为可选的VkPipelineCache对象的引用。管道缓存可用于存储和重用与多个vkCreateGraphicsPipelines调用相关的管道创建相关数据，如果这些缓存存储在文件中的话，甚至可以用来跨程序使用，这使得可以在以后显着加速管道创建的过程。我们将在之后的管道缓存章节中来讨论这个问题。

所有常见的绘图操作都需要图形管道，因此它也在程序结束时才进行销毁：
```cpp
void cleanup() {
    vkDestroyPipeline(device, graphicsPipeline, nullptr);
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    ...
}
```
现在运行您的程序以确认所作的这些工作可以成功创建管道。现在距离在屏幕上看到某些东西只有一步之遥了。在接下来的几章中，我们将从交换链图像中设置实际的帧缓冲区并准备绘图命令。
















