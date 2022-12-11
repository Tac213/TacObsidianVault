[qsb](https://doc.qt.io/qt-6/qtshadertools-qsb.html)用于生成qt的跨平台shader文件。

可以直接从[清华镜像网站](https://mirrors.tuna.tsinghua.edu.cn/qt/official_releases/online_installers/)下载online installer，也可以通过[Qt官网](https://www.qt.io/download)下载(选择go open source，然后拉到下面的download the qt online installer)。

linux通过命令行安装：

```bash
wget -c https://mirrors.tuna.tsinghua.edu.cn/qt/official_releases/online_installers/qt-unified-linux-x64-online.run
chmod u+x qt-unified-linux-x64-online.run
./qt-unified-linux-x64-online.run
```

在下面这个页面中选择默认安装路径和Custom installation。

![](../Images/qt-default-install-folder.png)

macos和Linux的默认安装路径为`$HOME/Qt`，Windows的默认安装路径为`C:\Qt`。

然后重要的是下面这里，要选择Qt Designer Studio，不用cmake和ninja:

![](../Images/qt-component-selection.png)

macos在这个路径能找到qsb:

```
$QT/QtDesignStudio/qt6_design_studio_reduced_version/bin/qsb
```

windows和linux在这个路径:

```
$QT/Tools/QtDesignStudio/qt6_design_studio_reduced_version/bin/qsb
```
