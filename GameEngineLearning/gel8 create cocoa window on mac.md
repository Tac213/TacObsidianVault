为了有良好的写代码体验，可以先设置好编译配置。

首先因为cocoa使用objective-c进行编写，程序存在c / c++ / objective-c混合编译，为了支持混合编译，首先应该在根目录的CMakeLists.txt中增加编译选项：

```cmake
IF(${OS_MACOS})
    add_compile_options(-x objective-c++)
ENDIF(${OS_MACOS})
```

然后在Core/CMakeLists.txt中，寻找并链接Cocoa库：

```cmake
IF(${OS_MACOS})
    find_library(COCOA_LIBRARY Cocoa required)
    set(PLATFORM_ONLY_SOURCES
    Private/Application/Mac/CocoaApplication.mm
    Private/Application/Mac/AppDelegate.m
    Private/Application/Mac/WindowDelegate.m
    )
ENDIF(${OS_MACOS})

IF(${OS_MACOS})
    target_link_libraries(Core
        ${COCOA_LIBRARY}
    )
ENDIF(${OS_MACOS})
```

首先定义应用代理类AppDelegate。

头文件：

```objective-c
#pragma once

#import <Cocoa/Cocoa.h>

@interface AppDelegate : NSObject <NSApplicationDelegate>
@end

```

实现文件：

```objective-c
#import "Application/Mac/AppDelegate.h"

@interface AppDelegate ()

@end

@implementation AppDelegate

- (void)applicationDidFinishLaunching:(NSNotification *)notification {
}

- (BOOL)applicationShouldTerminateAfterLastWindowClosed:(NSApplication *)sender {
    return YES;
}

- (void)applicationWillTerminate:(NSNotification *)notification {
    // Insert code here to tear down your application
}

@end

```

然后定义窗口代理类WindowDelegate。

头文件；

```objective-c
#pragma once

#import <Cocoa/Cocoa.h>

@interface WindowDelegate : NSObject <NSWindowDelegate>
@end

```

实现文件：

```objective-c
#import "Application/Mac/WindowDelegate.h"

@interface WindowDelegate ()

@end

@implementation WindowDelegate

- (void)windowWillClose:(NSNotification *)notification {
    [NSApp terminate:self];
}

@end

```

最后定义应用类CocoaApplication。

头文件：

```objective-c
#pragma once

#include "Application/Application.hpp"
#include <Cocoa/Cocoa.h>

namespace Tamashii {
class CocoaApplication : public Application {
public:
    CocoaApplication(const TamashiiConfiguration& config)
        : Application(config){};
    int initialize() override;
    void finalize() override;
    void tick() override;

protected:
    NSWindow* mWindow = nullptr;
};
}  // namespace Tamashii

```

实现文件：

```objective-c
#include "Application/Mac/CocoaApplication.h"

#import "Application/Mac/AppDelegate.h"
#import "Application/Mac/WindowDelegate.h"

using namespace Tamashii;

int CocoaApplication::initialize() {
    int result = 0;

    [NSApplication sharedApplication];

    // Menu
    NSString* appName = [NSString stringWithFormat:@"%s", mConfig.appName];
    id menubar = [[NSMenu alloc] initWithTitle:appName];
    id appMenuItem = [NSMenuItem new];
    [menubar addItem:appMenuItem];
    [NSApp setMainMenu:menubar];

    id appMenu = [NSMenu new];
    id quitMenuItem = [[NSMenuItem alloc] initWithTitle:@"Quit"
                                          action:@selector(terminate:)
                                          keyEquivalent:@"q"];
    [appMenu addItem:quitMenuItem];
    [appMenuItem setSubmenu:appMenu];

    id appDelegate = [AppDelegate new];
    [NSApp setDelegate:appDelegate];
    [NSApp activateIgnoringOtherApps:YES];
    [NSApp finishLaunching];

    NSInteger style = NSWindowStyleMaskTitled | NSWindowStyleMaskClosable | 
                      NSWindowStyleMaskMiniaturizable | NSWindowStyleMaskResizable;

    mWindow = [[NSWindow alloc] initWithContentRect:CGRectMake(mConfig.positionX, mConfig.positionY, mConfig.screenWidth, mConfig.screenHeight)
                                                    styleMask:style
                                                    backing:NSBackingStoreBuffered
                                                    defer:NO];
    [mWindow setTitle:appName];
    [mWindow makeKeyAndOrderFront:nil];
    id windowDelegate = [WindowDelegate new];
    [mWindow setDelegate:windowDelegate];

    return result;
}

void CocoaApplication::finalize() {
    [mWindow release];
}

void CocoaApplication::tick() {
    NSEvent* event = [NSApp nextEventMatchingMask:NSEventMaskAny untilDate:nil inMode:NSDefaultRunLoopMode dequeue:YES];
    switch([(NSEvent*) event type]) {
        case NSEventTypeKeyDown: {
            break;
        }
        default: {
            break;
        }
    }
    [NSApp sendEvent:event];
    [NSApp updateWindows];
    [event release];
}

```

然后还是在Game.cpp中实例化CocoaApplication，注意因为CocoaApplication是objective-c中定义的类，所以要用static_cast将其cast为C++中的类：

```cpp
#elif defined(OS_MACOS)
const TamashiiConfiguration CONFIG(8, 8, 8, 8, 32, 0, 0, 20, 20, 960, 540, 10, "Tamashii(MacOS)");
extern IApplication* const G_APPLICATION = static_cast<IApplication*>(new CocoaApplication(CONFIG));
```
