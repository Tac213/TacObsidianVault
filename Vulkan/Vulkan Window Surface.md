Vulkan是一个平台无关的API，不能直接和窗口系统交互，为了将Vulkan渲染的图像显示在窗口上，需要使用WSI(Window System Integration)拓展。比如VK_KHR_surface拓展通过`VkSurfaceKHR`对象抽象出可供Vulkan渲染的window surface。

创建Window Surface需要先声明对应的变量：

```cpp
VkSurfaceKHR mWindowSurface;
```

window surface的创建依赖于各平台的窗口系统，比如Windows系统上需要依赖于HWND和HMODULE，因此不同平台创建window surface的方法是不同的。glfw对这个过程做了封装，只需要像下面这样调用即可：

```cpp
void Application::createWindowSurface()
{
    if (glfwCreateWindowSurface(mVulkanInstance, mWindow, nullptr, &mWindowSurface) != VK_SUCCESS)
    {
        std::cerr << "Failed to create window surface!" << std::endl;
        mbQuit = true;
    }
}
```

不过实测，1.3.216.0版本的VulkanSDK，在MacOS上调用这个函数会闪退，已经把问题写到[stack overflow](https://stackoverflow.com/questions/72824168/glfwcreatewindowsurface-got-an-exception-on-macos)上，等待大佬帮忙解决。

然后，我们需要确保平台上的Physical Device能支持在Window Surface上显示图像，此时就需要拓展`QueueFamilyIndices`:

```cpp
#pragma once

#include <cstdint>
#include <optional>

namespace LearnVulkan
{
    struct QueueFamilyIndices
    {
        std::optional<uint32_t> graphicsFamily;
        std::optional<uint32_t> presentFamily;

        bool isComplete() const { return graphicsFamily.has_value() && presentFamily.has_value(); }
    };
}  // namespace LearnVulkan
```

`findQueueModuleIndices`函数修改成下面这样：

```cpp
QueueFamilyIndices Application::findQueueFamilyIndices(VkPhysicalDevice device)
{
    QueueFamilyIndices indices;

    uint32_t queueFamilyCount = 0;
    vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);

    std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
    vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());

    uint32_t i = 0;
    for (const auto& queueFamily : queueFamilies)
    {
        if (queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT)
        {
            indices.graphicsFamily = i;
        }

        VkBool32 bPresentSupport = false;
        vkGetPhysicalDeviceSurfaceSupportKHR(device, i, mWindowSurface, &bPresentSupport);
        if (bPresentSupport)
        {
            indices.presentFamily = i;
        }

        if (indices.isComplete())
        {
            break;
        }

        i++;
    }

    return indices;
}
```

不过通常`graphicsFamily`和`presentFamily`都是同一个Family队列族，因此调用这个函数的地方可以这样修改（使用set)：

```cpp
QueueFamilyIndices indices = findQueueFamilyIndices(mPhysicalDevice);

std::vector<VkDeviceQueueCreateInfo> queueCreateInfos;
std::set<uint32_t> uniqueQueueFamilies = {indices.graphicsFamily.value(), indices.presentFamily.value()};
const float QUEUE_PRIORITY = 1.0f;
for (uint32_t queueFamilyIndex : uniqueQueueFamilies)
{
    VkDeviceQueueCreateInfo queueCreateInfo {};
    queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
    queueCreateInfo.queueFamilyIndex = queueFamilyIndex;
    queueCreateInfo.queueCount = 1;
    queueCreateInfo.pQueuePriorities = &QUEUE_PRIORITY;
    queueCreateInfos.push_back(queueCreateInfo);
}
```

最后记得在销毁Vulkan Instance前销毁Window Surface:

```cpp
vkDestroySurfaceKHR(mVulkanInstance, mWindowSurface, nullptr);
vkDestroyInstance(mVulkanInstance, nullptr);
```
