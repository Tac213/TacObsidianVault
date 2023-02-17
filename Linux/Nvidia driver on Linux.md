## Arch

先确定显卡型号，通过下面命令可以确定：

```bash
lspci -k | grep -A 2 -E "(VGA|3D)"
```

对于[Maxwell系列](https://nouveau.freedesktop.org/CodeNames.html#NV110)或者更高的版本，安装`nvidia`即可，前提是内核为`linux`时，如果内核为`linux-lts`则应该安装`nvidia-lts`。

如果是[Turing系列](https://nouveau.freedesktop.org/CodeNames.html#NV160)或更高的版本，也可以安装`nvidia-open`(linux内核)，其他内核安装`nvidia-open-dkms`。

如果是[Kepler系列](https://nouveau.freedesktop.org/CodeNames.html#NVE0)，安装`nvidia-470xx-dkms`。

再老的基本不怎么支持，有一些可能可以用，参考[Unsupported drivers](https://wiki.archlinux.org/title/NVIDIA#Unsupported_drivers)。

如果要支持32位应用，还要从pacman的multilib仓库下载`lib32-nvidia-utils`。

接着，将`kms`从`/etc/mkinitcpio.conf`的`HOOKS`中移除，并执行(如果失败尝试sudo)：

```bash
mkinitcpio -p linux
```

接着reboot即可。

如果要启用[DRM](https://en.wikipedia.org/wiki/Direct_Rendering_Manager)，还要将`nvidia_drm.modeset=1`加到[内核参数](https://wiki.archlinux.org/title/Kernel_parameters)里，以grub为例，方法就是在`/etc/default/grub`的`GRUB_CMDLINE_LINUX_DEFAULT`里增加，接着执行：

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```
