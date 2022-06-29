通过下面的代码可以获取Vulkan当前支持的拓展数量：

```cpp
uint32_t vulkanExtensionCount = 0;
vkEnumerateInstanceExtensionProperties(nullptr, &vulkanExtensionCount, nullptr);
```

拿到支持的拓展数量之后，就可以声明一个数组，获取具体支持的拓展，并遍历之:

```cpp
std::vector<VkExtensionProperties> extensions(vulkanExtensionCount);
vkEnumerateInstanceExtensionProperties(nullptr, &vulkanExtensionCount, extensions.data());
std::cout << "available extensions:" << std::endl;
for (const VkExtensionProperties& extension : extensions)
{
    std::cout << '\t' << extension.extensionName << std::endl;
}
```

glfw可以使用vulkan时需要vulkan支持相关拓展，调用下面的函数可以获得glfw要求的拓展:

```cpp
uint32_t glfwRequiredExtensionCount = 0;
const char** glfwRequiredExtensions = glfwGetRequiredInstanceExtensions(&glfwRequiredExtensionCount);
for (int i = 0; i < glfwRequiredExtensionCount; i++) {
    std::cout << glfwRequiredExtensions[i] << std::endl;
}
```

vulkan初始化时，可以判断glfw要求的vulkan拓展是否被支持，如果不支持可以直接结束程序，避免`vkCreateInstance`返回`VK_ERROR_EXTENSION_NOT_PRESENT`的错误：

```cpp
bool         bAllSupported              = true;
const char*  unsupportedExtensionName   = nullptr;
for (int i = 0; i < glfwRequiredExtensionCount; i++)
{
    bool bSupported = false;
    for (const VkExtensionProperties& extension : extensions)
    {
        if (strcmp(glfwRequiredExtensions[i], extension.extensionName) == 0)
        {
            bSupported = true;
            break;
        }
    }
    if (!bSupported)
    {
        bAllSupported            = false;
        unsupportedExtensionName = glfwRequiredExtensions[i];
        break;
    }
}
if (bAllSupported)
{
    std::cout << "All glfw required extensions is supported!" << unsupportedExtensionName << std::endl;
}
else
{
    std::cerr << "Unsupported extension: " << unsupportedExtensionName << std::endl;
}
```
