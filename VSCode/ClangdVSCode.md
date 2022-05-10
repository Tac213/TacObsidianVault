[llvm-vs-code-extensions.vscode-clangd](https://marketplace.visualstudio.com/items?itemName=llvm-vs-code-extensions.vscode-clangd)拓展使用Clangd作为language server做cpp的静态代码分析和代码提示。

## clangd.arguments

这个配置clangd的一些参数，可以通过`clangd -h`查看参数说明，比如可以配置成这样：

```json
{
    "clangd.arguments": [
        "--compile-commands-dir=${workspaceFolder}/build",
        "--completion-style=detailed",
        "--header-insertion=never",
        "--background-index",
        "-j=12"
    ]
}
```

- `--compile-commands-dir`: 指定cmake编译路径，读取cmake生成的compile-commands.json文件，通过这些编译命令给到正确的代码分析；
- `--completion-style=detailed`: clangd代码提示显示参数类型；
- `--header-insertion=never`: 从不自动导入头文件；
- `--background-index`: 在后台完成index；
- `-j=12`: 同时最多开启12个任务；

## Clang tidy

需要在clangd.arguments中增加下面的参数才能启用Clang tidy的功能：

```json
{
    "clangd.arguments": [
        "--clang-tidy"
    ]
}
```

> 启用clang tidy功能后，项目根目录中必须要有.clang-tidy文件，该功能才能正常显示黄线和红线

使用clang-tidy之前，需要安装clang-tidy。windows下载时已经带有clang-tidy，无需下载。

Linux下载(Ubuntu):

```shell
sudo apt install clang-tidy
```

Mac下载:

```shell
brew install fmenezes/tap/clang-tidy
```

Clang Tidy是类似于[[ESLint & Prettier#ESLint]]的代码规范检查工具，类似ESLint，可以直接运行于命令行，比如：

```shell
clang-tidy hello.cpp -checks=cppcoreguidelines-*
```

上面的命令行可以检查所有cppcoreguidelines的相关错误。

如果需要自定义检查规则，需要在项目根目录创建一个`.clang-tidy`文件。这个文件是个yaml文件，使用前，可以现在settings.json增加下面的配置：

```json
{
    "files.associations": {
        ".clang-tidy": "yaml"
    }
}
```

`.clang-tidy`可配置项参考[官方文档](https://clang.llvm.org/extra/clang-tidy/)、[clang-tidy-checks](https://clang.llvm.org/extra/clang-tidy/checks/list.html)、[llvm项目clang-tidy配置](https://github.com/llvm/llvm-project/blob/main/.clang-tidy)。

变量名规范有下面几种([参考文档](https://clang.llvm.org/extra/clang-tidy/checks/readability-identifier-naming.html))：

- `lower_case`
- `UPPER_CASE`
- `camelBack`
- `CamelCase`
- `camel_Snake_Back`
- `Camel_Snake_Case`
- `aNy_CasE`

变量名检查项有下面几种：

- ClassCase: 类
- StructCase: 结构体
- TypedefCase: typedef
- EnumCase: 枚举
- EnumConstantCase: 枚举常量
- UnionCase: 联合
- NamespaceCase: 命名空间
- FunctionCase: 函数
- VariableCase: 变量
- ConstantCase: 常量

示例`.clang-tidy`:

```yaml
Checks: '-*,clang-diagnostic-*,llvm-*,misc-*,-misc-unused-parameters,-misc-non-private-member-variables-in-classes,-misc-no-recursion,readability-identifier-naming,cppcoreguidelines-*'
CheckOptions:
  - key: readability-identifier-naming.ClassCase
    value: CamelCase
  - key: readability-identifier-naming.EnumCase
    value: CamelCase
  - key: readability-identifier-naming.FunctionCase
    value: lower_case
  - key: readability-identifier-naming.MemberCase
    value: lower_case
  - key: readability-identifier-naming.ParameterCase
    value: lower_case
  - key: readability-identifier-naming.UnionCase
    value: CamelCase
  - key: readability-identifier-naming.VariableCase
    value: lower_case
  - key: readability-identifier-naming.IgnoreMainLikeFunctions
    value: 1
  - key: readability-redundant-member-init.IgnoreBaseInCopyConstructors
    value: 1
  - key: modernize-use-default-member-init.UseAssignment
    value: 1
```

Checks字段对应的就是命令行中传的checks参数，CheckOptions就是一些Check字段的额外可配置参数。如果有了`.clang-tidy`，命令行执行`clang-tidy`时就不需要传参数了。

示例例中用到的checks:

- misc-unused-parameters: 未使用函数参数检查，示例中去掉了这个检查；
- misc-non-private-member-variables-in-classes: 类中存在非私有成员变量检查，示例中去掉了这个检查；
- misc-no-recursion: 不能使用递归的检查，示例中去掉了这个检查；
- readability-identifier-naming: 变量名命名规范检查；

可以用`//NOLINT`之类的注释忽略lint，比如:

```cpp
#include <iostream>

int main() {
    auto array = {0, 1, 2, 3, 4};

    int /* NOLINTBEGIN */ v1 /* NOLINTEND */, v2;
    std::cin >> v1 >> v2;
    for (int val : array) {
        std::printf("%d, ", val); // NOLINT(cppcoreguidelines-pro-type-vararg)
    }

    // NOLINTNEXTLINE
    std::printf("Hello world\n");
    return 0;
}
```

也可以在checks中，`-`开头的参数表示不适用该规则，比如`-*`去掉默认的所有规则，`-misc-unused-parameters`去掉未使用参数的规则。

还有一些见过的配置，比如：

```yaml
# 函数的statement不超过50个
- key: readability-funcion-size.StatementThreshold
  value: 50
# 函数的参数不超过5个
- key: readability-funcion-size.ParameterThreshold
  value: 5
```

语句必须在大括号内：

```yaml
Checks: 'readability-braces-around-statements'
```

魔鬼数字检查：

```yaml
Checks: 'readability-magic-numbers'
```

使用nullptr而不是NULL或0：

```yaml
Checks: 'modernize-use-nullptr'
```

禁止使用auto_ptr：

```yaml
Checks: 'modernize-replace-auto-ptr'
```

不使用异常机制：

```yaml
Checks: 'modernize-use-noexcept'
```

重写虚函数时需要使用override关键字：

```yaml
Checks: 'modernize-use-override'
```

禁止使用std::move操作const对象：

```yaml
Checks: 'performance-move-const-arg'
```

不使用C风格转换：

```yaml
Checks: 'cppcoreguidelines-pro-type-cstyle-cast'
```

不使用reinterpret_cast：

```yaml
Checks: 'cppcoreguidelines-pro-type-reinterpret-cast'
```

不使用const_cast：

```yaml
Checks: 'cppcoreguidelines-pro-type-const-cast'
```

## Clang Format

类似于[[ESLint & Prettier#Prettier]]的代码格式化工具，配置`.clang-format`是个yaml文件，可配置项[参考文档](https://clang.llvm.org/docs/ClangFormatStyleOptions.html)。

使用前，可以先在settings.json增加下面的配置：

```json
{
    "files.associations": {
        ".clang-format": "yaml"
    },
    "[cpp]": {
        "editor.defaultFormatter": "llvm-vs-code-extensions.vscode-clangd",
        "editor.formatOnSave": true
    },
    "[c]": {
        "editor.defaultFormatter": "llvm-vs-code-extensions.vscode-clangd",
        "editor.formatOnSave": true
    }
}
```

这样在编辑器里面save的时候就会自动格式化代码。`.clang-format`文件也有代码高亮。

简单的`.clang-format`配置如下：

```yaml
BasedOnStyle: LLVM
UseTab: Never
IndentWidth: 4
AccessModifierOffset: -4
```

详细可以参考[这篇文章](https://zhuanlan.zhihu.com/p/356143396)或者[官方文档](https://clang.llvm.org/docs/ClangFormatStyleOptions.html)。

摘抄如下：

```yaml
# 语言: None, Cpp, Java, JavaScript, ObjC, Proto, TableGen, TextProto
Language: Cpp
# BasedOnStyle:	LLVM

# 访问说明符(public、private等)的偏移
AccessModifierOffset: -4

# 开括号(开圆括号、开尖括号、开方括号)后的对齐: Align, DontAlign, AlwaysBreak(总是在开括号后换行)
AlignAfterOpenBracket: Align

# 连续赋值时，对齐所有等号
AlignConsecutiveAssignments: false

# 连续声明时，对齐所有声明的变量名
AlignConsecutiveDeclarations: false

# 右对齐逃脱换行(使用反斜杠换行)的反斜杠
AlignEscapedNewlines: Right

# 水平对齐二元和三元表达式的操作数
AlignOperands: true

# 对齐连续的尾随的注释
AlignTrailingComments: true

# 不允许函数声明的所有参数在放在下一行
AllowAllParametersOfDeclarationOnNextLine: false

# 不允许短的块放在同一行
AllowShortBlocksOnASingleLine: true

# 允许短的case标签放在同一行
AllowShortCaseLabelsOnASingleLine: true

# 允许短的函数放在同一行: None, InlineOnly(定义在类中), Empty(空函数), Inline(定义在类中，空函数), All
AllowShortFunctionsOnASingleLine: None

# 允许短的if语句保持在同一行
AllowShortIfStatementsOnASingleLine: true

# 允许短的循环保持在同一行
AllowShortLoopsOnASingleLine: true

# 总是在返回类型后换行: None, All, TopLevel(顶级函数，不包括在类中的函数), 
# AllDefinitions(所有的定义，不包括声明), TopLevelDefinitions(所有的顶级函数的定义)
AlwaysBreakAfterReturnType: None

# 总是在多行string字面量前换行
AlwaysBreakBeforeMultilineStrings: false

# 总是在template声明后换行
AlwaysBreakTemplateDeclarations: true

# false表示函数实参要么都在同一行，要么都各自一行
BinPackArguments: true

# false表示所有形参要么都在同一行，要么都各自一行
BinPackParameters: true

# 大括号换行，只有当BreakBeforeBraces设置为Custom时才有效
BraceWrapping:
  # class定义后面
  AfterClass: false
  # 控制语句后面
  AfterControlStatement: false
  # enum定义后面
  AfterEnum: false
  # 函数定义后面
  AfterFunction: false
  # 命名空间定义后面
  AfterNamespace: false
  # struct定义后面
  AfterStruct: false
  # union定义后面
  AfterUnion: false
  # extern之后
  AfterExternBlock: false
  # catch之前
  BeforeCatch: false
  # else之前
  BeforeElse: false
  # 缩进大括号
  IndentBraces: false
  # 分离空函数
  SplitEmptyFunction: false
  # 分离空语句
  SplitEmptyRecord: false
  # 分离空命名空间
  SplitEmptyNamespace: false

# 在二元运算符前换行: None(在操作符后换行), NonAssignment(在非赋值的操作符前换行), All(在操作符前换行)
BreakBeforeBinaryOperators: NonAssignment

# 在大括号前换行: Attach(始终将大括号附加到周围的上下文), Linux(除函数、命名空间和类定义，与Attach类似), 
#   Mozilla(除枚举、函数、记录定义，与Attach类似), Stroustrup(除函数定义、catch、else，与Attach类似), 
#   Allman(总是在大括号前换行), GNU(总是在大括号前换行，并对于控制语句的大括号增加额外的缩进), WebKit(在函数前换行), Custom
#   注：这里认为语句块也属于函数
BreakBeforeBraces: Custom

# 在三元运算符前换行
BreakBeforeTernaryOperators: false

# 在构造函数的初始化列表的冒号后换行
BreakConstructorInitializers: AfterColon

#BreakInheritanceList: AfterColon

BreakStringLiterals: false

# 每行字符的限制，0表示没有限制
ColumnLimit: 0

CompactNamespaces: true

# 构造函数的初始化列表要么都在同一行，要么都各自一行
ConstructorInitializerAllOnOneLineOrOnePerLine: false

# 构造函数的初始化列表的缩进宽度
ConstructorInitializerIndentWidth: 4

# 延续的行的缩进宽度
ContinuationIndentWidth: 4

# 去除C++11的列表初始化的大括号{后和}前的空格
Cpp11BracedListStyle: true

# 继承最常用的指针和引用的对齐方式
DerivePointerAlignment: false

# 固定命名空间注释
FixNamespaceComments: true

# 缩进case标签
IndentCaseLabels: false

IndentPPDirectives: None

# 缩进宽度
IndentWidth: 4

# 函数返回类型换行时，缩进函数声明或函数定义的函数名
IndentWrappedFunctionNames: false

# 保留在块开始处的空行
KeepEmptyLinesAtTheStartOfBlocks: false

# 连续空行的最大数量
MaxEmptyLinesToKeep: 1

# 命名空间的缩进: None, Inner(缩进嵌套的命名空间中的内容), All
NamespaceIndentation: None

# 指针和引用的对齐: Left, Right, Middle
PointerAlignment: Right

# 允许重新排版注释
ReflowComments: true

# 允许排序#include
SortIncludes: false

# 允许排序 using 声明
SortUsingDeclarations: false

# 在C风格类型转换后添加空格
SpaceAfterCStyleCast: false

# 在Template 关键字后面添加空格
SpaceAfterTemplateKeyword: true

# 在赋值运算符之前添加空格
SpaceBeforeAssignmentOperators: true

# SpaceBeforeCpp11BracedList: true

# SpaceBeforeCtorInitializerColon: true

# SpaceBeforeInheritanceColon: true

# 开圆括号之前添加一个空格: Never, ControlStatements, Always
SpaceBeforeParens: ControlStatements

# SpaceBeforeRangeBasedForLoopColon: true

# 在空的圆括号中添加空格
SpaceInEmptyParentheses: false

# 在尾随的评论前添加的空格数(只适用于//)
SpacesBeforeTrailingComments: 1

# 在尖括号的<后和>前添加空格
SpacesInAngles: false

# 在C风格类型转换的括号中添加空格
SpacesInCStyleCastParentheses: false

# 在容器(ObjC和JavaScript的数组和字典等)字面量中添加空格
SpacesInContainerLiterals: true

# 在圆括号的(后和)前添加空格
SpacesInParentheses: false

# 在方括号的[后和]前添加空格，lamda表达式和未指明大小的数组的声明不受影响
SpacesInSquareBrackets: false

# 标准: Cpp03, Cpp11, Auto
Standard: Cpp11

# tab宽度
TabWidth: 4

# 使用tab字符: Never, ForIndentation, ForContinuationAndIndentation, Always
UseTab: Never
```

一个推荐的clang-format配置：

```yaml
Language:        Cpp
# BasedOnStyle:  Google
AccessModifierOffset: -1
AlignAfterOpenBracket: AlwaysBreak
AlignConsecutiveAssignments: false
AlignConsecutiveDeclarations: false
AlignEscapedNewlines: Left
AlignOperands:   true
AlignTrailingComments: true
AllowAllParametersOfDeclarationOnNextLine: false
AllowShortBlocksOnASingleLine: true
AllowShortCaseLabelsOnASingleLine: true
AllowShortFunctionsOnASingleLine: All
AllowShortIfStatementsOnASingleLine: true
AllowShortLoopsOnASingleLine: true
AlwaysBreakAfterDefinitionReturnType: None
AlwaysBreakAfterReturnType: None
AlwaysBreakBeforeMultilineStrings: true
AlwaysBreakTemplateDeclarations: Yes
BinPackArguments: false
BinPackParameters: false
BraceWrapping:   
  AfterClass:      false
  AfterControlStatement: false
  AfterEnum:       false
  AfterFunction:   false
  AfterNamespace:  false
  AfterObjCDeclaration: false
  AfterStruct:     false
  AfterUnion:      false
  AfterExternBlock: false
  BeforeCatch:     false
  BeforeElse:      false
  IndentBraces:    false
  SplitEmptyFunction: true
  SplitEmptyRecord: true
  SplitEmptyNamespace: true
BreakBeforeBinaryOperators: None
BreakBeforeBraces: Attach
BreakBeforeInheritanceComma: false
BreakInheritanceList: BeforeColon
BreakBeforeTernaryOperators: true
BreakConstructorInitializersBeforeComma: false
BreakConstructorInitializers: BeforeColon
BreakAfterJavaFieldAnnotations: false
BreakStringLiterals: true
ColumnLimit:     80
CommentPragmas:  '^ IWYU pragma:'
CompactNamespaces: false
ConstructorInitializerAllOnOneLineOrOnePerLine: true
ConstructorInitializerIndentWidth: 4
ContinuationIndentWidth: 4
Cpp11BracedListStyle: true
DerivePointerAlignment: true
DisableFormat:   false
ExperimentalAutoDetectBinPacking: false
FixNamespaceComments: true
ForEachMacros:   
  - foreach
  - Q_FOREACH
  - BOOST_FOREACH
IncludeBlocks:   Preserve
IncludeCategories: 
  - Regex:           '^<ext/.*\.h>'
    Priority:        2
  - Regex:           '^<.*\.h>'
    Priority:        1
  - Regex:           '^<.*'
    Priority:        2
  - Regex:           '.*'
    Priority:        3
IncludeIsMainRegex: '([-_](test|unittest))?$'
IndentCaseLabels: true
IndentPPDirectives: None
IndentWidth:     2
IndentWrappedFunctionNames: false
JavaScriptQuotes: Leave
JavaScriptWrapImports: true
KeepEmptyLinesAtTheStartOfBlocks: false
MacroBlockBegin: ''
MacroBlockEnd:   ''
MaxEmptyLinesToKeep: 1
NamespaceIndentation: None
ObjCBinPackProtocolList: Never
ObjCBlockIndentWidth: 2
ObjCSpaceAfterProperty: false
ObjCSpaceBeforeProtocolList: true
PenaltyBreakAssignment: 2
PenaltyBreakBeforeFirstCallParameter: 1
PenaltyBreakComment: 300
PenaltyBreakFirstLessLess: 120
PenaltyBreakString: 1000
PenaltyBreakTemplateDeclaration: 10
PenaltyExcessCharacter: 1000000
PenaltyReturnTypeOnItsOwnLine: 30
PointerAlignment: Left
RawStringFormats: 
  - Language:        Cpp
    Delimiters:      
      - cc
      - CC
      - cpp
      - Cpp
      - CPP
      - 'c++'
      - 'C++'
    CanonicalDelimiter: ''
    BasedOnStyle:    google
  - Language:        TextProto
    Delimiters:      
      - pb
      - PB
      - proto
      - PROTO
    EnclosingFunctions: 
      - EqualsProto
      - EquivToProto
      - PARSE_TEST_PROTO
      - PARSE_TEXT_PROTO
      - ParseTextOrDie
    CanonicalDelimiter: ''
    BasedOnStyle:    google
ReflowComments:  true
SortIncludes:    true
SortUsingDeclarations: true
SpaceAfterCStyleCast: false
SpaceAfterTemplateKeyword: true
SpaceBeforeAssignmentOperators: true
SpaceBeforeCpp11BracedList: false
SpaceBeforeCtorInitializerColon: true
SpaceBeforeInheritanceColon: true
SpaceBeforeParens: ControlStatements
SpaceBeforeRangeBasedForLoopColon: true
SpaceInEmptyParentheses: false
SpacesBeforeTrailingComments: 2
SpacesInAngles:  false
SpacesInContainerLiterals: true
SpacesInCStyleCastParentheses: false
SpacesInParentheses: false
SpacesInSquareBrackets: false
Standard:        Auto
TabWidth:        4
UseTab:          Never
```

Mac下如果格式化功能失效，可以尝试安装clang-format

```shell
brew install clang-format
```
