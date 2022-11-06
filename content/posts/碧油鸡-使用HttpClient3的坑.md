---
title: '[碧油鸡]使用HttpClient3的坑'
author: 土川
tags:
  - BUG
categories:
  - Java基础
slug: 589437945
date: 2018-03-22 17:57:00
---
> 特么登录状态老是丢失。。。

<!--more-->

# 场景
使用阿里开源配置中心的diamond-sdk时需要登录配置中心才有权限进行操作。diamond-sdk是使用HttpClient3来进行信息传输的，于是在进行配置的操作之前客户端会先模拟登录获得session。然而在接下来的操作却一直报"未登录"之类的提示。

debug大法后发现`Cookie`对象好像`id`一直在变，返回的`jsessionid`也没保存。一波探索发现`HttpClient3`需要手动设置cookie策略。

# 解决方法
`HttpClient3` 默认的cookie策略是每次新建一个`Cookie`对象，复用Cookie的话，要进行如下设置。

	client.getParams().setCookiePolicy(CookiePolicy.BROWSER_COMPATIBILITY)
(完)