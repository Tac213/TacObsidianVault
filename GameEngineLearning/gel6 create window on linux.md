- [参考1](https://zhuanlan.zhihu.com/p/28665581)
- [参考2](https://lists.freedesktop.org/archives/xcb/2010-December/006714.html)

首先需要配置好linux显示窗口的环境。参考[[Run WSL GUI Apps]][传送门](../Linux/Run%20WSL%20GUI%20Apps.md)

另外，还要安装xcb的库，否则编译时找不到xcb。ubuntu参考下面的命令安装：

```shell
sudo apt update
sudo apt install libx11-xcb-dev
```

安装完成后，修改Core/CMakeLists.txt，find_libirary，同时链接库：

```cmake
IF(${OS_LINUX})
    find_library(XCB_LIBRARY xcb)
    find_library(X11_LIBRARY X11)
    find_library(X11_XCB_LIBRARY X11-xcb)
    set(PLATFORM_ONLY_SOURCES
    Private/Application/XcbApplication.cpp
    )
ENDIF(${OS_LINUX})

add_library(Core
Private/Application/Application.cpp
Private/Allocator.cpp

${PLATFORM_ONLY_SOURCES}
)

IF(${OS_LINUX})
    target_link_libraries(Core
        ${XCB_LIBRARY} 
        ${X11_LIBRARY} 
        ${X11_XCB_LIBRARY} 
    )
ENDIF(${OS_LINUX})
```

接着参考1和2两篇文章写XcbApplication。

header:

```cpp
#pragma once

#include "Application/Application.hpp"
#include <xcb/xcb.h>

namespace Tamashii {
class XcbApplication : public Application {
public:
    XcbApplication(const TamashiiConfiguration& config)
        : Application(config){};

    int initialize() override;
    void finalize() override;
    void tick() override;

protected:
    xcb_connection_t* mpConn = nullptr;
    xcb_screen_t* mpScreen = nullptr;
    xcb_window_t mWindow = 0;
    xcb_intern_atom_reply_t* mCloseWindowReply = nullptr;
};
}  // namespace Tamashii
```

cpp file:

```cpp
#include "Application/XcbApplication.hpp"
#include <array>
#include <cstring>

using namespace Tamashii;

int XcbApplication::initialize() {
    int result;
    result = Application::initialize();
    if (result != 0) {
        exit(result);
    }

    xcb_gcontext_t foreground;
    xcb_gcontext_t background;
    uint32_t mask = 0;
    std::array<uint32_t, 2> values{0, 0};

    // establish connection to X Server
    mpConn = xcb_connect(nullptr, nullptr);

    // get the first screen
    mpScreen = xcb_setup_roots_iterator(xcb_get_setup(mpConn)).data;

    // get the root window
    mWindow = mpScreen->root;

    // create black (foreground) graphic context
    foreground = xcb_generate_id(mpConn);
    mask = XCB_GC_FOREGROUND | XCB_GC_GRAPHICS_EXPOSURES;
    values[0] = mpScreen->black_pixel;
    values[1] = 0;
    xcb_create_gc(mpConn, foreground, mWindow, mask, &values);

    // create white (background) graphic context
    background = xcb_generate_id(mpConn);
    mask = XCB_GC_BACKGROUND | XCB_GC_GRAPHICS_EXPOSURES;
    values[0] = mpScreen->white_pixel;
    values[1] = 0;
    xcb_create_gc(mpConn, foreground, mWindow, mask, &values);

    // create window
    mWindow = xcb_generate_id(mpConn);
    mask = XCB_CW_BACK_PIXEL | XCB_CW_EVENT_MASK;
    values[0] = mpScreen->white_pixel;
    values[1] = XCB_EVENT_MASK_EXPOSURE | XCB_EVENT_MASK_KEY_PRESS;
    xcb_create_window(mpConn,                         // connection
                      XCB_COPY_FROM_PARENT,           // depth
                      mWindow,                        // window id
                      mpScreen->root,                 // parent window
                      mConfig.positionX,              // position-x
                      mConfig.positionY,              // position-y
                      mConfig.screenWidth,            // width
                      mConfig.screenHeight,           // height
                      mConfig.borderWidth,            // border width
                      XCB_WINDOW_CLASS_INPUT_OUTPUT,  // class
                      mpScreen->root_visual,          // visual
                      mask,                           // mask
                      &values);

    const char* protocolCookieName = "WM_PROTOCOLS";
    xcb_intern_atom_cookie_t cookie = xcb_intern_atom(mpConn, true, strlen(protocolCookieName), protocolCookieName);
    xcb_intern_atom_reply_t* reply = xcb_intern_atom_reply(mpConn, cookie, nullptr);
    const char* deleteWindowCookieName = "WM_DELETE_WINDOW";
    xcb_intern_atom_cookie_t deleteWindowCookie = xcb_intern_atom(mpConn, false, strlen(deleteWindowCookieName), deleteWindowCookieName);
    mCloseWindowReply = xcb_intern_atom_reply(mpConn, deleteWindowCookie, nullptr);
    xcb_change_property(mpConn,                 // connection
                        XCB_PROP_MODE_REPLACE,  // mode
                        mWindow,                // window id
                        reply->atom,            // property
                        XCB_ATOM_ATOM,          // property type
                        32,                     // format NOLINT
                        1,                      // data length
                        &mCloseWindowReply->atom);

    // set the title of the window
    xcb_change_property(mpConn,                   // connection
                        XCB_PROP_MODE_REPLACE,    // mode
                        mWindow,                  // window id
                        XCB_ATOM_WM_NAME,         // property
                        XCB_ATOM_STRING,          // property type
                        8,                        // format NOLINT
                        strlen(mConfig.appName),  // data length
                        mConfig.appName);

    // set the icon of the window
    xcb_change_property(mpConn,                   // connection
                        XCB_PROP_MODE_REPLACE,    // mode
                        mWindow,                  // window id
                        XCB_ATOM_WM_ICON_NAME,    // property
                        XCB_ATOM_STRING,          // property type
                        8,                        // format NOLINT
                        strlen(mConfig.appName),  // data length
                        mConfig.appName);

    // map the window on the screen
    xcb_map_window(mpConn, mWindow);

    xcb_flush(mpConn);

    return result;
}

void XcbApplication::finalize() {
    xcb_destroy_window(mpConn, mWindow);
    xcb_disconnect(mpConn);
}

void XcbApplication::tick() {
    if (xcb_generic_event_t* event = xcb_poll_for_event(mpConn)) {
        switch (event->response_type & ~0x80) {  // NOLINT(cppcoreguidelines-avoid-magic-numbers)
        case XCB_EXPOSE: {
            break;
        }
        case XCB_KEY_PRESS: {
            break;
        }
        case XCB_CLIENT_MESSAGE: {
            if (xcb_client_message_event_t* clientEvent = reinterpret_cast<xcb_client_message_event_t*>(event)) {  // NOLINT(cppcoreguidelines-pro-type-reinterpret-cast)
                if (clientEvent->data.data32[0] == mCloseWindowReply->atom) {
                    mbQuit = true;
                    break;
                }
            }
        }
        default: {
            break;
        }
        }
        free(event);  // NOLINT(cppcoreguidelines-no-malloc)
    }
}
```

最后修改Game.cpp，改法和windows基本相同，就是使用cmake定义的宏：

```cpp
#include "config.hpp"
#if defined(OS_WINDOWS)
#include "Application/WindowsApplication.hpp"
#elif defined(OS_LINUX)
#include "Application/XcbApplication.hpp"
#else
#include "Application/Application.hpp"
#endif
#include "Application/Configuration/TamashiiConfiguration.hpp"

namespace Tamashii {
#if defined(OS_WINDOWS)
const TamashiiConfiguration CONFIG(8, 8, 8, 8, 32, 0, 0, 20, 20, 960, 540, 10, "Tamashii(Windows)");
extern IApplication* const G_APPLICATION = new WindowsApplication(CONFIG);
#elif defined(OS_LINUX)
const TamashiiConfiguration CONFIG(8, 8, 8, 8, 32, 0, 0, 20, 20, 960, 540, 10, "Tamashii(Linux)");
extern IApplication* const G_APPLICATION = new XcbApplication(CONFIG);
#else
const TamashiiConfiguration CONFIG(8, 8, 8, 8, 32, 0, 0, 20, 20, 960, 540, 10, "Tamashii(Default)");
extern IApplication* const G_APPLICATION = new Application(CONFIG);
#endif
}  // namespace Tamashii
```
