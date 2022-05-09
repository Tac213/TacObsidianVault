## windows开发环境搭建

windows默认情况下没有编译器环境，如果需要编译器需要自行安装并配置。可以参考[这个文档](https://www.msys2.org/)下载g++，下载完成后将`C:\msys64\mingw64\bin`目录配置到系统环境变量中。

## c_cpp_properties.json

这个文件用于配置vscode的C/C++拓展，如果项目运行于多个平台，可以在这个文件中配置不同平台的配置，比如:

```json
{
    "version": 4,
    "configurations": [
        {
            "name": "Win32",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "defines": [
                "_DEBUG",
                "UNICODE",
                "_UNICODE"
            ],
            "windowsSdkVersion": "10.0.19041.0",
            "compilerPath": "C:/Program Files (x86)/Microsoft Visual Studio/2019/Community/VC/Tools/MSVC/14.29.30133/bin/Hostx64/x64/cl.exe",
            "cStandard": "c17",
            "cppStandard": "c++17",
            "intelliSenseMode": "windows-msvc-x64"
        },
        {
            "name": "Win32GCC",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "defines": [
                "_DEBUG",
                "UNICODE",
                "_UNICODE"
            ],
            "windowsSdkVersion": "10.0.19041.0",
            "compilerPath": "C:/msys64/mingw64/bin/g++.exe",
            "cStandard": "c17",
            "cppStandard": "c++17",
            "intelliSenseMode": "windows-gcc-x64"
        },
        {
            "name": "Mac",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "defines": [],
            "macFrameworkPath": [
                "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/System/Library/Frameworks"
            ],
            "compilerPath": "/usr/bin/clang",
            "cStandard": "c11",
            "cppStandard": "c++17",
            "intelliSenseMode": "clang-x64"
        },
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "defines": [],
            "compilerPath": "/usr/bin/gcc",
            "cStandard": "c17",
            "cppStandard": "c++17",
            "intelliSenseMode": "clang-x64"
        }
    ]
}
```

## task.json

这个文件主要用于配置编译命令，一次编译被认为是一个task。这个文件不支持operating system specific properties，我们可以把同一个任务在不同操作系统上的编译命令拆分成拖个任务配置在这个文件中。

示例：

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "type": "cppbuild",
            "label": "build prog1 on windows",
            "command": "g++",
            "args": [
                "-fdiagnostics-color=always",
                "-g",
                "prog1.cpp",
                "-o",
                "${workspaceFolder}\\prog1.exe"
            ],
            "options": {
                "cwd": "${workspaceFolder}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        },
        {
            "type": "cppbuild",
            "label": "build prog1 on mac",
            "command": "clang++",
            "args": [
                "-std=c++17",
                "-stdlib=libc++",
                "-g",
                "prog1.cpp",
                "-o",
                "${workspaceFolder}\\prog1"
            ],
            "options": {
                "cwd": "${workspaceFolder}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        },
        {
            "type": "cppbuild",
            "label": "build prog1 on linux",
            "command": "g++",
            "args": [
                "-fdiagnostics-color=always",
                "-g",
                "prog1.cpp",
                "-o",
                "${workspaceFolder}\\prog1"
            ],
            "options": {
                "cwd": "${workspaceFolder}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        },
    ]
}
```

type字段需要配置为cppbuild，或者配置为shell也可以因为本质上是运行一个shell命令。label就是任务的名字，类似任务id，可以根据自己的喜好配置。command配置编译器的路径即可，如果编译器已经在环境变量里面直接写编译器的名字就好了。windows字段里面可以写windows系统下override的配置，osx字段里面可以下mac系统写override的配置，linux字段里面可以写linux系统下override的配置。

## launch.json

启动配置的最主要任务是配置以下这2项:

- 启动前要执行哪项编译任务
- 使用哪个debugger

launch.json支持Operating system specific properties，可以把不同操作系统的启动写在同一个configuration中，比如使用gcc编译器可以使用以下配置：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Build & Debug prog1",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/prog1",
            "args": [],
            "stopAtEntry": false, // true if you want breakpoint at start of main function
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "lldb",
            "preLaunchTask": "build prog1 on mac",
            "windows": {
                "program": "${workspaceFolder}/prog1.exe",
                "MIMode": "gdb",
                "miDebuggerPath": "gdb",
                "setupCommands": [
                    {
                        "description": "Enable pretty-printing for gdb",
                        "text": "-enable-pretty-printing",
                        "ignoreFailures": true
                    }
                ],
                "preLaunchTask": "build prog1 on windows"
            },
            "linux": {
                "MIMode": "gdb",
                "miDebuggerPath": "/usr/bin/gdb",
                "setupCommands": [
                    {
                        "description": "Enable pretty-printing for gdb",
                        "text": "-enable-pretty-printing",
                        "ignoreFailures": true
                    }
                ],
                "preLaunchTask": "build prog1 on linux"
            },
        }
    ]
}
```

type配置为cppdbg，request配置为launch表明启动调试cpp程序，preLaunchTask配置运行之前要执行的编译任务。MIMode和miDebuggerPath配置debugger的类型和路径。MI是Machine Interface的意思。

## 统一使用clang tool chain

上面的配置方法在不同的平台使用的不同的tool chain，在开发过程中为了适配这些不同的tool chain会带来额外的麻烦，所以最好是统一个平台的tool chain。

比较常用的tool chain就是gcc和clang，到底哪个好坏褒贬不一，网上也有很多的评价可以自行搜索，选择clang主要是因为它更新也更流行，而且gcc编译下面这个函数不会报错:

```cpp
int foo() {}
```

在int函数没有返回值时gcc默认返回了0，这种隐含的规则会带来很多问题，出了bug也很难找。既然苹果选择了clang，那不如就跟随大佬的脚步。

以下笔记已过时，可参考[[CMakeClangVSCode]] [CMakeClangVSCode](CMakeClangVSCode.md)

### 各平台clang环境搭建

#### osx

这个最简单，App Store下载XCode即可。

####  windows

根据[msys2官网](https://www.msys2.org/)的指引，安装msys2，安装目录选择默认的`C:\msys64`即可，通常都用这个默认的路径。

第7步是安装mingw-w64 gcc，如果不想用gcc可以不安装，如果想要用gdb来debug的话就需要安装。安装完成后需要把`C:\msys64\mingw64\bin`目录加到PATH。

下面这个执行可以安装clang/llvm:

```shell
pacman -S mingw-w64-clang-x86_64-toolchain
```

安装完成后需要把`C:\msys64\clang64\bin`目录加到PATH。

#### linux

TODO

### c_cpp_properties.json

这个文件很简单，参考[vscode官网的mac配置](https://code.visualstudio.com/docs/cpp/config-clang-mac#_cc-configuration)即可。

```json
{
    "configurations": [
        {
            "name": "win32 clang",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "defines": [
                "_DEBUG",
                "UNICODE",
                "_UNICODE"
            ],
            "windowsSdkVersion": "10.0.19041.0",
            "compilerPath": "C:/msys64/clang64/bin/clang++.exe",
            "cStandard": "c17",
            "cppStandard": "c++20",
            "intelliSenseMode": "clang-x64"
        }
    ],
    "version": 4
}
```

### task.json

这个文件很简单，参考[vscode官网的mac配置](https://code.visualstudio.com/docs/cpp/config-clang-mac#_build-helloworldcpp)即可。


```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "type": "cppbuild",
            "label": "build prog1 on windows with clang",
            "command": "clang++",
            "args": [
                "-std=c++20",
                "-stdlib=libc++",
                "-g",
                "prog1.cpp",
                "-o",
                "${workspaceFolder}\\prog1.exe",
                // "--debug"  // this is an optional arg for debug
            ],
            "options": {
                "cwd": "${workspaceFolder}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        }
    ]
}
```

### launch.json

vscode官方的插件并不支持使用lldb进行调试，所以如果要调试的话只能用gdb，配置如下：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Build & Debug prog1 with clang",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/prog1",
            "terminal": "integrated",
            "args": [],
            "stopAtEntry": false, // true if you want breakpoint at start of main function
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "lldb",
            "stdio": null,
            "preLaunchTask": "build prog1 on mac",
            "windows": {
                "program": "${workspaceFolder}/prog1.exe",
                "MIMode": "gdb",
                "miDebuggerPath": "gdb",
                "setupCommands": [
                    {
                        "description": "Enable pretty-printing for gdb",
                        "text": "-enable-pretty-printing",
                        "ignoreFailures": true
                    }
                ],
                "preLaunchTask": "build prog1 on windows with clang"
            },
            "linux": {
                "MIMode": "gdb",
                "miDebuggerPath": "/usr/bin/gdb",
                "setupCommands": [
                    {
                        "description": "Enable pretty-printing for gdb",
                        "text": "-enable-pretty-printing",
                        "ignoreFailures": true
                    }
                ],
                "program": "${workspaceFolder}/prog1",
                "preLaunchTask": "build prog1 on linux with clang"
            },
        }
    ]
}
```

和原本的配置基本相同，只不过编译的preLaunchTask是用clang++来编译的。

#### 使用lldb debug

> 14.0.0版本以上的clang在windows平台至今未成功过...如果需要在windows使用最高只能到13.0.0版本

如果使用微软的c/cpp拓展，需要自行编译[lldb-mi](https://github.com/lldb-tools/lldb-mi)，然后将`miDebuggerPath`配置为编译出来的lldb-mi.exe的路径。也有个[官方文档](https://code.visualstudio.com/docs/cpp/lldb-mi)可以参考，不过至今没有成功过...

也可以安装[vadimcn.vscode-lldb](https://github.com/vadimcn/vscode-lldb)，然后launch.json按照如下配置进行配置：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug",
            "program": "${workspaceFolder}/prog1.exe",
            "args": [],
            "cwd": "${workspaceFolder}",
            "stdio": null,
            "preLaunchTask": "build prog1 on windows with clang"
        },
    ]
}
```

不过这个方法在windows上面断点无法命中，原因是[vadimcn.vscode-lldb](https://github.com/vadimcn/vscode-lldb)的当前最新版本1.7.0不支持14.0.0以上版本的clang调试，如果需要在windows使用最高只能到[13.0.0版本](https://github.com/brechtsanders/winlibs_mingw/releases/download/11.2.0-9.0.0-ucrt-r3/winlibs-x86_64-posix-seh-gcc-11.2.0-llvm-13.0.0-mingw-w64ucrt-9.0.0-r3.7z)。
