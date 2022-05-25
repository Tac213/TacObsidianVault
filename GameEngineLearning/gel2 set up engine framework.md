在根目录下创建这些目录(模仿Unreal):

```
.
└── Engine
    └── Source
        ├── Programs
        └── Runtime
            ├── Core
            │   ├── Private
            │   └── Public
            └── Interface
                └── Public
```

Source目录同级会放ThirdParty和Plugins预留。

Programs目录放最终的executable文件，Runtime目录则是引擎的Runtime，里面和Unreal类似，每个目录是一个模块。模块内部的Public目录下放头文件，Private目录下放实现文件。每个模块都会有一个CMakeLists.txt，也是类似Unreal。

为了支持这个目录结构，需要重新配置一下CMake，根目录的CMakeLists.txt改成如下：

```
cmake_minimum_required(VERSION 3.0.0)
project(TamashiiScratch VERSION 0.1.0)

set(CPACK_PROJECT_NAME ${PROJECT_NAME})
set(CPACK_PROJECT_VERSION ${PROJECT_VERSION})
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_EXTENSIONS OFF)

add_subdirectory(Engine)
```

主要就是add_executable改成了add_subdirectory。

上面的目录递归到Engine子目录，所以在Engine子目录创建CMakeLists.txt:

```
set(TAMASHII_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/Source")
include_directories("${TAMASHII_SOURCE_DIR}/Runtime/Interface/Public")
include_directories("${TAMASHII_SOURCE_DIR}/Runtime/Core/Public")
add_subdirectory(Source/Programs)
add_subdirectory(Source/Runtime)
```

Engine子目录主要用于设置include的搜索路径，并递归到Programs和Runtime子目录中。因此还需要在这2个目录创建CMakeLists.txt，首先是Programs，先创建一个空的即可，后续会解释如何配置。直接看Runtime:

```
add_subdirectory(Core)
add_subdirectory(Interface)
```

这个很简单，就是递归到各模块即可，每加一个模块都要加一个add_subdirectory。对应的现在Core和Interface目录下创建空的CMakeLists.txt。也是在后面代码完成后说明如何配置。

然后开始像[原文](https://zhuanlan.zhihu.com/p/28619982)一样写接口宏、模块接口、应用接口、应用。其中应用我写在了Core模块下。

写完之后配置Core的CMakeLists，将Core作为lib:

```
add_library(Core
Private/Application/Application.cpp
)
```

然后将main.cpp move到Programs目录下：

```shell
git mv main.cpp Engine/Source/Programs/Game.cpp
```

接着改成这样：

```cpp
#include "Application/Application.hpp"

namespace Tamashii {
Application* const G_APPLICATION = new Application();
}  // namespace Tamashii

using namespace Tamashii;

int main() {
    int ret;

    if ((ret = G_APPLICATION->initialize()) != 0) {
        return ret;
    }

    while (!G_APPLICATION->isQuit()) {
        G_APPLICATION->tick();
    }

    G_APPLICATION->finalize();

    return 0;
}
```

修改Programs的CMakeLists.txt，将Game作为executable，并为Game链接Core:

```
add_executable(Game Game.cpp)
target_link_libraries(Game Core)
```

接着按F5即可编译、运行。
