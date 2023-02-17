`flutter_tools`这个dart package负责执行`flutter`命令行的任务，入口在`executable.dart`中的`main`函数。有以下几个概念：

- verbose: log详细运行记录，传`-v`或者`--verbose`或者`-vv`(very vebose)的时候为`true`
- prefixedErros: 传`prefixedErrors`的时候为`true`，`ERROR:`开头的输出会输出到`stderr`
- doctor: 运行`flutter doctor`或者`flutter -v doctor`时候为`true`
- help: 传`-h`或者`--help`，或者运行`flutter help xxx`的时候为true
- muteCommandLogging: `(help || doctor) && !veryVerbose`
- verboseHelp: verbose和help同时为`true`的时候为`true`
- daemon: flutter命令行带有`daemon`的时候为`true`
- machine: 传`--machine`并且执行`flutter run`，或者传`--machine`并且执行`flutter attach`，为`true`

`executable.dart`中的`generateCommands`函数提供了flutter命令行可以执行的command列表。

`runner.dart`中的`run`函数接收上述command列表，实例化`FlutterCommandRunner`，将command列表中的Command加入`FlutterCommandRunner`的实例，然后调用`runner.run(args)`函数执行command。执行command的最终实现在`FlutterCommandRunner`的`runCommand`函数中，这个函数实际上只是对一些通用参数做了一些封装，比如`--version`, `--suppress-analytics`, `--quiet`等等，最终调用的还是`FlutterCommand`实例的`run`函数。

`FlutterCommand`的子类不应该override `run`函数，而是应该override `verifyThenRunCommand`和`runCommand`函数，大多数情况下只需要override后者。所有的command都在`flutter_tools/lib/src/commands`目录下。

## doctor

真正的实现在`flutter_tools/lib/src/doctor.dart`。

首先调用`globals.flutterVersion.fetchTagsAndUpdate()`获取flutter版本信息。

如果有传`--check-for-remote-artifacts`参数，则调用`globals.doctor?.checkRemoteArtifacts(engineRevision)`检查远端zip好的artifact对应的url是否存在，检查函数如下:

```dart
  Future<bool> checkForArtifacts(String? engineVersion) async {
    engineVersion ??= version;
    final String url = '${cache.storageBaseUrl}/flutter_infra_release/flutter/$engineVersion/';

    bool exists = false;
    for (final String pkgName in getPackageDirs()) {
      exists = await cache.doesRemoteExist('Checking package $pkgName is available...', Uri.parse('$url$pkgName.zip'));
      if (!exists) {
        return false;
      }
    }

    for (final List<String> toolsDir in getBinaryDirs()) {
      final String cacheDir = toolsDir[0];
      final String urlPath = toolsDir[1];
      exists = await cache.doesRemoteExist('Checking $cacheDir tools are available...',
          Uri.parse(url + urlPath));
      if (!exists) {
        return false;
      }
    }
    return true;
  }
```

其中packageDir和binaryDir为：

```dart
  @override
  List<String> getPackageDirs() => const <String>['sky_engine'];

  @override
  List<List<String>> getBinaryDirs() {
    // Currently only Linux supports both arm64 and x64.
    final String arch = cache.getHostPlatformArchName();
    return <List<String>>[
      <String>['common', 'flutter_patched_sdk.zip'],
      <String>['common', 'flutter_patched_sdk_product.zip'],
      if (cache.includeAllPlatforms) ...<List<String>>[
        <String>['windows-x64', 'windows-x64/artifacts.zip'],
        <String>['linux-$arch', 'linux-$arch/artifacts.zip'],
        <String>['darwin-x64', 'darwin-x64/artifacts.zip'],
      ]
      else if (_platform.isWindows)
        <String>['windows-x64', 'windows-x64/artifacts.zip']
      else if (_platform.isMacOS)
        <String>['darwin-x64', 'darwin-x64/artifacts.zip']
      else if (_platform.isLinux)
        <String>['linux-$arch', 'linux-$arch/artifacts.zip'],
    ];
  }

  @override
  List<String> getLicenseDirs() => const <String>[];
```

对应的url为：

```
https://storage.googleapis.com/flutter_infra_release/flutter/$ENGINE_VERSION/$DART_ZIP_NAME
```

比如某个3.3.10的darwin x64的artifacts(复制链接到浏览器可以下载):

```
https://storage.googleapis.com/flutter_infra_release/flutter/3316dd8728419ad3534e3f6112aa6291f587078a/darwin-x64/artifacts.zip
```

各平台的二进制:

```dart
const List<List<String>> _windowsDesktopBinaryDirs = <List<String>>[
  <String>['windows-x64', 'windows-x64/windows-x64-flutter.zip'],
  <String>['windows-x64', 'windows-x64/flutter-cpp-client-wrapper.zip'],
  <String>['windows-x64-profile', 'windows-x64-profile/windows-x64-flutter.zip'],
  <String>['windows-x64-release', 'windows-x64-release/windows-x64-flutter.zip'],
];

const List<List<String>> _macOSDesktopBinaryDirs = <List<String>>[
  <String>['darwin-x64', 'darwin-x64/FlutterMacOS.framework.zip'],
  <String>['darwin-x64', 'darwin-x64/gen_snapshot.zip'],
  <String>['darwin-x64-profile', 'darwin-x64-profile/FlutterMacOS.framework.zip'],
  <String>['darwin-x64-profile', 'darwin-x64-profile/artifacts.zip'],
  <String>['darwin-x64-profile', 'darwin-x64-profile/gen_snapshot.zip'],
  <String>['darwin-x64-release', 'darwin-x64-release/FlutterMacOS.framework.zip'],
  <String>['darwin-x64-release', 'darwin-x64-release/artifacts.zip'],
  <String>['darwin-x64-release', 'darwin-x64-release/gen_snapshot.zip'],
];

const List<List<String>> _osxBinaryDirs = <List<String>>[
  <String>['android-arm-profile/darwin-x64', 'android-arm-profile/darwin-x64.zip'],
  <String>['android-arm-release/darwin-x64', 'android-arm-release/darwin-x64.zip'],
  <String>['android-arm64-profile/darwin-x64', 'android-arm64-profile/darwin-x64.zip'],
  <String>['android-arm64-release/darwin-x64', 'android-arm64-release/darwin-x64.zip'],
  <String>['android-x64-profile/darwin-x64', 'android-x64-profile/darwin-x64.zip'],
  <String>['android-x64-release/darwin-x64', 'android-x64-release/darwin-x64.zip'],
];

const List<List<String>> _linuxBinaryDirs = <List<String>>[
  <String>['android-arm-profile/linux-x64', 'android-arm-profile/linux-x64.zip'],
  <String>['android-arm-release/linux-x64', 'android-arm-release/linux-x64.zip'],
  <String>['android-arm64-profile/linux-x64', 'android-arm64-profile/linux-x64.zip'],
  <String>['android-arm64-release/linux-x64', 'android-arm64-release/linux-x64.zip'],
  <String>['android-x64-profile/linux-x64', 'android-x64-profile/linux-x64.zip'],
  <String>['android-x64-release/linux-x64', 'android-x64-release/linux-x64.zip'],
];

const List<List<String>> _windowsBinaryDirs = <List<String>>[
  <String>['android-arm-profile/windows-x64', 'android-arm-profile/windows-x64.zip'],
  <String>['android-arm-release/windows-x64', 'android-arm-release/windows-x64.zip'],
  <String>['android-arm64-profile/windows-x64', 'android-arm64-profile/windows-x64.zip'],
  <String>['android-arm64-release/windows-x64', 'android-arm64-release/windows-x64.zip'],
  <String>['android-x64-profile/windows-x64', 'android-x64-profile/windows-x64.zip'],
  <String>['android-x64-release/windows-x64', 'android-x64-release/windows-x64.zip'],
];

const List<List<String>> _iosBinaryDirs = <List<String>>[
  <String>['ios', 'ios/artifacts.zip'],
  <String>['ios-profile', 'ios-profile/artifacts.zip'],
  <String>['ios-release', 'ios-release/artifacts.zip'],
];

const List<List<String>> _androidBinaryDirs = <List<String>>[
  <String>['android-x86', 'android-x86/artifacts.zip'],
  <String>['android-x64', 'android-x64/artifacts.zip'],
  <String>['android-arm', 'android-arm/artifacts.zip'],
  <String>['android-arm-profile', 'android-arm-profile/artifacts.zip'],
  <String>['android-arm-release', 'android-arm-release/artifacts.zip'],
  <String>['android-arm64', 'android-arm64/artifacts.zip'],
  <String>['android-arm64-profile', 'android-arm64-profile/artifacts.zip'],
  <String>['android-arm64-release', 'android-arm64-release/artifacts.zip'],
  <String>['android-x64-profile', 'android-x64-profile/artifacts.zip'],
  <String>['android-x64-release', 'android-x64-release/artifacts.zip'],
  <String>['android-x86-jit-release', 'android-x86-jit-release/artifacts.zip'],
];

const List<List<String>> _dartSdks = <List<String>> [
  <String>['darwin-x64', 'dart-sdk-darwin-x64.zip'],
  <String>['linux-x64', 'dart-sdk-linux-x64.zip'],
  <String>['windows-x64', 'dart-sdk-windows-x64.zip'],
];
```

远程url检查没问题，则调用`globals.doctor?.diagnose`诊断当前环境。

doctor实现了一个`_DefaultDoctorValidatorsProvider`，里面有个`validators` getter提供了当前平台的所有`DoctorValidator`。具体实现如下：

```dart
  @override
  List<DoctorValidator> get validators {
    if (_validators != null) {
      return _validators!;
    }

    final List<DoctorValidator> ideValidators = <DoctorValidator>[
      if (androidWorkflow!.appliesToHostPlatform)
        ...AndroidStudioValidator.allValidators(globals.config, globals.platform, globals.fs, globals.userMessages),
      ...IntelliJValidator.installedValidators(
        fileSystem: globals.fs,
        platform: globals.platform,
        userMessages: userMessages,
        plistParser: globals.plistParser,
        processManager: globals.processManager,
      ),
      ...VsCodeValidator.installedValidators(globals.fs, globals.platform, globals.processManager),
    ];
    final ProxyValidator proxyValidator = ProxyValidator(platform: globals.platform);
    _validators = <DoctorValidator>[
      FlutterValidator(
        fileSystem: globals.fs,
        platform: globals.platform,
        flutterVersion: () => globals.flutterVersion,
        devToolsVersion: () => globals.cache.devToolsVersion,
        processManager: globals.processManager,
        userMessages: userMessages,
        artifacts: globals.artifacts!,
        flutterRoot: () => Cache.flutterRoot!,
        operatingSystemUtils: globals.os,
      ),
      if (androidWorkflow!.appliesToHostPlatform)
        GroupedValidator(<DoctorValidator>[androidValidator!, androidLicenseValidator!]),
      if (globals.iosWorkflow!.appliesToHostPlatform || macOSWorkflow.appliesToHostPlatform)
        GroupedValidator(<DoctorValidator>[XcodeValidator(xcode: globals.xcode!, userMessages: userMessages), globals.cocoapodsValidator!]),
      if (webWorkflow.appliesToHostPlatform)
        ChromeValidator(
          chromiumLauncher: ChromiumLauncher(
            browserFinder: findChromeExecutable,
            fileSystem: globals.fs,
            operatingSystemUtils: globals.os,
            platform:  globals.platform,
            processManager: globals.processManager,
            logger: globals.logger,
          ),
          platform: globals.platform,
        ),
      if (linuxWorkflow.appliesToHostPlatform)
        LinuxDoctorValidator(
          processManager: globals.processManager,
          userMessages: userMessages,
        ),
      if (windowsWorkflow!.appliesToHostPlatform)
        visualStudioValidator!,
      if (ideValidators.isNotEmpty)
        ...ideValidators
      else
        NoIdeValidator(),
      if (proxyValidator.shouldShow)
        proxyValidator,
      if (globals.deviceManager?.canListAnything ?? false)
        DeviceValidator(
          deviceManager: globals.deviceManager,
          userMessages: globals.userMessages,
        ),
      HttpHostValidator(
        platform: globals.platform,
        featureFlags: featureFlags,
        httpClient: globals.httpClientFactory?.call() ?? HttpClient(),
      ),
    ];
    return _validators!;
  }
```

其中`FlutterValidation`会检测Artifact是否存在并下载。

`FLUTTER_GIT_URL`环境变量可以指定自定义的flutter git url。

## run

`RunCommand`的`createRunner`函数会根据命令行是否需要热更新来创建`HotRunner`或者`ColdRunner`，这两个runner分别会调用`FlutterDevice`的`runHot`和`runCold`函数(`resisdent_runner.dart`)。这2个函数都会通过下面这句拿到`package`，并通过`package`启动App。

```dart
package = await ApplicationPackageFactory.instance!.getPackageForPlatform(
  targetPlatform,
  buildInfo: coldRunner.debuggingOptions.buildInfo,
  applicationBinary: coldRunner.applicationBinary,
);
```

其中`getPackageForPlatform`由`FlutterApplicationPackageFactory`实现(`flutter_application_package.dart`)，这个函数根据不同的target platform返回对应的`ApplicationPackage`(也可能没有，也就是空的)。

启动App通过不同的`Device`的`startApp`实现，比如`DesktopDevice`, `AndroidDevice`, 等等，位于`flutter_tools/lib/src`。该函数根据用`ApplicationPackage`的`executable`启动App，并传入dart入口函数的对应参数：

```dart
final List<String> command = <String>[
  executable,
  ...debuggingOptions.dartEntrypointArgs,
];
```

同时会根据`debuggingOptions.buildInfo`来决定是否启用`DartDevTools`来调试dart代码。

`DesktopDevice`在启动App之前，如果没有`ApplicationPackage`，会先调用`buildForDevice`方法在对应的平台编译出可执行文件(带有flutter engine)，三个平台分别对应`buildLinux`, `buildMacOS`, `buildWindows`这3个函数。

Android是下面这句:

```dart
      await androidBuilder!.buildApk(
          project: project,
          target: mainPath ?? 'lib/main.dart',
          androidBuildInfo: AndroidBuildInfo(
            debuggingOptions.buildInfo,
            targetArchs: <AndroidArch>[androidArch],
            fastStart: debuggingOptions.fastStart,
            multidexEnabled: (platformArgs['multidex'] as bool?) ?? false,
          ),
      );
```

`IOSDevice`位于`devices.dart`，编译是下面这句：

```dart
      final XcodeBuildResult buildResult = await buildXcodeProject(
          app: package as BuildableIOSApp,
          buildInfo: debuggingOptions.buildInfo,
          targetOverride: mainPath,
          activeArch: cpuArchitecture,
          deviceID: id,
      );
```

对于使用cmake的平台(windows, linux)，在编译前会先通过下面这句生成一个cmake配置:

```dart
writeGeneratedCmakeConfig(Cache.flutterRoot!, windowsProject, buildInfo, environment);
```

作用是生成`linux/flutter/ephemeral/generated_config.cmake`或`windows/flutter/ephemeral/generated_config.cmake`，写入了一些本地配置的环境变量以及一些编译时的必要变量。

编译时，`flutter/CMakeLists.txt`会通过下面这句引入这个生成的配置:

```cmake
# Configuration provided via flutter tool.
include(${EPHEMERAL_DIR}/generated_config.cmake)
```

然后通过下面这里调用`tool_backend`生成`flutter_assemble`(其中`FLUTTER_TOOL_ENVIRONMENT`就是生成配置中所描述的环境变量):

```cmake
# === Flutter tool backend ===
# _phony_ is a non-existent file to force this command to run every time,
# since currently there's no way to get a full input/output list from the
# flutter tool.
set(PHONY_OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/_phony_")
set_source_files_properties("${PHONY_OUTPUT}" PROPERTIES SYMBOLIC TRUE)
add_custom_command(
  OUTPUT ${FLUTTER_LIBRARY} ${FLUTTER_LIBRARY_HEADERS}
    ${CPP_WRAPPER_SOURCES_CORE} ${CPP_WRAPPER_SOURCES_PLUGIN}
    ${CPP_WRAPPER_SOURCES_APP}
    ${PHONY_OUTPUT}
  COMMAND ${CMAKE_COMMAND} -E env
    ${FLUTTER_TOOL_ENVIRONMENT}
    "${FLUTTER_ROOT}/packages/flutter_tools/bin/tool_backend.bat"
      windows-x64 $<CONFIG>
  VERBATIM
)
add_custom_target(flutter_assemble DEPENDS
  "${FLUTTER_LIBRARY}"
  ${FLUTTER_LIBRARY_HEADERS}
  ${CPP_WRAPPER_SOURCES_CORE}
  ${CPP_WRAPPER_SOURCES_PLUGIN}
  ${CPP_WRAPPER_SOURCES_APP}
)
```

而`tool_backend`这个脚本实际上只是对`flutter assemble`命令的一个封装，调用的正是这个命令(源码见`tool_backend.dart`)。

## assemble

该命令的本质是在下面这些已经实现的`Target`中找到当前平台的已实现的`Target`并执行其`build`方法:

```dart
/// All currently implemented targets.
List<Target> _kDefaultTargets = <Target>[
  // Shared targets
  const CopyAssets(),
  const KernelSnapshot(),
  const AotElfProfile(TargetPlatform.android_arm),
  const AotElfRelease(TargetPlatform.android_arm),
  const AotAssemblyProfile(),
  const AotAssemblyRelease(),
  // macOS targets
  const DebugMacOSFramework(),
  const DebugMacOSBundleFlutterAssets(),
  const ProfileMacOSBundleFlutterAssets(),
  const ReleaseMacOSBundleFlutterAssets(),
  // Linux targets
  const DebugBundleLinuxAssets(TargetPlatform.linux_x64),
  const DebugBundleLinuxAssets(TargetPlatform.linux_arm64),
  const ProfileBundleLinuxAssets(TargetPlatform.linux_x64),
  const ProfileBundleLinuxAssets(TargetPlatform.linux_arm64),
  const ReleaseBundleLinuxAssets(TargetPlatform.linux_x64),
  const ReleaseBundleLinuxAssets(TargetPlatform.linux_arm64),
  // Web targets
  WebServiceWorker(globals.fs, globals.cache),
  const ReleaseAndroidApplication(),
  // This is a one-off rule for bundle and aot compat.
  const CopyFlutterBundle(),
  // Android targets,
  const DebugAndroidApplication(),
  const ProfileAndroidApplication(),
  // Android ABI specific AOT rules.
  androidArmProfileBundle,
  androidArm64ProfileBundle,
  androidx64ProfileBundle,
  androidArmReleaseBundle,
  androidArm64ReleaseBundle,
  androidx64ReleaseBundle,
  // Deferred component enabled AOT rules
  androidArmProfileDeferredComponentsBundle,
  androidArm64ProfileDeferredComponentsBundle,
  androidx64ProfileDeferredComponentsBundle,
  androidArmReleaseDeferredComponentsBundle,
  androidArm64ReleaseDeferredComponentsBundle,
  androidx64ReleaseDeferredComponentsBundle,
  // iOS targets
  const DebugIosApplicationBundle(),
  const ProfileIosApplicationBundle(),
  const ReleaseIosApplicationBundle(),
  // Windows targets
  const UnpackWindows(),
  const DebugBundleWindowsAssets(),
  const ProfileBundleWindowsAssets(),
  const ReleaseBundleWindowsAssets(),
];
```

比如linux和windows平台都有一个unpack的target，该命令是把对应平台的artifact解压到`flutter/ephemeral`目录，即引擎的动态库二进制文件以及头文件，引擎在编译对应平台的可执行文件时被链接进去。
