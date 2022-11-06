---
title: '[建站]Hexo+Nginx+VPS实现HTTPS建站'
tags:
  - Hexo
  - Https
categories:
  - 建站
slug: 711179553
date: 2018-03-08 18:02:00
---
> 拿人家hexo来建站，参考[hexo建站](https://www.jianshu.com/p/834d7cc0668d)写的，加上一些自己的东西

系统：centos6，是vultr的一个vps，1000g流量100m带宽可以跑满，最低每个月5美刀，点击就送屠龙宝刀嘿嘿[vultr.com](https://www.vultr.com/?ref=7229832)
使用工具：nginx用来做反向代理和https重定向、certbot来做免费证书申请、GitHub pages 同步博客、hexo做静态资源、hey hexo做博客管理

# GitHub Pages
hexo可以将文件同步到GitHub page上，可以同步后，输入`[你的帐户名].github.io` 来访问

![upload successful](/images/pasted-16.png)

> 没有GitHub账号吗，现在你有了

命名格式是:[你的帐户名].github.io，不然不行哦
# Hexo
最好照着[官网](https://hexo.io/zh-cn/docs/)来，因为这个东西更新有点快,链接已经说明一切
装node

    curl --silent --location https://rpm.nodesource.com/setup_9.x | sudo bash -
    yum -y install nodejs
装git
    
    yum -y install git

安装完后，正常情况下的hexo命令会加入/usr/bin中，
> 不正常情况下...我的的命令在`/etc/node/lib/node_modules/hexo/bin`下且没加入环境变量，找了半天，第二次安装就正常了，可能是第一次没有npm没加`-g`参数。

这时建个站点，执行`hexo init your_blog_name`，这里假设你要建立一个叫叫`foo`的博客，执行`hexo init foo | cd foo | npm install`, `ls`可以看到`foo`目录。
> 这里`foo`路径是相对于你执行命令的路径。

进入`foo`目录，vim打开`_config.yml`，并滚动到最下面添加如下配置信息（注意最下边有`deploy`和`type`字段，覆盖这两个字段或者删除这两个字段然后复制下面的四个字段也行。）：

    deploy:
        type: git
        repo: git@github.com:hz8080/hz8080.github.io.git
        branch: master

> 我配了ssh验证，避免密码登陆

`foo`目录的结构

    ├── _config.yml
    ├── package.json
    ├── scaffolds
    ├── source
    |  ├── _drafts
    |  └── _posts 
    └── themes
`themes`是我们待会主题放的地方

接着在`foo`目录执行`hexo s -p 5000`看到输出，输入ip:5000就可以看到你的博客了。
> s是server的意思，-p是端口，默认4000。
不要把`s`写成`-s`，`-s`可以加，但是`s` 是启动必须的。
  
# 发布
> hexo clean
hexo g
hexo d

`hexo clean`是清楚缓存
`hexo g`是生成本地发布文件夹
`hexo d`是发布到deploy到 _config.yml设置的目标地址

可以把三条写进bash脚本~

    #!/bin/bash
    hexo clean
    hexo g
    hexo d

> `hexo d` 遇到`not found` 问题, 输入`npm install hexo-deployer-git --save`，再执行`hexo d`

# 主题
hexo有很多主题，比如我用的是[next](https://github.com/theme-next/hexo-theme-next)
# 给菜单增加标签、类目
在站点内，执行`hexo new page tags`，这时在`sources/tags`有`index.md`，vim编辑之，

    ---
    title: tags
    date: 2016-11-11 21:40:58
    type: "tags"
    ---
在站点的`_config.yml `把标签打开

    menu:
      home: /
      #categories: /categories
      #about: /about
      archives: /archives
      tags: /tags    //确保标签页已打开
      #schedule: /schedule   
      #commonweal: /404.html 
分类同理，在站点内，执行`hexo new page categories`，这时在`sources/categories`有`index.md`，vim编辑之，然后在`_config.yml `把# 去掉

最后！在主题文件里（`themes/next/`）的_config.yml的tags和categories记得把注释去掉，菜单栏才会显示`标签`和`分类`
```bash
menu:
  home: / || home
  #about: /about/ || user
  tags: /tags/ || tags
  categories: /categories/ || th
  archives: /archives/ || archive
  #schedule: /schedule/ || calendar
  #sitemap: /sitemap.xml || sitemap
  #commonweal: /404/ || heartbeat
```
重新部署，即可生效。 

> 其他换图标、头像什么的可以搞搞

> hexo 修改了站点名字、介绍什么的，好像重启才生效，不知道是不是bug

# 设置‘阅读全文’
hexo默认显示全部，设置阅读全文有两种方法，
* 第一种，在主题的`_config.yml`的`auto_excerpt.enable`改为true，`length`是预览长度
* 第二种，在文章加入`<!--more-->`，首页就会预览到`<!--more-->`的位置

第一种会有`...`，且预览长度是一样的
# 域名
我从godaddy 买了一个.me后缀的，年费挺便宜的，不过续费就不便宜了，可以试用一波。具体绑定方法自找。

买完后在dns管理把`A`类名指向自己的服务器ip，配置完毕浏览器打开`你的域名:hexo端口`试试

![upload successful](/images/pasted-17.png)


# 博客文章管理
* [hexo-hey](https://github.com/nihgwu/hexo-hey )，不推荐，已经停止更新，有莫名其妙的bug
* [hexo-admin](https://jaredforsyth.com/hexo-admin) ，用法简单粗暴，还能自动保存，用的这个 

# nginx安装
安装不赘述，直接从配置动手
在`/etc/nginx/conf.d/default.conf`是一个配置模板，在`/etc/nginx/conf.d/`下的*.conf都会被nginx读取。我只有一个应用，直接在`default.conf`动手
```
upstream hexo_host{
    server localhost:4000;
}
#
# The default serve

server {

    server_name  htchz.me;
    # root        /usr/share/nginx/html;

    # Load configuration files for the default server block.
    # include /etc/nginx/default.d/*.conf;

    location /admin {
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://hexo_host;
    }

    // 非后台管理直接走静态文件
    location / {
        root /root/htc/public/;
        expires  30m;
    }  

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }


```
执行`nginx -s reload`， 重启nginx，输入`域名`（http默认80端口），验证nginx转发端口是否生效

> 如果请求资源报403权限不足，可能是`/etc/nginx/nginx.conf`的`user`权限不足，改成`root`就可以

# certbot申请证书
> 为什么要博客升级https，其实只是为了装逼。chrome打开的时候地址栏一个绿色的`安全|https` 看着挺舒服的

终端输入

    wget https://dl.eff.org/certbot-auto
    chmod a+x certbot-auto
    ./certbot-auto --nginx
期间要输入你的邮箱，选择申请证书的域名，选择是否原端口重定向到`https`（443），选是

然后certbot会自动申请证书、修改你的nginx配置。

下边是80转443的配置。
```
server {
    if ($host = htchz.me) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen      80;
    server_name  htchz.me;
    return 404; # managed by Certbot
}
```
也可以这么写
```
server {
    listen      80;
    server_name  htchz.me;
    return 301 https://$server_name$request_uri;
}
```
> 记得开通你的80、443端口。。。
# 自动续期
certbot申请的证书只有三个月，这里写个定时任务，命令行：

    crontab -e
写入

    0 0 * * 0 /root/www/certbot-auto renew
这条命令的意思是每周日的0点0分执行`/root/www/certbot-auto renew`这条命令。执行下面这条命令查看定时任务列表中是否有刚才添加的任务

    [root@California_VPS etc]# crontab -l 
    0 0 * * 0 /root/www/certbot-auto renew
 

# 后来发现的问题
到这里整个博客差不多了。

但还是有问题。没加图片的时候，chrome还是一把小绿锁，知道加了图片之后，小绿锁就没了。
打开chrome控制台，看到类似下面warning，

    Mixed Content: The page at 'https://domain.com/w/a?id=074ac65d-70db-422d-a6d6-a534b0f410a4' was loaded over HTTPS, but requested an insecure image 'http://img.domain.com/images/2016/5/3/2016/058c5085-21b0-4b1d-bb64-23a119905c84_cf0d97ab-bbdf-4e25-bc5b-868bdfb581df.jpg'. This content should also be served over HTTPS.
原来是https站点下，图片依旧是http访问。

## 解决方法
这个简单，在服务器返回响应时加响应头，nginx配置如下

    server {
      ...
      add_header Content-Security-Policy upgrade-insecure-requests;
      ...
    }

还有一个方法是在html加入meta 

    <meta http-equiv="Content-Security-Policy" content="upgrade-insecure-requests" />
在hexo这个要去改模板太麻烦


然后重新加载配置文件，顺利解决~

# 优化
1. [优化url过长](https://liziczh.com/hexo-submit.html)
1. [标题自动编号](https://www.tuicool.com/articles/7BnIVnI)