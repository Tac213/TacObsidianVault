在创建Graphics pipeline之前，还需要对用于渲染的frame buffer attachment进行配置。需要指定使用的颜色和深度缓冲，以及采样数，渲染操作如何处理frame buffer的内容。所有的这些信息被称为Render Pass。

Render Pass应被声明为一个成员函数：

```cpp
VkRenderPass mRenderPass;
```

创建：

```cpp
void Application::createRenderPass()
{
    VkAttachmentDescription colorAttachment {};
    colorAttachment.format = mSwapChainImageFormat;
    colorAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
    colorAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
    colorAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
    colorAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    colorAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    colorAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    colorAttachment.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;

    VkAttachmentReference colorAttachmentRef {};
    colorAttachmentRef.attachment = 0;
    colorAttachmentRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;

    VkSubpassDescription subpass {};
    subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
    subpass.colorAttachmentCount = 1;
    subpass.pColorAttachments = &colorAttachmentRef;

    VkRenderPassCreateInfo createInfo {};
    createInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
    createInfo.attachmentCount = 1;
    createInfo.pAttachments = &colorAttachment;
    createInfo.subpassCount = 1;
    createInfo.pSubpasses = &subpass;

    if (vkCreateRenderPass(mLogicalDevice, &createInfo, nullptr, &mRenderPass) != VK_SUCCESS)
    {
        std::cerr << "Failed to create render pass" << std::endl;
        mbQuit = true;
    }
}
```

销毁：

```cpp
vkDestroyRenderPass(mLogicalDevice, mRenderPass, nullptr);
```
