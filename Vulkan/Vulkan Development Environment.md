## cmake find vulkan

```cmake
# Find Vulkan Path using CMake's Vulkan Module
# This will return Boolean 'Vulkan_FOUND' indicating the status of find as success(ON) or fail(OFF).
# Include directory path - 'Vulkan_INCLUDE_DIRS' and 'Vulkan_LIBRARY' with required libraries.
find_package(Vulkan)

# Try extracting VulkanSDK path from ${Vulkan_INCLUDE_DIRS}
if (NOT ${Vulkan_INCLUDE_DIRS} STREQUAL "")
    set(VULKAN_PATH ${Vulkan_INCLUDE_DIRS})
    STRING(REGEX REPLACE "/Include" "" VULKAN_PATH ${VULKAN_PATH})
endif()

MESSAGE(STATUS ${Vulkan_INCLUDE_DIRS})
     
if(NOT Vulkan_FOUND)
    # CMake may fail to locate the libraries but could be able to 
    # provide some path in Vulkan SDK include directory variable
    # 'Vulkan_INCLUDE_DIRS', try to extract path from this.
    MESSAGE(STATUS "Failed to locate Vulkan SDK, retrying again...")
    if(EXISTS "${VULKAN_PATH}")
        MESSAGE(STATUS "Successfully located the Vulkan SDK: ${VULKAN_PATH}")
    else()
        MESSAGE("Error: Unable to locate Vulkan SDK. Please turn ensure 'VULKAN_SDK' has been set as Environment Variable.")
        return()
    endif()
endif()
```

执行成功后会得到3个变量：

- `VULKAN_PATH`: vulkan的路径，比如`C:/VulkanSDK/1.3.216.0`
- `Vulkan_LIBRARY`: vulkan库的路径，比如`C:/VulkanSDK/1.3.216.0/Lib/vulkan-1.lib`
- `Vulkan_INCLUDE_DIR`: vulkan的include目录，比如`C:/VulkanSDK/1.3.216.0/Include`

可以使用这些变量链接库和增加include:

```cmake
target_link_libraries(${TARGET_NAME} PUBLIC ${Vulkan_LIBRARY})

target_include_directories(${TARGET_NAME} PUBLIC ${Vulkan_INCLUDE_DIR})
```

## glfw and glm as git submodule

以glfw为例:

```shell
git submodule add git@github.com:glfw/glfw.git ThirdParty/glfw
```

执行上面的命令后会在根目录下创建`.gitmodules`文件，内容如下：

```
[submodule "ThirdParty/glfw"]
	path = ThirdParty/glfw
	url = git@github.com:glfw/glfw.git
```

默认会使用master分支，此时对该文件做修改：

```
[submodule "ThirdParty/glfw"]
	path = ThirdParty/glfw
	url = git@github.com:glfw/glfw.git
	branch = 3.3-stable
```

然后执行：

```shell
git submodule update --remote
```

glm类似处理。

最终的`.gitmodules`如下：

```
[submodule "ThirdParty/glfw"]
	path = ThirdParty/glfw
	url = git@github.com:glfw/glfw.git
	branch = 3.3-stable
[submodule "ThirdParty/glm"]
	path = ThirdParty/glm
	url = git@github.com:g-truc/glm.git
	branch = cmake3
```

使用glm和glfw本身的cmake脚本编译:

```cmake
set(THIRD_PARTY_FOLDER "${CMAKE_CURRENT_SOURCE_DIR}")

MESSAGE(STATUS ${THIRD_PARTY_FOLDER})

if(NOT TARGET glm)
    option(BUILD_STATIC_LIBS "" ON)
    option(BUILD_TESTING "" OFF)
    option(GLM_TEST_ENABLE "" OFF)
    add_subdirectory(glm)
    set_target_properties(glm_static PROPERTIES FOLDER ${THIRD_PARTY_FOLDER}/glm)
endif()

if(NOT TARGET glfw)
    option(GLFW_BUILD_EXAMPLES "" OFF)
    option(GLFW_BUILD_TESTS "" OFF)
    option(GLFW_BUILD_DOCS "" OFF)
    option(GLFW_INSTALL "" OFF)
    add_subdirectory(glfw)
    set_target_properties(glfw PROPERTIES FOLDER ${THIRD_PARTY_FOLDER}/glfw)
    set_target_properties(update_mappings PROPERTIES FOLDER ${THIRD_PARTY_FOLDER}/glfw)
endif()
```

cmake链接glｍ和glfw:

```cmake
target_link_libraries(${TARGET_NAME} PUBLIC glm)
target_link_libraries(${TARGET_NAME} PUBLIC glfw)

target_include_directories(${TARGET_NAME} PUBLIC ${THIRD_PARTY_DIR}/glm)
```

## Windows

通过[这个链接](https://sdk.lunarg.com/sdk/download/latest/windows/vulkan-sdk.exe)安装最新版本的VulkanSDK，安装完成后确保`VULKAN_SDK`在系统环境变量里。

然后在cmake里找到vulkan，并配置好glm和glfw即可。

## Linux

安装vulkan相关依赖：

```shell
sudo apt install vulkan-tools
sudo apt install libvulkan-dev
sudo apt install vulkan-validationlayers-dev
sudo apt install glslang-tools
```

也可以下载vulkan-sdk:

```shell
wget -qO - http://packages.lunarg.com/lunarg-signing-key-pub.asc | sudo apt-key add -
sudo wget -qO /etc/apt/sources.list.d/lunarg-vulkan-focal.list http://packages.lunarg.com/vulkan/lunarg-vulkan-focal.list
sudo apt update
sudo apt install vulkan-sdk
```

下载vulkan-sdk后的相关配置在`/usr/share/vulkan`目录下。

安装glfw相关依赖：

```shell
sudo apt-get install libgl1-mesa-dev
sudo apt-get install libxrandr-dev
sudo apt-get install libxinerama-dev
sudo apt-get install libx11-xcb-dev
sudo apt-get install libxcursor-dev
sudo apt-get install libxi-dev
```

然后在cmake里找到vulkan，并配置好glm和glfw即可。

## MacOS

通过[这个链接](https://sdk.lunarg.com/sdk/download/latest/mac/vulkan-sdk.dmg)安装最新版本的VulkanSDK，安装完之后确保`VULKAN_SDK`在系统环境变量里：

```shell
export VULKAN_SDK=/path/to/your/vulkan/sdk/macOS
export VK_ADD_LAYER_PATH="$VULKAN_SDK/share/vulkan/explicit_layer.d"
export VK_ICD_FILENAMES="$VULKAN_SDK/share/vulkan/icd.d/MoltenVK_icd.json"
export VK_DRIVER_FILES="$VULKAN_SDK/share/vulkan/icd.d/MoltenVK_icd.json"
```

注意`VULKAN_SDK`最后一定要加一个macOS。实际上是在这个macOS子目录下。

然后cd到`VULKAN_SDK`的父目录，执行:

```shell
python3 install_vulkan.py
```
