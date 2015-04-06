---
layout: post
title:  "Emulate RetroPi on Mac OS"
date:   2015-02-05 12:00:00
categories: bash
tags: mac retropi bash
header-img: assets/article_images/server-cover1.jpg
---

* http://www.pihomeserver.fr/2013/11/30/raspberry-pi-home-server-utiliser-le-rapsberry-pi-avec-qemu-sous-mac-os-x/
* http://xecdesign.com/compiling-qemu/
* http://xecdesign.com/qemu-emulating-raspberry-pi-the-easy-way/
* https://gist.github.com/bdsatish/7476239
* [good one] http://blogduyax.madyanne.fr/emuler-la-raspbian-avec-qemu.html
* http://www.raspberrypi.org/forums/viewtopic.php?f=63&t=42279
* http://www.raspbian.org/RaspbianQuake3
* https://mike632t.wordpress.com/2014/06/26/raspberry-pi-camera-setup/

Raspberry > ARM11(ARMv6) 

Choose macune for `-M` argument : `qemu-system-arm -machine help | sort`

Do not try to use more than 256 MB of RAM, the value is hard-coded in and QEMU will not work correctly.




```
brew search zlib1g
brew install pkg-config
brew install glib
brew link glib
brew unlink glib && brew link glib #(if necessary)
brew install apple-gcc42
brew install qemu --env=std --use-gcc

mkdir retropi & cd retropi
wget http://xecdesign.com/downloads/linux-qemu/kernel-qemu
wget http://......./RetroPieImage_ver2.3.img

qemu-img resize RetroPieImage_ver2.3.img +2G

qemu-system-arm -kernel zImage -cpu arm1176 -m 256 -M versatilepb -no-reboot -serial stdio -append "root=/dev/sda2 rootfstype=ext4 elevator=deadline rw panic=1 console=ttyAMA0,115200 console=tty1 init=/bin/bash" -hda RetroPieImage_ver2.3.img

> nano /etc/ld.so.preload
	> #/usr/lib/arm-linux-gnueabihf/libcofi_rpi.so
	
> nano /etc/udev/rules.d/90-qemu.rules
	> KERNEL=="sda", SYMLINK+="mmcblk0"
	> KERNEL=="sda?", SYMLINK+="mmcblk0p%n"
	> KERNEL=="sda2", SYMLINK+="root"
> sudo ln -snf mmcblk0p2 /dev/root

> echo 'SUBSYSTEM=="vchiq",GROUP="video",MODE="0660"' > /etc/udev/rules.d/10-vchiq-permissions.rules
> usermod -a -G video pi

> halt

qemu-system-arm -kernel zImage -cpu arm1176 -m 256 -M versatilepb -no-reboot -serial stdio -append "root=/dev/sda2 rootfstype=ext4 elevator=deadline rw panic=1 console=ttyAMA0,115200 console=tty1" -hda RetroPieImage_ver2.3.img

> vi $HOME/.bashrc
	> export PATH=$PATH:/opt/vc/bin
	> export LD_LIBRARY_PATH=$LD_LIBRARYPATH:/opt/vc/lib
> sudo raspi-config
	select the expand fs option to fix everything up.

```




Emulate Raspberry image (2)

```
dd if=debian6-17-02-2012/debian6-17-02-2012.img of=rootfs_debian6_rpi.ext4 skip=157696 count=3256320
qemu-system-arm -M versatilepb -cpu arm1176 -m 256 -hda rootfs_debian6_rpi.ext4 -kernel zImage_3.1.9 -append "root=/dev/sda" -serial stdio -redir tcp:2222::22 -net none
```