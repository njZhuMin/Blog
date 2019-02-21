---
title: Android搞机那些事-从SuperSU到Magisk
date: 2017-08-12 01:00:00
tags: [android,root]
categories:
- Android
- misc
---

# 从SuperSU到Magisk
喜欢折腾Android手机的小伙伴一定都很熟悉Chainfire大神的SuperSU。Android操作系统刚刚兴起时，市面上root权限管理软件五花八门，但在Android M之后大部分已经Root过的手机当中权限管理软件可能都换成了SuperSU。最近SuperSU在Google Play上的下载量更是突破了一亿。

然而，随着SuperSU被出售给CCMT，到Chainfire大神的离开，SuperSU的安全感也在逐渐降低。于是，XDA诞生了另一款神器：[Magisk](https://forum.xda-developers.com/apps/magisk)。Magisk兼具root、root权限管理以及对module的支持，可以实现类似Xposed框架的功能。

<!-- more -->

> Magisk is a Magic Mask to Alter System Systemless-ly.
> Magisk can ROOT your device, along with standard common patches.
> It packs with a super powerful Universal Systemless Interface, allowing unlimited potential!

从SuperSU切换到Magisk也很简单，首先需要fully unroot。需要注意的是，当刷入Magisk时会检查boot.img是否为stock，如果曾经有app modify过boot镜像，需要手动刷入stock boot.img即可。

刷机过程中遇到一个神奇的fastboot的问题，下面来详细说一说。

# fastboot卡在waiting for device
一直使用的是Arch官方源里的`android-tools`工具包，之前从来没有遇到fastboot命令卡住的问题。
```bash
$ fastboot flash boot boot.img
< waiting for device >
```
查看一下device：
```bash
$ fastboot devices
no permissions  fastboot
```
隐约觉得是权限问题：
```bash
$ sudo chown root:root /bin/fastboot
$ sudo chmod +s /bin/fastboot
$ fastboot devices
    7D89BCD6        fastboot
$ fastboot flash boot boot.img
    target reported max download size of 465031168 bytes
    sending 'boot' (9409 KB)...
    OKAY [  0.665s]
    writing 'boot'...
    OKAY [  0.961s]
    finished. total time: 1.666s
```
至此问题解决了。顺便说一句，在Windows上出现问题一般是驱动的问题，而Linux上命令本身的权限也会引起类似的问题。

# Android Studio错误处理
先说说Android Studio在更新SDK时的一个错误：
```bash
An error occurred while preparing SDK package Google APIs
Intel x86 Atom_64 System Image: No space left on device.
```
但硬盘空间是足够的。尝试使用`rm -rf /tmp/*`清理临时目录也没有效果。

在Arch论坛上找到增大`/tmp`分区最大容量的[解决方法](https://bbs.archlinux.org/viewtopic.php?id=225680)：
```bash
# sudo nano /etc/fstab
tmpfs            /tmp    tmpfs   size=7G,nodev,nosuid   0 0
```
