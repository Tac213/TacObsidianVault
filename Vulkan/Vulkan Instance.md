Vulkan Instance用于指定驱动程序需要使用的应用程序信息。为了创建VulkanInstance，首先需要在应用程序的类上初始化一个私有成员：

```cpp
private:
    VkInstance mVulkanInstance;
```

注意不能把这个变量初始化为空指针，否则会导致VulkanInstance创建失败。

接着是创建VulkanInstance，比较简单就是传递一些必要的信息，这些信息都是通过结构体来定义:

```cpp
void Application::createVulkanInstance()
{
    VkApplicationInfo appInfo {};
    appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
    appInfo.pApplicationName = mConfig.windowTitle;
    appInfo.applicationVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.pEngineName = "Tamashii";
    appInfo.engineVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.apiVersion = VK_API_VERSION_1_0;

    VkInstanceCreateInfo createInfo {};
    createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
    createInfo.pApplicationInfo = &appInfo;
    uint32_t glfwRequiredExtensionCount = 0;
    const char** glfwRequiredExtensions = glfwGetRequiredInstanceExtensions(&glfwRequiredExtensionCount);
    createInfo.enabledExtensionCount = glfwRequiredExtensionCount;
    createInfo.ppEnabledExtensionNames = glfwRequiredExtensions;
    createInfo.enabledLayerCount = 0;

    if (vkCreateInstance(&createInfo, nullptr, &mVulkanInstance) != VK_SUCCESS)
    {
        mbQuit = true;
        std::cerr << "Failed to create Vulkan instance!" << std::endl;
    }
}
```

然后在应用被关闭时应当清理VulkanInstance:

```cpp
void Application::finalize()
{
    vkDestroyInstance(mVulkanInstance, nullptr);
}
```

MacOS的1.3.216.0版本以上的sdk需要注意，需要在VkInstanceCreateInfo中指定flag:

```cpp
#ifdef OS_MACOS
    createInfo.flags = VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR;
#endif
```

而且要启用下面的extension:

```cpp
#ifdef OS_MACOS
    extensions.push_back(VK_KHR_PORTABILITY_ENUMERATION_EXTENSION_NAME);
#endif
```
