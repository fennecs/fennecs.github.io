tags:
  - OOP
categories:
  - Dubbo
title: '[Dubbo]Dubbo SPI 机制'
slug: 2436052280
date: 2019-03-25 11:44:00
---
# 前言
之前一篇[[Java基础]Java的SPI机制](./754409717.html)讲到Java spi的缺陷是在查找所需实现的时候，会实例化无关的实现，那么这篇看看Dubbo是怎么规避这个问题的。

# Dubbo spi的特点
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface SPI {

    /**
     * default extension name
     */
    String value() default "";

}
```
由于配置文件是以key-value配置的，这里可以为SPI接口指定一个默认实现的key

**Dubbo spi有以下特点**
1. 不需要遍历所有实例化所有实现类
2. 增加了对扩展点IoC和AOP的支持，一个扩展点可以直接setter注入其它扩展点。

# Demo
接口,需要加上`org.apache.dubbo.common.extension.SPI`注解
```java
@SPI
public interface Runner {
    void run(String name);
}
```
两个实现
```java
public class DefaultRunner implements Runner {
    private Logger log = LoggerFactory.getLogger(getClass());

    @Override
    public void run(String name) {
        log.info("I'm a DefaultRunner");
    }
}

public class ExcitedRunner implements Runner {
    private Logger log = LoggerFactory.getLogger(getClass());

    @Override
    public void run(String name) {
        log.info("I'm a ExcitedRunner!");
    }
}
```
在资源目录`META-INF/dubbo`下创建文件`com.htc.learning.api.Runner`

![upload successful](/images/pasted-172.png)

内容是一行一行的键值对，value是实现类，我们只要通过key就能直接拿到所需类。
```java
default=com.htc.learning.api.impl.DefaultRunner
excited=com.htc.learning.api.impl.ExcitedRunner
```
测试方法
```java
@Test
public void dubboSpiTest() {
    ExtensionLoader<Runner> extensionLoader = ExtensionLoader.getExtensionLoader(Runner.class);
    Runner defaultRunner = extensionLoader.getExtension("default");
    defaultRunner.run("htc");
    Runner excitedRunner = extensionLoader.getExtension("excited");
    excitedRunner.run("htc");
}
```
输出
    
    2019-03-23 12:20:50.681  INFO   --- [           main] com.htc.learning.api.impl.DefaultRunner  : I'm a DefaultRunner
    2019-03-23 12:20:50.681  INFO   --- [           main] com.htc.learning.api.impl.ExcitedRunner  : I'm a ExcitedRunner!
    
# 原理
## getExtensionLoader
Dubbo spi与java spi的`ServiceLoader`对应的，是`ExtensionLoader`。不同于`ServiceLoader`的`load`方法每次返回都要实例化一个对象，`ExtensionLoader`每次`getExtensionLoader`会进行缓存。
```java

    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        if (type == null) {
            throw new IllegalArgumentException("Extension type == null");
        }
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
        }
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type(" + type +
                    ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
        }

        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```
`ExtensionLoader`构造函数
```java
    private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```
其中`objectFactory`是一个接口,与IOC相关，该接口根据type和name找到一个bean，这个bean可以是Spring的bean，也可以是一个dubbo spi实现类。
```java
@SPI
public interface ExtensionFactory {

    /**
     * Get extension.
     *
     * @param type object type.
     * @param name object name.
     * @return object instance.
     */
    <T> T getExtension(Class<T> type, String name);

}
```
可以看出，这是也是一个SPI接口，接下来调用的`getExtensionLoader`逻辑就是通过他实现的。

## getExtension

```java
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    if ("true".equals(name)) {
        // 获取默认的拓展实现类
        return getDefaultExtension();
    }
    // Holder，顾名思义，用于持有目标对象
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    // 双重检查
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                // 创建拓展实例
                instance = createExtension(name);
                // 设置实例到 holder 中
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

```java
private T createExtension(String name) {
    // 从配置文件中加载所有的拓展类，可得到“配置项名称”到“配置类”的映射关系表
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            // 通过反射创建实例
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // 向实例中注入依赖
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            // 循环创建 Wrapper 实例
            for (Class<?> wrapperClass : wrapperClasses) {
                // 将当前 instance 作为参数传给 Wrapper 的构造方法，并通过反射创建 Wrapper 实例。
                // 然后向 Wrapper 实例中注入依赖，最后将 Wrapper 实例再次赋值给 instance 变量
                instance = injectExtension(
                    (T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("...");
    }
}
```
这里按逻辑可以拆分成四步
1. `getExtensionClasses()`获取接口的所有实现类。这个方法会在很多地方被调用，保证所有类被获取到。由于多次调用缓存也是必须的。
1. 实例化目标类的对象。
1. IOC注入。注入其他SPI实现。
1. AOP实现。如果需要，把对象实例包裹在Wrapper中。

### getExtensionClasses
```java
private Map<String, Class<?>> getExtensionClasses() {
    // 从缓存中获取已加载的拓展类
    Map<String, Class<?>> classes = cachedClasses.get();
    // 双重检查
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                // 加载拓展类
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```
典型的双重校验，这里的缓存key是实现类的key，值就是实现类了。如果缓存为`null`，调用`loadExtensionClasses`初始化缓存。
```java
    // synchronized in getExtensionClasses
    private Map<String, Class<?>> loadExtensionClasses() {
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if (defaultAnnotation != null) {
            // 如果设置了默认实现，就缓存以下这个默认实现的key
            String value = defaultAnnotation.value();
            if ((value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if (names.length > 1) {
                    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                            + ": " + Arrays.toString(names));
                }
                if (names.length == 1) {
                    cachedDefaultName = names[0];
                }
            }
        }

        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
        loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());
        loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
        loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());
        loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
        return extensionClasses;
    }
```
java spi的配置只能放在一个目录`META-INF/services/`,dubbo spi的配置可以放在三个目录
```java
    private static final String SERVICES_DIRECTORY = "META-INF/services/";

    private static final String DUBBO_DIRECTORY = "META-INF/dubbo/";

    private static final String DUBBO_INTERNAL_DIRECTORY = DUBBO_DIRECTORY + "internal/";
```
这里的依赖我用的是apache-dubbo，相对于alibaba-dubbo三个目录，估计为了兼容，apache-dubbo版本的`loadDirectory`方法进行了一个包名的替换。

```java
private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir) {
    // fileName = 文件夹路径 + type 全限定名 
    String fileName = dir + type.getName();
    try {
        Enumeration<java.net.URL> urls;
        ClassLoader classLoader = findClassLoader();
        // 根据文件名加载所有的同名文件
        if (classLoader != null) {
            urls = classLoader.getResources(fileName);
        } else {
            urls = ClassLoader.getSystemResources(fileName);
        }
        if (urls != null) {
            while (urls.hasMoreElements()) {
                java.net.URL resourceURL = urls.nextElement();
                // 加载资源
                loadResource(extensionClasses, classLoader, resourceURL);
            }
        }
    } catch (Throwable t) {
        logger.error("...");
    }
}
```
接下来“真·读取配置文件”
```java
private void loadResource(Map<String, Class<?>> extensionClasses, 
	ClassLoader classLoader, java.net.URL resourceURL) {
    try {
        BufferedReader reader = new BufferedReader(
            new InputStreamReader(resourceURL.openStream(), "utf-8"));
        try {
            String line;
            // 按行读取配置内容
            while ((line = reader.readLine()) != null) {
                // 定位 # 字符
                final int ci = line.indexOf('#');
                if (ci >= 0) {
                    // 截取 # 之前的字符串，# 之后的内容为注释，需要忽略
                    line = line.substring(0, ci);
                }
                line = line.trim();
                if (line.length() > 0) {
                    try {
                        String name = null;
                        int i = line.indexOf('=');
                        if (i > 0) {
                            // 以等于号 = 为界，截取键与值
                            name = line.substring(0, i).trim();
                            line = line.substring(i + 1).trim();
                        }
                        if (line.length() > 0) {
                            // 加载类，并通过 loadClass 方法对类进行缓存
                            loadClass(extensionClasses, resourceURL, 
                                      Class.forName(line, true, classLoader), name);
                        }
                    } catch (Throwable t) {
                        IllegalStateException e = new IllegalStateException("Failed to load extension class...");
                    }
                }
            }
        } finally {
            reader.close();
        }
    } catch (Throwable t) {
        logger.error("Exception when load extension class...");
    }
}
```
读完配置文件，下面不单是对类进行加载，而且还对`Adaptive`、`Wrapper`类进行缓存。
```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("Error when load extension class(interface: " +
                type + ", class line: " + clazz.getName() + "), class "
                + clazz.getName() + "is not subtype of interface.");
    }
    // 只能有一个自适应实现类
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        if (cachedAdaptiveClass == null) {
            cachedAdaptiveClass = clazz;
        } else if (!cachedAdaptiveClass.equals(clazz)) {
            throw new IllegalStateException(...);
        }
    } else if (isWrapperClass(clazz)) {
        Set<Class<?>> wrappers = cachedWrapperClasses;
        if (wrappers == null) {
            cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
            wrappers = cachedWrapperClasses;
        }
        wrappers.add(clazz);
    } else {
        clazz.getConstructor();
        if (name == null || name.length() == 0) {
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                throw new IllegalStateException(...);
            }
        }
        String[] names = NAME_SEPARATOR.split(name);
        if (names != null && names.length > 0) {
            Activate activate = clazz.getAnnotation(Activate.class);
            if (activate != null) {
                cachedActivates.put(names[0], activate);
            } else {
                // support com.alibaba.dubbo.common.extension.Activate
                com.alibaba.dubbo.common.extension.Activate oldActivate = clazz.getAnnotation(com.alibaba.dubbo.common.extension.Activate.class);
                if (oldActivate != null) {
                    cachedActivates.put(names[0], oldActivate);
                }
            }
            for (String n : names) {
                if (!cachedNames.containsKey(clazz)) {
                    cachedNames.put(clazz, n);
                }
                Class<?> c = extensionClasses.get(n);
                if (c == null) {
                    extensionClasses.put(n, clazz);
                // 实现类重复报错
                } else if (c != clazz) {
                    throw new IllegalStateException(...);
                }
            }
        }
    }
}
```
要注意的几点：
1. `@Adaptive`注解是标明一个SPI实现类是属于自适应类，一个SPI接口只能有一个自适应实现，这从代码逻辑可以看出来，如果`!cachedAdaptiveClass.equals(clazz)`则报错。具体这个自适应类可以做什么，下面会说。
1. `Wrapper`类是AOP类
1. 这里还有一个`@Active`注解，标明实现类的激活条件，是一种条件机制。

### IOC
```java
    private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                for (Method method : instance.getClass().getMethods()) {
                    if (method.getName().startsWith("set")
                            && method.getParameterTypes().length == 1
                            && Modifier.isPublic(method.getModifiers())) {
                        /**
                         * Check {@link DisableInject} to see if we need auto injection for this property
                         */
                        if (method.getAnnotation(DisableInject.class) != null) {
                            continue;
                        }
                        Class<?> pt = method.getParameterTypes()[0];
                        if (ReflectUtils.isPrimitives(pt)) {
                            continue;
                        }
                        try {
                            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                            Object object = objectFactory.getExtension(pt, property);
                            if (object != null) {
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("fail to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```
dubbo的IOC目前只对setter方法支持，如果set的方法参数只有一个，那么就拿参数类型`pt`和setProperty的`property`去`objectFactory`找。`objectFactory`是`ExtensionFactory`接口，有两个实现，

上面提到`ExtensionLoader`构造函数
```java
    private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```
这里的`getAdaptiveExtension()`是什么？

### @Adaptive
`getAdaptiveExtension()`的实现如下，
```java
public T getAdaptiveExtension() {
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                // 这是前面spi load class缓存的
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        // 到这里说明没有自适应扩展类,需要动态创建
                        instance = ();
                        cachedAdaptiveInscreateAdaptiveExtensiontance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException(...);
                    }
                }
            }
        } else {
            throw new IllegalStateException(...);
        }
    }

    return (T) instance;
}
```
dubbo的Adaptive机制如下
1. 如果一个SPI接口实现类存在自适应实现，那么直接拿这个类；
1. 否则动态创建自适应类(手动拼接代码字符，并转为字节码，加载到jvm中)，由于dubbo是以url为协议的，所以创建的自适应代码是根据url中的内容决定使用那种实现。

> 当 Adaptive 注解在类上时，Dubbo 不会为该类生成代理类。注解在方法（接口方法）上时，Dubbo 则会为该方法生成代理逻辑。Adaptive 注解在类上的情况很少，在 Dubbo 中，仅有两个类被 Adaptive 注解了，分别是 AdaptiveCompiler 和 AdaptiveExtensionFactory。此种情况，表示拓展的加载逻辑由人工编码完成。更多时候，Adaptive 是注解在接口方法上的，表示拓展的加载逻辑需由框架自动生成。

下面是alibaba dubbo 的中文注释
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Adaptive {

    /**
     * 从{@link URL}的Key名，对应的Value作为要Adapt成的Extension名。
     * <p>
     * 如果{@link URL}这些Key都没有Value，使用 用 缺省的扩展（在接口的{@link SPI}中设定的值）。<br>
     * 比如，<code>String[] {"key1", "key2"}</code>，表示
     * <ol>
     * <li>先在URL上找key1的Value作为要Adapt成的Extension名；
     * <li>key1没有Value，则使用key2的Value作为要Adapt成的Extension名。
     * <li>key2没有Value，使用缺省的扩展。
     * <li>如果没有设定缺省扩展，则方法调用会抛出{@link IllegalStateException}。
     * </ol>
     * <p>
     * 如果不设置则缺省使用Extension接口类名的点分隔小写字串。<br>
     * 即对于Extension接口{@code com.alibaba.dubbo.xxx.YyyInvokerWrapper}的缺省值为<code>String[] {"yyy.invoker.wrapper"}</code>
     *
     * @see SPI#value()
     */
    String[] value() default {};

}
```
也就是说，如果@Adaptive上的key从url获取不到，就以@SPI上的默认扩展值作为key去url找，如果@SPI都没默认值，就将接口类名用`.`分隔，作为接口缺省值（这么奇怪的key吗 =。=）

以下面代码为例，
```java
@SPI("dubbo")
public interface Protocol {
    int getDefaultPort();

    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
    
    void destroy();
}
```
```java
Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```
这里会返回一个`Protocol$Adaptive`类，因为`Protocol`没有一个自适应实现类，所以dubbo动态生成了一个自适应类，通过debug得
```java
package org.apache.dubbo.rpc;

import org.apache.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        // 这里默认使用dubbo
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }

    public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg1;
        // 这里默认使用dubbo
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }
}
```
这里`Protocol$Adaptive`代码里，如果接口方法没有设置`@Adaptive`注解，以抛异常处理。

在生成的方法中，以`protocol`为key去url里查找这个key的值，如果url里没有设置，就以`Protocol`接口上的@SPI注解默认值——“dubbo”作为自适应类要找的扩展的name。

> 创建code代码太长，不贴。


拿`ExtensionFactory`来说，他有三个实现（有一个废弃的）

![upload successful](/images/pasted-173.png)

一个自适应实现类，以`@Adaptive`标注
```java
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    @Override
    public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }

}
```
一个用于获取dubbo Spi实现类
```java
public class SpiExtensionFactory implements ExtensionFactory {

    @Override
    public <T> T getExtension(Class<T> type, String name) {
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
            if (!loader.getSupportedExtensions().isEmpty()) {
                return loader.getAdaptiveExtension();
            }
        }
        return null;
    }

}
```
一个是用于获取Spring的bean，代码太长不贴
### @Active
@Active注解用于实现类上，表示一些激活条件。
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Activate {
    /**
     * Activate the current extension when one of the groups matches. The group passed into
     * {@link ExtensionLoader#getActivateExtension(URL, String, String)} will be used for matching.
     *
     * @return group names to match
     * @see ExtensionLoader#getActivateExtension(URL, String, String)
     */
    String[] group() default {};

    /**
     * Activate the current extension when the specified keys appear in the URL's parameters.
     * <p>
     * For example, given <code>@Activate("cache, validation")</code>, the current extension will be return only when
     * there's either <code>cache</code> or <code>validation</code> key appeared in the URL's parameters.
     * </p>
     *
     * @return URL parameter keys
     * @see ExtensionLoader#getActivateExtension(URL, String)
     * @see ExtensionLoader#getActivateExtension(URL, String, String)
     */
    String[] value() default {};

    /**
     * Relative ordering info, optional
     * Deprecated since 2.7.0
     *
     * @return extension list which should be put before the current one
     */
    @Deprecated
    String[] before() default {};

    /**
     * Relative ordering info, optional
     * Deprecated since 2.7.0
     *
     * @return extension list which should be put after the current one
     */
    @Deprecated
    String[] after() default {};

    /**
     * Absolute ordering info, optional
     *
     * @return absolute ordering info
     */
    int order() default 0;
}
```
同样由于dubbo是面向url的协议，所以这些激活条件需要通过url匹配。

在加载类的时候，有这么几行代码，以扩展名字为key，以扩展上的注解为值进行map缓存。
```java
// org.apache.dubbo.common.extension.ExtensionLoader#loadClass
Activate activate = clazz.getAnnotation(Activate.class);
if (activate != null) {
    cachedActivates.put(names[0], activate);
} else {
    // support com.alibaba.dubbo.common.extension.Activate
    com.alibaba.dubbo.common.extension.Activate oldActivate = clazz.getAnnotation(com.alibaba.dubbo.common.extension.Activate.class);
    if (oldActivate != null) {
        cachedActivates.put(names[0], oldActivate);
    }
}
```
获取激活的的扩展可以通过以下方法调用
```java
    public List<T> getActivateExtension(URL url, String key) {
        return getActivateExtension(url, key, null);
    }

    /**
     * This is equivalent to <pre>
     *     getActivateExtension(url, values, null);
     * </pre>
     *
     * @param url    url
     * @param values extension point names
     * @return extension list which are activated
     * @see #getActivateExtension(com.alibaba.dubbo.common.URL, String[], String)
     */
    public List<T> getActivateExtension(URL url, String[] values) {
        return getActivateExtension(url, values, null);
    }

    /**
     * This is equivalent to <pre>
     *     getActivateExtension(url, url.getParameter(key).split(","), null);
     * </pre>
     *
     * @param url   url
     * @param key   url parameter key which used to get extension point names
     * @param group group
     * @return extension list which are activated.
     * @see #getActivateExtension(com.alibaba.dubbo.common.URL, String[], String)
     */
    public List<T> getActivateExtension(URL url, String key, String group) {
        String value = url.getParameter(key);
        return getActivateExtension(url, value == null || value.length() == 0 ? null : Constants.COMMA_SPLIT_PATTERN.split(value), group);
    }
```
这些方法的具体实现如下
```java
    /**
     * Get activate extensions.
     *
     * @param url    url
     * @param values extension point names
     * @param group  group
     * @return extension list which are activated
     * @see com.alibaba.dubbo.common.extension.Activate
     */
    public List<T> getActivateExtension(URL url, String[] values, String group) {
        List<T> exts = new ArrayList<T>();
        List<String> names = values == null ? new ArrayList<String>(0) : Arrays.asList(values);
        if (!names.contains(Constants.REMOVE_VALUE_PREFIX + Constants.DEFAULT_KEY)) {
            getExtensionClasses();
            for (Map.Entry<String, Activate> entry : cachedActivates.entrySet()) {
                String name = entry.getKey();
                Activate activate = entry.getValue();
                if (isMatchGroup(group, activate.group())) {
                    T ext = getExtension(name);
                    if (!names.contains(name)
                            && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)
                            && isActive(activate, url)) {
                        exts.add(ext);
                    }
                }
            }
            // order 值越小越靠前
            Collections.sort(exts, ActivateComparator.COMPARATOR);
        }
        List<T> usrs = new ArrayList<T>();
        for (int i = 0; i < names.size(); i++) {
            String name = names.get(i);
            if (!name.startsWith(Constants.REMOVE_VALUE_PREFIX)
                    && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)) {
                if (Constants.DEFAULT_KEY.equals(name)) {
                    if (usrs.size() > 0) {
                        exts.addAll(0, usrs);
                        usrs.clear();
                    }
                } else {
                    T ext = getExtension(name);
                    usrs.add(ext);
                }
            }
        }
        if (usrs.size() > 0) {
            exts.addAll(usrs);
        }
        return exts;
    }
```
```java
    private boolean isActive(String[] keys, URL url) {
        if (keys.length == 0) {
            return true;
        }
        // 两次for循环
        for (String key : keys) {
            for (Map.Entry<String, String> entry : url.getParameters().entrySet()) {
                String k = entry.getKey();
                String v = entry.getValue();
                // 如果key存在且对应的value不为空
                if ((k.equals(key) || k.endsWith("." + key))
                        && ConfigUtils.isNotEmpty(v)) {
                    return true;
                }
            }
        }
        return false;
    }
```
匹配条件的逻辑是，先匹配组，再匹配键，键只要都存在值，那么即匹配成功。当注解上order值越小，这个实现类的排序越靠前。

关于排序，这里的一个具体例子是Dubbo服务的过滤器往往加上了@Active注解，这时候如果要设置过滤器的处理顺序，就可以通过该注解上的order属性设置。

# 结束语
还差Wrapper机制没看，wrapper就是一种aop机制，通过手工编码的方式完成aop。关于下面那段话我也没看到具体的例子。

> 如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK标准的ScriptEngine，通过getName();获取脚本类型的名称，但如果RubyScriptEngine因为所依赖的jruby.jar不存在，导致RubyScriptEngine类加载失败，这个失败原因被吃掉了，和ruby对应不起来，当用户执行ruby脚本时，会报不支持ruby，而不是真正失败的原因。

# 参考
1. [Apache Dubbo SPI](https://dubbo.apache.org/zh-cn/docs/source_code_guide/dubbo-spi.html)