首先如[原文](https://zhuanlan.zhihu.com/p/29127264)所述，增加一个引擎配置的结构体，增加到Engine/Core/Application/Configuration目录下。

TamashiiConfiguration.hpp:

```cpp
#pragma once

#include <cstdint>
#include <iostream>

namespace Tamashii {
struct TamashiiConfiguration {
    /**
     * @param r the red color depth in bits
     * @param g the green color depth in bits
     * @param b the red color depth in bits
     * @param a the alpha color depth in bits
     * @param d the depth buffer depth in bits
     * @param s the stencil buffer depth in bits
     * @param msaa the msaa sample count
     * @param width the screen width in pixel
     * @param height the screen height in pixel
     * @param appName the name of application
     */
    TamashiiConfiguration(
        uint32_t r = 8,   // NOLINT(cppcoreguidelines-avoid-magic-numbers)
        uint32_t g = 8,   // NOLINT(cppcoreguidelines-avoid-magic-numbers)
        uint32_t b = 8,   // NOLINT(cppcoreguidelines-avoid-magic-numbers)
        uint32_t a = 8,   // NOLINT(cppcoreguidelines-avoid-magic-numbers)
        uint32_t d = 24,  // NOLINT(cppcoreguidelines-avoid-magic-numbers)
        uint32_t s = 0,
        uint32_t msaa = 0,
        int width = 1920,   // NOLINT(cppcoreguidelines-avoid-magic-numbers)
        int height = 1080,  // NOLINT(cppcoreguidelines-avoid-magic-numbers)
        const char* appName = "Tamashii")
        : redBits(r)
        , greenBits(g)
        , blueBits(b)
        , alphaBits(a)
        , depthBits(d)
        , stencilBits(s)
        , msaaSamples(msaa)
        , screenWidth(width)
        , screenHeight(height)
        , appName(appName) {
    }

    uint32_t redBits;      // red color channel depth in bits
    uint32_t greenBits;    // green color channel depth in bits
    uint32_t blueBits;     // blue color channel depth in bits
    uint32_t alphaBits;    // alpha color channel depth in bits
    uint32_t depthBits;    // depth buffer depth in bits
    uint32_t stencilBits;  // stencil buffer depth in bits
    uint32_t msaaSamples;  // MSAA samples
    int screenWidth;       // width of screen
    int screenHeight;      // height of screen
    const char* appName;

    friend std::ostream& operator<<(std::ostream& out, const TamashiiConfiguration& gfxConf) {
        out << "GfxConfiguration:"
            << " R:" << gfxConf.redBits
            << " G:" << gfxConf.greenBits
            << " B:" << gfxConf.blueBits
            << " A:" << gfxConf.alphaBits
            << " D:" << gfxConf.depthBits
            << " S:" << gfxConf.stencilBits
            << " M:" << gfxConf.msaaSamples
            << " W:" << gfxConf.screenWidth
            << " H:" << gfxConf.screenHeight
            << std::endl;
        return out;
    }
};
}  // namespace Tamashii

```

然后实现WindowsApplication，头文件：

```cpp
#pragma once

#include "Application/Application.hpp"
#include <windows.h>
#include <windowsx.h>

namespace Tamashii {
class WindowsApplication : public Application {
public:
    WindowsApplication(const TamashiiConfiguration& config)
        : Application(config){};

    int initialize() override;
    void finalize() override;
    void tick() override;

    static LRESULT CALLBACK windowProc(HWND hwnd, UINT message, WPARAM wParam, LPARAM lParam);
};
}  // namespace Tamashii

```

实现文件：

```cpp
#include "Application/WindowsApplication.hpp"
#include <tchar.h>

using namespace Tamashii;

int WindowsApplication::initialize() {
    int result;
    result = Application::initialize();
    if (result != 0) {
        exit(result);
    }
    // get the HINSTANCE of the Console Program
    HINSTANCE hInstance = GetModuleHandle(nullptr);

    // the handle for the window, filled by a function
    HWND hwnd;

    // this struct holds information for the window class
    WNDCLASSEX wc;

    // clear out the window class for use
    ZeroMemory(&wc, sizeof(WNDCLASSEX));

    // fill in the struct with the needed information
    wc.cbSize = sizeof(WNDCLASSEX);
    wc.style = CS_HREDRAW | CS_VREDRAW;
    wc.lpfnWndProc = windowProc;
    wc.hInstance = hInstance;
    wc.hCursor = LoadCursor(nullptr, IDC_ARROW);  // NOLINT(cppcoreguidelines-pro-type-cstyle-cast)
    wc.hbrBackground = (HBRUSH)COLOR_WINDOW;      // NOLINT(cppcoreguidelines-pro-type-cstyle-cast)
    wc.lpszClassName = _T("Tamashii");

    // register the window class
    RegisterClassEx(&wc);

    // create the window and use the result as the handle
    hwnd = CreateWindowEx(0,
                          "Tamashii",            // name of the window class
                          mConfig.appName,       // title of the window
                          WS_OVERLAPPEDWINDOW,   // window style
                          CW_USEDEFAULT,         // x-position of the window
                          CW_USEDEFAULT,         // y-position of the window
                          mConfig.screenWidth,   // width of the window
                          mConfig.screenHeight,  // height of the window
                          nullptr,               // parent window hwnd
                          nullptr,               // menus
                          hInstance,             // application handle
                          nullptr                // used with multiple
    );

    // display the window on the screen
    ShowWindow(hwnd, SW_SHOW);

    return result;
}

void WindowsApplication::finalize() {
}

void WindowsApplication::tick() {
    // this struct holds Windows event messages
    MSG msg;

    // we use PeekMessage instead of GetMessage here
    // because we should not block the thread at anywhere
    // except the engine execution driver module
    if (PeekMessage(&msg, nullptr, 0, 0, PM_REMOVE)) {
        // translate keystroke messages into the right format
        TranslateMessage(&msg);

        // send the message to the windowProc function
        DispatchMessage(&msg);
    }
}

LRESULT CALLBACK WindowsApplication::windowProc(HWND hwnd, UINT message, WPARAM wParam, LPARAM lParam) {
    // sort through and find what code to run for the message given
    switch (message) {
    case WM_PAINT:
        // we will replace this part with Rendering Module
        {
            break;
        }
    case WM_DESTROY: {
        // close the application entirely
        PostQuitMessage(0);
        Application::mbQuit = true;
        return 0;
    }
    }

    // Handle any messages the switch statement didn't
    return DefWindowProc(hwnd, message, wParam, lParam);
}
```

最重要的是怎么编译这个WindowsApplication。

首先根目录的CMakeLists.txt，需要设置一些变量，判断当前操作系统环境：

```cmake
IF(${UNIX})
    IF(${APPLE})
        set(TAMASHII_TARGET_PLATFORM "Darwin")
        set(OS_MACOS 1)
    ELSEIF(${CMAKE_SYSTEM_NAME} MATCHES Android)
        set(TAMASHII_TARGET_PLATFORM "Android")
        set(OS_ANDROID 1)
    ELSE(${APPLE})
        set(TAMASHII_TARGET_PLATFORM "Linux")
        set(OS_LINUX 1)
    ENDIF(${APPLE})
ELSEIF(${WIN32})
    set(TAMASHII_TARGET_PLATFORM "Windows")
    set(OS_WINDOWS 1)
ENDIF(${UNIX})
```

然后再根目录配置一个`config.hpp.in`，写入如下内容：

```
#cmakedefine OS_WINDOWS
#cmakedefine OS_ANDROID
#cmakedefine OS_LINUX
#cmakedefine OS_ANDROID
```

还是再CMakeLists.txt，在设置好操作系统环境后，增加下面这2行：

```cmake
include_directories("${PROJECT_SOURCE_DIR}/")
configure_file(${PROJECT_SOURCE_DIR}/config.hpp.in ${PROJECT_SOURCE_DIR}/config.hpp)
```

这样就会在根目录根据当前操作系统环境生成一个config.hpp文件，里面会根据cmake的变量设置对应的宏，这个config.hpp由于是自动生成的所以可以加到.gitignore中。

config.hpp:

```cpp
#define OS_WINDOWS
/* #undef OS_ANDROID */
/* #undef OS_LINUX */
/* #undef OS_ANDROID */

```

接着在Core/CmakeLists.txt中，将整个文件改成下面这样：

```cmake
set(PLATFORM_ONLY_SOURCES)
IF(${OS_WINDOWS})
    set(PLATFORM_ONLY_SOURCES
    Private/Application/WindowsApplication.cpp
    )
ENDIF(${OS_WINDOWS})

add_library(Core
Private/Application/Application.cpp
Private/Allocator.cpp

${PLATFORM_ONLY_SOURCES}
)
```

其实就是根据根目录设置好的变量，设置一些只在部分平台编译的文件，并把这些平台相关的文件加到lib里面去。

最后改Programs/Game.cpp，根据宏来决定IApplication是哪个：

```cpp
#include "config.hpp"
#if defined(OS_WINDOWS)
#include "Application/WindowsApplication.hpp"
#else
#include "Application/Application.hpp"
#endif
#include "Application/Configuration/TamashiiConfiguration.hpp"

namespace Tamashii {
#if defined(OS_WINDOWS)
const TamashiiConfiguration CONFIG(8, 8, 8, 8, 32, 0, 0, 960, 540, "Tamashii(Windows)");
extern IApplication* const G_APPLICATION = new WindowsApplication(CONFIG);
#else
const TamashiiConfiguration CONFIG(8, 8, 8, 8, 32, 0, 0, 960, 540, "Tamashii(Default)");
extern IApplication* const G_APPLICATION = new Application(CONFIG);
#endif
}  // namespace Tamashii
```

注意windows.h和windowsx.h是不需要find_library的，windows下编译好的llvm会带有这2个头文件，在clang64/include这个目录下能找到。
