Vulkan和OpenGL一样使用glsl，但Vulkan运行时不编译人类可读的glsl文件，而是读取提前编译好的二进制文件SPIR-V。当然也可以通过[shaderc](https://github.com/google/shaderc)在运行时编译glsl为SPIR-V。

使用glslangValidator或者glslc都可以将glsl编译为SPIR-V，比如glslc的使用方法：

```shell
glslc shader.vert -o vert.spv
```

glslc会随VulkanSDK一起安装。

然后可以通过下面的方法读取这个二进制文件中的所有数据：

```cpp
#include <fstream>
#include <vector>

using namespace std;

vector<char> readFile(const string& filename)
{
    ifstream file(filename, ios::ate | ios::binary);

    if (!file.is_open())
    {
        throw runtime_error("Failed to open file: " + filename);
    }

    size_t fileSize = static_cast<size_t>(file.tellg());
    vector<char> buffer(fileSize);

    file.seekg(0);
    file.read(buffer.data(), fileSize);

    file.close();
    return buffer;
}
```

读取到SPIR-V文件中的二进制数据之后，就可以实现一个这样的函数来创建Shader Module:

```cpp
VkShaderModule Application::createShaderModule(std::vector<char> shaderCode)
{
    VkShaderModuleCreateInfo createInfo {};
    createInfo.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO;
    createInfo.codeSize = shaderCode.size();
    createInfo.pCode = reinterpret_cast<const uint32_t*>(shaderCode.data());

    VkShaderModule shaderModule;

    if (vkCreateShaderModule(mLogicalDevice, &createInfo, nullptr, &shaderModule) != VK_SUCCESS)
    {
        throw std::runtime_error("Failed to create shader module!");
    }

    return shaderModule;
}
```

Shader Module用于创建Graphics Pipeline中的Shader Stage:

```cpp
void Application::createGraphicsPipeline()
{
    auto vertShaderCode = readFile("Shader/Vert.spv");
    auto fragShaderCode = readFile("Shader/Frag.spv");

    VkShaderModule vertShaderModule = createShaderModule(vertShaderCode);
    VkShaderModule fragShaderModule = createShaderModule(fragShaderCode);

    VkPipelineShaderStageCreateInfo vertShaderStageInfo {};
    vertShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
    vertShaderStageInfo.stage = VK_SHADER_STAGE_VERTEX_BIT;
    vertShaderStageInfo.module = vertShaderModule;
    vertShaderStageInfo.pName = "main";

    VkPipelineShaderStageCreateInfo fragShaderStageInfo {};
    fragShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
    fragShaderStageInfo.stage = VK_SHADER_STAGE_FRAGMENT_BIT;
    fragShaderStageInfo.module = fragShaderModule;
    fragShaderStageInfo.pName = "main";

    VkPipelineShaderStageCreateInfo shaderStageInfos[] = {vertShaderStageInfo, fragShaderStageInfo};

    vkDestroyShaderModule(mLogicalDevice, vertShaderModule, nullptr);
    vkDestroyShaderModule(mLogicalDevice, fragShaderModule, nullptr);
}
```

其中`VkPipelineShaderStageCreateInfo`中的`pName`用于指定shader的入口函数，也就是说可以编译多个不同的shader到同一个Shader Module中，并通过不同的入口函数名字启用不同的shader。
