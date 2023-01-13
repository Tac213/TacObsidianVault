## MacOS

直接从git仓库下载即可:

```bash
git clone https://github.com/flutter/flutter.git -b stable
```

把clone下来的flutter目录的bin目录放到`$PATH`变量中，接着执行：

```bash
flutter doctor
```

如果提示http无法访问google网站则为科学上网问题，比如通过如下方法：

```bash
export HTTPS_PROXY=http://127.0.0.1:7890 HTTP_PROXY=http://127.0.0.1:7890 ALL_PROXY=socks5://127.0.0.1:7891
export NO_PROXY=localhost,127.0.0.1,::1
```

然后再执行`flutter doctor`即可。

MacOS还需要安装CocoaPods，执行如下命令：

```bash
sudo gem install cocoapods
```

注意如果是从git仓库下载的话会flutter会下载Dart SDK并编译flutter tools，如果直接从网站上下载Flutter SDK则不需要这一步。

## Windows

和MacOS一致也是直接从git仓库下载即可。需要安装最新的Visual Studio。

## WSL

和MacOS一致也是直接从git仓库下载即可。需要额外安装的软件参考[官方文档](https://docs.flutter.dev/get-started/install/linux)。

但是WSL上在ide运行应用失败，提示：

```
Error connecting to the service protocol: failed to connect to http://127.0.0.1:33497/GVQTN9cS5u0=/
```

表现是界面能弹出来但是里面没有任何东西，但是在项目内部运行下面的命令是可以正常弹出app的，虽然也会弹上面这个错：

```bash
flutter run -d linux
```

## Arch

```bash
sudo pacman -S git unzip xz zip glu which
sudo pacman -S clang cmake ninja pkg-config gtk3
```
