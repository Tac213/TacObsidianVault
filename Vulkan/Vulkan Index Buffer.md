Index buffer的创建与Vertex Buffer类似：

```cpp
void Application::createIndexBuffer()
{
    VkDeviceSize bufferSize = sizeof(indices[0]) * indices.size();

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(mLogicalDevice, stagingBufferMemory, 0, bufferSize, 0, &data);
    memcpy(data, indices.data(), static_cast<size_t>(bufferSize));
    vkUnmapMemory(mLogicalDevice, stagingBufferMemory);

    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_INDEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, mIndexBuffer, mIndexBufferMemory);
    copyBuffer(stagingBuffer, mIndexBuffer, bufferSize);

    vkDestroyBuffer(mLogicalDevice, stagingBuffer, nullptr);
    vkFreeMemory(mLogicalDevice, stagingBufferMemory, nullptr);
}
```

同样需要基于staging buffer来创建。

销毁也与Vertex Buffer类似：

```cpp
vkDestroyBuffer(mLogicalDevice, mIndexBuffer, nullptr);
vkFreeMemory(mLogicalDevice, mIndexBufferMemory, nullptr);
vkDestroyBuffer(mLogicalDevice, mVertexBuffer, nullptr);
vkFreeMemory(mLogicalDevice, mVertexBufferMemory, nullptr);
```

使用index buffer绘制时，不需要使用vertex buffer:

```cpp
VkBuffer vertexBuffers[] = {mVertexBuffer};
VkDeviceSize offsets[] = {0};
vkCmdBindVertexBuffers(commandBuffer, 0, 1, vertexBuffers, offsets);
vkCmdBindIndexBuffer(commandBuffer, mIndexBuffer, 0, VK_INDEX_TYPE_UINT16);
// vkCmdDraw(commandBuffer, static_cast<uint32_t>(vertices.size()), 1, 0, 0);
vkCmdDrawIndexed(commandBuffer, static_cast<uint32_t>(indices.size()), 1, 0, 0, 0);
```

在调用`vkCmdBindIndexBuffer`时应当注意，如果索引数据类型为`uint16_t`，则最后一个参数应为`VK_INDEX_TYPE_UINT16`，如果索引数据类型为`uint32_t`，则最后一个参数应为`VK_INDEX_TYPE_UINT32`。
