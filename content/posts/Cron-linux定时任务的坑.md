---
title: '[Cron]linux定时任务的坑'
author: 土川
tags: []
categories:
  - Linux
slug: 3058313917
date: 2018-04-18 02:23:00
---
> 写了个脚本定时执行，有天发现命令没有正确找到我配在PATH的环境变量。

<!--more-->
用了[gdrive](https://github.com/prasmussen/gdrive)来备份博客，日志看到
	
    /root/gdrive-bak-blog.sh: line 2: gdrive: command not found
百度一波发现cron读取环境变量有点搓，有两个解决方案

* crontab里的命令先执行`source`


	0 8 * * * source /etc/profile &&  /root/gdrive-bak-blog.sh > /root/blog-bak.out 2>&1
*  脚本里执行加载环境变量


	#!/bin/sh
	source /etc/profile
    ...

附上备份脚本

	#! /bin/bash
    oldFileId=`gdrive list | grep 'hzblog.bak.tar.gz' | awk '{print $1}'` 
    if [ -z $oldFileId ] 
    then 
     echo "no backup for hzblog.bak.tar.gz"
    else
     gdrive delete $oldFileId
     echo "hzblog.bak.tar.gz[id:$oldFileId] deleted"
    fi

    rm -f /root/hzblog.bak.tar.gz
    tar -czPf hzblog.bak.tar.gz /root/hzblog
    parentId=`gdrive list | grep 'hzblog' | awk '{print $1}'`
    gdrive upload -p $parentId /root/hzblog.bak.tar.gz
    echo "hzblog.bak.tar.gz uploaded"