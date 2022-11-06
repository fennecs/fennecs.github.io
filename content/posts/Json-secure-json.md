---
title: '[Json]secure json'
author: 土川
tags:
  - Json
categories: []
slug: 235536411
date: 2018-06-08 13:31:00
---
> 安全的Json..

<!--more-->
学习go的后端框架gin的时候，[官方文档](https://github.com/gin-gonic/gin)提到了这种代码来返回安全的json
```
func main() {
	r := gin.Default()

	// You can also use your own secure json prefix
	// r.SecureJsonPrefix(")]}',\n")

	r.GET("/someJSON", func(c *gin.Context) {
		names := []string{"lena", "austin", "foo"}

		// Will output  :   while(1);["lena","austin","foo"]
		c.SecureJSON(http.StatusOK, names)
	})

	// Listen and serve on 0.0.0.0:8080
	r.Run(":8080")
}
```
在不设置前缀的情况下，默认在返回的json加上`while(1);`  
[stackoverflow](https://stackoverflow.com/questions/2669690/why-does-google-prepend-while1-to-their-json-responses)查阅一下，说是为了防止json hijacking（json劫持）  

> 关于这方面的资料不是很多，中文的更少了。目前这些漏洞大都修复了。

假设有一个`http://www.safe.com/contacts`的接口， 返回的内容如下
```
[
  {
    "id": 1,
    "name": "Riven"
  },
  {
    "id": 2,
    "name": "Miss Fortune"
  }
]
```

由于浏览器的同源策略，使得和安全网站的域名、端口、协议不同的页面是不能直接发起上面请求的，这样恶意网站`http://www.evil.com/index.html`并不能直接请求安全接口。  
以下情况是没有同源策略限制的：
1. **页面中的链接。**很常见的就是导航页面中的链接。
2. **跨域资源获取，当然，浏览器限制了Javascript不能读写加载的内容**。这种允许的跨域请求有:`src`属性(服务器可以拒绝)、`<iframe>`（服务器可以拒绝）

json劫持是通过`<script>`来进行的。

首先，恶意者在`http://www.evil.com/index.html`的写入一行
	
   	<script src="http://www.safe.com/contacts"></script>
由于这种接口一般利用cookie来保证登录状态，所以恶意者把这个钓鱼网站以邮件的形式发到受害者的邮箱，  

受害者懵逼点开网站之后，`<script>`标签就跑起来了，一个Get请求带上还没过期的cookie信息，返回了json数组。

重点来了，虽然浏览器不允许对资源的操作加载的内容，可是`<script>`是会运行加载到的东西的(这也是jsonp的运行机制),而json数组文本是可以被js引擎运行的，所以浏览器会构造一个`Array`，但是并没有赋值给谁。

于是恶意者就从构造函数开始动手，如这篇[文章](http://www.thespanner.co.uk/2011/05/30/json-hijacking/)，重写`Array`的构造函数，或者重写`Objects.prototype.__defineSetter__`来进行原型函数的替换，从而加入自己的恶意代码，如把数据发送到自己的服务器去。

如果`http://www.safe.com/contacts`接口返回的是**对象**而不是**数组**呢，比如该接口返回`{"id": 2, "name": "batman"}`，那么js引擎就会报错，无法运行这个文本。

或者，在json数组前加上一些**垃圾串、死循环代码**，再用自己的**json解析器**解析出正确的json数组。比如在**Gmail网页版**里，就有这样的带有垃圾串的**json数组**返回。

![upload successful](/images/pasted-126.png)

**在es5之后，这些漏洞都不能被利用了**，但是如果我们要接别人这种返回带前缀json的api怎么办呢，这位[老哥](http://blackbe.lt/safely-handling-json-hijacking-prevention-methods-with-jquery/)说JQuery用一个过滤器处理（只针对`//`, `while(true);`, `for(;;);`）
```
$.ajaxSetup({
    dataFilter: function(data, type) {
        var prefixes = ['//', 'while(true);', 'for(;;);'],
            i,
            l,,
            pos;

        if (type != 'json' && type != 'jsonp') {
            return data;
        }

        for (i = 0, l = prefixes.length; i < l; i++) {
            pos = data.indexOf(prefixes[i]);
            if (pos === 0) {
                return data.substring(prefixes[i].length);
            }
        }

        return data;
    }
});
```