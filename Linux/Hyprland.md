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
