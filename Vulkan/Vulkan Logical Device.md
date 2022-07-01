Logical Device是Vulkan中与Physical Device进行交互的一个对象。

Logical Device的创建过程和Vulkan Instance的创建过程非常像。首先需要一个`VkDevice`对象：

```cpp
VkDevice mLogicalDevice;
```

创建Logical Device时，队列会被一起创建，可以在创建Logical Device后获取队列的handle，因此可以再声明一个`VkQueue`对象：

```cpp
VkQueue mGraphicsQueue;
```

完整的创建代码：

```cpp
void Application::createLogicalDevice()
{
    QueueFamilyIndices indices = findQueueFamilyIndices(mPhysicalDevice);

    VkDeviceQueueCreateInfo queueCreateInfo {};
    queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
    queueCreateInfo.queueFamilyIndex = indices.graphicsFamily.value();
    queueCreateInfo.queueCount = 1;
    const float QUEUE_PRIORITY = 1.0f;
    queueCreateInfo.pQueuePriorities = &QUEUE_PRIORITY;

    VkPhysicalDeviceFeatures deviceFeatures {};

    VkDeviceCreateInfo createInfo {};
    createInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;
    createInfo.pQueueCreateInfos = &queueCreateInfo;
    createInfo.queueCreateInfoCount = 1;
    createInfo.pEnabledFeatures = &deviceFeatures;
#ifdef DEBUG
    createInfo.enabledLayerCount = static_cast<uint32_t>(VALIDATION_LAYERS.size());
    createInfo.ppEnabledLayerNames = VALIDATION_LAYERS.data();
#else
    createInfo.enabledLayerCount = 0;
#endif

#ifdef OS_MACOS
    const char* deviceExtension = "VK_KHR_portability_subset";
    createInfo.enabledExtensionCount = 1;
    createInfo.ppEnabledExtensionNames = &deviceExtension;
#endif

    if (vkCreateDevice(mPhysicalDevice, &createInfo, nullptr, &mLogicalDevice) != VK_SUCCESS)
    {
        std::cerr << "Failed to create logical device!" << std::endl;
        mbQuit = true;
        return;
    }
    vkGetDeviceQueue(mLogicalDevice, indices.graphicsFamily.value(), 0, &mGraphicsQueue);
}
```

注意mac要enable `VK_KHR_portability_subset`这个extension。
