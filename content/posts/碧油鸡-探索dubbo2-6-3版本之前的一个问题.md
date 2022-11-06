---
title: '[Dubbo]探索dubbo2.6.3版本之前的一个问题'
author: 土川
tags:
  - BUG
categories:
  - Dubbo
  - ''
slug: 1632594101
date: 2018-10-20 10:55:00
---
> 由于`dubbo`版本较低遇到了一个诡异的问题。

<!--more-->
# 场景
`dubbo`的`RpcContext`存放了一个`attachments`属性，用于隐式传递参数，每次发起调用之后会`clear`清空。

使用了cat监控之后，可以在`dubbo`服务调用之间传递一个id作为链路跟踪。而这个参数就是在`Comsumer`发起请求前放入`attachments`，在`Provider`接收到请求后从`attachments`拿出。

测试环境调用某服务会出现这个id时有时无的情况，于是分析日志，发现没有这个id的情况下，dubbo协议走的是hessian协议。原来此服务提供了hessian访问，于是客户端会随机走dubbo或者hessian协议。

为什么hessian协议会丢`attachments`？这个id是通过实现`dubbo`的`Filter`塞进去的，理论上Filter应该是协议无关的。通过debug发现，在发送请求之前的`RpcContext.attachments`里也的确是有这个id。

# 探索
## dubbo协议
dubbo的consumer调用抽象为Invoker和Invocation，在Invoker执行invoke方法之前，需要执行负载均衡、重试计数、拦截器调用链，最后执行抽象类`AbstractInvoker.invoke`方法，继而调用不同协议的`doInvoke`方法

看看dubbo协议的doInvoke方法，

![upload successful](/images/pasted-154.png)
可以看到请求是由一个`HeaderExchangeClient`去发起，
![upload successful](/images/pasted-153.png)
这里的的channel是一个`nettyClient`，而`request`的内容如下：
![upload successful](/images/pasted-155.png)
dubbo协议的实现是利用`socket长连接`，将整个`request`对象发送过去的,没有丢`attachments`

## hessian协议
hessian协议下是通过`com.caucho.hessian.client.HessianProxy#invoke`来发起请求，这个方法签名如下：
```java
public Object invoke(Object proxy, Method method, Object[] args)
```
第一个参数不知道是干嘛的，第二个参数和第三个参数分别是调用服务的方法对象和参数。反正这个方法是没地方放`attchments`这种**隐式参数**含义的东西。

debug看他的调用栈，

![upload successful](/images/pasted-156.png)

定位到这个`com.alibaba.dubbo.rpc.proxy.AbstractProxyInvoker#invoke`方法

```java
    public Result invoke(Invocation invocation) throws RpcException {
        try {
            return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
        } catch (InvocationTargetException e) {
            return new RpcResult(e.getTargetException());
        } catch (Throwable e) {
            throw new RpcException("Failed to invoke remote proxy method " + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```

可以看到，从这里开始
```java
return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
```
就把invocation里的`attachments`丢了

![upload successful](/images/pasted-157.png)


# 解决方法
dubbo版本至少升级直2.6.3，这个版本对提供了对hessian协议的`attachments`支持。

![upload successful](/images/pasted-158.png)

下面是部分代码，
```java
public class DubboHessianURLConnectionFactory extends HessianURLConnectionFactory {

    @Override
    public HessianConnection open(URL url) throws IOException {
        HessianConnection connection = super.open(url);
        RpcContext context = RpcContext.getContext();
        for (String key : context.getAttachments().keySet()) {
            connection.addHeader(Constants.DEFAULT_EXCHANGER + key, context.getAttachment(key));
        }

        return connection;
    }
}
```
dubbo通过继承hessian库的类，在处理`URL`的时候把`attachments`放到header里去了，接收请求时再从header里拿出来。
