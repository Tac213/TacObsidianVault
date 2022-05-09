## Windows

[参考文章](https://huguoyang.cn/2021/12/10/CMAKE%E9%85%8D%E7%BD%AEC-C-%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83/)

### 安装mingw64/llvm/clang

首先是下载mwing64 + llvm + clang，可以在[这个网站](https://winlibs.com/)下载别人提起编译好的。不过需要注意，笔记撰写时[CodeLLDB](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb)最新版本为1.7.0，该版本的不支持14.0.0及以上的LLVM/Clang/LLD/LLDB，只能下载[13.0.0版本](https://github.com/brechtsanders/winlibs_mingw/releases/download/11.2.0-9.0.0-ucrt-r3/winlibs-x86_64-posix-seh-gcc-11.2.0-llvm-13.0.0-mingw-w64ucrt-9.0.0-r3.7z)。

后续的CodeLLDB支持哪个版本，可以关注CodeLLDB的最新发布时间，支持的llvm版本的发布时间通常早于CodeLLDB最新版本的发布时间。

下载完成后将7z解压缩到任意目录，比如`C:\softwares`，得到`C:\softwares\mingw64`目录。

解压完成后将`C:\softwares\mingw64\bin`这个目录加到系统PATH中。

通过下面这个命令检查安装是否成功：

```shell
$ clang --version
(built by Brecht Sanders) clang version 13.0.0
Target: x86_64-w64-windows-gnu
Thread model: posix
InstalledDir: C:\softwares\mingw64\bin
```

安装完成后clang / llvm / lldb / ninja / gcc这些都会在bin目录下了。

### 安装cmake

[cmake官网上](https://cmake.org/download/)找到最新的64为安装包一路next即可，注意要把安装目录加到系统PATH中。

通过下面这个命令检查安装是否成功：

```shell
$ cmake --version
cmake version 3.23.1

CMake suite maintained and supported by Kitware (kitware.com/cmake).
```

### 安装vscode拓展

- [llvm-vs-code-extensions.vscode-clangd](https://marketplace.visualstudio.com/items?itemName=llvm-vs-code-extensions.vscode-clangd): 与ms官方的cpp-tools冲突，不过这个只是language server，提供代码提示和代码补全功能，如果嫌麻烦也可以用ms官方的cpp-tools，就是要额外配置c_cpp_properties.json文件；
- [ms-vscode.cmake-tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cmake-tools): ms官方的cmake工具；
- [twxs.cmake](https://marketplace.visualstudio.com/items?itemName=twxs.cmake): 为`CMakeLists.txt`文件提供语法高亮支持，如果不需要这个功能也可以不安装；
- [vadimcn.vscode-lldb](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb): lldb调试插件，如果需要调试的话要注意这个插件所支持的llvm版本；

### 配置cmake项目

配置一个cmake项目。

配置完成后，在vscode内输入`Ctrl + Shift + P`，输入`CMake: Configure`，让vscode初始化vscode项目。如果想在每次打开vscode时自动完成这一步，需要在`settings.json`里面增加配置(generator设成Ninja是额外的配置，与该配置无关，只是把项目的cmake生成器设置为Ninja而已)：

```json
{
    "cmake.configureOnOpen": true,
    "cmake.generator": "Ninja",
}
```

Configure完成后就会在项目的根目录增加一个build目录，编译生成的文件都会放到这个目录下，可以把这个目录放到`.gitignore`中。

接着，继续`Ctrl + Shift + P`，输入`CMake: Select a Kit`，将编译器选为刚才下载的clang，如果在列表中没找到，也可以选择`[Scan for kits]`扫描编译器，等扫描完成后再重新选择。

### 调试

配置launch.json:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug",
            "program": "${command:cmake.launchTargetPath}",
            "args": [],
            "cwd": "${workspaceFolder}",
            "internalConsoleOptions": "neverOpen",
            "console": "integratedTerminal"
        }
    ]
}
```

注意此处的launch.json是将[ms-vscode.cmake-tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cmake-tools)和[vadimcn.vscode-lldb](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb)这2个工具搭配起来使用的，`"type": "lldb"`是CodeLLDB提供的功能，而`"program": "${command:cmake.launchTargetPath}"`则是cmake-tools提供的功能，这个指令会返回cmake编译的exe的路径，而且如果settings.json中`cmake.buildBeforeRun`配置为`true`(默认为`true`)，会在返回exe的路径前先编译完成，再返回完整路径。

详情可以阅读[CodeLLDB文档](https://github.com/vadimcn/vscode-lldb/blob/master/MANUAL.md)和[cmake-tools文档](https://github.com/microsoft/vscode-cmake-tools/blob/main/docs/README.md)。
