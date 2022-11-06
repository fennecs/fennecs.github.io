---
title: '[Java代理]Spring的cglib与final方法的坑'
tags:
  - AOP
  - BUG
categories:
  - Proxy
slug: 4249712641
date: 2020-01-12 13:51:39
---
# 前言
cglib是采用生成目标类subclass方式来代理目标类的，所以如果目标类的方法是`final`的话，就会直接调用目标类的方法。Spring的cglib代理实现有点坑，就是生成的代理对象是没有调用父类构造函数的，这个代理对象的成员变量不会初始化。
# null
Spring的aop是基于代理的，而采用cglib的实现下，对于一个bean，假设只代理一次的情况下（增强一次），内存实际是有两个对象，一个代理对象，一个目标对象。当我们调用代理对象的非fianl方法，代理对象调用完切面逻辑，不是调用自己父类方法，而是调用目标对象的目标方法；如果调用代理对象的final方法，代理对象会直接调用目标方法。

看下面的例子：
```java
public class DefaultRunner implements Runner {
    protected final Logger log = LoggerFactory.getLogger(getClass());

    @Override
    public final void run(String name) {
        log.info("I'm {}", name);
    }
}

public class CglibProxy implements MethodInterceptor {
    private Enhancer enhancer = new Enhancer();
·
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        objects[0] = "hijacked!";
        return methodProxy.invokeSuper(o, objects);
    }

    public Object newProxy(Class klass) {
        enhancer.setSuperclass(klass);
        enhancer.setCallback(new CglibProxy());
        return enhancer.create();
    }
}

@Test
public void testCglib() {
    CglibProxy cglibProxy = new CglibProxy();
    DefaultRunner runner = (DefaultRunner) cglibProxy.newProxy(DefaultRunner.class);
    runner.run("proxy");
}
```
如果我们调用代理对象的`runner.run("Tony")`，日志输出的将会是“I'm Tony”而不是“I'm hijacked!”。

而Spring使用`org.springframework.aop.framework.CglibAopProxy`对目标对象进行代理，我的程序在调用一个fianl方法的时候，报了一个NPE异常，debug进去一看，成员变量`logger`居然是`null`。
![](../images/20200112142005.png)
事实上，Spring的`CglibAopProxy`生成的代理对象的成员变量都是null，因为从Spring设计上，代理类只负责增强逻辑，然后再调用目标对象的方法，所以并没有初始化成员变量的必要。

![](../images/20200112142143.png)

实际上我也不需要这个对象被代理。。手动new一个对象就正常了。

# 为什么没有初始化成员变量？
我们在反射生成代理对象的时候，会调用构造方法。如果子类无参构造函数没有写super()，编译的时候，是会写进一句`super()`的。

但是由于是代理类是用子节码生成的，所以代理类构造函数是没有调用`super()`的，故没有初始化成员变量。

# 参考
1. [Spring AOP避坑指南](https://zhuanlan.zhihu.com/p/131584403)
