title: '[Java基础]Java的SPI机制'
author: 土川
slug: 754409717
tags:
  - OOP
categories:
  - Java基础
date: 2018-11-29 16:04:00
---
# 为什么要SPI
SPI, Service Provider Interface, 简单来说就是调用方提供接口，接入方提供实现。比如一个应用程序调用JDBC的接口，你要是使用mysql，就得提供mysql提供的jdbc实现。这和我们定义**接口**然后写**实现类**差不多，只不过**实现类**是可以在jar/war外提供。

# 原理
java的实现无非是读取文件，按类名加载。

## 调用方定义接口
```java
public interface Name {
    String getName();
}
```
## 第三方实现接口
```java
public class HtcName implements Name {
    @Override
    public String getName() {
        return "htc";
    }
}
```
```java
public class DefaultName implements Name {
    @Override
    public String getName() {
        return "default";
    }
}
```
## 声明第三方实现
在`CLASSPATH`下建`META-INF/services`,这个路径是java的代码写死的。  
然后新建一个文件，文件名为接口名。

![upload successful](/images/pasted-160.png)

文件的内容就是声明要加载的实现类。

    com.htc.learning.api.impl.DefaultName
    com.htc.learning.api.impl.HtcName

## 调用方加载实现类
使用`ServiceLoader`类加载实现类，他会搜索`CLASSPATH`下的所有的"META-INF/services/com.htc.learning.api.Name"文件，获取所有声明一一加载。**注意,类的实例化发生在遍历的时候**
```java
public class SpiTest {
    public static void main(String[] args) {
        ServiceLoader<Name> serviceLoader = ServiceLoader.load(Name.class);
        for (Name name : serviceLoader) {
            System.out.println(name.getName());
        }
    }
}
```

# 源码
看看`ServiceLoader.load(Name.class)`做了什么事。
```java
    public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }
```
> 这里使用了线程上下文类加载器，因为内置Spi接口都是由Bootstrap类加载器加载，Bootstrap类加载器又加载不了第三方实现类，所以要使用线程上下文类加载器（默认是App类加载器）

进入方法后，获取了`ClassLoader`,接下来继续进入`ServiceLoader.load(service, cl);`方法。
```java
    public static <S> ServiceLoader<S> load(Class<S> service,
                                            ClassLoader loader)
    {
        return new ServiceLoader<>(service, loader);
    }
```
返回了一个实例，那么查看他的构造方法。
```java
    public void reload() {
        providers.clear();
        lookupIterator = new LazyIterator(service, loader);
    }
    
    private ServiceLoader`(Class<S> svc, ClassLoader cl) {
        service = Objects.requireNonNull(svc, "Service interface cannot be null");
        loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
        acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
        reload();
    }
```
到这里，完全没有任何实例化的代码。

前面说到是实例化发生在遍历的时候，在构造函数里也有实例化一个`LazyIterator`的类，我们转到`ServiceLoader`的`iterator()`方法。
```java
public Iterator<S> iterator() {
    return new Iterator<S>() {

        Iterator<Map.Entry<String,S>> knownProviders
            = providers.entrySet().iterator();

        public boolean hasNext() {
            if (knownProviders.hasNext())
                return true;
            return lookupIterator.hasNext();
        }

        public S next() {
            if (knownProviders.hasNext())
                return knownProviders.next().getValue();
            return lookupIterator.next();
        }

        public void remove() {
            throw new UnsupportedOperationException();
        }

    };
}
```
先是`hasNext()`,有一个`knownProviders`的变量，它从`providers`属性获得，这个属性是一个`LinkedHashMap<String,S>`，起到一个缓存的作用，保证实现类只被加载一次。我们可以不关注缓存，看`lookupIterator.hasNext()`。`lookupIterator`在上面`lookupIterator = new LazyIterator(service, loader);`被赋值了的，下面是`LazyIterator`的代码
```java
public boolean hasNext() {
    if (acc == null) {
        return hasNextService();
    } else {
        PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
            public Boolean run() { return hasNextService(); }
        };
        return AccessController.doPrivileged(action, acc);
    }
}
```
这里有java `SecurityManager`的管理，直接看`hasNextService()`的代码就好了。
```java
private boolean hasNextService() {
    //nextName在上一次查询提前缓存实现类的名字，做到快速判断
    if (nextName != null) {
        return true;
    }
    if (configs == null) {
        try {
            // 这里的 PREFIX 就是  "META-INF/services/"
            String fullName = PREFIX + service.getName();
            if (loader == null)
                configs = ClassLoader.getSystemResources(fullName);
            else
                configs = loader.getResources(fullName);
        } catch (IOException x) {
            fail(service, "Error locating configuration files", x);
        }
    }
    while ((pending == null) || !pending.hasNext()) {
        if (!configs.hasMoreElements()) {
            return false;
        }
        pending = parse(service, configs.nextElement());
    }
    // 提前缓存实现类的名字，下一次查询做到快速判断
    nextName = pending.next();
    return true;
}
```
接下来看`next()`操作。
```java
public S next() {
    if (knownProviders.hasNext())
        return knownProviders.next().getValue();
    return lookupIterator.next();
}
```
转到`lookupIterator.next()`
```java
public S next() {
    if (acc == null) {
        return nextService();
    } else {
        PrivilegedAction<S> action = new PrivilegedAction<S>() {
            public S run() { return nextService(); }
        };
        return AccessController.doPrivileged(action, acc);
    }
}
```
看`nextService()`
```java
private S nextService() {
    // 这里会执行hasNextService()的判断
    if (!hasNextService())
        throw new NoSuchElementException();
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service,
             "Provider " + cn + " not found");
    }
    if (!service.isAssignableFrom(c)) {
        fail(service,
             "Provider " + cn  + " not a subtype");
    }
    try {
        S p = service.cast(c.newInstance());
        providers.put(cn, p);
        return p;
    } catch (Throwable x) {
        fail(service,
             "Provider " + cn + " could not be instantiated",
             x);
    }
    throw new Error();          // This cannot happen
}
```
实际上`nextService() `没参与遍历，实现类的遍历是交给了`hasNextService()`并把遍历到的`ClassName`存放到`nextName`属性，`nextService() `只负责把`nextName`实例化，并且放入缓存中。

# 缺点
java的SPI是有缺点的，这也是dubbo为什么要实现自己的SPI机制。

JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。

# 题外话
我们java程序加载数据库驱动的时候，利用的就是java的spi机制。如果我们引入了多个数据库的jdbc驱动jar，那么java怎么知道加载哪一个呢。

我们我的数据库驱动是由`java.sql.DriverManager`管理的，现在看他的一个`getDriver()`方法，

```java
@CallerSensitive
public static Driver getDriver(String url)
    throws SQLException {
    ...
    // who understands the given URL.
    for (DriverInfo aDriver : registeredDrivers) {
        // If the caller does not have permission to load the driver then
        // skip it.
        if(isDriverAllowed(aDriver.driver, callerClass)) {
            try {
                // 就是这一行！
                if(aDriver.driver.acceptsURL(url)) {
                    // Success!
                    println("getDriver returning " + aDriver.driver.getClass().getName());
                return (aDriver.driver);
                }

            } catch(SQLException sqe) {
                // Drop through and try the next driver.
            }
        } else {
            println("    skipping: " + aDriver.driver.getClass().getName());
        }

    }
    ...
}
```
看`if(aDriver.driver.acceptsURL(url))`这一行，`DriverManager`用连接的url对每一个驱动进行尝试，尝试得通就是这个驱动没跑了。。。

# 结束语
无。