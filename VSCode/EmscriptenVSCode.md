与[CMakeClangVSCode](./CMakeClangVSCode.md) [[CMakeClangVSCode]]基本相同。

首先要配置Emscripten的工具集，如果用emsdk安装的话位于以下目录：

```
emsdk/upstream/emscripten/cmake/Modules/Platform/Emscripten.cmake
```

有2种方式可以配置，一种是加到项目的`.vscode/cmake-kits.json`里，另一种是加到VSCode的个人设置，Windows位于`C:\Users\<user_name>\AppData\Local\CMakeTools\cmake-tools-kits.json`，Mac位于`/Users/<user_name>/.local/share/CMakeTools/cmake-tools-kits.json`，Linux位于`/home/<user_name>/.local/share/CMakeTools/cmake-tools-kits.json`增加一个如下配置即可：

```json
{
    "name": "Emscripten",
    "toolchainFile": "/path/to/emsdk/upstream/emscripten/cmake/Modules/Platform/Emscripten.cmake"
}
```

接着就可以正常使用cmake configure。

emscripten有一个额外的sysroot，位于这个目录：

```
emsdk/upstream/emscripten/cache/sysroot
```

当`#include <emscripten.h>`时实际上是在这个目录中include的，但是目前没有找到clangd配置额外sysroot的方法，可以在`compile_commands.json`文件里为需要跳转的文件手动加上`-Ipath/to/emscripten/sysroot`这个参数，比如：

```json
{
    "directory": "D:/Private/EmscriptenIDE/Build/Debug",
    "command": "D:\\Private\\emsdk\\upstream\\emscripten\\em++.bat -ID:/Private/emsdk/upstream/emscripten/cache/sysroot/include -g -std=c++20 -o CMakeFiles\\EmscriptenIDE.dir\\test.cpp.o -c D:\\Private\\EmscriptenIDE\\test.cpp",
    "file": "D:\\Private\\EmscriptenIDE\\test.cpp"
}
```

缺点就是每次cmake configure之后要手撸一遍这个文件。后续找到支持的方法再做更新。
