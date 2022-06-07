## GLFW

[github地址](https://github.com/glfw/glfw)

可以clone下来用cmake + cl编译，编译出来得到的lib要链接到自己的程序的话也自己的程序也必须用cl编译。

也可以[直接下载编译好的glfw](https://www.glfw.org/download.html)。

## Glad

参考[[gel10 set up glad env]]中的[setup脚本](../GameEngineLearning/gel10%20set%20up%20glad%20env.md)。

## 目录结构

```
.
|-- CMakeLists.txt
|-- External
|   |-- Source
|   |   `-- glad
|   |       |-- include
|   |       |   |-- KHR
|   |       |   |   `-- khrplatform.h
|   |       |   `-- glad
|   |       |       |-- glad.h
|   |       |       |-- glad_glx.h
|   |       |       `-- glad_wgl.h
|   |       `-- src
|   |           |-- glad.c
|   |           |-- glad_glx.c
|   |           `-- glad_wgl.c
|   `-- Windows
|       |-- include
|       |   `-- GLFW
|       |       |-- glfw3.h
|       |       `-- glfw3native.h
|       `-- lib
|           `-- libglfw3.a
|-- Scripts
|   |-- Linux
|   |   `-- Setup.sh
|   |-- Mac
|   |   `-- Setup.command
|   |-- Universal
|   |   |-- install_certificates.py
|   |   |-- path_handler.py
|   |   |-- senv.py
|   |   |-- setup.py
|   |   `-- setup_glad.py
|   `-- Windows
|       `-- Setup.bat
|-- config.hpp
|-- config.hpp.in
`-- main.cpp
```

目前只在windows上开发就只搞了windows的include和lib。关键的CMakeLists.txt如下：

```cmake
include_directories("${PROJECT_SOURCE_DIR}/External/${PROJECT_TARGET_PLATFORM}/include")
include_directories("${PROJECT_SOURCE_DIR}/External/Source/glad/include")
add_library(glfw SHARED IMPORTED) # or STATIC instead of SHARED
set_target_properties(glfw PROPERTIES
    IMPORTED_LOCATION "${PROJECT_SOURCE_DIR}/External/${PROJECT_TARGET_PLATFORM}/lib/libglfw3.a"
    IMPORTED_IMPLIB "${PROJECT_SOURCE_DIR}/External/${PROJECT_TARGET_PLATFORM}/lib/libglfw3.a"
    INTERFACE_INCLUDE_DIRECTORIES "${PROJECT_SOURCE_DIR}/External/${PROJECT_TARGET_PLATFORM}/include"
)

add_executable(LearnOpenGL
    main.cpp
    "${PROJECT_SOURCE_DIR}/External/Source/glad/src/glad.c"
)
target_link_libraries(LearnOpenGL glfw) # also adds the required include path

configure_file(${PROJECT_SOURCE_DIR}/config.hpp.in ${PROJECT_SOURCE_DIR}/config.hpp)
MESSAGE( STATUS "PROJECT_TARGET_PLATFORM: " ${PROJECT_TARGET_PLATFORM} )
```

glad.c一定要加到executable里面，库文件用libglfw3.a是为了可以用clang编译。

## main.cpp

[参考文章](https://learnopengl.com/Getting-started/Hello-Window)

glad的include一定要在glfw的上面，否则编译失败。

文章的代码也可以[参考这里](https://learnopengl.com/code_viewer_gh.php?code=src/1.getting_started/1.2.hello_window_clear/hello_window_clear.cpp)。

```cpp
#include "config.hpp"
#include <glad/glad.h>  // NOLINT(llvm-include-order)
#include <GLFW/glfw3.h>

/**
 * handle window size change event
 */
void framebufferSizeCallback(GLFWwindow* window, int width, int height) {
    glViewport(0, 0, width, height);
}

/**
 * handle input event
 */
void processInput(GLFWwindow* window) {
    if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS) {
        glfwSetWindowShouldClose(window, true);
    }
}

/**
 * render frame
 */
void renderFrame(GLFWwindow* window) {
    glClearColor(0.2f, 0.3f, 0.3f, 1.0f);  // NOLINT(cppcoreguidelines-avoid-magic-numbers)
    glClear(GL_COLOR_BUFFER_BIT);
}

/**
 * main function
 */
int main() {
    // glfw: initialize and configure
    // ------------------------------
    glfwInit();
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
#if defined(OS_MACOS)
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
#endif

    const int WINDOW_WIDTH = 800;
    const int WINDOW_HEIGHT = 600;

    // glfw window creation
    // --------------------
    GLFWwindow* window = glfwCreateWindow(WINDOW_WIDTH, WINDOW_HEIGHT, "Learn OpenGL", nullptr, nullptr);
    if (!window) {
        glfwTerminate();
        return -1;
    }
    glfwMakeContextCurrent(window);

    // glad: load all OpenGL function pointers
    // ---------------------------------------
    if (!gladLoadGLLoader(reinterpret_cast<GLADloadproc>(glfwGetProcAddress))) {  // NOLINT(cppcoreguidelines-pro-type-reinterpret-cast)
        return -1;
    }

    glViewport(0, 0, WINDOW_WIDTH, WINDOW_HEIGHT);
    glfwSetFramebufferSizeCallback(window, framebufferSizeCallback);

    // render loop
    // -----------
    while (!glfwWindowShouldClose(window)) {
        // input
        processInput(window);

        // rendering commands here
        renderFrame(window);

        // check and call events and swap the buffers
        // glfw: swap buffers and poll IO events (keys pressed/released, mouse moved etc.)
        // -------------------------------------------------------------------------------
        glfwPollEvents();
        glfwSwapBuffers(window);
    }

    // glfw: terminate, clearing all previously allocated GLFW resources.
    // ------------------------------------------------------------------
    glfwTerminate();

    return 0;
}
```

note:

- `glViewport`中的数据会被用于[[Viewport transformation]]
