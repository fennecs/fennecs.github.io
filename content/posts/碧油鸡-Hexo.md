---
title: '[碧油鸡]Hexo-admin的404探索'
author: 土川
tags:
  - BUG
categories:
  - 建站
slug: 3915904827
date: 2018-03-10 10:58:00
---
> 作为一个不懂js的后端开始了瞎搞。。

hexo-admin在写的时候，一般是把图片放到图床，再获得图片链接加入md，但是笔者习惯了为知笔记直接把图片复制粘贴进md里。不过hexo-admin确实加入了这个直接粘贴图片的功能，把图片复制进md里，有时会出现一个404的问题，一直百度谷歌无果，同事老哥一直叫我用gitbook，但是不找出这个404的谜底实在难受，于是有了下面的记录。
# First Blood 第一张图

![upload successful](/images/pasted-19.png)
如上图，上传成功后出现了404，点击404发生的代码，如下：
![upload successful](/images/pasted-21.png)
hexo-admin直接生成html进行html内容的替换。
可以看到笔者把图片粘贴之后，hexo-admin做了三个操作：
1. 上传图片，返回`![upload successful](/images/pasted-8.png)`的字符串给客户端
1. 客户端根据返回的字符串重新渲染
1. 把新的文章内容post到服务器

**为什么上传成功还404???**
这时笔者点开图片的链接

![upload successful](/images/pasted-22.png)
没问题啊，诡异。
# 第二张图
粘贴第二张图片的时候

![upload successful](/images/pasted-23.png)

可以看到第一次获取失败的`Nubia Z17`获取成功了，证明图片是上传成功，然而`林允儿`在上传完成后，并没有404
# 第三张图片
粘贴Gakki，又是404

![upload successful](/images/pasted-30.png)
# 其他现象
* 上面是在宿舍的操作，在公司操作的时候完全没出现这种404。不过有一个现象是，公司网络ping这台vps的时候有点慢，宿舍网ping这台vps非常快。
* 新建了另一台vps，同样的环境，也没有404。但是这台vps在洛杉矶，网络时延有点大。

# 各种小动作
于是在404的js打上断点，

![upload successful](/images/pasted-24.png)
运行到这里之后，继续执行所有js，没发生404。

于是乎猜想：
> 服务器保存图片的接下来，返回了response，肯定还进行了`某个操作`，但是短时间内这个操作没完成，导致短时间内访问的时候还不能通。

这可以解释上面：
1. 第一次操作是404，第二次却是两张图片都成功请求到。原因是第二次访问时，由于我的`Nubia`没成功加载，他会去再访问一次，这就导致了`林允儿`的请求会稍晚一点，而这点时间间隔里，对`林允儿`的`某个操作`已经完成，所以第二次操作不存在404，而第三次操作，`Gakki`又没给`某个操作`留时间完成，请求结果明显404。
2. 公司访问我的vps比较慢，网络上的时延足够`某个操作完成`
3. 断点的时延足够`某个操作完成`

那么上vps看日志，三次操作依次三张图。
> `hexo server --debug`可以输出日志

![upload successful](/images/pasted-26.png)


![upload successful](/images/pasted-27.png)


![upload successful](/images/pasted-28.png)
利用小学找规律题目的尿性，我发觉**404发生在这坨Generator操作之前**
> 规律不是3次操作总结的，其实复现了很多次了

网上看到`hexo-server`会jian视`source`文件夹的变动，我手动把一张图片加入`source/images`目录下，他也自动跑出`Generator`日志。于是我猜想这个`Generator`不跑完，就不能通过`hexo-server`访问到图片。

怎么验证，琢磨去改`hexo-admin`代码，上[github](https://github.com/jaredly/hexo-admin)，不懂node.js，看到一个`api.js`的就点进去，惊喜找到这个接口

![upload successful](/images/pasted-29.png)

> 推荐个看GitHub的神器`Octotree`，我在chrome商店装的，可以直接看到GitHub的目录。

这个方法拉到最底
```javascript
...
    var dataURI = req.body.data.slice('data:image/png;base64,'.length)
    var buf = new Buffer(dataURI, 'base64')
    hexo.log.d(`saving image to ${outpath}`)
    fs.writeFile(outpath, buf, function (err) {
      if (err) {
        console.log(err)
      }
      hexo.source.process().then(function () {
        res.done({
          src: path.join(hexo.config.root + filename),
          msg: msg
        })
      });
    })
...
```
把服务器发送响应`res.done`语句延迟50ms执行后，再也没有发生过过404
```javascript
...
    var dataURI = req.body.data.slice('data:image/png;base64,'.length)
    var buf = new Buffer(dataURI, 'base64')
    hexo.log.d(`saving image to ${outpath}`)
    fs.writeFile(outpath, buf, function (err) {
      if (err) {
        console.log(err)
      }
      hexo.source.process().then(function () {
        setTimeout(function(){ // 百度来的延迟执行的方法。。
          res.done({
            src: path.join(hexo.config.root + filename),
            msg: msg
          })
        }, 50);
      });
    })
...
```
> 为什么是50ms？看日志的时间，100ms左右可以保证`Generator`操作完成，于是我直接设置50ms，到现在也没再404过。

# 后记
我看`hexo-admin`也没人提这个issue，估计直接粘贴图片的人不多，毕竟这种方法不能清理md不再引用的图片。
我能想到的解决方法就是延迟执行再响应请求了，毕竟这个线程也不清楚`Generator`什么时候才执行完。。。