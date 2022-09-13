```cmake
cmake_minimum_required(VERSION 3.15 FATAL_ERROR)
project(libTamashii.js VERSION 0.1.0)

IF(NOT EMSCRIPTEN)
    message(FATAL_ERROR "You're trying to compile ${PROJECT_NAME} without emscripten.")
ENDIF()

IF(${CMAKE_BUILD_TYPE} STREQUAL "")
    message(STATUS "CMAKE_BUILD_TYPE is empty, assuming build type is Release")
    set(CMAKE_BUILD_TYPE Release)
ENDIF()

set(CPACK_PROJECT_NAME ${PROJECT_NAME})
set(CPACK_PROJECT_VERSION ${PROJECT_VERSION})
set(CMAKE_C_FLAGS "-Wall")
set(CMAKE_C_FLAGS_DEBUG "-O0 -g")
set(CMAKE_C_FLAGS_MINSIZEREL "-Os -DNDEBUG")
set(CMAKE_C_FLAGS_RELEASE "-O3 -DNDEBUG")
set(CMAKE_C_FLAGS_RELWITHDEBINFO "-O2 -g -DNDEBUG")
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_FLAGS "-Wall -std=gnu++20")
set(CMAKE_CXX_FLAGS_DEBUG "-O0 -g")
set(CMAKE_CXX_FLAGS_MINSIZEREL "-Os -DNDEBUG")
set(CMAKE_CXX_FLAGS_RELEASE "-O3 -DNDEBUG")
set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-O2 -g -DNDEBUG")

IF(PROJECT_SOURCE_DIR STREQUAL PROJECT_BINARY_DIR)
  message(
    FATAL_ERROR
      "In-source builds not allowed. Please make a new directory (called a build directory) and run CMake from there."
  )
ENDIF()

add_executable(
    Tamashii
    "cpp-src/WebgpuRHI.cpp"
)

target_link_libraries(Tamashii "-s MODULARIZE=1")
target_link_libraries(Tamashii "-s EXPORT_NAME=\"initializeTamashiiJs\"")
target_link_libraries(Tamashii "-s EXPORTED_FUNCTIONS=['_malloc','_free']")
target_link_libraries(Tamashii "-s EXPORTED_RUNTIME_METHODS=['allocateUTF8','allocateUTF8OnStack','UTF8ToString']")

set_target_properties(Tamashii PROPERTIES PREFIX "lib")

set(POST_BUILD_COMMANDS
    COMMAND ${CMAKE_COMMAND} -E copy "${PROJECT_BINARY_DIR}/libTamashii.js" "${PROJECT_SOURCE_DIR}/public"
    COMMAND ${CMAKE_COMMAND} -E copy "${PROJECT_BINARY_DIR}/libTamashii.wasm" "${PROJECT_SOURCE_DIR}/public"
)

add_custom_command(TARGET Tamashii ${POST_BUILD_COMMANDS})
```