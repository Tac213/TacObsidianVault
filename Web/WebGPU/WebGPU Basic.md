WebGPU整体用起来比较像简化版的Vulkan。在C++里面使用需要用到`webgpu/webgpu_cpp.h`这个头文件，该头文件是对`webgpu/webgpu.h`的一个C++封装，实际调用的还是C的接口，而且整体设计很像[wgpu](https://wgpu.rs/)，有些代码不知道怎么写可以参考wgpu的rust代码，现阶段C++的webgpu文档基本没有，想要参考的话只能参考下面这些：

- TypeScript和Rust的相关代码，将其翻译为C++，遇到报错猜测其原因并想办法修复
- Vulkan，毕竟是简化版本，由很多概念都可以参考Vulkan

## 判断当前环境是否支持WebGPU

WebGPU还是个实验性的功能，要使用WebGPU需要先判断当前环境是否支持，可以通过以下C++代码完成：

```cpp
int supportWebgpu = EM_ASM_INT({return "gpu" in navigator});
if (supportWebgpu)
{
    // WebGPU is supported
}
```

## 获取adapter和device

adapter即显卡，相当于Vulkan里面的Physical Device，device即逻辑设备，相当于Vulkan里面的Logical Device。在webgpu里，获取这2个对象的过程是异步的，在JavaScript里需要用到`await`这个关键字调用一个异步函数来获取，在C++里则需要用到回调，代码如下：

```cpp
void WebgpuRHI::requestAdapter()
{
    // 'wgpuCreateInstance' is unimplemented in Emscripten.
    // Leave as comment (until supported in Emscripten)
    // mInstance = wgpu::CreateInstance();
    wgpu::RequestAdapterOptions adapterOptions {};
    adapterOptions.powerPreference = wgpu::PowerPreference::HighPerformance;
    mInstance.RequestAdapter(
        &adapterOptions, [](WGPURequestAdapterStatus status, WGPUAdapter adapter, const char* message, void* userdata) {
            assert(status == WGPURequestAdapterStatus_Success);
            reinterpret_cast<WebgpuRHI*>(userdata)->onAdapterRequested(wgpu::Adapter::Acquire(adapter));
        },
        reinterpret_cast<void*>(this));
}
void WebgpuRHI::onAdapterRequested(wgpu::Adapter adapter)
{
    mAdapter = adapter;
    mAdapter.RequestDevice(
        nullptr, [](WGPURequestDeviceStatus status, WGPUDevice dev, const char* message, void* userdata) {
            assert(status == WGPURequestDeviceStatus_Success);
            wgpu::Device device = wgpu::Device::Acquire(dev);
            reinterpret_cast<WebgpuRHI*>(userdata)->onDeviceRequested(device);
        },
        reinterpret_cast<void*>(this));
}
void WebgpuRHI::onDeviceRequested(wgpu::Device device)
{
    mDevice = device;
    initializeWebgpu();
}
```

成员变量定义如下：

```cpp
wgpu::Instance mInstance;
wgpu::Adapter mAdapter;
wgpu::Device mDevice;
```

注意不需要创建Instance，因为Emscripten(3.1.21版本)暂时没有实现`wgpuCreateInstance`这个函数，等Emscripten实现了之后才能调用，否则会报错。

## 收集报错信息

和Vulkan一样，默认不显示报错，如果需要收集报错的话需要自行制定，代码如下：

```cpp
mDevice.SetUncapturedErrorCallback(
    [](WGPUErrorType errorType, const char* message, void* userdata) {
        reinterpret_cast<WebgpuRHI*>(userdata)->onUncapturedError(errorType, message);
    },
    reinterpret_cast<void*>(this));
```

## Queue

Queue只需要从Device里面获取即可得到：

```cpp
mQueue = mDevice.GetQueue();
```

## Surface

与Vulkan一样Surface值的是绘制窗口，WebGPU需要一个`canvas`元素作为绘制窗口，wgpu支持通过一个CSS选择器来选择到对应的html元素，具体代码如下：

```cpp
void WebgpuRHI::createSurface()
{
    wgpu::SurfaceDescriptorFromCanvasHTMLSelector canvasDescriptor {};
    canvasDescriptor.selector = mCANVAS_SELECTOR;
    wgpu::SurfaceDescriptor surfaceDescriptor {};
    surfaceDescriptor.nextInChain = &canvasDescriptor;
    mSurface = mInstance.CreateSurface(&surfaceDescriptor);
}
```

## SwapChain

与Vulkan一样有交换链SwapChain，不同的是在创建Swap Chain的时候会改变surface对应的`canvas`元素的尺寸，代码如下：

```cpp
void WebgpuRHI::createSwapChain(uint32_t width, uint32_t height)
{
    wgpu::SwapChainDescriptor swapCahinDescriptor {};
    swapCahinDescriptor.usage = wgpu::TextureUsage::RenderAttachment | wgpu::TextureUsage::CopySrc;
    swapCahinDescriptor.format = getSwapChainFormat();
    swapCahinDescriptor.width = width;
    swapCahinDescriptor.height = height;
    swapCahinDescriptor.presentMode = wgpu::PresentMode::Fifo;
    mSwapChain = mDevice.CreateSwapChain(mSurface, &swapCahinDescriptor);
}
```

### 处理Surface尺寸变化

与Vulkan一样，当Surface的尺寸发生变化时，需要重新创建Swap Chain，需要我们自己在tick时通过Emscripten的接口手动获取当前窗口的尺寸变化：

```cpp
// When the size changes, we need to recreate the swap chain.
int canvasWidth, canvasHeight;
emscripten_get_canvas_element_size(mCANVAS_SELECTOR, &canvasWidth, &canvasHeight);
if (canvasWidth != mSwapChainWidth || canvasHeight != mSwapChainHeight)
{
    if (!mSurface)
    {
        return EM_TRUE;
    }
    mSwapChainWidth = canvasWidth;
    mSwapChainHeight = canvasHeight;
    clearSwapChain();
    recreateSwapChain();
}
```

## Texture和TextureView

对应Vulkan的Image和ImageView，一样可以封装成一个函数，代码如下：

```cpp
void WebgpuRHI::createTexture(
    wgpu::Texture& texture,
    uint32_t width,
    uint32_t height,
    uint32_t depthOrArrayLayers,
    wgpu::TextureFormat format,
    wgpu::TextureUsage usage,
    uint32_t sampleCount,
    uint32_t mipLevelCount,
    wgpu::TextureDimension dimension) const
{
    wgpu::TextureDescriptor textureDescriptor {};
    textureDescriptor.size = {width, height, depthOrArrayLayers};
    textureDescriptor.format = format;
    textureDescriptor.usage = usage;
    textureDescriptor.dimension = dimension;
    textureDescriptor.mipLevelCount = mipLevelCount;
    textureDescriptor.sampleCount = sampleCount;
    texture = mDevice.CreateTexture(&textureDescriptor);
}
void WebgpuRHI::createTextureView(
    wgpu::TextureView& textureView,
    wgpu::Texture& texture,
    wgpu::TextureFormat format,
    wgpu::TextureAspect aspectFlags,
    uint32_t mipLevelCount)
{
    wgpu::TextureViewDescriptor textureViewDescriptor {};
    textureViewDescriptor.format = format;
    textureViewDescriptor.aspect = aspectFlags;
    textureViewDescriptor.mipLevelCount = mipLevelCount;
    textureView = texture.CreateView(&textureViewDescriptor);
}
```

### 写入像素数据

通过上面的方法创建的图片没有任何的像素数据，如果需要图片中有像素数据，需要通过下面的方法写入：

```cpp
void imageToTexture(wgpu::Texture texture, void* pixelData, wgpu::Extend3D size, uint32_t channels)
{
    const uint64_t dataSize = size.width * size.height * size.depthOrArrayLayers * channels * sizeof(uint8_t);
    wgpu::ImageCopyTexture destination {};
    destination.texture = texture;
    destination.mipLevel = 0;
    destination.origin = {0, 0, 0};
    destination.aspect = wgpu::TextureAspect::All;
    wgpu::TextureDataLayout dataLayout {};
    dataLayout.offset = 0;
    dataLayout.bytesPerRow = size.width * channels * sizeof(uint8_t);
    dataLayout.rowsPerImage = size.height;
    wgpu::Extent3D writeSize {size.width, size.height, size.depthOrArrayLayers};
    mQueue.WriteTexture(&destination, pixelData, dataSize, &dataLayout, &writeSize);
}
```

### TextureSampler

比较简单，只是一些简单的配置：

```cpp
void WebgpuRHI::createTextureSampler(uint32_t mipLevel)
{
    wgpu::SamplerDescriptor samplerDescriptor {};
    samplerDescriptor.magFilter = wgpu::FilterMode::Linear;
    samplerDescriptor.minFilter = wgpu::FilterMode::Linear;
    samplerDescriptor.addressModeU = wgpu::AddressMode::Repeat;
    samplerDescriptor.addressModeV = wgpu::AddressMode::Repeat;
    samplerDescriptor.addressModeW = wgpu::AddressMode::Repeat;
    // samplerDescriptor.compare = wgpu::CompareFunction::Always;
    samplerDescriptor.mipmapFilter = wgpu::FilterMode::Linear;
    samplerDescriptor.lodMinClamp = 0.0f;
    samplerDescriptor.lodMaxClamp = static_cast<float>(mipLevel);
    mTextureSampler = mDevice.CreateSampler(&samplerDescriptor);
}
```

## BindGroupLayout和BindGroup

相当于Vulkan里的DescriptorSetLayout和DescriptorSet，在创建PipelineLayout的时候需要用到BindGroupLayout，类似Vulkan创建PipelineLayout需要DescriptorSetLayout。

与DescriptorSet的作用相同，用来定义当前Pipeline中shader的binding信息。

通过下面的代码可以创建一个有2个binding，第1个binding为ubo，第2个binding为image sampler的BindGroupLayout:

```cpp
wgpu::BufferBindingLayout uniformBufferBindingLayout {};
uniformBufferBindingLayout.type = wgpu::BufferBindingType::Uniform;

wgpu::BindGroupLayoutEntry uniformBindGroupLayoutEntry {};
uniformBindGroupLayoutEntry.binding = 0;
uniformBindGroupLayoutEntry.visibility = wgpu::ShaderStage::Vertex;
uniformBindGroupLayoutEntry.buffer = uniformBufferBindingLayout;

wgpu::SamplerBindingLayout samplerBindingLayout {};
samplerBindingLayout.type = wgpu::SamplerBindingType::Filtering;
wgpu::BindGroupLayoutEntry samplerBindGroupLayoutEntry {};
samplerBindGroupLayoutEntry.binding = 1;
samplerBindGroupLayoutEntry.visibility = wgpu::ShaderStage::Fragment;
samplerBindGroupLayoutEntry.sampler = samplerBindingLayout;

std::array<wgpu::BindGroupLayoutEntry, 2> bindGroupLayoutEntries {uniformBindGroupLayoutEntry, samplerBindGroupLayoutEntry};
wgpu::BindGroupLayoutDescriptor uniformBindGroupLayoutDescriptor {};
uniformBindGroupLayoutDescriptor.entries = bindGroupLayoutEntries.data();
uniformBindGroupLayoutDescriptor.entryCount = bindGroupLayoutEntries.size();
wgpu::BindGroupLayout uniformBindGroupLayout = mDevice.CreateBindGroupLayout(&uniformBindGroupLayoutDescriptor);
```

通过下面的代码可以给上面这个BindGroupLayout写入数据得到BindGroup:

```cpp
wgpu::BindGroupEntry uniformBindGroupEntry {};
uniformBindGroupEntry.binding = 0;
uniformBindGroupEntry.buffer = mUniformBuffer;
uniformBindGroupEntry.size = sizeof(float);

wgpu::BindGroupEntry samplerBindGroupEntry {};
samplerBindGroupEntry.binding = 1;
samplerBindGroupEntry.sampler = mTextureSampler;

std::array<wgpu::BindGroupEntry, 2> bindGroupEntries {uniformBindGroupEntry, samplerBindGroupEntry};
wgpu::BindGroupDescriptor uniformBindGroupDescriptor {};
uniformBindGroupDescriptor.layout = uniformBindGroupLayout;
uniformBindGroupDescriptor.entries = bindGroupEntries.data();
uniformBindGroupDescriptor.entryCount = bindGroupEntries.size();
mUniformBindGroup = mDevice.CreateBindGroup(&uniformBindGroupDescriptor);
```

### Dynamic offset

有时候可能对于不同的对象使用同一个uniform buffer，只是每个对象对应的buffer的offset不同，此时虽然可以给每个对象都设置一个BindGroup，但是需要提前准备好BindGroup，因为创建BindGroup的开销比draw call要大。所以比较常见的办法是使用同一个BindGroup但是设置dynamic offset，也就是说在SetBindGroup的时候才决定offset，而不是创建时决定。

使用方法如下，首先BufferBindingLayout需要设置`hasDynamicOffset`字段为`true`：

```cpp
wgpu::BufferBindingLayout uniformBufferBindingLayout {};
uniformBufferBindingLayout.type = wgpu::BufferBindingType::Uniform;
uniformBufferBindingLayout.hasDynamicOffset = true;
```

然后使用BindGroup时，传入offset，注意传入的offset是一个数组，因为一个BindGroup可能有多个ubo，需要定义每个ubo的offset。

```cpp
std::array<uint32_t, 2> dynamicOffest {0, 0};
renderPassEncoder.SetBindGroup(0, mUniformBindGroup, dynamicOffset.size(), dynamicOffset.data());
```

## Buffer

创建Buffer很简单，直接调用Device上的接口即可：

```cpp
void WebgpuRHI::createBuffer(wgpu::Buffer& buffer, uint64_t size, wgpu::BufferUsage usage) const
{
    wgpu::BufferDescriptor bufferDescriptor {};
    bufferDescriptor.size = size;
    bufferDescriptor.usage = usage;
    buffer = mDevice.CreateBuffer(&bufferDescriptor);
}
```

注意size传的是整个buffer的字节大小，比如一个`uint32_t`的长度为4的数组，因为`uint32_t`是4个字节，所以size为16，也可以用`sizeof(uint32_t) * 4`。

## 写入Buffer

最简单的方法是用`Queue::WriteBuffer`方法，该方法的作用类似Vulkan里面通过staging buffer将buffer数据进行一次拷贝，Vulkan的做法：

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

WebGPU的做法：

```cpp
uint16_t const INDEX_DATA[] = {0, 1, 2, 0};
createBuffer(mIndexBuffer, sizeof(INDEX_DATA), wgpu::BufferUsage::Index | wgpu::BufferUsage::CopyDst);
mQueue.WriteBuffer(mIndexBuffer, 0, INDEX_DATA, sizeof(INDEX_DATA));
```

如果不拷贝，也可以用将显存映射到内存的方法，用Vulkan那种memcpy的方法映射过去。可以同步或异步处理，同步处理需要`mappedAtCreation`设为`true`。

```cpp
wgpu::BufferDescriptor bufferDescriptor {};
bufferDescriptor.size = size;
bufferDescriptor.usage = wgpu::BufferUsage::Index;
bufferDescriptor.mappedAtCreation = true;
buffer = mDevice.CreateBuffer(&bufferDescriptor);

void* data = buffer.GetMappedRange();
// do memcpy here
buffer.Unmap();
```

异步处理则需要使用如下的方法(注意需要在创建时增加MapWrite)：

```cpp
wgpu::BufferDescriptor bufferDescriptor {};
bufferDescriptor.size = size;
bufferDescriptor.usage = wgpu::BufferUsage::Index | wgpu::BufferUsage::MapWrite;
buffer = mDevice.CreateBuffer(&bufferDescriptor);

buffer.MapAsync(
    wgpu::MapMode::Write, 0, sizeof(VERT_DATA), [](WGPUBufferMapAsyncStatus status, void* userdata) -> void {
        assert(status == WGPUBufferMapAsyncStatus_Success);
        wgpu::Buffer* buffer = reinterpret_cast<wgpu::Buffer*>(userdata);
        void* data = buffer->GetMappedRange();
        // do memcpy here
        buffer->Unmap();
    },
    static_cast<void*>(&buffer));
```

## Shader

WebGPU的shader可以有2种：

- WGSL: 专门为WebGPU设计的shader语言，[官方文档](https://www.w3.org/TR/WGSL/)
- SPIR-V: 编译后的二进制shader，可以由glsl编译而来，与Vulkan一致，使用[tint](https://dawn.googlesource.com/tint)可以完成WGSL和SPIR-V之间的互相编译

具体创建ShaderModule的代码如下：

```cpp
wgpu::ShaderModule WebgpuRHI::createWGSLShaderModule(const char* shaderCode) const
{
    wgpu::ShaderModuleWGSLDescriptor wgsl {};
    wgsl.source = shaderCode;
    wgpu::ShaderModuleDescriptor shaderModuleDescriptor {};
    shaderModuleDescriptor.nextInChain = &wgsl;
    return mDevice.CreateShaderModule(&shaderModuleDescriptor);
}
wgpu::ShaderModule WebgpuRHI::createSPIRVShaderModule(const uint32_t* shaderCode, uint32_t size) const
{
    wgpu::ShaderModuleSPIRVDescriptor spirv {};
    spirv.codeSize = size / sizeof(uint32_t);
    spirv.code = shaderCode;
    wgpu::ShaderModuleDescriptor shaderModuleDescriptor {};
    shaderModuleDescriptor.nextInChain = &spirv;
    return mDevice.CreateShaderModule(&shaderModuleDescriptor);
}
```

注意wgsl只需要传入`const char*`的源代码即可，spirv则需要传入`uint32_t*`类型的二进制代码数组并传入数组长度。

## RenderPipeline

对应Vulkan里的GraphicsPipeline，创建需要定义PipelineLayout已经Pipeline里的各个阶段的信息。示例代码如下：

```cpp
// Input Assembly
std::array<wgpu::VertexAttribute, 2> vertexAttributes {};
vertexAttributes[0].shaderLocation = 0;  // [[location(0)]]
vertexAttributes[0].offset = 0;
vertexAttributes[0].format = wgpu::VertexFormat::Float32x2;

vertexAttributes[1].shaderLocation = 1;  // [[location(1)]]
vertexAttributes[1].offset = 2 * sizeof(float);
vertexAttributes[1].format = wgpu::VertexFormat::Float32x3;

wgpu::VertexBufferLayout vertexBufferLayout {};
vertexBufferLayout.attributes = vertexAttributes.data();
vertexBufferLayout.attributeCount = 2;
vertexBufferLayout.arrayStride = sizeof(float) * 5;
vertexBufferLayout.stepMode = wgpu::VertexStepMode::Vertex;

// Depth
wgpu::DepthStencilState depthStencil {};
depthStencil.depthWriteEnabled = true;
depthStencil.depthCompare = wgpu::CompareFunction::Less;
depthStencil.format = findDepthFormat();

// Pipeline Layout
wgpu::PipelineLayoutDescriptor pipelineLayoutDescriptor {};
pipelineLayoutDescriptor.bindGroupLayouts = &uniformBindGroupLayout;
pipelineLayoutDescriptor.bindGroupLayoutCount = 1;
wgpu::PipelineLayout pipelineLayout = mDevice.CreatePipelineLayout(&pipelineLayoutDescriptor);

// Vertex Shader State
wgpu::VertexState vertexState {};
vertexState.module = mVertexShaderModule;
vertexState.entryPoint = mVERTEX_SHADER_ENTRY;
vertexState.buffers = &vertexBufferLayout;
vertexState.bufferCount = 1;

// Color / Blend state
wgpu::BlendState blendState {};
blendState.color.operation = wgpu::BlendOperation::Add;
blendState.color.srcFactor = wgpu::BlendFactor::One;
blendState.color.dstFactor = wgpu::BlendFactor::One;
blendState.alpha.operation = wgpu::BlendOperation::Add;
blendState.alpha.srcFactor = wgpu::BlendFactor::One;
blendState.alpha.dstFactor = wgpu::BlendFactor::One;

wgpu::ColorTargetState colorTargetState {};
colorTargetState.format = getSwapChainFormat();
colorTargetState.blend = &blendState;
colorTargetState.writeMask = wgpu::ColorWriteMask::All;

// Fragment shader state
wgpu::FragmentState fragmentState {};
fragmentState.module = mFragmentShaderModule;
fragmentState.entryPoint = mFRAGMENT_SHADER_ENTRY;
fragmentState.targets = &colorTargetState;
fragmentState.targetCount = 1;

// Rasterization
wgpu::PrimitiveState primitiveState {};
primitiveState.frontFace = wgpu::FrontFace::CCW;
primitiveState.cullMode = wgpu::CullMode::Back;
primitiveState.topology = wgpu::PrimitiveTopology::TriangleList;

// Multisampling
wgpu::MultisampleState multisampleState {};
multisampleState.alphaToCoverageEnabled = false;
multisampleState.count = MSAA_SAMPLE_COUNT;

// Graphics Pipeline
wgpu::RenderPipelineDescriptor pipelineDescriptor {};
pipelineDescriptor.layout = pipelineLayout;
pipelineDescriptor.vertex = vertexState;
pipelineDescriptor.fragment = &fragmentState;
pipelineDescriptor.primitive = primitiveState;
pipelineDescriptor.depthStencil = &depthStencil;
pipelineDescriptor.multisample = multisampleState;

mRenderPipeline = mDevice.CreateRenderPipeline(&pipelineDescriptor);
```

## RenderPass与draw call

Vulkan里RenderPass的概念被简化。在每次绘制的时候才会被创建，每次绘制之前需要先创建一个CommandEncoder:

```cpp
wgpu::CommandEncoder commandEncoder = mDevice.CreateCommandEncoder();
```

然后定义render pass的描述，调用`commandEncoder.BeginRenderPass(&renderPassDescriptor)`得到`RenderPassEncoder`:

```cpp
wgpu::RenderPassColorAttachment colorAttachment {};
colorAttachment.view = mColorTextureView;
// Set current texture view as resolve target, otherwise the image view will not be presented.
colorAttachment.resolveTarget = mSwapChain.GetCurrentTextureView();
colorAttachment.clearValue = getClearColor();
colorAttachment.loadOp = wgpu::LoadOp::Clear;
colorAttachment.storeOp = wgpu::StoreOp::Store;

wgpu::RenderPassDepthStencilAttachment depthStencilAttachment {};
depthStencilAttachment.view = mDepthTextureView;
depthStencilAttachment.depthClearValue = 1;
depthStencilAttachment.depthLoadOp = wgpu::LoadOp::Clear;
depthStencilAttachment.depthStoreOp = wgpu::StoreOp::Store;
depthStencilAttachment.stencilClearValue = 0;
depthStencilAttachment.stencilLoadOp = wgpu::LoadOp::Clear;
depthStencilAttachment.stencilStoreOp = wgpu::StoreOp::Store;

wgpu::RenderPassDescriptor renderPassDescriptor {};
renderPassDescriptor.colorAttachments = &colorAttachment;
renderPassDescriptor.colorAttachmentCount = 1;
renderPassDescriptor.depthStencilAttachment = &depthStencilAttachment;

wgpu::RenderPassEncoder renderPassEncoder = commandEncoder.BeginRenderPass(&renderPassDescriptor);
```

实际上，这个`RenderPassDescriptor`也不一定需要每次绘制时重新创建，只需要在SwapChain重新创建时重新创建即可，因为它只跟SwapChain有关系。

这个`RenderPassEncoder`可以用来调用各种绘制指令，比如设置Pipeline、Viewport、BindGroup、VertexBuffer、IndexBuffer等等，示例代码如下：

```cpp
renderPassEncoder.SetPipeline(mRenderPipeline);
renderPassEncoder.SetViewport(0, 0, mSwapChainWidth, mSwapChainHeight, 0, 1);
renderPassEncoder.SetScissorRect(0, 0, mSwapChainWidth, mSwapChainHeight);
renderPassEncoder.SetBindGroup(0, mUniformBindGroup);
renderPassEncoder.SetVertexBuffer(0, mVertexBuffer);
renderPassEncoder.SetIndexBuffer(mIndexBuffer, wgpu::IndexFormat::Uint16);
renderPassEncoder.DrawIndexed(3);
renderPassEncoder.End();
renderPassEncoder.Release();
```

绘制指令调用完成后调用`commandEncoder.Finish()`即可得到`CommandBuffer`，将其通过queue进行submit即可完成绘制：

```cpp
wgpu::CommandBuffer commandBuffer = commandEncoder.Finish();
commandEncoder.Release();
mQueue.Submit(1, &commandBuffer);
commandBuffer.Release();

// 'wgpuSwapChainPresent' is unsupported
// mSwapChain.Present();
```

本来最后应该调用`swapChain.Present()`，不过Emscripten(3.1.21版本)暂时不支持这个接口，就不用调了。

### RenderBundleEncoder

通常来说每帧创建的command buffer的command是一样的，如果觉得没必要每帧都重复创建，可以使用RenderBundleEncoder。作用是只需要提前设置一次command到RenderBundle，然后每帧执行该bundle，从而实现了command buffer的复用。

RenderBundleEncoder还支持一次使用多个。

记录代码：

```cpp
void recordRenderBundle()
{
    std::array<wgpu::TextureFormat, 1> colorFormats {getSwapChainFormat()};
    wgpu::RenderBundleEncoderDescriptor renderBundleEncoderDescriptor {};
    renderBundleEncoderDescriptor.colorFormatsCount = colorFormats.size();
    renderBundleEncoderDescriptor.colorFormats = colorFormats.data();
    renderBundleEncoderDescriptor.depthFormats = findDepthFormat();
    wgpu::RenderBundleEncoder renderBundleEncoder = mDevice.CreateRenderBundleEncoder(&renderBundleEncoderDescriptor);

    renderBundleEncoder.SetPipeline(mRenderPipeline);
    renderBundleEncoder.SetBindGroup(0, mUniformBindGroup);
    renderBundleEncoder.SetVertexBuffer(0, mVertexBuffer);
    renderBundleEncoder.SetIndexBuffer(mIndexBuffer, wgpu::IndexFormat::Uint16);
    renderBundleEncoder.DrawIndexed(3);

    mRenderBundle = renderBundleEncoder.Finish();
}
```

绘制代码：

```cpp
wgpu::CommandEncoder commandEncoder = mDevice.CreateCommandEncoder();
// set up render pass descriptor hear
wgpu::RenderPassEncoder renderPassEncoder = commandEncoder.BeginRenderPass(&renderPassDescriptor);

std::array<wgpu::RenderBundle, 1> renderBundles {mRenderBundle};
renderPassEncoder.ExecuteBundles(renderBundles.size(), renderBundles.data());
renderPassEncoder.End();
renderPassEncoder.Release();

wgpu::CommandBuffer commandBuffer = commandEncoder.Finish();
commandEncoder.Release();
mQueue.Submit(1, &commandBuffer);
commandBuffer.Release();

// 'wgpuSwapChainPresent' is unsupported
// mSwapChain.Present();
```

### subpass

目前暂时不支持subpass，[这个issue](https://github.com/gpuweb/gpuweb/issues/435#issuecomment-592259867)有给出Proposal，但当前版本(3.1.21)还没有实装。

subpass主要用于DefferredRendering，[示例代码](https://austin-eng.com/webgpu-samples/samples/deferredRendering)有给出DefferedRendering的详细代码，并没有用subpass，而是用多个render pipeline，把gbuffer对应的pipeline得到的几张TextureView写入`gBufferTexturesBindGroup`，并把这个`gBufferTexturesBindGroup`作为最终绘制render target的pipeline的bind group，也就是可以在这个pipeline中读取gbuffer对应的图像信息。
