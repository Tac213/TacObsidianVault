执行flutter sdk中提供的flutter脚本或者dart脚本时，最终执行的都是`flutter/bin/internal/shared`脚本中的`shared::execute`这个函数。

下面这几个路径很重要：

- `flutter_tools`路径: `flutter/packages/flutter_tools`
- `flutter_tools`的snapshot路径: `flutter/bin/cache/flutter_tools.snapshot`
- `flutter_tools`的stamp文件路径: `flutter/bin/cache/flutter_tools.stamp`，里面存了当前flutter版本的git提交sha1
- `flutter_tools`的入口脚本路径: `flutter/packages/flutter_tools/bin/flutter_tools.dart`
- dart sdk路径: `flutter/bin/cache/dart-sdk`
- dart可执行二进制文件路径: `flutter/bin/cache/dart-sdk/dart`

这个函数要求flutter必须是一个git目录而且本地有git。

接着调用`upgrade_flutter`函数来更新flutter，这个函数在每次执行flutter脚本或者dart脚本时都会被执行。

该函数首先通过`git rev-parse HEAD`拿到当前HEAD的sha1，准备和stamp文件中缓存的进行对比。当以下任意条件成立时，说明要重新生成cache中的内容：

- 不存在`flutter_tools`的snapshot
- `flutter_tools`的stamp文件不存在
- `flutter_tools`的stamp文件内容为空
- `flutter_tools`的stamp文件内容和当前HEAD的sha1不匹配(里面完整的内容是`$sha1:$FLUTTER_TOOLS_ARGS`，`FLUTTER_TOOLS_ARGS`可以取消注释flutter脚本中的第18和第19行用于允许debug)
- `flutter_tools`的`pubspec.yaml`的修改时间晚于`pubspec.lock`

生成cache时，会移除`flutter/version`文件(该文件存有当前flutter信息)，生成`flutter/bin/cache/.dartignore`文件，并执行`flutter/bin/internal/update_dart_sdk.sh`脚本来获取dart sdk。

dart获取完之后，在`flutter_tools`目录执行`dart pub upgrade --no-precompile`升级pub依赖。接着，执行`dart --verbosity=error --disable-dart-dev $FLUTTER_TOOL_ARGS --snapshot="$SNAPSHOT_PATH" --packages="$FLUTTER_TOOLS_DIR/.dart_tool/package_config.json" --no-enable-mirrors "$SCRIPT_PATH"`编译flutter tool，编译完成后写入stamp文件。

flutter更新完成后，如果是从`dart`脚本调用过来的，直接把参数原封不动传给dart可执行二进制文件来执行程序，如果是从`flutter`脚本调用过来，仍然是执行dart二进制可执行文件，只不过程序参数有点区别，前面加了一些修饰:

```bash
exec "$DART" --disable-dart-dev --packages="$FLUTTER_TOOLS_DIR/.dart_tool/package_config.json" $FLUTTER_TOOL_ARGS "$SNAPSHOT_PATH" "$@"
```

dart执行的是这个:

```bash
exec "$DART" "$@"
```

## 下载dart sdk

`flutter/bin/internal/update_dart_sdk.sh`脚本用来更新或下载dart sdk。

下面这几个路径很重要：

- dart sdk路径：`flutter/bin/cache/dart-sdk`
- 旧dart sdk路径：`flutter/bin/cache/dart-sdk.old`
- dart引擎stamp文件：`flutter/bin/cache/engine-dart-sdk.stamp`
- dart引擎版本：`flutter/bin/internal/engine.version`

以下任意条件满足才走后续，否则退出脚本：

- 引擎stamp文件不存在
- 引擎stamp文件的内容与引擎版本文件内容不相同

该脚本要求本地装有`curl`和`unzip`。

接着根据本地系统和架构(x64 arm64)，分别下载这些zip：

- dart-sdk-darwin-${ARCH}.zip
- dart-sdk-linux-${ARCH}.zip
- dart-sdk-windows-x64.zip(windows只下载x64版本)

具体下载链接为：

```
https://storage.googleapis.com/flutter_infra_release/flutter/$ENGINE_VERSION/$DART_ZIP_NAME
```

如果有设置环境变量`FLUTTER_STORAGE_BASE_URL`(见[文档](https://docs.flutter.dev/community/china))，则为：

```
$FLUTTER_STORAGE_BASE_URL/flutter_infra_release/flutter/$ENGINE_VERSION/$DART_ZIP_NAME
```

下载前会先把dart sdk备份到旧dart sdk路径，然后开始用curl下载，下载完成后unzip，并删掉zip文件和旧dart sdk备份目录。
