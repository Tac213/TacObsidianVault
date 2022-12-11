pip install PySide6之后就会有pyside6-qmllint和pyside6-qmlformat这2个可执行文件。

## qmllint

可以在项目根目录增加一个`.qmllint.ini`文件对qmlformat进行配置：

```ini
[General]
AdditionalQmlImportPaths=
DisableDefaultImports=false
DisablePlugins=
OverwriteImportTypes=
ResourcePath=

[Warnings]
AttachedPropertyReuse=disable
BadSignalHandler=warning
CompilerWarnings=disable
ControlsSanity=disable
DeferredPropertyId=warning
Deprecated=warning
ImportFailure=warning
InheritanceCycle=warning
MultilineStrings=info
PropertyAlias=warning
RequiredProperty=warning
TypeError=warning
UnknownProperty=warning
UnqualifiedAccess=warning
UnusedImports=info
WithStatement=warning
LintPluginWarnings=warning
```

该配置文件可以通过以下命令自动生成：

```bash
pyside6-qmllint --write-defaults
```

General中的配置和pyside6-qmllint中的部分参数是等价的：

- AdditionalQmlImportPaths: `--qmldirs`, Look for QML modules in specified directory
- DisableDefaultImports: `--bare`, Do not include default import directories or the current directory. This may be used to run qmllint on a project using a different Qt version.
- DisablePlugins: `--disable-plugins`, List of qmllint plugins to disable (all to disable all plugins)
- OverwriteImportTypes: `--qmltypes`, Import the specified qmldir files. By default, the qmldir file found in the current directory is used if present. If no qmldir file is found,but qmltypes files are, those are imported instead. When this option is set, you have to explicitly add the qmldir or any qmltypes files in the current directory if you want it to be used. Importing qmltypes files without their corresponding qmldir file is inadvisable.
- ResourcePath: `--resource`, Look for related files in the given resource file

Warnings用来配置不同分类报错的等级，有3个等级：

- disable: 不报错
- info: 显示错误但不影响pyside6-qmllint的returncode
- warning: 显示错误且影响pyside6-qmllint的returncode

Warnings配置和以下pyside6-qmllint中的以下参数是等价的：

- AttachedPropertyReuse:
- BadSignalHandler: `--signal`,  Warn about bad signal handler parameters (default: warning)
- CompilerWarnings: `--compliler`, Warn about compiler issues (default: disable)
- ControlsSanity: `--controls-sanity`, Performance checks used for QuickControl's implementation (default: disable)
- DeferedPropertyId: `--deferred-property-id`, Warn about making deferred properties immediate by giving them an id. (default: warning)
- Deprecated: `--deprecated`, Warn about deprecated properties and types (default: warning)
- ImportFailure: `--import`, Warn about failing imports and deprecated qmltypes (default: warning)
- InheritanceCycle: `--inheritance-cycle`, Warn about inheritance cycles (default: warning)
- LintPluginWarnings: `--plugin`, Warn if a qmllint plugin finds an issue (default: warning)
- MultilineStrings: `--multiline-strings`, Warn about multiline strings (default: info)
- PropertyAlias: `--alias`, Warn about alias errors (default: warning)
- RequiredProperty: `--required`,  Warn about required properties (default: warning)
- TypeError: `--type`,  Warn about unresolvable types and type mismatches (default: warning)
- UnknownProperty: `--property`,  Warn about unknown properties (default: warning)
- UnqualifiedAccess: `--unqualified`, Warn about unqualified identifiers and how to fix them (default: warning)
- UnusedImports: `--unsed-imports`, Warn about unused imports (default: info)
- WithStatement: `--with`, Warn about with statements as they can cause false positives when checking for unqualified access (default: warning)

有些错误可能是qmllint误报，或者确实可以这么用，可以使用以下方法disable:

```qml
ToolTip {
    visible: parent.hovered  // qmllint disable type
    text: qsTr("clear output window")
    delay: 1000
    timeout: 5000
}

InteractiveInterpreter {  // qmllint disable type import
    id: interactiveInterpreter
}

Item {
    property string foo
    Item {
        // qmllint disable unqualified
        property string bar: foo
        property string bar2: foo
        // qmllint enable unqualified
    }
}
```

disable后面跟的是报错category，报错category指的是Warnings配置下各配置对应的参数去掉前面的`--`。比如property type unsed-imports之类的。disable后面不写东西就是全部忽略。

如果有多个报错，报错category之间用空格隔开即可。

通过qmllint，可以标记一些不再使用的属性或者类型：

```qml
@Deprecated { reason: "Use NewCustomText instead" }
Text {
    @Deprecated { reason: "Use newProperty instead" }
    property int oldProperty
    property int newProperty
    Component.onCompleted: console.log(oldProperty);  // Warning: XY.qml:8:40: Property "oldProperty" is deprecated (Reason: Use newProperty instead)
}
```

## qmlformat

可以在项目根目录增加一个`.qmlformat.ini`文件对qmlformat进行配置：

```ini
[General]
IndentWidth=4
NewlineType=native
NormalizeOrder=
UseTabs=
```

配置项和qmlformat的参数是一一对应的:

- IndentWidth: `--indent-width`, How many spaces are used when indenting.
- NewlineType: `--newline`, Override the new line format to use (native macos unix windows).
- NormalizeOrder: `--normalize`, Reorders the attributes of the objects according to the QML Coding Guidelines. 等号后面写个`true`就会重新排序所有的属性
- UseTabs: `--tabs`, Use tabs instead of spaces. 等号后面写个`true`就会用tab

可以同时处理多个文件，如果不加`--inplace`参数的话是不会重写原本文件的只会将结果输出到stdout。

如果加`--inplace`参数就会重写原本的文件，但原本的文件也会备份到`xxx.qml~`中。
