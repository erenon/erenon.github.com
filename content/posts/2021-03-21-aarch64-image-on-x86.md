---
title: "Bootstrap aarch64 image on X86"
date: 2021-03-21
---

The current (2021-Feb-05) aarch64 [Arch Linux ARM][alarm] image has [issues][forum] on 8GB Raspberry Pi 4 models:
It does not bring up the network interface, and the USB ports are not working,
making any interactive fix difficult (without serial cable).

[alarm]: https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-4
[forum]: https://archlinuxarm.org/forum/viewtopic.php?f=67&t=15190

A possible fix is to replace the default kernel with a Raspberry specific one.
It still results in a 64bit system, only with a non-upstream kernel.
If there's a different ARM system available, it is trivial to mount/boot the SD card there and apply the required changes.
The solution below, however, only requires a single working, linux supported hardware, e.g: x86.
This description is Arch Linux specific. For ubuntu, see [this blogpost on Disconnected Systems][disconnectedsys].

[disconnectedsys]: https://disconnected.systems/blog/raspberry-pi-archlinuxarm-setup/

## Install Qemu

Qemu makes it possible to run arm binaries on x86 (and many more combinations).
Install [qemu-user-static][] from AUR. To make the resulting package smaller,
add `--target-list=aarch64-linux-user` to the configure invocation in the `PKGBUILD` file.
This package requires glib-static and pcre-static, the former takes some time to build.
Due to the number of dependencies, consider building in a [clean chroot][].
(Do not forget to [set `-j` in `MAKEFLAGS`][makepkg])

[qemu-user-static]: https://aur.archlinux.org/packages/qemu-user-static/
[clean chroot]: https://wiki.archlinux.org/index.php/DeveloperWiki:Building_in_a_clean_chroot
[makepkg]: https://wiki.archlinux.org/index.php/Makepkg

Install the built qemu-user-static package. Read the [introduction of binfmt][binfmt],
then setup the config accordingly:

    $ cat /etc/binfmt.d/qemu-static.conf
    :qemu-aarch64:M::\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\xb7:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff:/usr/bin/qemu-aarch64-static:CF

Finally, restart the service that - if there's a config file - mounts the filesystem described by the doc above:

    $ sudo systemctl restart systemd-binfmt

Now the system can run aarch64 binaries using qemu.

[binfmt]: https://www.kernel.org/doc/html/latest/admin-guide/binfmt-misc.html

## Install Arch Linux ARM

Follow the [install guide][alarm]. Use the `ArchLinuxARM-rpi-aarch64-latest.tar.gz` archive,
but do not change `/etc/fstab`! Also, before unmounting root and boot, do:

    $ sudo arch-chroot root

This will drop you into the installed ARM system. Add the alarm repos, and install the raspberry kernel:

    $ pacman-key --init
    $ pacman-key --populate archlinuxarm
    $ pacman -S --needed linux-raspberrypi4 raspberrypi-bootloader raspberrypi-bootloader-x raspberrypi-firmware firmware-raspberrypi

This is a good time to also complete the official [Installation Guide][].
After done, copy root/boot over boot again, and unmount the partitions.
Again, do not modify `etc/fstab`, contrary to the alarm docs.

[Installation Guide]: https://wiki.archlinux.org/index.php/installation_guide

With everything done correctly, using the patched image, the network and USB ports of the 8GB Raspberry Pi 4
are fine again!
