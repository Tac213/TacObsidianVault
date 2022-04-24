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

type配置为cppdbg，request配置为launch表明启动调试cpp程序，preLaunchTask配置运行之前要执行的编译任务。MIMode和miDebuggerPath配置debugger的类型和路径。
