首先当然需要clone Unreal的仓库:

```shell
git clone git@github.com:EpicGames/UnrealEngine.git
```

仓库很大，需要clone很长的时间，可能会由于Mac的自动锁屏导致缺少网络连接而clone失败，为了解决这个问题可以提前在Preference里面把电池设置为较长的时间不锁屏，clone完成后再恢复原样。

clone完成后，确保已经安装了XCode，再走后续步骤。

接着cd到Unreal目录，比如：

```shell
cd ~/UnrealEngine
```

然后像windows一样，跑脚本，可以在Finder上双击command文件也可以在terminal直接运行shell:

```shell
sh "./Setp.sh"
```

上面这一步需要执行较长的时间，接着再执行下面这个脚本

```shell
sh "./GenerateProjectFiles.sh"
```

然后就在unreal目录下生成了UE5.xcworkspace，接着再执行这个：

```shell
xed ./UE5.xcworkspace
```

即可在XCode中打开Unreal项目。

点击左上角的播放按钮，即可开始编译。

编译完成后，clone自己的项目，然后要生成项目对应的xcworkspace，首先cd到下面的目录：

```shell
cd ~/UnrealEngine/Engine/Build/BatchFiles/Mac
```

然后执行下面的命令：

```shell
sh "./GenerateProjectFiles.sh" -project="Path/To/Project/YourProject.uproject" -game
```

执行完成后即可生成YourProject.xcworkspace，可以继续在terminal中用xed开启这个项目，或者在Finder中双击这个文件也可以开启XCode。