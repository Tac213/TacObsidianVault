Allocator是一个管理内存的工具，主要是为了减少new的次数，因为new是要向操作系统申请内存空间，这是异步操作，如果频繁地使用new，将造成程序效率降低等问题。

主要参考文章：[Memory Management part 1 of 3: The Allocator](http://allenchou.net/2013/05/memory-management-part-1-of-3-the-allocator/)

最终这个Allocator是像下面这样使用地：

```cpp
// create allocator
Allocator alloc(sizeof(unsigned), 1024, 4);
 
// allocate memory
unsigned *i = reinterpret_cast<unsigned *>(alloc.allocate());
 
// manipulate memory
*i = 0u;
 
// free memory
alloc.free(i);
```

Allocator内部有Page和Block的概念，Block是实际存放对象内存数据的内存空间，而Page则是一系列Block的组合。Allocator每次向操作系统申请内存的时候，是以一个Page为基础单位的，所以PageSize直接影响了Allocator向操作系统申请内存空间的次数。

一个Allocator会维护多个Page，Page和Allocator都是单向链表。

根据上面的描述，得到Page和Allocator的定义：

```cpp
struct BlockHeader {
    BlockHeader* next;
};

struct PageHeader {
    PageHeader* next;
    BlockHeader* blocks() {
        return reinterpret_cast<BlockHeader*>(this + 1);  // NOLINT
    }
};
```

每次分配内存时，先看当前的这个page有没有空闲的block，如果有则把这个block分配出来给对象使用，如果没有则新建一个page，把newpage的next设成当前page，再从newpage的首个block开始分配。(每一个Page的最后一个Block的next是nullptr)

每个block的内存空间大小是固定的，会被align为一个具二进制前两位最多2个1的值，align宏：

```cpp
#ifndef ALIGN
#define ALIGN(x, a) (((x) + ((a)-1)) & ~((a)-1))
#endif
```

所以每个Block虽然有mBlockSize的空间，但只有mBlockSize - mAlignmentSize的空间用来存堆空间内存数据，有mAlignmentSize的空间是用来align的空间。

最终实现如下。

Allocator.hpp:

```cpp
#include <cstddef>
#include <cstdint>

namespace Tamashii {

struct BlockHeader {
    BlockHeader* next;
};

struct PageHeader {
    PageHeader* next;
    BlockHeader* blocks() {
        return reinterpret_cast<BlockHeader*>(this + 1);  // NOLINT
    }
};

class Allocator {
public:
    // debug patterns
    static const uint8_t PATTER_ALIGN = 0xFC;
    static const uint8_t PATTER_ALLOC = 0xFD;
    static const uint8_t PATTER_FREE = 0xFE;

    Allocator(size_t dataSize, size_t pageSize, size_t alignment);
    ~Allocator();

    // resets the allocator to a new configuration
    void reset(size_t dataSize, size_t pageSize, size_t alignment);

    // allocate and free blocks
    void* allocate();
    void free(void* p);
    void freeAll();

private:
#if defined(_DEBUG)
    // fill a free page with debug patterns
    void fillFreePage(PageHeader* page);

    // fill a block with debug patterns
    void fillFreeBlock(BlockHeader* block);

    // fill an allocated block with debug patterns
    void fillAllocatedBlock(BlockHeader* block);
#endif

    // gets the next block
    BlockHeader* nextBlock(BlockHeader* block);

    // page list
    PageHeader* mPageList;

    // free block list
    BlockHeader* mFreeBlockList;

    size_t mDataSize;
    size_t mPageSize;
    size_t mAlignmentSize;
    size_t mBlockSize;
    uint32_t mBlocksPerPage;

    // statistics
    uint32_t mPagesCount;
    uint32_t mBlocksCount;
    uint32_t mFreeBlocksCount;

    // disable copy & assignment
    Allocator(Allocator const&) = delete;
    Allocator& operator=(Allocator const&) = delete;

    // disable move & assignment
    Allocator(Allocator&&) = delete;
    Allocator& operator=(Allocator&&) = delete;
};
}  // namespace Tamashii
```

Allocator.cpp:

```cpp
#include "Allocator.hpp"
#include <cassert>
#include <cstring>

#ifndef ALIGN
#define ALIGN(x, a) (((x) + ((a)-1)) & ~((a)-1))
#endif

using namespace Tamashii;

Allocator::Allocator(size_t dataSize, size_t pageSize, size_t alignment)
    : mPageList(nullptr)
    , mFreeBlockList(nullptr)
    , mDataSize(0)
    , mPageSize(0)
    , mAlignmentSize(0)
    , mBlockSize(0)
    , mBlocksPerPage(0)
    , mPagesCount(0)
    , mBlocksCount(0)
    , mFreeBlocksCount(0) {
    reset(dataSize, pageSize, alignment);
}

Allocator::~Allocator() {
    freeAll();
}

void Allocator::reset(size_t dataSize, size_t pageSize, size_t alignment) {
    freeAll();

    mDataSize = dataSize;
    mPageSize = pageSize;

    size_t minimalSize = (sizeof(BlockHeader) > mDataSize) ? sizeof(BlockHeader) : mDataSize;
    // this magic only works when alignment is 2^n, which should general be the case
    // because most CPU/GPU also requires the aligment be in 2^n
    // but still we use a assert to guarantee it

#if defined(_DEBUG)
    assert(alignment > 0 && (alignment & (alignment - 1)) == 0);
#endif

    mBlockSize = ALIGN(minimalSize, alignment);
    mAlignmentSize = mBlockSize - minimalSize;
    mBlocksPerPage = (mPageSize - sizeof(PageHeader)) / mBlockSize;
}

void* Allocator::allocate() {
    if (!mFreeBlockList) {
        // allocate a new page
        PageHeader* newPage = reinterpret_cast<PageHeader*>(new int8_t[mPageSize]);  // NOLINT
        mPagesCount++;
        mBlocksCount += mBlocksPerPage;
        mFreeBlocksCount += mBlocksPerPage;

#if defined(_DEBUG)
        fillFreePage(newPage);
#endif

        if (mPageList) {
            newPage->next = mPageList;
        }
        mPageList = newPage;

        BlockHeader* block = newPage->blocks();
        // link each block in this page
        for (uint32_t i = 0; i < mBlocksPerPage; i++) {
            block->next = nextBlock(block);
            block = nextBlock(block);
        }
        block->next = nullptr;

        mFreeBlockList = newPage->blocks();
    }

    BlockHeader* freeBlock = mFreeBlockList;
    mFreeBlockList = mFreeBlockList->next;
    mFreeBlocksCount--;

#if defined(_DEBUG)
    fillAllocatedBlock(freeBlock);
#endif

    return reinterpret_cast<void*>(freeBlock);  // NOLINT
}

void Allocator::free(void* p) {
    BlockHeader* block = reinterpret_cast<BlockHeader*>(p);  // NOLINT

#if defined(_DEBUG)
    fillFreeBlock(block);
#endif

    block->next = mFreeBlockList;
    mFreeBlockList = block;
    ++mFreeBlocksCount;
}

void Allocator::freeAll() {
    PageHeader* page = mPageList;
    while (page) {
        PageHeader* pageToDelete = page;
        page = page->next;
        delete[] reinterpret_cast<uint8_t*>(pageToDelete);  // NOLINT
    }
    mPageList = nullptr;
    mFreeBlockList = nullptr;

    mDataSize = 0;
    mPageSize = 0;
    mAlignmentSize = 0;
    mBlockSize = 0;
    mBlocksPerPage = 0;

    mPagesCount = 0;
    mBlocksCount = 0;
    mFreeBlocksCount = 0;
}

#if defined(_DEBUG)
void Allocator::fillFreePage(PageHeader* page) {
    page->next = nullptr;
    BlockHeader* block = page->blocks();
    for (uint32_t i = 0; i < mBlocksPerPage; i++) {
        fillFreeBlock(block);
        block = nextBlock(block);
    }
}

void Allocator::fillFreeBlock(BlockHeader* block) {
    // block header + data
    std::memset(block, PATTER_FREE, mBlockSize - mAlignmentSize);

    // alignment
    std::memset(reinterpret_cast<uint8_t*>(block) + mBlockSize - mAlignmentSize, PATTER_ALIGN, mAlignmentSize);  // NOLINT
}

void Allocator::fillAllocatedBlock(BlockHeader* block) {
    // block header + data
    std::memset(block, PATTER_ALLOC, mBlockSize - mAlignmentSize);

    // alignment
    std::memset(reinterpret_cast<uint8_t*>(block) + mBlockSize - mAlignmentSize, PATTER_ALIGN, mAlignmentSize);  // NOLINT
}
#endif

BlockHeader* Allocator::nextBlock(BlockHeader* block) {
    return reinterpret_cast<BlockHeader*>(reinterpret_cast<uint8_t*>(block + mBlockSize));  // NOLINT
}
```
