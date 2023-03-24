---
title: "PA1 在愉快的PA之旅之前"
date: 2023-03-21T00:44:20+08:00
draft: true
slug: f7e2562e
---
计算机系统基础，冲！

选择路线: x86

照着讲义`git clone`之后，一通折腾，最后执行`make ARCH=native run mainargs=mario`成功运行，but

```shell
...

 PRG ROM:    2 x 16KiB
 CHR ROM:    1 x  8KiB
 ROM MD5:  0x8e3630186e35d477231bf8fd50e54cdd
 Mapper #:  0
 Mapper name: NROM
 Mirroring: Vertical
 Battery-backed: No
 Trained: No

Power on
Initializing video...

```
卡在`Initializing video...`了，盲猜是没有装GUI，装上xface、[xrdp](https://github.com/neutrinolabs/xrdp)、[远程连接客户端](https://apps.apple.com/us/app/microsoft-remote-desktop/id1295203466?l=zh&mt=12)，成功运行again。

![](/images/20221113115300.png)

不过怎么没有声音呢？虽然讲义说不是必选，可是作为一个计(CTRL)算(C)机(V)科(工)学(程)家(师)，你既然提了可以做到，我就必须得开这个声音了，由于用的`xrdp`，于是去(github)[https://github.com/neutrinolabs/xrdp]一波，看到README.md有这样的内容:

![](/images/20221113220237.png)

顺着**Audio redirection**的链接过去，是一个[pulseaudio-module-xrdp](https://github.com/neutrinolabs/pulseaudio-module-xrdp/wiki/README)模块，需要编译两个.so，我的是**debian**，编译过程一直出现

```shell
Reading package lists...
Building dependency tree...
E: Unable to locate package sudo
E: Unable to locate package lsb-release
/bin/sh: 1: cannot create /etc/sudoers.d/nopasswd-ohuang: Directory nonexistent
chmod: cannot access '/etc/sudoers.d/nopasswd-ohuang': No such file or directory
/wrapped_script: 55: lsb_release: not found
/wrapped_script: 55: lsb_release: not found
```
在[issue](https://github.com/neutrinolabs/pulseaudio-module-xrdp/issues/73)里找到了解决办法，是apt源的原因，轻量云用的源是腾讯的镜像源。

`configure`的时候遇到了波浪线在双引号内无法解释的问题，命令改为`./bootstrap && ./configure PULSE_DIR=/home/$USER/pulseaudio.src`，最后`make && sudo make install`即可，执行
```bash
ls $(pkg-config --variable=modlibexecdir libpulse) | grep xrdp
显示
```
```shell
module-xrdp-sink.la
module-xrdp-sink.so
module-xrdp-source.la
module-xrdp-source.so
```
重启一下xrdp，可以在声音里看到**xrdp sink**
![](/images/20221113222651.png)

再运行一波虚拟机，声来！