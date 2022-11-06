---
title: '[建站]Linode开启Google BBR的正确方法'
author: 土川
tags:
  - bbr
categories:
  - 建站
slug: 1807419896
date: 2018-03-26 23:55:00
---
> 在linode一直开启不了bbr加速，在一个老哥的博客看到这个[传送门](https://malash.me/201702/the-right-way-to-enable-google-bbr/)

<!-- more -->
简单来说，Linode服务器的内核是要在配置页面修改才能正确引导的，而且4.9.0以上的liux内核都编译了`tcp_bbr`加速模块，所以不用自己配置了。

![upload successful](/images/pasted-92.png)
还有一点，`lsmod`找不到`tcp_bbr`，博主说有一种说法是，**Linode自带的内核都是把模块都编译一块的，所以lsmod里看不到正常，lsmod是看额外加载的模块的**。

> Linode冲5刀送20刀，快去啊，不过填个人信息得填优惠码和另一个忘了什么鬼码，否则没有20刀。