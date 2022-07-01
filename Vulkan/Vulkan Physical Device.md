Physical Device是物理设备，也就是显卡，有时候一台机器上会又多张显卡，vulkan允许开发者遍历这些显卡，选择一张合适的显卡来使用。

通过`VkPhysicalDevice`对象来储存显卡信息，可以这样初始化这个对象：

```cpp
VkPhysicalDevice physicalDevice = VL_NULL_HANDLE;
```

通过下面的代码可以拿到当前设备的所有显卡：

```cpp
void Application::pickPysicalDevice()
{
    uint32_t deviceCount = 0;
    vkEnumeratePhysicalDevices(mVulkanInstance, &deviceCount, nullptr);

    if (deviceCount == 0)
    {
        std::cerr << "Failed to find GPUs with Vulkan support!" << std::endl;
        return;
    }

    std::vector<VkPhysicalDevice> devices(deviceCount);
    vkEnumeratePhysicalDevices(mVulkanInstance, &deviceCount, devices.data());
}
```

选择物理设备时，可以根据我们指定的选择规则对各显卡进行打分，最终选择打分最高的显卡。比如下面是Piccolo引擎选择显卡的策略(很简单，独显1000分，集显100分)：

```cpp
void VulkanRHI::initializePhysicalDevice()
{
    uint32_t physical_device_count;
    vkEnumeratePhysicalDevices(m_instance, &physical_device_count, nullptr);
    if (physical_device_count == 0)
    {
        throw std::runtime_error("enumerate physical devices");
    }
    else
    {
        // find one device that matches our requirement
        // or find which is the best
        std::vector<VkPhysicalDevice> physical_devices(physical_device_count);
        vkEnumeratePhysicalDevices(m_instance, &physical_device_count, physical_devices.data());
        std::vector<std::pair<int, VkPhysicalDevice>> ranked_physical_devices;
        for (const auto& device : physical_devices)
        {
            VkPhysicalDeviceProperties physical_device_properties;
            vkGetPhysicalDeviceProperties(device, &physical_device_properties);
            int score = 0;
            if (physical_device_properties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU)
            {
                score += 1000;
            }
            else if (physical_device_properties.deviceType == VK_PHYSICAL_DEVICE_TYPE_INTEGRATED_GPU)
            {
                score += 100;
            }
            ranked_physical_devices.push_back({score, device});
        }
        std::sort(ranked_physical_devices.begin(),
                  ranked_physical_devices.end(),
                  [](const std::pair<int, VkPhysicalDevice>& p1, const std::pair<int, VkPhysicalDevice>& p2) {
                      return p1 > p2;
                  });
        for (const auto& device : ranked_physical_devices)
        {
            if (isDeviceSuitable(device.second))
            {
                m_physical_device = device.second;
                break;
            }
        }
        if (m_physical_device == VK_NULL_HANDLE)
        {
            throw std::runtime_error("failed to find suitable physical device");
        }
    }
}
```

其实还可以增加一些别的标准，比如根据最多支持的2D纹理数量：

```cpp
score += deviceProperties.limits.maxImageDimension2D;
```

## Physical Device Queue Family

Vulkan的几乎所有操作，从绘制到加载纹理都需要将操作指令提交给一个队列，然后才能执行。Vulkan又多种不同类型的丢列，它们属于不同的Queue Family队列族。需要检测物理设备支持的队列族，以及这些队列族中哪些支持开发者需要使用的指令。

首先定义一个结构体作为检测队列族函数的返回值类型：

```cpp
#include <cstdint>
#include <optional>

namespace LearnVulkan
{
    struct QueueFamilyIndices
    {
        std::optional<uint32_t> graphicsFamily;

        bool isComplete() const { return graphicsFamily.has_value(); }
    };
}  // namespace LearnVulkan
```

然后实现findQueueFamily:

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
        i++;
    }

    return indices;
}
```

最后，使用这个函数来判断物理设备能否使用，参考Piccolo:

```cpp
bool VulkanRHI::isDeviceSuitable(VkPhysicalDevice physical_device)
{
    auto queue_indices           = findQueueFamilies(physical_device);
    bool is_extensions_supported = checkDeviceExtensionSupport(physical_device);
    bool is_swapchain_adequate   = false;
    if (is_extensions_supported)
    {
        SwapChainSupportDetails swapchain_support_details = querySwapChainSupport(physical_device);
        is_swapchain_adequate =
            !swapchain_support_details.m_formats.empty() && !swapchain_support_details.m_presentModes.empty();
    }
    VkPhysicalDeviceFeatures physical_device_features;
    vkGetPhysicalDeviceFeatures(physical_device, &physical_device_features);
    if (!queue_indices.isComplete() || !is_swapchain_adequate || !physical_device_features.samplerAnisotropy)
    {
        return false;
    }
    return true;
}
```
