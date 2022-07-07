在使用Render Pass对象时，Render Pass对应的Color Attachment对象要被绑定到framebuffer对象上使用。需要为Swapchain中的每个ImageView创建framebuffer，并在渲染时渲染到对应的framebuffer上。

因此首先应该声明一个成员变量来储存所有framebuffer对象：

```cpp
std::vector<VkFramebuffer> mSwapchainFramebuffers;
```

创建代码：

```cpp
void Application::createCommandPool()
{
    QueueFamilyIndices queueFamilyIndices = findQueueFamilyIndices(mPhysicalDevice);
    VkCommandPoolCreateInfo poolInfo {};
    poolInfo.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
    poolInfo.flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT;
    poolInfo.queueFamilyIndex = queueFamilyIndices.graphicsFamily.value();
    if (vkCreateCommandPool(mLogicalDevice, &poolInfo, nullptr, &mCommandPool) != VK_SUCCESS)
    {
        std::cerr << "Failed to create command pool!" << std::endl;
        mbQuit = true;
    }
}
```

当然也需要销毁这些对象：

```cpp
for (VkFramebuffer framebuffer : mSwapchainFramebuffers)
{
    vkDestroyFramebuffer(mLogicalDevice, framebuffer, nullptr);
}
```

## Command Buffer

Vulkan的绘制指令、内存传输指令并不是直接通过函数调用执行的，需要将所有要执行的操作记录在一个Command Buffer对象上，然后提交给可以执行这些操作的队列才能执行指令。

### Command Pool

在创建CommandBuffer之前需要创建Command Pool，用于管理Command Buffer对象使用的内存，并负责Command Buffer对象的分配。

CommandPool也是定义为成员变量：

```cpp
VkCommandPool mCommandPool;
```

创建Command Pool时，可以指定一些flag，有如下flag可以使用：

- VK_COMMAND_POOL_CREATE_TRANSITENT_BIT: 使用它分配的Command Buffer对象被频繁用来记录新的指令(使用这一标记可能会改变Framebuffer对象的内存分配策略)
- VK_COMMAND_POLL_CREATE_RESET_COMMAND_BUFFER_BIT: Command Buffer对象之间相互独立，不会被一起重置。不使用这一标记，Command Buffer对象会被放在一起重置

```cpp
void Application::createCommandPool()
{
    QueueFamilyIndices queueFamilyIndices = findQueueFamilyIndices(mPhysicalDevice);
    VkCommandPoolCreateInfo poolInfo {};
    poolInfo.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
    poolInfo.flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT;
    poolInfo.queueFamilyIndex = queueFamilyIndices.graphicsFamily.value();
    if (vkCreateCommandPool(mLogicalDevice, &poolInfo, nullptr, &mCommandPool) != VK_SUCCESS)
    {
        std::cerr << "Failed to create command pool!" << std::endl;
        mbQuit = true;
    }
}
```

销毁Command Pool:

```cpp
vkDestroyCommandPool(mLogicalDevice, mCommandPool, nullptr);
```

### Command Buffer Allocation

Command Buffer用于记录绘制指令。

Command Buffer也是定义为成员变量：

```cpp
VkCommandBuffer mCommandBuffer;
```

在创建时，level成员变量用于指定指定分配的Command Buffer对象时主要对象还是辅助对象，辅助对象可以存储一些常用的指令，然后在主要对象中调用执行：

- VK_COMMAND_BUFFER_LEVEL_PRIMARY: 可以被提交到队列进行执行，但不能被其它Command Buffer对象调用
- VK_COMMAND_BUFFER_LEVEL_SECONDARY: 不能直接被提交到队列进行执行，但可以被主要Command Buffer对象调用执行

```cpp
void Application::createCommandBuffer()
{
    VkCommandBufferAllocateInfo allocInfo {};
    allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
    allocInfo.commandPool = mCommandPool;
    allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    allocInfo.commandBufferCount = 1;

    if (vkAllocateCommandBuffers(mLogicalDevice, &allocInfo, &mCommandBuffer) != VK_SUCCESS)
    {
        std::cerr << "Failed to allocate command buffers!" << std::endl;
        mbQuit = true;
    }
}
```

接着，实现下面这个函数记录Command Buffer:

```cpp
void Application::recordCommandBuffer(VkCommandBuffer commandBuffer, uint32_t imageIndex)
{
    VkCommandBufferBeginInfo beginInfo {};
    beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
    beginInfo.flags = 0;
    beginInfo.pInheritanceInfo = nullptr;

    if (vkBeginCommandBuffer(commandBuffer, &beginInfo) != VK_SUCCESS)
    {
        throw std::runtime_error("Failed to begin recording command buffer!");
    }

    VkRenderPassBeginInfo renderPassInfo {};
    renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
    renderPassInfo.renderPass = mRenderPass;
    renderPassInfo.framebuffer = mSwapchainFramebuffers[imageIndex];
    renderPassInfo.renderArea.offset = {0, 0};
    renderPassInfo.renderArea.extent = mSwapChainExtent;

    VkClearValue clearColor = {{{0.0f, 0.0f, 0.0f, 1.0f}}};
    renderPassInfo.clearValueCount = 1;
    renderPassInfo.pClearValues = &clearColor;

    vkCmdBeginRenderPass(commandBuffer, &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE);
    vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, mGraphicsPipeline);
    vkCmdDraw(commandBuffer, 3, 1, 0, 0);
    vkCmdEndRenderPass(commandBuffer);

    if (vkEndCommandBuffer(commandBuffer) != VK_SUCCESS)
    {
        throw std::runtime_error("Failed to record command buffer!");
    }
}
```

`VkCommandBufferBeginInfo`可以设置如下flag:

- VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT: Command Buffer会在执行后被记录一次
- VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT: 这是一个辅助CommandBuffer对象
- VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT: Command Buffer可以被重复提交

`VkCommandBufferBeginInfo`的pInheritanceInfo字段用于设置辅助CommandBuffer的BeginInfo。

`VkRenderPassBeginInfo`对象的`clearValueCount`和`pClearValues`用于指定使用`VK_ATTACHMENT_LOAD_OP_CLEAR`标记后的清除值，在[[Vulkan Render Pass]]创建时定义的这个标记。

`vkCmdBegineRenderPass`函数的最后一个参数可以传2种不同的flag:

- VK_SUBPASS_CONTENTS_INLINE: 所有要执行的指令都在主要指令缓冲中，没有辅助指令缓冲需要执行
- VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS: Render Pass的指令会从辅助command buffer中执行

`vkCmdDraw`有5个参数:

- commandBuffer: Command Buffer对象
- vertextCount: 顶点数量
- instanceCount: 渲染实例数量，为1时表示不进行实例渲染
- firstVertext: 定义shader变量`gl_VertextIndex`的值
- firstInstance: 定义shader变量`gl_InstanceIndex`的值
