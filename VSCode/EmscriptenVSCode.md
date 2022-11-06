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

接着就可以正常使用cmake configure。通过`CMake: Edit User-Local CMake Kits`也可以编辑该文件。

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

缺点就是每次cmake configure之后要手撸一遍这个文件。比较好的方法是在根目录增加一个`.clangd`配置文件，增加如下内容([参考链接](https://github.com/clangd/vscode-clangd/issues/309)):

```yaml
CompileFlags:
  Add:
    [
      "-D__EMSCRIPTEN__",
      "-isysroot/path/to/emsdk/upstream/emscripten/cache/sysroot",
      "-iwithsysroot/include/compat",
      "-isystem/path/to/emsdk/upstream/emscripten/cache/sysroot/include/c++/v1",
      "-isystem/path/to/emsdk/upstream/emscripten/cache/sysroot/include",
    ]
```

`.clangd`文件可配置内容[参考官方文档](https://clangd.llvm.org/config)。

上面的这些问题，也可以通过CLion这个IDE来解决，CLion对于Emscripten的适配比VSCode当前的clangd插件要好(当然前提是要有JetBrains的LICENSE)。

首先肯定还是要指定toolchain，在`Settings->Build, Execution, Development->Toolchains`界面中设定一个名为Emsciprten的路径，将C Compiler和C++ Compiler的路径分别设置为emcc和em++所在的路径。

接着在`Settings->Build, Execution, Development->CMake`界面中，将Toolchain设置为刚才的Toolchain，并在CMake Options这一项中，增加下面这个编译选项：

```
-DCMAKE_TOOLCHAIN_FILE=/path/to/emsdk/upstream/emscripten/cmake/Modules/Platform/Emscripten.cmake
```

最后，最好设置一下Build directory，将它设置到一个会被git ignore而且最好和VSCode的构建路径不同的地方。

另外，最好在CLion中配置一下保存自动clang-format。首先是在`Settings->Editor->Code Style`界面的最下方勾选Enable ClangFormat。然后在`Settings->Tools->Actions on Save`中勾选Reformat code。
