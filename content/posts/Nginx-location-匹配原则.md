---
title: '[Nginx]Nginx location 匹配原则'
author: 土川
tags:
  - Nginx
categories:
  - 服务器
slug: 3180078468
date: 2018-03-10 15:16:00
---
Location block 的基本语法形式是：`location [=|~|~*|^~|@] pattern { ... }`

`[=|~|~*|^~|@]`被称作 location modifier ，这会定义 Nginx 如何去匹配其后的 pattern ，以及该 pattern 的最基本的属性（简单字符串或正则表达式）

# location modifier详解

* 1、`=`

```
server {
    server_name htchz.com;
    location = /abcd {
    […]
    }
}
```
匹配情况：
   
    http://website.com/abcd        # 正好完全匹配
    http://website.com/ABCD        # 如果运行 Nginx server 的系统本身对大小写不敏感，比如 Windows ，那么也匹配
    http://website.com/abcd?param1m2    # 忽略查询串参数（query string arguments），这里就是 /abcd 后面的 ?param1m2
    http://website.com/abcd/    # 不匹配，因为末尾存在反斜杠（trailing slash），Nginx 不认为这种情况是完全匹配
    http://website.com/abcde    # 不匹配，因为不是完全匹配
* 2、(None)

不写 location modifier ，Nginx 仍然能去匹配 pattern 。这种情况下，匹配那些以指定的 patern 开头的 URI，注意这里的 URI 只能是普通字符串，不能使用正则表达式。
```
server {
    server_name website.com;
    location /abcd {
    […]
    }
}
```
匹配情况：

    http://website.com/abcd        # 正好完全匹配
    http://website.com/ABCD        # 如果运行 Nginx server 的系统本身对大小写不敏感，比如 Windows ，那么也匹配
    http://website.com/abcd?param1m2    # 忽略查询串参数（query string arguments），这里就是 /abcd 后面的 ?param1m2
    http://website.com/abcd/    # 末尾存在反斜杠（trailing slash）也属于匹配范围内
    http://website.com/abcde    # 仍然匹配，因为 URI 是以 pattern 开头的
* 3、`~`

```
server {
    server_name website.com;
    location ~ ^/abcd$ {
    […]
    }
}
```
匹配情况：

    http://website.com/abcd        # 完全匹配
    http://website.com/ABCD        # 不匹配，~ 对大小写是敏感的
    http://website.com/abcd?param1m2    # 忽略查询串参数（query string arguments），这里就是 /abcd 后面的 ?param1m2
    http://website.com/abcd/    # 不匹配，因为末尾存在反斜杠（trailing slash），并不匹配正则表达式 ^/abcd$
    http://website.com/abcde    # 不匹配正则表达式 ^/abcd$

> 对于一些对大小写不敏感的系统，比如 Windows ，~ 和 ~* 都是不起作用的，这主要是操作系统的原因。

* 4、` ~*`

与 ~ 类似，但这个 location modifier 不区分大小写，pattern 须是正则表达式
```
server {
    server_name website.com;
    location ~* ^/abcd$ {
    […]
    }
}
```

    http://website.com/abcd        # 完全匹配
    http://website.com/ABCD        # 匹配，这就是它不区分大小写的特性
    http://website.com/abcd?param1m2    # 忽略查询串参数（query string arguments），这里就是 /abcd 后面的 ?param1m2
    http://website.com/abcd/    # 不匹配，因为末尾存在反斜杠（trailing slash），并不匹配正则表达式 ^/abcd$
    http://website.com/abcde    # 不匹配正则表达式 ^/abcd$

* 5、`^~`

匹配情况类似 2. (None) 的情况，以指定匹配模式开头的 URI 被匹配，不同的是，一旦匹配成功，那么 Nginx 就停止去寻找其他的 Location 块进行匹配了（与 Location 匹配顺序有关）
* 6、`@`

用于定义一个 Location 块，且该块不能被外部 Client 所访问，只能被 Nginx 内部配置指令所访问，比如 try_files or error_page

# 搜索顺序以及生效优先级
因为可以定义多个 Location 块，每个 Location 块可以有各自的 pattern 。因此就需要明白（不管是 Nginx 还是你），当 Nginx 收到一个请求时，它是如何去匹配 URI 并找到合适的 Location 的。

要注意的是，写在配置文件中每个 Server 块中的 Location 块的次序是不重要的，Nginx 会按 location modifier 的优先级来依次用 URI 去匹配 pattern ，顺序如下：
    
    1. =
    2. (None)    如果 pattern 完全匹配 URI（不是只匹配 URI 的头部）
    3. ^~
    4. ~ 或 ~*
    5. (None)    pattern 匹配 URI 的头部
    

# 实际使用建议
所以实际使用中，个人觉得至少有三个匹配规则定义，如下：
直接匹配网站根，通过域名访问网站首页比较频繁，使用这个会加速处理，官网如是说。
这里是直接转发给后端应用服务器了，也可以是一个静态首页

* 第一个必选规则

```
location = / {
    proxy_pass http://tomcat:8080/index
}
```
* 第二个必选规则是处理静态文件请求，这是nginx作为http服务器的强项,有两种配置模式，目录匹配或后缀匹配,任选其一或搭配使用

```
location ^~ /static/ {
    root /webroot/static/;
}
```

```
location ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {
    root /webroot/res/;
}
```

* 第三个规则就是通用规则，用来转发动态请求到后端应用服务器，非静态文件请求就默认是动态请求，自己根据实际把握

```
location / {
    proxy_pass http://tomcat:8080/
}
```