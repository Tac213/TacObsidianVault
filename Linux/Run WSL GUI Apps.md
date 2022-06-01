## windows 10

[参考文章](https://techcommunity.microsoft.com/t5/windows-dev-appconsult/running-wsl-gui-apps-on-windows-10/ba-p/1493242)

首先在[这个地址](https://sourceforge.net/projects/vcxsrv/)下载VcXsrv Windows X Server。下载过程中如果遇到下载缓慢的问题可以选择其他镜像。

安装过程中一路next。

启动VcXsrv Windows X Server的过程中需要配置，需要勾选"Disable access control"，弹出防火墙提示时，需要给到所有全选。

然后在wsl端，配置DISPLAY环境变量，如果用zsh可以配置到`~/.zshrc`中，增加下面这一行：

```bash
export DISPLAY="`grep nameserver /etc/resolv.conf | sed 's/nameserver //'`:0"
```

继续在wsl端，执行下面的shell:

```
echo xfce4-session > ~/.xsession
```

然后保证windows端的VcXsrv Windows X Server有开启，就可以正常运行wsl上的gui app。

## windows 11

[参考文章](https://docs.microsoft.com/en-us/windows/wsl/tutorials/gui-apps)

首先确保显卡驱动有安装，然后**管理员模式**进入powershell，执行下面的命令：

```shell
wsl --update
wsl --shutdown
```

就是先将wsl更新到最新，再重启wsl。

接着重启wsl即可，可以随意运行wsl上的gui app。

运行之前最好先执行下面的命令：

```shell
sudo apt update
```
