---
title: '[Java代理]Jdk代理'
tags:
  - AOP
  - ''
categories:
  - Proxy
  - ''
slug: 68869360
date: 2019-03-05 00:16:00
draft: true
---
# 前言
相比静态代理需要手动写代理类，动态代理可以通过抽象代码完成对一定规则的类的代理，生成的代理类直接以字节码的形式存在于内存中。Spring里Aop的实现使用了两种动态代理方案，一种是jdk代理，一种是`cglib`代理。

jdk代理是从目标类的接口生成实现类，cglib是继承目标类生成子类。

# Demo
```java
/**
 * 接口
 */
public interface Runner {
    void run(String name);
}
```
```java
/**
 * 接口实现类
 * created by Huang.Zhen on 2019-02-22
 */
public class RunnerDefault implements Runner {
    private Logger log = LoggerFactory.getLogger(getClass());

    @Override
    public void run(String name) {
        log.info("run: " + name);
    }
}
```
```java
/**
 * 代理类
 * created by Huang Zhen on 2019-02-22
 */
public class JdkProxyHandler implements InvocationHandler {
    private Logger log = LoggerFactory.getLogger(getClass());

    private Object target;

    // 保存目标类的引用
    public JdkProxyHandler(Object target) {
        this.target = target;
    }

    /**
     * 对newProxyInstance方法的封装
     * @return 代理类
     */
    public Object getProxy() {
        // 生成代理类
        return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        args[0] = "jdk " + args[0];
        return method.invoke(target, args);
    }
}
```
用下面的代码获取代理类并运行
```java
    @Test
    public void testJdk() {
        Runner runner = (Runner) new JdkProxyHandler(new RunnerDefault()).getProxy();
        runner.run("proxy");
    }
```
输出 

    2019-03-01 16:02:17.623  INFO   --- [           main] com.htc.learning.api.impl.RunnerDefault  : run: jdk proxy

# 原理
## 怎么生成代理类class文件
直捣`Proxy.newProxyInstance`方法，
```java
/*
 * Look up or generate the designated proxy class.
 */
Class<?> cl = getProxyClass0(loader, intfs);
```
进入`getProxyClass0`，从`proxyClassCache`字面上理解，jdk代理是有缓存的
```java
// If the proxy class defined by the given loader implementing
// the given interfaces exists, this will simply return the cached copy;
// otherwise, it will create the proxy class via the ProxyClassFactory
return proxyClassCache.get(loader, interfaces);
```
进入get方法。可以看到出现了jdk8级别的代码，说明jdk8里jdk代理又被优化了
```java
// create subKey and retrieve the possible Supplier<V> stored by that
// subKey from valuesMap
// subKeyFactory 其实是 java.lang.reflect.Proxy.ProxyClassFactory
Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
Supplier<V> supplier = valuesMap.get(subKey);
...
V value = supplier.get();
if (value != null) {
    return value;
}
...
```
进入`java.lang.reflect.Proxy.ProxyClassFactory#apply`方法
```java
...
/*
 * Generate the specified proxy class.
 */
byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
    proxyName, interfaces, accessFlags);
...
```
```java
public static byte[] generateProxyClass(final String var0, Class<?>[] var1, int var2) {
    ProxyGenerator var3 = new ProxyGenerator(var0, var1, var2);
    // 生成字节码的方法，不想看
    final byte[] var4 = var3.generateClassFile();
    // 这里可以通过命令行参数设置要不要存储生成的class文件
    if (saveGeneratedFiles) {
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
                try {
                    int var1 = var0.lastIndexOf(46);
                    Path var2;
                    if (var1 > 0) {
                        Path var3 = Paths.get(var0.substring(0, var1).replace('.', File.separatorChar));
                        Files.createDirectories(var3);
                        var2 = var3.resolve(var0.substring(var1 + 1, var0.length()) + ".class");
                    } else {
                        var2 = Paths.get(var0 + ".class");
                    }

                    Files.write(var2, var4, new OpenOption[0]);
                    return null;
                } catch (IOException var4x) {
                    throw new InternalError("I/O exception saving generated file: " + var4x);
                }
            }
        });
    }

    return var4;
}
```

> 以往都是说jdk代理比cglib性能差，其实优化到现在都没差多少了，更多的时候是从两者的特性按需求采取不同的。

## 缓存
jdk代理获取Class的时候使用了缓存
```java
private static Class<?> getProxyClass0(ClassLoader loader,
                                       Class<?>... interfaces) {
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }

    // If the proxy class defined by the given loader implementing
    // the given interfaces exists, this will simply return the cached copy;
    // otherwise, it will create the proxy class via the ProxyClassFactory
    return proxyClassCache.get(loader, interfaces);
}
```
`proxyClassCache`的声明是这样的
```java
/**
     * a cache of proxy classes
     */
    private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
```
主要的成员变量
```java
// 引用队列
private final ReferenceQueue<K> refQueue
    = new ReferenceQueue<>();
// 缓存本存
// the key type is Object for supporting null key
private final ConcurrentMap<Object, ConcurrentMap<Object, Supplier<V>>> map
    = new ConcurrentHashMap<>();
// 反向索引，用来快速判断一个对象是否存在缓存里
private final ConcurrentMap<Supplier<V>, Boolean> reverseMap
    = new ConcurrentHashMap<>();
// 两个二元运算方法
// subKeyFactory = new KeyFactory()
private final BiFunction<K, P, ?> subKeyFactory;
// valueFactory = new ProxyClassFactory()
private final BiFunction<K, P, V> valueFactory;
```
这里里的缓存`map`的value又是一个`ConcurrentMap`,说明这个缓存是一个二级缓存。

|字段名|说明|
| ------ | ------ |
| 一级缓存key   | 一个CacheKey类型的对象，以ClassLoader作为hash |
| 一级缓存value | 一级缓存 |

|字段名|说明|
| ------ | ------ |
| 二级缓存key | 以interfaces为key |
| 二级缓存value | Supplier接口，可能是CacheValue 或者 代理工厂对象 |

看代码
```java
// 在Proxy类的代码中，key是ClassLoader，parameter是interface数组
public V get(K key, P parameter) {
    Objects.requireNonNull(parameter);

    expungeStaleEntries();

    Object cacheKey = CacheKey.valueOf(key, refQueue);

    // lazily install the 2nd level valuesMap for the particular cacheKey
    // 初始化二级缓存，用了双重校验，保证所有线程拿到的是同一个实例
    ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
    if (valuesMap == null) {
        ConcurrentMap<Object, Supplier<V>> oldValuesMap
            = map.putIfAbsent(cacheKey,
                              valuesMap = new ConcurrentHashMap<>());
        if (oldValuesMap != null) {
            valuesMap = oldValuesMap;
        }
    }

    // create subKey and retrieve the possible Supplier<V> stored by that
    // subKey from valuesMap
    Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
    Supplier<V> supplier = valuesMap.get(subKey);
    Factory factory = null;

    while (true) {
        if (supplier != null) {
            // supplier might be a Factory or a CacheValue<V> instance
            V value = supplier.get();
            if (value != null) {
                return value;
            }
        }
        // else no supplier in cache
        // or a supplier that returned null (could be a cleared CacheValue
        // or a Factory that wasn't successful in installing the CacheValue)

        // lazily construct a Factory
        if (factory == null) {
            factory = new Factory(key, parameter, subKey, valuesMap);
        }

        if (supplier == null) {
            supplier = valuesMap.putIfAbsent(subKey, factory);
            if (supplier == null) {
                // successfully installed Factory
                supplier = factory;
            }
            // else retry with winning supplier
        } else {
            if (valuesMap.replace(subKey, supplier, factory)) {
                // successfully replaced
                // cleared CacheEntry / unsuccessful Factory
                // with our Factory
                supplier = factory;
            } else {
                // retry with current supplier
                supplier = valuesMap.get(subKey);
            }
        }
    }
}
```
* 这里的代码主要是维护`Map.put`操作多线程下的一些同步，防止重复实例化。虽然`map`是`ConcurrentHashMap`,但重复put还是得避免的。
* 二级缓存`Map`的`value`为`Supplier`类型，第一次访问是 `Factory`对象，第二次访问就可能是`CacheValue `，因为`Factory`存有二级缓存map的引用，会把`value`从`this`（`Factory`本身）替换为`CacheValue`

> 其实不太明白这种机制,可能为了提高并发性能？先返回值，再为值构造缓存。

## 缓存过期机制
`CacheKey`是一个`WeakReference`，当gc时就会被清理掉引用的对象，这时需要把`CacheKey`从`Map`里`remove`，下面这个方法在`WeakCache`执行读操作的时候会执行一遍。
```java
    private void expungeStaleEntries() {
        CacheKey<K> cacheKey;
        while ((cacheKey = (CacheKey<K>)refQueue.poll()) != null) {
            cacheKey.expungeFrom(map, reverseMap);
        }
    }
```

> `CacheValue`也是虚引用。


## InvocationHandler注入
代理类已经生成了，我们写的`InvocationHandler`还没有注入，所以生成代理类的时候是不包含代理逻辑的。

我们回到`Proxy.newProxyInstance`方法，这时已经获取到`class`，
```java
...
final Constructor<?> cons = cl.getConstructor(constructorParams);
final InvocationHandler ih = h;
if (!Modifier.isPublic(cl.getModifiers())) {
    AccessController.doPrivileged(new PrivilegedAction<Void>() {
        public Void run() {
            cons.setAccessible(true);
            return null;
        }
    });
}
return cons.newInstance(new Object[]{h});
```
`InvocationHandler`是通过构造参数注入的。

## 代理class文件的内容
我们生成基本的class文件只需要给一个自定义类名和一个目标类就可以了。
```java
 @Test
    public void saveJdkProxyClass() throws IOException {
        String path = "./$Proxy0.class";
        byte[] classFile = ProxyGenerator.generateProxyClass("$Proxy0", RunnerDefault.class.getInterfaces());
        FileOutputStream out = null;
        try {
            out = new FileOutputStream(path);
            out.write(classFile);
            out.flush();
        } finally {
            if (out != null) {
                out.close();
            }
        }
    }
```
```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

import com.htc.learning.api.Runner;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements Runner {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void run(String var1) throws  {
        try {
            super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("com.htc.learning.api.Runner").getMethod("run", Class.forName("java.lang.String"));
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```
> 那么cglib方案的代理类class文件又要怎么获取呢，下篇再说。

可以看到字节码里每个目标方法都有一个同名的代理方法包着，代理逻辑已经写在`InvocationHandler`，代理方法直接调用`InvocationHandler`就可以了。

# 从代理类获取原始对象的Class
在Spring里，bean被代理是很常见的，假如我们要获取目标bean上的注解，这时候我们拿到的如果是代理类，是获取不到的目标bean上的注解的。所以这时我们得从代理类获取原始对象，再获得对应的Class。

## 我的做法
在`InvocationHandler`实现类里，把目标类对象放入了一个`target`成员变量，然后当我们拿到代理类后，通过调用`java.lang.reflect.Proxy#getInvocationHandler`方法，再通过反射即可获取到原始对象`target`。
## Spring的做法
Spring有一个AopProxyUtils的工具，其中有个方法可以获取到jdk代理或cglib代理的原始对象。关于这个工具更多使用参考[AopProxyUtils详解](https://www.jianshu.com/p/273d8e2bb992)
```java
    // candidate即传入的代理类实例
	public static Class<?> ultimateTargetClass(Object candidate) {
		Assert.notNull(candidate, "Candidate object must not be null");
		Object current = candidate;
		Class<?> result = null;
        // Spring的代理类都实现了 TargetClassAware，调用getTargetClass()可获取到目标对象，注意，这里不一定是原始对象，因为可能切面切了很多次，生成了很多层的代理类，这也是为什么需要一个while循环
		while (current instanceof TargetClassAware) {
			result = ((TargetClassAware) current).getTargetClass();
			current = getSingletonTarget(current);
		}
		if (result == null) {
            // 如果是cglib代理，则获取对象父类，否则是jdk代理，直接获取对象类型
			result = (AopUtils.isCglibProxy(candidate) ? candidate.getClass().getSuperclass() : candidate.getClass());
		}
		return result;
	}
```
```java
	public static Object getSingletonTarget(Object candidate) {
		if (candidate instanceof Advised) {
			TargetSource targetSource = ((Advised) candidate).getTargetSource();
			if (targetSource instanceof SingletonTargetSource) {
				return ((SingletonTargetSource) targetSource).getTarget();
			}
		}
		return null;
	}
```
这里的代码逻辑看起来不难，但是涉及了Spring Aop的接口概念，所以具体调用我也不太懂是干嘛的。

> Spring在代理逻辑中拦截了`getTargetClass()`等切面方法，将这些方法转发给`Advised`去执行。


# 缺陷
从class文件看，由于代理类继承了`Proxy`类（其实这个类看起来也只有一个`java.lang.reflect.Proxy#getInvocationHandler`比较通用的方法，其实我觉得这个`InvocationHandler`可以通过反射拿到，不懂为什么非要继承这个类，喵？），导致jdk代理不能通过继承目标类来达到代理的目的。

# 关于Spring
Spring有个属性是`proxy-target-class`，默认值是`false`，表示默认使用jdk代理，这时使用常常会发生类型转换的错误，因为最终bean的class已经不是最初的bean的类型。
在Springboot里，`proxy-target-class`使用`spring.aop.proxy-target-class`属性来配置，默认为`true`，即都使用`cglib`来代理。如果配置为`false`，Springboot会对实现接口的bean使用jdk代理，对于没有实现接口的类依旧使用`cglib`代理。

# 参考
1. [AopProxyUtils详解](https://www.jianshu.com/p/273d8e2bb992)