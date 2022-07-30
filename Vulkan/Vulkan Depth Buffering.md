Depth Buffering实际上就是生成一张深度图，所以需要定义以下成员变量，与其他图片类似：

```cpp
VkImage mDepthImage;
VkDeviceMemory mDepthImageMemory;
VkImageView mDepthImageView;
```

在生成图片之前，需要完成以下helper函数：

```cpp
VkFormat Application::findSupportedFormat(const std::vector<VkFormat>& candidates, VkImageTiling tiling, VkFormatFeatureFlags features)
{
    for (VkFormat format : candidates)
    {
        VkFormatProperties formatProperties;
        vkGetPhysicalDeviceFormatProperties(mPhysicalDevice, format, &formatProperties);

        if (tiling == VK_IMAGE_TILING_LINEAR && (formatProperties.linearTilingFeatures & features) == features)
        {
            return format;
        }
        else if (tiling == VK_IMAGE_TILING_OPTIMAL && (formatProperties.optimalTilingFeatures & features) == features)
        {
            return format;
        }
    }

    throw std::runtime_error("Failed to find supported format!");
}

VkFormat Application::findDepthFormat()
{
    return findSupportedFormat({VK_FORMAT_D32_SFLOAT, VK_FORMAT_D32_SFLOAT_S8_UINT, VK_FORMAT_D24_UNORM_S8_UINT}, VK_IMAGE_TILING_OPTIMAL, VK_FORMAT_FEATURE_DEPTH_STENCIL_ATTACHMENT_BIT);
}

bool Application::hasStencilComponent(VkFormat format)
{
    return format == VK_FORMAT_D32_SFLOAT_S8_UINT || format == VK_FORMAT_D24_UNORM_S8_UINT;
}
```

接着就可以完成深度图的创建：

```cpp
void Application::createDepthResources()
{
    VkFormat depthFormat = findDepthFormat();
    createImage(
        mSwapchainExtent.width,
        mSwapchainExtent.height,
        depthFormat,
        VK_IMAGE_TILING_OPTIMAL,
        VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT,
        VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT,
        mDepthImage,
        mDepthImageMemory);
    mDepthImageView = createImageView(mDepthImage, depthFormat, VK_IMAGE_ASPECT_DEPTH_BIT);

    transitionImageLayout(mDepthImage, depthFormat, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL);
}
```

最后一行也可以不需要，如果加这一行的话需要改一下`transitionImageLayout`函数。

接着需要对Render Pass / Framebuffer / CommandBuffer / GraphicsPipeline做一并的修改，使其支持Depth Buffering。

由于与GraphicsPipeline相关，所以Depth Resources的创建和清楚都要和swapchain的重新创建和清除联系起来，否则无法支持应用修改窗口大小。

需要注意Depth Resources必须要Command Pool之后、Framebuffer之前创建。
