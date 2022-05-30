## Windows

[参考文章](https://huguoyang.cn/2021/12/10/CMAKE%E9%85%8D%E7%BD%AEC-C-%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83/)

[参考视频](https://www.bilibili.com/video/BV1sW411v7VZ)

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
    "cmake.buildDirectory": "${workspaceFolder}/Build/${buildType}",
}
```

Configure完成后就会在项目的根目录增加一个Build目录，编译生成的文件都会放到这个目录下，可以把这个目录放到`.gitignore`中。Build目录的名字也可以自定义，详见上述settings。`${buildType}`表示当前编译的类型，比如Debug/Release等等。

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

## Linux

除了llvm / clang / cmake的安装方式不一样，其他都是一样的。

下面是ubuntu的安装：

```shell
sudo apt install llvm
sudo apt install clang
sudo apt install clangd
sudo apt install cmake
sudo apt install ninja-build
```

如果是在WSL下开发，那么在安装拓展时应当把拓展安装到WSL上。

## Mac

```shell
brew install llvm cmake ninja pkgconfig
```

通过上面的指令安装llvm/cmake/ninja/pkgconfig

llvm的tool chain也可以是使用XCode自带的，如果使用XCode自带的这里就不用重复安装了，但需要额外配置，`Command + Shift + P`输入: `CMake: Edit User-Local CMake Kits`，在`"Apple Clang"`里面的compilers里面的C和CXX都要配置为绝对路径(原本是`/usr/bin/clang`和`/usr/bin/clang++`：

```json
[
    {
        "name": "Apple Clang",
        "compilers": {
            "C": "/Library/Developer/CommandLineTools/usr/bin/clang",
            "CXX": "/Library/Developer/CommandLineTools/usr/bin/clang++"
        }
    }
]
```

其他和windows平台一样。

## extensions.json

使用这套开发环境，extensions.jon配置如下：

```json
{
    "recommendations": [
        "llvm-vs-code-extensions.vscode-clangd",
        "vadimcn.vscode-lldb",
        "ms-vscode.cmake-tools",
        "twxs.cmake"
    ],
    "unwantedRecommendations": [
        "ms-vscode.cpptools"
    ]
}
```

## attach process

lldb同样支持attach一个进程进行调试，但和调试python不一样，没法在进程里写脚本开放端口给VSCode去attach，必须要用`"${command:pickProcess}"`或者`"${command:pickMyProcess}"`这2个指令去选进程对应的pid来attach，对应的配置如下：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "attach",
            "name": "Attach",
            "program": "${workspaceFolder}/build/hello_world",
            "windows": {
                "program": "${workspaceFolder}/build/hello_world.exe"
            },
            "pid": "${command:pickProcess}",
            "waitFor": true
        }
    ]
}
```

实际上`pid`是个optional字段，也可以通过执行一段指令来告知vscode当前进程的pid，比如：

```cpp
char command[256];
snprintf(command, sizeof(command), "code --open-url \"vscode://vadimcn.vscode-lldb/launch/config?{'request':'attach','pid':%d}\"", getpid());
system(command);
sleep(1); // Wait for debugger to attach
```

还有一个很重要的字段`program`，这个字段必须为被调试的进程的可执行文件的路径，如果C++代码只是作为库被调用，并不编译出可执行文件，而是被python或者node之类的脚本调用，那么program应该设置为python或者node的可执行文件路径，比如`"/usr/bin/python3"`。

当然`program`字段也可以配置成cmake的`${command:cmake.launchTargetPath}`，也能找到正确的路径，只不过如果配置成这样那就意味着每次attach之前都要cmake编译一次，如果有这个需求就可以配置成这样，如果没有这个需求就还是配置完整的可执行文件路径即可。

## 还原core dump现场

[参考CodeLLDB的MANUAL](https://github.com/vadimcn/vscode-lldb/blob/master/MANUAL.md#inspecting-a-core-dump)，launch.json配置成这样：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "custom",
            "name": "core",
            "initCommands": [
                "target create ${workspaceFolder}/build/hello_world.exe -c ${input:coreFileName}"
            ]
        }
    ],
    "inputs": [
        {
          "id": "coreFileName",
          "type": "promptString",
          "description": "Enter core file path"
        }
    ] 
}
```

这样按下f5之后就会输入core文件的路径，输入正确后即可还原现场。
