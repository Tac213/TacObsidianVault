Sampler是一个采样器，用于处理图形学中的oversampling和undersampling问题。

需要创建对应的成员变量：

```cpp
VkSampler mTextureSampler;
```

创建函数如下：

```cpp
void Application::createTextureSampler()
{
    VkSamplerCreateInfo samplerInfo {};
    samplerInfo.sType = VK_STRUCTURE_TYPE_SAMPLER_CREATE_INFO;
    samplerInfo.magFilter = VK_FILTER_LINEAR;
    samplerInfo.minFilter = VK_FILTER_LINEAR;
    samplerInfo.addressModeU = VK_SAMPLER_ADDRESS_MODE_REPEAT;
    samplerInfo.addressModeV = VK_SAMPLER_ADDRESS_MODE_REPEAT;
    samplerInfo.addressModeW = VK_SAMPLER_ADDRESS_MODE_REPEAT;

    samplerInfo.anisotropyEnable = VK_TRUE;
    samplerInfo.maxAnisotropy = mPhysicalDeviceProperties.limits.maxSamplerAnisotropy;

    samplerInfo.borderColor = VK_BORDER_COLOR_INT_OPAQUE_BLACK;
    samplerInfo.unnormalizedCoordinates = VK_FALSE;

    samplerInfo.compareEnable = VK_FALSE;
    samplerInfo.compareOp = VK_COMPARE_OP_ALWAYS;

    samplerInfo.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
    samplerInfo.mipLodBias = 0.0f;
    samplerInfo.minLod = 0.0f;
    samplerInfo.maxLod = 0.0f;

    if (vkCreateSampler(mLogicalDevice, &samplerInfo, nullptr, &mTextureSampler) != VK_SUCCESS)
    {
        throw std::runtime_error("Failed to create texture sampler");
    }
}
```

字段解释：

- `magFilter`: oversampling对应的方式
- `minFilter`: undersampling对应的方式
- `borderColor`: 设置超出边界时的固定颜色
- `addressModeX`: u或v或w的超出边界的处理方式，有以下4种
    - `VK_SAMPLER_ADDRESS_MODE_REPEAT`: 超出边界部分重复图片
    - `VK_SAMPLER_ADDRESS_MODE_MIRRORED_REPEAT`: 超出边界的部分镜像重复图片
    - `VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE`: 超出边界的部分取边界的颜色
    - `VK_SAMPLER_ADDRESS_MODE_MIRROR_CLAMP_TO_EDGE`: 超出边界的部分取镜像边界的颜色
    - `VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_BORDER`: 超出边界的部分取固定颜色，是之前设置的固定的边界颜色
- `anisotropyEnable`: 是否启用各向异性优化
- `maxAnisotropy`: 计算最终颜色所使用的最大样本数量
- `unnormalizedCoordinates`: 如果设置为true则坐标取值范围为图片的像素大小，否则是标准化的`[0, 1)`这个区间，通常都用标准化的坐标
- `mipmapMode` / `mipLodBias` / '`minLod` / `maxLod`: 与mipmap有关

为了使用各向异性优化，需要启用Logical Device对应的Feature:

```cpp
VkPhysicalDeviceFeatures deviceFeatures {};
deviceFeatures.samplerAnisotropy = VK_TRUE;
```

同时最好要求物理设备支持各向异性优化：

```cpp
VkPhysicalDeviceFeatures supportedFeatures;
vkGetPhysicalDeviceFeatures(device, &supportedFeatures);

if (!supportedFeatures.samplerAnisotropy)
{
    return;
}
```
