与OpenGL一样，Vulkan采用Vertex Buffer将数据从cpu传到gpu。

## Vertex抽象

首先为了方便，可以将Vertex抽象成一个结构体：

```cpp
#include <array>
#include <glm/glm.hpp>
#include <vulkan/vulkan.h>


namespace LearnVulkan
{
    struct Vertex
    {
        glm::vec2 pos;
        glm::vec3 color;
        static VkVertexInputBindingDescription getBindingDescription();
        static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions();
    };
}  // namespace LearnVulkan
```

两个static函数的实现如下：

```cpp
#include "Vertex.hpp"

using namespace LearnVulkan;

VkVertexInputBindingDescription Vertex::getBindingDescription()
{
    VkVertexInputBindingDescription bindingDescription {};
    bindingDescription.binding = 0;
    bindingDescription.stride = sizeof(Vertex);
    bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;

    return bindingDescription;
}

std::array<VkVertexInputAttributeDescription, 2> Vertex::getAttributeDescriptions()
{
    std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions {};

    attributeDescriptions[0].binding = 0;
    attributeDescriptions[0].location = 0;
    attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
    attributeDescriptions[0].offset = offsetof(Vertex, pos);

    attributeDescriptions[1].binding = 0;
    attributeDescriptions[1].location = 1;
    attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
    attributeDescriptions[1].offset = offsetof(Vertex, color);

    return attributeDescriptions;
}
```

`VkVertexInputBindingDescription`用来描述要用什么rate来加载内存中的顶点数据。

- binding字段：数组中启始数据的索引值
- stride: 顶点数据的步长
- inputRate:
    - `VERTEX_INPUT_RATE_VERTEX`: 每经过一个顶点，移动到下一个数据
    - `VERTEX_INPUT_RATE_INSTANCE`: 每经过一个instance，移动到下一个数据

`VkVertexInputAttributeDescription`用来描述顶点数据的具体属性。

- binding字段: tells vulkan from which binding the per-vertex data comes
- location字段: 属性在vertex shader中的location
- format字段: 属性对应的数据格式
    - float: `VK_FORMAT_R32_SFLOAT`
    - vec2: `VK_FORMAT_R32G32_SFLOAT`
    - vec3: `VK_FORMAT_R32G32B32_SFLOAT`
    - vec4: `VK_FORMAT_R32G32B32A32_SFLOAT`
    - ivec2: `VK_FORMAT_R32G32_SINT`
    - uvec4: `VK_FORMAT_R32G32B32A32_UINT`
    - float: `VK_FORMAT_R64_SFLOAT`
- offset字段: 属性数值在数组中的offset是多少

## Graphics Pipeline vertex input

Graphics Pipeline中的vertex input要修改成下面这样：

```cpp
VkPipelineVertexInputStateCreateInfo vertexInputInfo {};

auto bindingDescription = Vertex::getBindingDescription();
auto attributeDescriptions = Vertex::getAttributeDescriptions();

vertexInputInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO;
vertexInputInfo.vertexBindingDescriptionCount = 1;
vertexInputInfo.pVertexBindingDescriptions = &bindingDescription;
vertexInputInfo.vertexAttributeDescriptionCount = static_cast<uint32_t>(attributeDescriptions.size());
vertexInputInfo.pVertexAttributeDescriptions = attributeDescriptions.data();
```

## Vertices and Vertex Shader

定义顶点属性:

```cpp
const std::vector<Vertex> vertices = {
    {{0.0f, -0.5f}, {1.0f, 0.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 1.0f, 0.0f}},
    {{-0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}}};
```

对应的Vertex Shader应当读取顶点数据:

```glsl
# version 460

layout (location = 0) in vec2 inPosition;
layout (location = 1) in vec3 inColor;

layout (location = 0) out vec3 fragColor;

void main() {
    gl_Position = vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

## Create Vertex Buffer

首先定义成员变量：

```cpp
VkBuffer mVertexBuffer;
```

可以先帮清理加上：

```cpp
vkDestroyBuffer(mLogicalDevice, mVertexBuffer, nullptr);
```

创建函数如下：

```cpp
void Application::createVertexBuffer()
{
    VkBufferCreateInfo vertexBufferInfo {};
    vertexBufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
    vertexBufferInfo.size = sizeof(vertices[0]) * vertices.size();
    vertexBufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;
    vertexBufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateBuffer(mLogicalDevice, &vertexBufferInfo, nullptr, &mVertexBuffer) != VK_SUCCESS)
    {
        std::cerr << "Failed to create Vertex Buffer" << std::endl;
        mbQuit = true;
        return;
    }
}
```

## Memory Requirements

显卡可以分配不同类型的内存作为缓冲使用。不同类型的内存所允许进行的操作以及操作的效率有所不同。需要结合自己的需求选择最合适的内存类型使用。添加一个函数来做件事:

```cpp
uint32_t Application::findPhysicalDeviceMemoryType(uint32_t typeFilter, VkMemoryPropertyFlags properties)
{
    VkPhysicalDeviceMemoryProperties memoryProperties;
    vkGetPhysicalDeviceMemoryProperties(mPhysicalDevice, &memoryProperties);

    for (uint32_t i = 0; i < memoryProperties.memoryTypeCount; i++)
    {
        if ((typeFilter & (1 << i)) && ((memoryProperties.memoryTypes[i].propertyFlags & properties) == properties))
        {
            return i;
        }
    }

    throw std::runtime_error("Failed to find suitable physical device memory type!");
}
```

## Allocate Memory

首先创建一个成员变量：

```cpp
VkDeviceMemory mVertexBufferMemory;
```

然后分配内存：

```cpp
void Application::createVertexBuffer()
{
    VkMemoryAllocateInfo allocInfo {};
    allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
    allocInfo.allocationSize = memoryRequirements.size;
    allocInfo.memoryTypeIndex = findPhysicalDeviceMemoryType(memoryRequirements.memoryTypeBits, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT);

    if (vkAllocateMemory(mLogicalDevice, &allocInfo, nullptr, &mVertexBufferMemory) != VK_SUCCESS)
    {
        std::cerr << "Failed to allocate Vertex Buffer Memory" << std::endl;
        mbQuit = true;
        return;
    }

    vkBindBufferMemory(mLogicalDevice, mVertexBuffer, mVertexBufferMemory, 0);
}
```

记得清理内存：

```cpp
vkFreeMemory(mLogicalDevice, mVertexBufferMemory, nullptr);
```

## Filling the vertex buffer

有了Vertex Buffer，也分配了内存，就可以将顶点数据写入buffer里了：

```cpp
void Application::createVertexBuffer()
{
    void* data;
    vkMapMemory(mLogicalDevice, mVertexBufferMemory, 0, vertexBufferInfo.size, 0, &data);
    memcpy(data, vertices.data(), static_cast<size_t>(vertexBufferInfo.size));
    vkUnmapMemory(mLogicalDevice, mVertexBufferMemory);
}
```

## Binding the vertex buffer

最后，需要在drawFrame调用recordCommandBuffer时，将VertexBuffer绑定上去：

```cpp
vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, mGraphicsPipeline);

VkBuffer vertexBuffers[] = {mVertexBuffer};
VkDeviceSize offsets[] = {0};
vkCmdBindVertexBuffers(commandBuffer, 0, 1, vertexBuffers, offsets);
vkCmdDraw(commandBuffer, static_cast<uint32_t>(vertices.size()), 1, 0, 0);
```
