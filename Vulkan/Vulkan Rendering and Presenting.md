Vulkan的drawFrame通常是这样操作的：

- 等待上一帧渲染结束
- 从Swapchain上获取一张ImageView
- 记录Command Buffer中的渲染指令
- 提交已记录的Command Buffer
- 返回渲染后的ImageView到Swapchain进行Present操作

这些操作每一个都通过一个函数调用来完成的，但是每个操作的实际执行是异步进行的。函数调用会在实际操作结束前返回，并且操作的实际执行顺序也是不确定的。如果需要操作的执行按照一定顺序，就需要使用以下2种对象：

- VkSemaphore: 用于保证queue操作的顺序
- VkFence: 用于保证cpu和gpu的执行顺序同步

VkSemaphore和VkFence都有Signaled状态和Unsignaled状态，比如对于VkSemaphore，可以在Queue Submit的时候传入要等待哪个VkSemaphore的信号，当这个VkSemaphore的Unsignaled变成Signaled时，这个Queue的work才会被执行。伪代码：

```
VkCommandBuffer A, B = ... // record command buffers
VkSemaphore S = ... // create a semaphore

// enqueue A, signal S when done - starts executing immediately
vkQueueSubmit(work: A, signal: S, wait: None)

// enqueue B, wait on S to start
vkQueueSubmit(work: B, signal: None, wait: S)
```

我们写的C++代码都是运行在cpu的，如果需要在某个函数执行的时候等待gpu的某个指令的结果，就需要用到Fence，Fence的伪代码如下：

```
VkCommandBuffer A = ... // record command buffer with the transfer

VkFence F = ... // create the fence  

// enqueue A, start work immediately, signal F when done
vkQueueSubmit(work: A, fence: F)

vkWaitForFence(F) // blocks execution until A has finished executing

save_screenshot_to_disk() // can't run until the transfer has finished
```

把需要用到的VkSemaphore和VkFence定义为成员变量：

```cpp
VkSemaphore mImageAvailableSemaphore;
VkSemaphore mRenderFinishedSemaphore;
VkFence mInFlightFence;
```

创建函数：

```cpp
void Application::createSyncronizationObject()
{
    VkSemaphoreCreateInfo semaphoreInfo {};
    semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;

    VkFenceCreateInfo fenceInfo {};
    fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
    fenceInfo.flags = VK_FENCE_CREATE_SIGNALED_BIT;

    if (vkCreateSemaphore(mLogicalDevice, &semaphoreInfo, nullptr, &mImageAvailableSemaphore) != VK_SUCCESS)
    {
        std::cerr << "Failed to create Image Available Semaphore!" << std::endl;
        mbQuit = true;
        return;
    }

    if (vkCreateSemaphore(mLogicalDevice, &semaphoreInfo, nullptr, &mRenderFinishedSemaphore) != VK_SUCCESS)
    {
        std::cerr << "Failed to create Render Finished Semaphore!" << std::endl;
        mbQuit = true;
        return;
    }

    if (vkCreateFence(mLogicalDevice, &fenceInfo, nullptr, &mInFlightFence) != VK_SUCCESS)
    {
        std::cerr << "Failed to create In Flight Fence!" << std::endl;
        mbQuit = true;
        return;
    }
}
```

其中`fenceInfo`的`flags`字段要设置为`VK_FENCE_CREATE_SIGNALED_BIT`，这样fence创建出来就是Signaled状态，以免第一次进入drawFrame时的死等。

当然还要记得清理掉这几个对象：

```cpp
vkDestroyFence(mLogicalDevice, mInFlightFence, nullptr);
vkDestroySemaphore(mLogicalDevice, mRenderFinishedSemaphore, nullptr);
vkDestroySemaphore(mLogicalDevice, mImageAvailableSemaphore, nullptr);
```

## 等待上一帧渲染结束

drawFrame函数中，首先应该等待上一帧完成，并在上一帧完成之后重新将fence设为Unsigaled状态。

```cpp
void drawFrame() {
    vkWaitForFences(device, 1, &inFlightFence, VK_TRUE, UINT64_MAX);
    vkResetFences(device, 1, &inFlightFence);
}
```

注意Semaphore的状态是不需要开发者维护的，只有Fence的状态需要。

## 从Swapchain上获取一张ImageView

```cpp
uint32_t imageIndex;
vkAcquireNextImageKHR(mLogicalDevice, mSwapChain, UINT64_MAX, mImageAvailableSemaphore, VK_NULL_HANDLE, &imageIndex);
```

## 记录Command Buffer中的渲染指令

首要要重置Command Buffer，然后调用之前写好的recordCommandBuffer记录渲染指令：

```cpp
// record the command buffer
vkResetCommandBuffer(mCommandBuffer, 0);
recordCommandBuffer(mCommandBuffer, imageIndex);
```

## 提交已记录的Command Buffer

提交之前要等待Image的Index获取完成，因此要等待`mImageAvailableSemaphore`的信号，配置好之后就调用`vkQueueSubmit`即可：

```cpp
// submit the command buffer
VkSubmitInfo submitInfo {};
submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

VkSemaphore waitSemaphores[] = {mImageAvailableSemaphore};
VkPipelineStageFlags waitStages[] = {VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT};
submitInfo.waitSemaphoreCount = 1;
submitInfo.pWaitSemaphores = waitSemaphores;
submitInfo.pWaitDstStageMask = waitStages;
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &mCommandBuffer;
VkSemaphore signalSemaphores[] = {mRenderFinishedSemaphore};
submitInfo.signalSemaphoreCount = 1;
submitInfo.pSignalSemaphores = signalSemaphores;

// submit the command buffer to the graphics queue
if (vkQueueSubmit(mGraphicsQueue, 1, &submitInfo, mInFlightFence) != VK_SUCCESS)
{
    std::cerr << "Failed to submit draw command buffer!" << std::endl;
    mbQuit = true;
    return;
}
```

## 返回渲染后的ImageView到Swapchain进行Present操作

Present之前要等待vkQueueSubmit完成，因此需要等待`mRenderFinishedSemaphore`的信号，配置好之后调用`vkQueuePresentKHR`即可：

```cpp
VkPresentInfoKHR presentInfo {};
presentInfo.sType = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR;
presentInfo.waitSemaphoreCount = 1;
presentInfo.pWaitSemaphores = signalSemaphores;

VkSwapchainKHR swapchains[] = {mSwapChain};
presentInfo.swapchainCount = 1;
presentInfo.pSwapchains = swapchains;
presentInfo.pImageIndices = &imageIndex;
presentInfo.pResults = nullptr;

vkQueuePresentKHR(mPresentQueue, &presentInfo);
```
