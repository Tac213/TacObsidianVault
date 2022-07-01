Swapchain用于缓冲渲染操作，不存在默认的Swapchain，所以必须显示创建。Swapchain本质上是一个包含了若干个等待呈现的图像的队列。

首先需要在Logical Device中启用`VK_KHR_swapchain`:

```cpp
const std::vector<const char*> Application::PHYSICAL_DEVICE_EXTENSIONS = {
    VK_KHR_SWAPCHAIN_EXTENSION_NAME,
#ifdef OS_MACOS
    "VK_KHR_portability_subset"
#endif
};
```

然后新增一个`checkPhysicalDeviceExtensionSupport`函数，检查物理设备是否支持这些拓展：

```cpp
bool Application::checkPhysicalDeviceSupport(VkPhysicalDevice device)
{
    uint32_t extensionCount;
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, nullptr);

    std::vector<VkExtensionProperties> availableExtensions(extensionCount);
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, availableExtensions.data());

    std::set<std::string> requiredExtensions(PHYSICAL_DEVICE_EXTENSIONS.begin(), PHYSICAL_DEVICE_EXTENSIONS.end());
    for (const auto& extension : availableExtensions)
    {
        requiredExtensions.erase(extension.extensionName);
    }

    return requiredExtensions.empty();
}
```

同时在创建Logical Device启用这些拓展：

```cpp
createInfo.enabledExtensionCount = static_cast<uint32_t>(PHYSICAL_DEVICE_EXTENSIONS.size());
createInfo.ppEnabledExtensionNames = PHYSICAL_DEVICE_EXTENSIONS.data();
```

Swapchain有可能跟Window Surface不兼容，需要查询Swapchain支持的细节。与`QueueFamilyIndicies`类似，增加一个`SwapChainSupportDetails`作为查询函数的返回值：

```cpp
#pragma once

#include "vulkan/vulkan.h"
#include <vector>

namespace LearnVulkan
{
    struct SwapChainSupportDetails
    {
        VkSurfaceCapabilitiesKHR capabilities;
        std::vector<VkSurfaceFormatKHR> formats;
        std::vector<VkPresentModeKHR> presentModes;
    };
}  // namespace LearnVulkan
```

接着实现`querySwapChainSupport`:

```cpp
SwapChainSupportDetails Application::querySwapChainSupport(VkPhysicalDevice device)
{
    SwapChainSupportDetails details;

    vkGetPhysicalDeviceSurfaceCapabilitiesKHR(device, mWindowSurface, &details.capabilities);

    uint32_t formatCount;
    vkGetPhysicalDeviceSurfaceFormatsKHR(device, mWindowSurface, &formatCount, nullptr);
    if (formatCount != 0)
    {
        details.formats.resize(formatCount);
        vkGetPhysicalDeviceSurfaceFormatsKHR(device, mWindowSurface, &formatCount, details.formats.data());
    }

    uint32_t presentModeCount;
    vkGetPhysicalDeviceSurfacePresentModesKHR(device, mWindowSurface, &presentModeCount, nullptr);
    if (presentModeCount != 0)
    {
        details.presentModes.resize(presentModeCount);
        vkGetPhysicalDeviceSurfacePresentModesKHR(device, mWindowSurface, &presentModeCount, details.presentModes.data());
    }

    return details;
}
```

实现完成后，修改物理设备评分函数：

```cpp
QueueFamilyIndices indices = findQueueFamilyIndices(device);
bool bExtensionsSupported = checkPhysicalDeviceSupport(device);
bool bSwapChainAdequate = false;
if (bExtensionsSupported)
{
    SwapChainSupportDetails swapChainSupportDetails = querySwapChainSupport(device);
    bSwapChainAdequate = !swapChainSupportDetails.formats.empty() && !swapChainSupportDetails.presentModes.empty();
}
if (!indices.isComplete() || !bExtensionsSupported || !bSwapChainAdequate)
{
    return 0;
}
```

这样就能保证Swapchain和Window Surface兼容。

接着，为了创建Swapchain，还需要实现3个函数：

- chooseSwapSurfaceFormat: 选择合适的Surface Format, 通常用format为`VK_FORMAT_B8G8R8A8_SRGB`和colorSpace为`VK_COLOR_SPACE_SRGB_NONLINEAR_KHR`的即可，如果没有符合这个条件的也可以对所有可用的进行打分，选择分数最高的，不过通常来说默认取第一个就行了，而且通常都会满足上面的这个条件
- chooseSwapPresentMode: 选择合适的Present Mode，有以下4种Present Mode，第2种时肯定能用的，第4种是最好的
    - VK_PRESENT_MODE_IMMEDIATE_KHR: 应用程序提交的图像会被立即传输到屏幕上，可能会导致撕裂现象。
    - VK_PRESENT_MODE_FIFO_KHR: 交换链变成一个先进先出的队列，每次从队列头部取出一张图像进行显示，应用程序渲染的图像提交给交换链后，会被放在队列尾部。当队列为满时，应用程序需要进行等待。这一模式非常类似现在常用的垂直同步。刷新显示的时刻也被叫做垂直回扫。
    - VK_PRESENT_MODE_FIFO_RELAXED_KHR: 这一模式和上一模式的唯一区别是，如果应用程序延迟，导致交换链的队列在上一次垂直回扫时为空，那么，如果应用程序在下一次垂直回扫前提交图像， 图像会立即被显示。这一模式可能会导致撕裂现象。
    - VK_PRESENT_MODE_MAILBOX_KHR: 这一模式是第二种模式的 另一个变种。它不会在交换链的队列满时阻塞应用程序，队列中的图 像会被直接替换为应用程序新提交的图像。这一模式可以用来实现三 倍缓冲，避免撕裂现象的同时减小了延迟问题。
- chooseSwapExtent: Swapchain种图像的分辨率，几乎总是与要显示图像的窗口的分辨率相同


```cpp
VkPresentModeKHR Application::chooseSwapPresentMode(const std::vector<VkPresentModeKHR>& availablePresentModes)
{
    for (const auto& availablePresentMode : availablePresentModes)
    {
        if (availablePresentMode == VK_PRESENT_MODE_MAILBOX_KHR)
        {
            return availablePresentMode;
        }
    }
    return VK_PRESENT_MODE_FIFO_KHR;
}

VkExtent2D Application::chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities)
{
    if (capabilities.currentExtent.width != std::numeric_limits<uint32_t>::max())
    {
        return capabilities.currentExtent;
    }
    int width, height;
    glfwGetFramebufferSize(mWindow, &width, &height);
    VkExtent2D actualExtent = {
        static_cast<uint32_t>(width),
        static_cast<uint32_t>(height)};
    actualExtent.width = std::clamp(actualExtent.width, capabilities.minImageExtent.width, capabilities.maxImageExtent.width);
    actualExtent.height = std::clamp(actualExtent.height, capabilities.minImageExtent.height, capabilities.maxImageExtent.height);
    return actualExtent;
}
```

注意`std::numeric_limits`要`#include <limits>`, `std::clamp`要`#include algorithm`。

最后就可以开始创建Swapchain了，注意应当在Logical Device之后创建。而且通常还需要获取Swapchain的图像，所以需要定义：

```cpp
VkSwapchainKHR mSwapChain;
std::vector<VkImage> mSwapChainImages;
VkFormat mSwapChainImageFormat;
VkExtent2D mSwapChainExtent;
```

完整函数：

```cpp
void Application::createSwapChain()
{
    SwapChainSupportDetails swapChainSupportDetails = querySwapChainSupport(mPhysicalDevice);
    VkSurfaceFormatKHR surfaceFormat = chooseSwapSurfaceFormat(swapChainSupportDetails.formats);
    VkPresentModeKHR presentMode = chooseSwapPresentMode(swapChainSupportDetails.presentModes);
    VkExtent2D extent = chooseSwapExtent(swapChainSupportDetails.capabilities);

    uint32_t imageCount = swapChainSupportDetails.capabilities.minImageCount + 1;
    if (swapChainSupportDetails.capabilities.maxImageCount > 0 && imageCount > swapChainSupportDetails.capabilities.maxImageCount)
    {
        imageCount = swapChainSupportDetails.capabilities.maxImageCount;
    }

    VkSwapchainCreateInfoKHR createInfo {};
    createInfo.sType = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR;
    createInfo.surface = mWindowSurface;
    createInfo.minImageCount = imageCount;
    createInfo.imageFormat = surfaceFormat.format;
    createInfo.imageColorSpace = surfaceFormat.colorSpace;
    createInfo.imageExtent = extent;
    createInfo.imageArrayLayers = 1;
    createInfo.imageUsage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT;

    QueueFamilyIndices indices = findQueueFamilyIndices(mPhysicalDevice);
    uint32_t queueFamilyIndices[] = {indices.graphicsFamily.value(), indices.presentFamily.value()};
    if (indices.graphicsFamily != indices.presentFamily)
    {
        createInfo.imageSharingMode = VK_SHARING_MODE_CONCURRENT;
        createInfo.queueFamilyIndexCount = 2;
        createInfo.pQueueFamilyIndices = queueFamilyIndices;
    }
    else
    {
        createInfo.imageSharingMode = VK_SHARING_MODE_EXCLUSIVE;
        createInfo.queueFamilyIndexCount = 0;      // Optional
        createInfo.pQueueFamilyIndices = nullptr;  // Optional
    }

    createInfo.preTransform = swapChainSupportDetails.capabilities.currentTransform;
    createInfo.compositeAlpha = VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR;
    createInfo.presentMode = presentMode;
    createInfo.clipped = VK_TRUE;
    createInfo.oldSwapchain = VK_NULL_HANDLE;

    if (vkCreateSwapchainKHR(mLogicalDevice, &createInfo, nullptr, &mSwapChain) != VK_SUCCESS)
    {
        std::cerr << "Failed to create swap chain" << std::endl;
        mbQuit = true;
        return;
    }

    vkGetSwapchainImagesKHR(mLogicalDevice, mSwapChain, &imageCount, nullptr);
    mSwapChainImages.resize(imageCount);
    vkGetSwapchainImagesKHR(mLogicalDevice, mSwapChain, &imageCount, mSwapChainImages.data());

    mSwapChainImageFormat = surfaceFormat.format;
    mSwapChainExtent = extent;
}
```

注意有个`oldSwapchain`成员变量，需要指定它，因为应用程序在运行过程中Swapchain很可能会失效，比如改变窗口大小之后Swapchain需要重新创建。
