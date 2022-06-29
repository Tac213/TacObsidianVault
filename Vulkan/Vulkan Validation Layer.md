默认情况下Vulkan API提供的错误检查功能非常有限，很多基本的错误都没有被Vulkan显式地处理。为了将错误检查加入，需要引入Validation Layer即校验层。

一般情况下只需要再Debug模式下使用Validation Layer，因此需要现在cmake中增加宏：

```cmake
target_compile_definitions(${TARGET_NAME} 
    PRIVATE
    $<$<CONFIG:Debug>:DEBUG>
)
```

也就是cmake配置为debug时才会定义`DEBUG`这个宏。

## 检测Validation Layer是否可以使用

LunarG地Vulkan SDK允许通过`VK_LAYER_KHRONOS_validation`来隐式地开启所有地Validation Layer，为此我们可以先定义：

```cpp
#ifdef DEBUG
const std::vector<const char*> VALIDATION_LAYERS = {"VK_LAYER_KHRONOS_validation"};
#endif
```

然后就可以用这个对象检测Validation Layer是否支持：

```cpp
#ifdef DEBUG
bool Application::checkValidationLayerSupport()
{
    uint32_t layerCount = 0;
    vkEnumerateInstanceLayerProperties(&layerCount, nullptr);

    std::vector<VkLayerProperties> availableLayers(layerCount);
    vkEnumerateInstanceLayerProperties(&layerCount, availableLayers.data());

    for (const char* layerName : VALIDATION_LAYERS)
    {
        bool bLayerFound = false;
        for (const VkLayerProperties& layerProperties : availableLayers)
        {
            if (strcmp(layerName, layerProperties.layerName) == 0)
            {
                bLayerFound = true;
                break;
            }
        }
        if (!bLayerFound)
        {
            return false;
        }
    }

    return true;
}
#endif
```

可以在创建Vulkan Instance之前调用这个函数。

```cpp
#ifdef DEBUG
if (!checkValidationLayerSupport())
{
    mbQuit = true;
    std::cerr << "Validation layers requested, but not available!" << std::endl;
    return;
}
#endif
```

## Extension支持

要使用Validation Layer还需要Vulkan有提供`VK_EXT_debug_utils`这个Extension支持:

```cpp
std::vector<const char*> Application::getRequiredExtensions()
{
    uint32_t glfwRequiredExtensionCount = 0;
    const char** glfwRequiredExtensions = glfwGetRequiredInstanceExtensions(&glfwRequiredExtensionCount);
    std::vector<const char*> extensions(glfwRequiredExtensions, glfwRequiredExtensions + glfwRequiredExtensionCount);
#ifdef DEBUG
    extensions.push_back(VK_EXT_DEBUG_UTILS_EXTENSION_NAME);
#endif
    return extensions;
}
```

实现这个函数后，修改一下原本检查Extension是否支持的函数即可判断Extension是否支持。

## Message Callback

像下面这样定义一个Message Callback:

```cpp
static VKAPI_ATTR VkBool32 VKAPI_CALL debugCallback(VkDebugUtilsMessageSeverityFlagBitsEXT messageSeverity,
                                                            VkDebugUtilsMessageTypeFlagsEXT messageType,
                                                            const VkDebugUtilsMessengerCallbackDataEXT* pCallbackData,
                                                            void* pUserData);
```

实现：

```cpp
VKAPI_ATTR VkBool32 VKAPI_CALL Application::debugCallback(VkDebugUtilsMessageSeverityFlagBitsEXT messageSeverity,
                                                          VkDebugUtilsMessageTypeFlagsEXT messageType,
                                                          const VkDebugUtilsMessengerCallbackDataEXT* pCallbackData,
                                                          void* pUserData)
{
    std::cerr << "validation layer: " << pCallbackData->pMessage << std::endl;
    return VK_FALSE;
}
```

## 初始化VkDebugUtilsMessengerCreateInfoEXT

这个结构体可以告诉vulkan我们要使用的Message Callback是哪个:

```cpp
void Application::populateDebugMessengerCreateInfo(VkDebugUtilsMessengerCreateInfoEXT& createInfo)
{
    createInfo = {};
    createInfo.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
    createInfo.messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
    createInfo.messageType = VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
    createInfo.pfnUserCallback = debugCallback;
    // createInfo.pUserData = nullptr;
}
```

如果指定`pUserData`指针的话，在Message Callback里面就可以读到这个指针对应的数据。

## 创建销毁VkDebugUtilsMessengerEXT

由于创建销毁VkDebugUtilsMessengerEXT的函数是一个Extension函数，不会被Vulkan自动加载，所以要自己使用`vkGetInstanceProcAddr`来加载这些函数：

```cpp
VkResult Application::createDebugUtilsMessengerEXT(VkInstance instance, const VkDebugUtilsMessengerCreateInfoEXT* pCreateInfo, const VkAllocationCallbacks* pAllocator, VkDebugUtilsMessengerEXT* pDebugMessenger)
{
    PFN_vkCreateDebugUtilsMessengerEXT func = reinterpret_cast<PFN_vkCreateDebugUtilsMessengerEXT>(vkGetInstanceProcAddr(instance, "vkCreateDebugUtilsMessengerEXT"));
    if (func == nullptr)
    {
        return VK_ERROR_EXTENSION_NOT_PRESENT;
    }
    return func(instance, pCreateInfo, pAllocator, pDebugMessenger);
}

void Application::destoryDebugUtilsMessengerEXT(VkInstance instance, VkDebugUtilsMessengerEXT debugMessenger, const VkAllocationCallbacks* pAllocator)
{
    PFN_vkDestroyDebugUtilsMessengerEXT func = reinterpret_cast<PFN_vkDestroyDebugUtilsMessengerEXT>(vkGetInstanceProcAddr(instance, "vkDestroyDebugUtilsMessengerEXT"));
    if (func == nullptr)
    {
        return;
    }
    func(instance, debugMessenger, pAllocator);
}
```

记得创建VkDebugUtilsMessengerEXT要晚于Vulkan Instance，销毁要早于VulkanInstance。
