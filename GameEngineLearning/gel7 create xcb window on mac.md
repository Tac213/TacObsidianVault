mac理论上是可以和linux一样用同一套代码进行编译的。也可以使用xcb创建窗口，但是效果比较古老，不是macos原生的。不过这么做也能创建窗口，因此记录以下。

首先需要下载并安装[XQuartz](https://www.xquartz.org/)。

然后下载依赖库：

```shell
brew install libx11
brew install libxcb
```

接着修改Game.cpp，在mac上也使用XcbApplication创建应用实例。

最后修改Core/CMakeLists.txt，跟linux一样，在mac上也用find_library找xcb和x11，并为Core额外链接库。

此时编译会报错，仍然找不到`xcb/xcb.h`这个文件，此时需要在根目录的CMakeLists.txt增加下面这一行，才能成功编译。

```cmake
include_directories("/usr/local/Cellar/libxcb/1.15/include")
```
