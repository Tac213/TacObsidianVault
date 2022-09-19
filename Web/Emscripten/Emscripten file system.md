使用emscritpen的文件系统，需要增加`-s FORCE_FILESYSTEM`的链接选项。如果使用NODEFS文件系统，还需要增加`-lnodefs.js`。

```cmake
target_link_libraries(${TARGET_NAME} "-s FORCE_FILESYSTEM")
target_link_libraries(${TARGET_NAME} "-lnodefs.js")
```

在node环境下、浏览器环境下emscripten的文件环境存在差异。

node环境下可以正常读写本地文件，浏览器环境则不行，最多只能通过emscripten提供的虚拟文件系统读取文件。

下面这个代码用于兼容不同的环境，包括node环境、浏览器环境、native环境：

```cpp
void FileSystem::initialize()
    {
#ifdef __EMSCRIPTEN__
    mbEmscripten = true;
    int inBrowser = EM_ASM_INT({return globalThis.process == undefined});
    mbInBrowser = static_cast<bool>(inBrowser);
    if (!mbInBrowser)
    {
        // clang-format off
        int cwdPointer = EM_ASM_INT({ return allocateUTF8(path.posix.join(...process.cwd().split(path.win32.sep))); });
        // clang-format on
        char* cwdCharArray = reinterpret_cast<char*>(cwdPointer);
        cwd = Path::normpath(std::string(cwdCharArray));
        free(cwdCharArray);
#ifdef NODERAWFS
        mNodeRootName = "Tamashii";
        // mount the current folder as a NODEFS instance
        // inside of emscripten
        EM_ASM({
            var rootName = UTF8ToString($0);
            FS.mkdir(`/${rootName}`);
            FS.mount(NODEFS, {root : '.'}, `/${rootName}`); }, mNodeRootName.c_str());
#enif
    }
    else
    {
        cwd = std::filesystem::current_path().string();
    }
#ele
    cwd = std::filesystem::current_path().string();
#enif
}
```

其中`FS.mkdir`和`FS.mount`在node环境下是必须的，相当于在node环境下有一套单独的路径标记为当前的工作环境，也就是说`/mNodeRootName`这个目录等价于`cwd`这个目录，而且node环境下不能用native路径读写文件，必须用`/mNodeRootName`这个目录作为根来读写文件。比如native工作目录是`D:\test`，在代码中mount为`/working`，那么native路径为`D:\test\test.txt`的文件必须通过`/working/test.txt`这个路径读取、写入。

浏览器环境下写入文件是没有意义的，只能通过emscripten提供的虚拟文件系统读取文件，要让浏览器加载虚拟文件，编译时需要增加如下链接选项：

```cmake
file(RELATIVE_PATH RELATIVE_ASSET "${CMAKE_BINARY_DIR}" "${PROJECT_SOURCE_DIR}/editor/public")
target_link_libraries(${TARGET_NAME} "--preload-file ${RELATIVE_ASSET}@/")
```

`--preload-file`选项后面要传文件或目录相对于编译工作目录的相对路径。通过在末尾增加`@x`可以设置这个路径在emscripten虚拟文件系统中的别名，比如`@/`说明该目录是虚拟文件系统的根目录，通过`/xxx`这个路径读取文件，比如`@config`说明该目录是虚拟文件系统中的`config`目录，通过`config`或`/config`这个路径读取文件。

可以设置多个`--preload-file`选项，则可以preload多个不同的目录，每个目录起一个不同的别名即可区分。

编译完成后会多出来一个`${TARGET_NAME}.data`的二进制文件，需要把这个文件和`${TARGET_NAME}.js`, `${TARGET_NAME}.wasm`放在同一个目录下，胶水代码才能正常加载。
