对[[gel3 set up allocator]]中实现的[Allocator](./gel3%20set%20up%20allocator.md)做封装，管理各个allocator。

源码参考[Memory Management part 2 of 3: C-Style Interface](http://allenchou.net/2013/05/memory-management-part-2-of-3-c-style-interface/)

实现很简单，就直接贴代码了：

MemoryManager.hpp:

```cpp
#pragma once

#include "Allocator.hpp"
#include "IModule.hpp"

namespace Tamashii {
class MemoryManager : _implements_ IModule {
public:
    template <typename T, typename... Arguments>
    T* newObject(Arguments... parameters) {
        return new (allocate(sizeof(T))) T(parameters...);
    }

    template <typename T>
    void deleteObject(T* object) {
        object->~T();
        free(object, sizeof(T));
    }

    void* allocate(size_t size);
    void free(void* object, size_t size);

    int initialize() override;
    void finalize() override;
    void tick() override;

private:
    static size_t* mBlockSizeLookUp;
    static Allocator* mAllocators;
    static Allocator* lookUpAllocator(size_t size);
};
}  // namespace Tamashii
```

MemoryManager.cpp:

```cpp
#include "MemoryManager.hpp"
#include <array>
#include <malloc.h>

using namespace Tamashii;

namespace Tamashii {
static const std::array<uint32_t, 47> BLOCK_SIZES = {
    // 4-increments
    4, 8, 12, 16, 20, 24, 28, 32, 36, 40, 44, 48,
    52, 56, 60, 64, 68, 72, 76, 80, 84, 88, 92, 96,

    // 32-increments
    128, 160, 192, 224, 256, 288, 320, 352, 384,
    416, 448, 480, 512, 544, 576, 608, 640,

    // 64-increments
    704, 768, 832, 896, 960, 1024};

static const uint32_t PAGE_SIZE = 8192;
static const uint32_t ALIGNMENT = 4;

static const uint32_t BLOCK_SIZES_COUNT = sizeof(BLOCK_SIZES) / sizeof(BLOCK_SIZES[0]);

static const uint32_t MAX_BLOCK_SIZE = BLOCK_SIZES[BLOCK_SIZES_COUNT - 1];
}  // namespace Tamashii

void* MemoryManager::allocate(size_t size) {
    Allocator* allocator = lookUpAllocator(size);
    if (allocator) {
        return allocator;
    }
    return malloc(size);  // NOLINT
}

void MemoryManager::free(void* object, size_t size) {
    Allocator* allocator = lookUpAllocator(size);
    if (allocator) {
        allocator->free(object);
    } else {
        free(object, size);
    }
}

int MemoryManager::initialize() {
    static bool bInitialized = false;
    if (!bInitialized) {
        mBlockSizeLookUp = new size_t[MAX_BLOCK_SIZE + 1];
        size_t j = 0;
        for (size_t i = 0; i <= MAX_BLOCK_SIZE; i++) {
            if (i > BLOCK_SIZES[j]) {  // NOLINT
                j++;
            }
            mBlockSizeLookUp[i] = j;  // NOLINT
        }
        mAllocators = new Allocator[BLOCK_SIZES_COUNT];
        for (size_t i = 0; i < BLOCK_SIZES_COUNT; i++) {
            mAllocators[i].reset(BLOCK_SIZES[i], PAGE_SIZE, ALIGNMENT);  // NOLINT
        }

        bInitialized = true;
    }
    return 0;
}

Allocator* MemoryManager::lookUpAllocator(size_t size) {
    if (size < MAX_BLOCK_SIZE) {
        return mAllocators + mBlockSizeLookUp[size];  // NOLINT
    }
    return nullptr;
}
```

注意Allocator要加一个默认构造，否则mAllocator数据无法初始化，编译不通过。
