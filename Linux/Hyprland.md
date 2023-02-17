## install

先确保有`yay`，然后通过`yay`安装：

```bash
yay -S hyprland  # or hyprland-git hyprland-bin
```

最好再装个`kitty`，默认使用`kitty`作为终端。

正常来说执行`Hyprland`即可启动，但一般不这么做，要导出一些环境变量，所以要用一个wrapper，比如命名为`wrappedhl`:

```bash
#!/bin/sh

cd ~

# Log WLR errors and logs to the hyprland log. Recommended
export HYPRLAND_LOG_WLR=1

# Tell XWayland to use a cursor theme
export XCURSOR_THEME=Bibata-Modern-Classic

# Set a cursor size
export XCURSOR_SIZE=24

# Example IME Support: fcitx
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
export SDL_IM_MODULE=fcitx
export GLFW_IM_MODULE=ibus

# Nvidia
export LIBVA_DRIVER_NAME=nvidia
export XDG_SESSION_TYPE=wayland
export GBM_BACKEND=nvidia-drm
export __GLX_VENDOR_LIBRARY_NAME=nvidia
export WLR_NO_HARDWARE_CURSORS=1

exec Hyprland
```

如果使用login manager，比如gdm，sddm(推荐)，则要把默认的destop文件也增加一个wrapper版本的，比如将`/usr/share/wayland-sessions/hyprland.desktop`另存一个到`/usr/share/wayland-sessions/hyprland-wrapped.desktop`，然后修改为以下内容：

```ini
[Desktop Entry]
Name=Hyprland Wrapped
Comment=An intelligent dynamic tiling Wayland compositor
Exec=wrappedhl
Type=Application
```

注意`wrappedhl`应该在`PATH`里，比如`/usr/local/bin`。

### nvidia

安装`nvidia-dkms`。如果不是laptop而且显卡在[这个列表](https://github.com/NVIDIA/open-gpu-kernel-modules#compatible-gpus)里，可以安装`nvidia-open-dkms`。

然后启用drm对应的内核参数。参考[[Nvidia driver on Linux#Arch]]

接着执行：

```bash
mkinitcpio --config /etc/mkinitcpio.conf --generate /boot/initramfs-custom.img
```

接着在`/etc/modprobe.d/nvidia.conf`中增加一行：`options nvidia-drm modeset=1`

如果运行electron app有crash，尝试：

```bash
pacman -S qt5-wayland qt5ct libva
yay -S nvidia-vaapi-driver-git
```

完成上面的东西之后reboot。记得在wrapper脚本里增加nvidia相关的东西。

## 科学上网

```bash
sudo pacman -S clash
```

然后下载配置文件放到`.config/clash`即可，如果提示要下载Country.mmdb，大概率下载不下来，可以从其他电脑拷贝过来。

## Must Have

首先需要notification daemon, 可以选择`dust`, `mako`, 加到hyprland.config的`exec-once`里。

如果需要分享屏幕比如使用obs，则需要安装:

```bash
sudo pacman -S pipewire wireplumber
yay -S xdg-desktop-protal-hyprland-git
```

注意安装时要选择`-gtk`后缀的依赖。

authentication agent可以选择`polkit-kde-agent`，注意可执行文件为`/usr/lib/polkit-kde-authentication-agent-1`。

最后还要下载qt对wayland的支持: `qt5-wayland`, `qt6-wayland`

科学上网也可以配置到`exec-once`中。

## status bar

直接使用waybar即可:

```bash
yay -S waybar-hyprland-git
```

使用之前应当先配置好字体，否则无法正确显示图标。

## app launcher

可以选择`rofi`或者`fuzzel`，都很简单，rofi的话最好先dump一下配置，然后执行下面的命令启动`rofi`:

```bash
rofi -show drun
```

## clipboard

直接安装`wl-clipboard`即可复制粘贴，如果需要管理剪切板可以可以参考[hyprland文档](https://wiki.hyprland.org/Useful-Utilities/Clipboard-Managers/)

## 其他常用软件

参考[Are we wayland yet](https://arewewaylandyet.com/)
