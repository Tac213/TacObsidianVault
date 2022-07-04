Fixed Functions指的是Graphics Pipeline中不可编程的部分，尽管不可编程，但这些功能依然是可以配置的，Vulkan要求开发者根据自身需求对这些功能进行配置。

## Vertex input

描述了顶点属性如何输入到Grphics pipeline。

```cpp
VkPipelineVertexInputStateCreateInfo vertexInputInfo {};
vertexInputInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO;
vertexInputInfo.vertexBindingDescriptionCount = 0;
vertexInputInfo.pVertexBindingDescriptions = nullptr;
vertexInputInfo.vertexAttributeDescriptionCount = 0;
vertexInputInfo.pVertexAttributeDescriptions = nullptr;
```

## Input Assembly

描述了需要根据顶点数据画什么样的几何图形

-   `VK_PRIMITIVE_TOPOLOGY_POINT_LIST`: points from vertices只画点
-   `VK_PRIMITIVE_TOPOLOGY_LINE_LIST`: line from every 2 vertices without reuse每两个顶点构成一个线段
-   `VK_PRIMITIVE_TOPOLOGY_LINE_STRIP`: the end vertex of every line is used as start vertex for the next line每两个顶点构成一个线段，除了第一个线段以外，每个线段使用上一个线段的末尾顶点作为起点
-   `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST`: triangle from every 3 vertices without reuse每三个顶点构成一个三角形
-   `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_STRIP`: the second and third vertex of every triangle are used as first two vertices of the next triangle每三个顶点构成一个三角形，每个三角形的第二和第三个顶点会被作为下一个三角形的第一和第二个顶点

```cpp
VkPipelineInputAssemblyStateCreateInfo inputAssembly {};
inputAssembly.sType = VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO;
inputAssembly.topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST;
inputAssembly.primitiveRestartEnable = VK_FALSE;
```

## Viewports and scissors

viewport用于描述被用来输出渲染结果的帧缓冲区域，scissor定义了哪一区域的像素实际被存储在帧缓存。下面这个图形象说明了viewport和scissor的关系:

![Viewports and scissors](https://vulkan-tutorial.com/images/viewports_scissors.png)

可以简单理解为scissor用于裁剪viewport。

有的显卡支持多个viewport和多个scissor，如果需要使用该功能则需要在选择物理设备时进行筛选，并在Logical Device创建时启用这些功能。

```cpp
VkViewport viewport {};
viewport.x = 0.0f;
viewport.y = 0.0f;
viewport.width = static_cast<float>(mSwapChainExtent.width);
viewport.height = static_cast<float>(mSwapChainExtent.height);
viewport.minDepth = 0.0f;
viewport.maxDepth = 1.0f;

VkRect2D scissor {};
scissor.offset = {0, 0};
scissor.extent = mSwapChainExtent;

VkPipelineViewportStateCreateInfo viewportState {};
viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
viewportState.viewportCount = 1;
viewportState.pViewports = &viewport;
viewportState.scissorCount = 1;
viewportState.pScissors = &scissor;
```

## Rasterize

光栅化是一个核心的过程，有很多参数可以配置。

- depthClampEnable: 设置为`VK_TRUE`表示在近平面和远平面外的片段会被截断为在近平面和远平面上，而不是直接丢弃这些片段
- rasterizerDiscardEnable: 设置为`VK_TRUE`表示所有几何图形都不能通过光栅化阶段
- polygonMode: 定义多边形产生fragment的方式，除了第一种，其他的都要启用GPU的特性
    - VK_POLYGON_MODE_FILL: 整个多边形，包括多边形内部都产生Fragment
    - VK_POLYGON_MODE_LINE: 只有多边形的边会产生Fragment
    - VK_POLYGON_MODE_POINT: 只有多边形的顶点会产生Fragment
- lineWidth: 光栅化后线段的宽度，最大值依赖于硬件，使用大于1.0f的线段宽度需要启用GPU的特性
- cullMode: 指定使用表面的剔除类型，可以剔除背面、正面、双面
- frontFace: 制定顺时针的顶点序是正面，还是逆时针的顶点序是正面
- depthBiasXXX: 定义shadow mapping时的bias

```cpp
VkPipelineRasterizationStateCreateInfo rasterizationState {};
rasterizationState.sType = VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO;
rasterizationState.depthClampEnable = VK_FALSE;
rasterizationState.rasterizerDiscardEnable = VK_FALSE;
rasterizationState.polygonMode = VK_POLYGON_MODE_FILL;
rasterizationState.lineWidth = 1.0f;
rasterizationState.cullMode = VK_CULL_MODE_BACK_BIT;
rasterizationState.frontFace = VK_FRONT_FACE_CLOCKWISE;
rasterizationState.depthBiasEnable = VK_FALSE;
rasterizationState.depthBiasConstantFactor = 0.0f;
rasterizationState.depthBiasClamp = 0.0f;
rasterizationState.depthBiasSlopeFactor = 0.0f;
```

## Multisampling

Multisampling是一种组合多个不同多边形产生的片段的颜色来决定最终的像素颜色的技术，它可以一定程度上减少多边形边缘的走样现象。

使用Multisampling需要启用GPU的特性。

```cpp
VkPipelineMultisampleStateCreateInfo multisampleState {};
multisampleState.sType = VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO;
multisampleState.sampleShadingEnable = VK_FALSE;
multisampleState.rasterizationSamples = VK_SAMPLE_COUNT_1_BIT;
multisampleState.minSampleShading = 1.0f;
multisampleState.pSampleMask = nullptr;
multisampleState.alphaToCoverageEnable = VK_FALSE;
multisampleState.alphaToOneEnable = VK_FALSE;
```

## Depth and stencil testing

## Color blending

Fragment shader返回的fragment颜色需要和原来帧缓冲中对应像素的颜色进行混合，这个过程称为Color blending。Color blending通常有2种方式：

- 混合旧值和新值产生最终的颜色
- 使用位运算组合旧值和新值

`VkPipelineColorBlendAttachmentState`结构体定义第一种方式，`VkPipelineColorBlendStateCreateInfo`结构体定义第二种方式。

第一类混合方式的运算过程类似下面的代码：

```cpp
if (blendEnable)
{
    finalColor.rgb = (srcColorBlendFactor * newColor.rgb) <colorBlendOp> (dstColorBlendFactor * oldColor.rgb);
    finalColor.a = (srcAlphaBlendFactor * newColor.a) <colorBlendOp> (dstAlphaBlendFactor * oldColor.a);
}
else
{
    finalColor = newColor;
}
```

如果用位运算的混合方式，第一种方式就会被禁用。

```cpp
VkPipelineColorBlendAttachmentState colorBlendAttachmentState {};
colorBlendAttachmentState.colorWriteMask = VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_G_BIT | VK_COLOR_COMPONENT_B_BIT | VK_COLOR_COMPONENT_A_BIT;
colorBlendAttachmentState.blendEnable = VK_FALSE;
colorBlendAttachmentState.srcColorBlendFactor = VK_BLEND_FACTOR_ONE;
colorBlendAttachmentState.dstColorBlendFactor = VK_BLEND_FACTOR_ZERO;
colorBlendAttachmentState.colorBlendOp = VK_BLEND_OP_ADD;
colorBlendAttachmentState.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE;
colorBlendAttachmentState.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO;
colorBlendAttachmentState.alphaBlendOp = VK_BLEND_OP_ADD;

VkPipelineColorBlendStateCreateInfo colorBlendState {};
colorBlendState.sType = VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO;
colorBlendState.logicOpEnable = VK_FALSE;
colorBlendState.logicOp = VK_LOGIC_OP_COPY;
colorBlendState.attachmentCount = 1;
colorBlendState.pAttachments = &colorBlendAttachmentState;
colorBlendState.blendConstants[0] = 0.0f;
colorBlendState.blendConstants[1] = 0.0f;
colorBlendState.blendConstants[2] = 0.0f;
colorBlendState.blendConstants[3] = 0.0f;
```

## Dynamic state

上面有些Fixed Functions的配置是可以在Graphics pipeline不重新创建的前提下动态修改的，比如Viewport大小、lineWidth、blend factor这些。如果需要动态修改，则需要定义：

```cpp
std::vector<VkDynamicState> dynamicStates = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_LINE_WIDTH
};

VkPipelineDynamicStateCreateInfo dynamicState{};
dynamicState.sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO;
dynamicState.dynamicStateCount = static_cast<uint32_t>(dynamicStates.size());
dynamicState.pDynamicStates = dynamicStates.data();
```

这样设置后会导致我们之前对这里使用的动态状态的设置被忽略掉，需要我们在进行绘制时重新指定它们的值。

## Pipeline layout

在Shader中使用的uniform变量需要在Graphics pipeline创建时通过`VkPipelineLayout`对象进行定义。

这个对象应声明为一个成员变量：

```cpp
VkPipelineLayout mPipelineLayout;
```

创建：

```cpp
VkPipelineLayoutCreateInfo pipelineLayoutInfo {};
    pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
    pipelineLayoutInfo.setLayoutCount = 0;
    pipelineLayoutInfo.pSetLayouts = nullptr;
    pipelineLayoutInfo.pushConstantRangeCount = 0;
    pipelineLayoutInfo.pPushConstantRanges = nullptr;

    if (vkCreatePipelineLayout(mLogicalDevice, &pipelineLayoutInfo, nullptr, &mPipelineLayout) != VK_SUCCESS)
    {
        std::cerr << "Failed to create pipeline layout" << std::endl;
        mbQuit = true;
        return;
    }
```

同时也要记得销毁：

```cpp
vkDestroyPipelineLayout(mLogicalDevice, mPipelineLayout, nullptr);
```

## Graphics pipeline

有了以上这些，再加上render pass，就可以开始创建Graphics pipeline了，首先还是声明一个成员函数：

```cpp
VkPipeline mGraphicsPipeline;
```

然后再创建信息的结构体中引用上面这些对象：

```cpp
VkGraphicsPipelineCreateInfo pipelineInfo {};
pipelineInfo.sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO;
pipelineInfo.stageCount = 2;
pipelineInfo.pStages = shaderStageInfos;
pipelineInfo.pVertexInputState = &vertexInputInfo;
pipelineInfo.pInputAssemblyState = &inputAssembly;
pipelineInfo.pViewportState = &viewportState;
pipelineInfo.pRasterizationState = &rasterizationState;
pipelineInfo.pMultisampleState = &multisampleState;
pipelineInfo.pDepthStencilState = nullptr;
pipelineInfo.pColorBlendState = &colorBlendState;
pipelineInfo.pDynamicState = nullptr;

pipelineInfo.layout = mPipelineLayout;

pipelineInfo.renderPass = mRenderPass;
pipelineInfo.subpass = 0;
```

另外还有2个字段，basePipelineHandle和basePipelineIndex，用于以一个创建好的图形管线为基础创建一个新的Graphics Pipeline图形管线。当要创建一个和已有管线大量设置相同的管线时，使用它的代价要比直接创建小，并且，对于从同一个管线衍生出的两个管线，在它们之间进行管线切换操作的效率也要高很多。可以用basePipelineHandle来指定已经创建好的管线，或是用basePipelineIndex来指定将要创建的管线作为基础管线，用于衍生新的管线。

设置好字段之后就可以创建对象：

```cpp
if (vkCreateGraphicsPipelines(mLogicalDevice, VK_NULL_HANDLE, 1, &pipelineInfo, nullptr, &mGraphicsPipeline) != VK_SUCCESS)
{
    std::cerr << "Failed to create graphics pipeline" << std::endl;
    mbQuit = true;
    return;
}
```

当然还要记得销毁：

```cpp
vkDestroyPipeline(mLogicalDevice, mGraphicsPipeline, nullptr);
```
