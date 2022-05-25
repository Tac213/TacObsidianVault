目标：

- 项目名字：TamashiiScratch
- 创建[项目仓库](https://github.com/Tac213/TamashiiScratch)
- 编辑器：VSCode
- 工具链：Clang + CMake

在github上创建好repo之后clone到本地，然后cd到仓库目录终端输入`code .`通过VSCode打开仓库目录。

新建`main.cpp`，输入以下内容：

```cpp
#include <stdio.h>

int main() {
    printf("Hello Engine!\n");
    return 0;
}

```

然后按照[[CMakeClangVSCode]]即[这个链接](../VSCode/CMakeClangVSCode.md)的指引，完成如下配置：

- 安装llvm cmake
- 安装相应的VSCode拓展
- 配置好CMakeLists.txt
- 配置好CMake + Clang工具链
- 配置好调试启动配置

CMakeLists.txt如下：

```
cmake_minimum_required(VERSION 3.0.0)
project(TamashiiScratch VERSION 0.1.0)

set(CPACK_PROJECT_NAME ${PROJECT_NAME})
set(CPACK_PROJECT_VERSION ${PROJECT_VERSION})
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_EXTENSIONS OFF)

add_executable(TamashiiScratch main.cpp)
```

此时按下F5，已经可以正常编译、调试项目。

接着按照[[ClangdVSCode]]即[这个链接](../VSCode/ClangdVSCode.md)的指引，完成如下配置：

- 配置`.clang-format`，尽可能早地配置这个文件，否则项目后期地blame会不准确
- 配置`.clang-tidy`

`.clang-format`配置如下：

```yaml
Language: Cpp
BasedOnStyle: LLVM
UseTab: Never
IndentWidth: 4
AccessModifierOffset: -4
AlignAfterOpenBracket: Align
AlignEscapedNewlines: Right
AlignOperands: true
AllowShortFunctionsOnASingleLine: None
AllowShortIfStatementsOnASingleLine: false
AllowShortLoopsOnASingleLine: false
ColumnLimit: 0
ConstructorInitializerAllOnOneLineOrOnePerLine: true
ConstructorInitializerIndentWidth: 4
KeepEmptyLinesAtTheStartOfBlocks: false
MaxEmptyLinesToKeep: 1
PointerAlignment: Left
SortIncludes: true
SortUsingDeclarations: true
SpaceBeforeParens: ControlStatements
SpacesBeforeTrailingComments: 2
SpacesInAngles: false
SpacesInContainerLiterals: true
SpacesInParentheses: false
SpacesInSquareBrackets: false
```

`.clang-tidy`配置如下：

```yaml
Checks: '-*,
clang-diagnostic-*,
llvm-*,
misc-*,
-misc-unused-parameters,
-misc-non-private-member-variables-in-classes,
-misc-no-recursion,
readability-identifier-naming,
cppcoreguidelines-*,
-cppcoreguidelines-pro-type-vararg,
readability-braces-around-statements,
modernize-use-nullptr,
modernize-use-override'
CheckOptions:
  - key: readability-identifier-naming.NamespaseCase
    value: CamelCase
  - key: readability-identifier-naming.ClassCase
    value: CamelCase
  - key: readability-identifier-naming.EnumCase
    value: CamelCase
  - key: readability-identifier-naming.FunctionCase
    value: camelBack
  - key: readability-identifier-naming.MemberCase
    value: camelBack
  - key: readability-identifier-naming.ParameterCase
    value: camelBack
  - key: readability-identifier-naming.UnionCase
    value: CamelCase
  - key: readability-identifier-naming.VariableCase
    value: camelBack
  - key: readability-identifier-naming.ConstantCase
    value: UPPER_CACE
  - key: readability-identifier-naming.EnumConstantCase
    value: UPPER_CACE
  - key: readability-identifier-naming.IgnoreMainLikeFunctions
    value: 1
  - key: readability-redundant-member-init.IgnoreBaseInCopyConstructors
    value: 1
  - key: modernize-use-default-member-init.UseAssignment
    value: 1
```

最后进行git的相关配置，首先是`.gitignore`:

```
# build files
build/

# clangd cache files
.cache/

# ide files
.vs/
.idea/

# vim files
*.swp
*.swo

# Compiled Object files
*.slo
*.lo
*.o
*.obj

# Compiled Dynamic libraries
*.so
*.dylib
*.dll

# Compiled Static libraries
*.lai
*.la
*.a
*.lib

# Executables
*.exe
*.out
*.app
```

然后是`.gitattributes`，目前只对eol进行配置：

```
# Set the default behavior, in case people don't have core.autocrlf set.
* text=auto
# Explicitly declare text files you want to always be normalized and converted
# to native line endings on checkout.
*.c text
*.cpp text
*.h text
*.hpp text
*.txt text
*.md text
LICENCE text
.clang-format text
.clang-tidy text
.gitattributes text
.gitignore text
# Declare files that will always have CRLF line endings on checkout
*.bat text eol=crlf
*.ps1 text eol=crlf
```
