---
layout: post
title:  "Docker On Raspberry Pi"
date:   2017-01-01 11:30:00 +0100
categories: [docker, raspberry]
---


Docker on your raspberry is more than an experiment today, we can use docker for easily deployment of some home services such as owncloud, torrent downloader or vpn server. docker allow us to reproduce environments for each application, avoiding library and binaries problems. all packages required for an application will be already configured on the image that will create the container.
With docker we can create applications that will have a high resilience behavior to errors using various hosts in a swarm cluster (with 3 raspberries you could have small services with a quick response to errors or even load balancing them in the cluster).


## Installing Alpine 3.5 on Raspberry
1) Download (https://fr.alpinelinux.org/alpine/v3.5/releases/armhf/alpine-rpi-3.5.0-armhf.tar.gz) alpine version for armf type hosts. We will download a tarball named alpine-rpi-3.5-armhf.tar.gz.

2) Mount an empty SD card on your computer and create a small partition with at least 512MB (this will become your /boot later).

3) Create a FAT32 filesystem on the newly created partition and mark it as bootable.

4) Check that the filesystem is ok for extracting de alpine OS there:
~~~
# fdisk -l

Disk /dev/sdd: 32.0 GB, 32010928128 bytes
  102 heads, 36 sectors/track, 17026 cylinders
  Units = cylinders of 3672 * 512 = 1880064 bytes

Device Boot Start End Blocks Id System
  /dev/sdd1 * 1 287 524288 b Win95 FAT32
~~~

 5) Mount the created partition on your host and extract the tarball on it.
~~~
$ cd /_WHEREVER_YOU_HAVE_MOUNTED_THE_PARTITION_
$ tar -zxvf /_WHEREVER_YOU_HAVE_YOUR_TAR_FILE/alpine-rpi-3.5.0-armhf.tar.gz
~~~

6) Now you are ready to boot from SD card on your raspberry.

7) Boot Alpine from SD will give us a On-Memory System so we will have to commit changes until we create a root partition and write down and change de root filesystem to boot.

8) Login as root without password and execute alpine-setup to configure your system.
~~~
# setup-alpine
~~~

9) Change your keyboard, timezone and network settings and commit the changes to the SD card. I always install openssh and chrony (raspberry doesn’t have hardware clock so I enable software clock on boot and disable hardware clock syncing).
~~~
# rc-update add swclock boot && rc-update del hwclock boot
~~~

10) Check if opensshd and crony are enabled on boot as well as swclock.
~~~
# rc-update
~~~
11) Update installed packages before commiting changes
~~~
# apk --update upgrade
~~~
12) Add support for creating ext4 filesystems for creating root filesystem in ext4 format.
~~~
# apk add e2fsprogs
~~~
13) Commit every change done to this point and reboot to check that everything is ok.
~~~
# lbu commit

# reboot
~~~

14) Login to the host and then we create a new partition for the root filesystem.

15) Then we will create a ext4 filesystem on the newly created partition
~~~
# fdisk -l /dev/mmcblk0

Disk /dev/mmcblk0: 32.0 GB, 32010928128 bytes
 102 heads, 36 sectors/track, 17026 cylinders
 Units = cylinders of 3672 * 512 = 1880064 bytes

Device Boot Start End Blocks Id System
 /dev/mmcblk0p1 * 1 287 524288 b Win95 FAT32
 /dev/mmcblk0p2 287 4542 7813800 83 Linux

# mkfs.ext4 /dev/mmcblk0p2
16) Now we will do a install to that partition using setup-disk and the data from the boot partition.

# mkdir /tmp_rootfs
 # mount /dev/mmcblk0p2 /tmp_rootfs
 # setup-disk -o /media/mmcblk0p1/$(hostname).apkovl.tar.gz /tmp_rootfs
NOTE: Execution should notice some errors about syslinux and extlinux but we can ignore them.
~~~

17) We will add the boot partition to the fstab file in /tmp_rootfs/etc/fstab
~~~
/dev/mmcblk0p1 /media/mmcblk0p1 vfat defaults 0 0
~~~

18) Then reboot (you can umount /tmp_rootfs, but you can not remount /media/mmcblk0p1 to make it writable).

19) Once logged again as root using On-Memory system, we will remount /media/mmcblk0p1 to make it writable for been able to change root fs.

~~~
# mount -o remount,rw /media/mmcblk0p1
~~~

20) Finally we will edit /media/mmcblk0p1/cmdline.txt to specify the root fs to use

We will add root=/dev/mmcblk0p2 at the end of the cmdline.

Reboot the system to check our locally installed Alpine Linux version.

At this point, we have alpine linux 3.5 installed on our raspberry (tested on raspberry 2, but changes may occur in cmdline.txt filename).

## Installing Docker
Installing Docker using official Alpine repository is quite simple, but this probably will not install the latest official Docker version.

In this guide we will install Alpine’s repository version.

1) We will edit /etc/apk/repositories to uncomment  the apline 3.5 community repository.
http://mirrors.2f30.org/alpine/v3.5/community

2) We will install now docker packages
~~~
# apk add --update docker
~~~

3) Once installed, enable Docker on startup (on default stage).
~~~
# rc-update add docker
~~~

4) We will start docker engine from init.d script.
~~~
# /etc/init.d/docker start
~~~

5) Finally we can check docker started version (at time of writing this article, alpine version was 1.12.3)
~~~
# docker version

Client:
  Version: 1.12.3
  API version: 1.24
  Go version: go1.7.3
  Git commit: v1.12.3
  Built: 
  OS/Arch: linux/arm

Server:
  Version: 1.12.3
  API version: 1.24
  Go version: go1.7.3
  Git commit: v1.12.3
  Built: 
  OS/Arch: linux/arm
~~~


Hope this will help someone as it does to me write down my steps 🙂

 

REFERENCES:

- Installing Alpine on Raspberry Pi
https://wiki.alpinelinux.org/wiki/Raspberry_Pi

- Docker on Alpine Linux
http://wiki.alpinelinux.org/wiki/Docker
