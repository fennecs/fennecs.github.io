---
title: '[Java代理]Cglib代理'
tags:
  - AOP
categories:
  - Proxy
slug: 3922793788
date: 2019-03-10 14:03:00
---
# Cglib
> CGLIB是一个强大的高性能的代码生成包。CGLIB包的底层是通过使用一个小而快的字节码处理框架ASM，来转换字节码并生成新的类。除了CGLIB包，脚本语言例如Groovy和BeanShell，也是使用ASM来生成java的字节码。当然不鼓励直接使用ASM，因为它要求你必须对JVM内部结构包括class文件的格式和指令集都很熟悉。

# Demo
直接上Demo～
```
/**
 * 目标类
 */
public class RunnerDefault implements Runner {
    private Logger log = LoggerFactory.getLogger(getClass());

    @Override
    public void run(String name) {
        log.info("run: " + name);
    }
}
```
指定Callback的顺序
```
public class CglibCallbackFilter implements CallbackFilter {
    @Override
    public int accept(Method method) {
        if ("toString".equals(method.getName())) {
            return 1;
        }
        return 0;
    }
}
```
代理逻辑及生成代理的封装
```
// 拦截所有方法
public class CglibProxy implements MethodInterceptor {
    private Enhancer enhancer = new Enhancer();

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        objects[0] = "cglib " + objects[0];
        return methodProxy.invokeSuper(o, objects);
    }

    public Object newProxy(Class klass) {
        enhancer.setSuperclass(klass);
        enhancer.setCallbackFilter(new CglibCallbackFilter());
        enhancer.setCallbacks(new Callback[]{new CglibProxy(), new CglibStringProxy()});
        return enhancer.create();
    }
} 
```
```java
// 为了拦截toString方法
public class CglibStringProxy implements MethodInterceptor {
    private Logger logger = LoggerFactory.getLogger(this.getClass());
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        logger.info("toString hijacked");
        return null;
    }
}
```
测试类
```java
@Test
public void testCglib() {
    // 生成class文件
    System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "./");
    CglibProxy cglibProxy = new CglibProxy();
    RunnerDefault runner = (RunnerDefault) cglibProxy.newProxy(RunnerDefault.class);
    runner.run("proxy");
    runner.toString();
}
```
输出:

    2019-03-07 16:48:44.248  INFO   --- [           main] RunnerDefault$$EnhancerByCGLIB$$5b557d48 : run: cglib proxy
    2019-03-07 16:48:44.254  INFO   --- [           main] com.htc.learning.proxy.CglibStringProxy  : toString hijacked

> 也可以不配置CallbackFilter，只能配一个Callback，Enhancer会把单个的Callback转为数组,并且把CallbackFilter设置为ALL_ZERO，固定返回0

下面是上面test执行过程中生成的文件

![upload successful](/images/pasted-169.png)

# NamingPolicy
上面的**RunnerDefault$$EnhancerByCGLIB$$16487fc**是cglib命名而来的，默认实现类是`net.sf.cglib.core.DefaultNamingPolicy`命名规则如下：

    目标ClassName + "$$" + 使用cglib处理的ClassName + "ByCGLIB" + "$$" + key的hashcode
    
# Key和缓存

## KeyFactory
先看`KeyFactory`，这个类可以生成一个代理类，这个代理类对于给定的参数，每次调用返回的对象的`equals`、`hashcode`方法都是返回相同的值。由于cglib的配置项比较多，所以使用这个类用于生成缓存key的。

> 目标类需要提供一个`public Object newInstance(...)`的声明，参数数量类型随意。

下面是cglib
```java
public class KeySample {
    private interface MyFactory {
        public Object newInstance(int a, char[] b, String d);
    }
    public static void main(String[] args) {
        // 源码没有这一行，加上这一行，cglib的debug模式打开，就可以输出生成代理类的class文件了。
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "./");
        
        MyFactory f = (MyFactory)KeyFactory.create(MyFactory.class);
        Object key1 = f.newInstance(20, new char[]{ 'a', 'b' }, "hello");
        Object key2 = f.newInstance(20, new char[]{ 'a', 'b' }, "hello");
        Object key3 = f.newInstance(20, new char[]{ 'a', '_' }, "hello");
        System.out.println(key1.equals(key2));// true
        System.out.println(key2.equals(key3));// false
    }
}
```
![upload successful](/images/pasted-168.png)
三次生成的是不同对象。
key1和key2是相等的，key2和key3是不等的。追踪代码可以看到`KeyFactory`重写了代理类的`equals`、`hashcode`

```java
// net.sf.cglib.core.KeyFactory.Generator#generateClass
...
// hash code
e = ce.begin_method(Constants.ACC_PUBLIC, HASH_CODE, null);
int hc = (constant != 0) ? constant : PRIMES[(int)(Math.abs(seed) % PRIMES.length)];
int hm = (multiplier != 0) ? multiplier : PRIMES[(int)(Math.abs(seed * 13) % PRIMES.length)];
e.push(hc);
for (int i = 0; i < parameterTypes.length; i++) {
    e.load_this();
    e.getfield(getFieldName(i));
    EmitUtils.hash_code(e, parameterTypes[i], hm, customizers);
}
e.return_value();
e.end_method();

// equals
e = ce.begin_method(Constants.ACC_PUBLIC, EQUALS, null);
Label fail = e.make_label();
e.load_arg(0);
e.instance_of_this();
e.if_jump(e.EQ, fail);
for (int i = 0; i < parameterTypes.length; i++) {
    e.load_this();
    e.getfield(getFieldName(i));
    e.load_arg(0);
    e.checkcast_this();
    e.getfield(getFieldName(i));
    EmitUtils.not_equals(e, parameterTypes[i], fail, customizers);
}
e.push(1);
e.return_value();
e.mark(fail);
e.push(0);
e.return_value();
e.end_method();
...
```
## 代理缓存

所有cglib代理类的缓存都存在于`net.sf.cglib.core.AbstractClassGenerator`的`static`变量里，
```java
private static volatile Map<ClassLoader, ClassLoaderData> CACHE = new WeakHashMap<ClassLoader, ClassLoaderData>();

private static final boolean DEFAULT_USE_CACHE =        Boolean.parseBoolean(System.getProperty("cglib.useCache", "true"));
```
这个缓存是一个`WeakHashMap`，key和jdk代理一样，也是以`ClassLoader为`为key，至于`ClassLoaderData`是一个关于`interfaces`的封装，到最底层其实是一个`ConcurrentHashMap`。看`net.sf.cglib.core.AbstractClassGenerator#create`的代码
```java
    protected Object create(Object key) {
        try {
            ClassLoader loader = getClassLoader();
            Map<ClassLoader, ClassLoaderData> cache = CACHE;
            ClassLoaderData data = cache.get(loader);
            // 维护多线程
            if (data == null) {
                synchronized (AbstractClassGenerator.class) {
                    cache = CACHE;
                    data = cache.get(loader);
                    if (data == null) {
                        Map<ClassLoader, ClassLoaderData> newCache = new WeakHashMap<ClassLoader, ClassLoaderData>(cache);
                        data = new ClassLoaderData(loader);
                        newCache.put(loader, data);
                        CACHE = newCache;
                    }
                }
            }
            this.key = key;
            // 这里发生了二级缓存的put操作
            Object obj = data.get(this, getUseCache());
            if (obj instanceof Class) {
                return firstInstance((Class) obj);
            }
            return nextInstance(obj);
        } catch (...) {
            ...
        }
    }
```
看`net.sf.cglib.core.AbstractClassGenerator.ClassLoaderData#get`方法
```java
public Object get(AbstractClassGenerator gen, boolean useCache) {
    // 不使用缓存直接生成
    if (!useCache) {
      return gen.generate(ClassLoaderData.this);
    } else {
      // 底层是去ConcurrentHashMap拿
      Object cachedValue = generatedClasses.get(gen);
      return gen.unwrapCachedValue(cachedValue);
    }
}
```
前面说到二级缓存其实是`ConcurrentHashMap`,那么key，value分别是什么？key是`AbstractClassGenerator`子类决定的，比如`KeyFactory`使用的是目标类名；至于value又有jdk代理的味道——value值不是固定的，可能是生成的代理类，也可能是一个`FutureTask`，多个线程下多个`FutureTask`调用`get()`只会有一个在执行，避免了重复生成字节码。
```java
    protected V createEntry(final K key, KK cacheKey, Object v) {
        FutureTask<V> task;
        boolean creator = false;
        if (v != null) {
            // Another thread is already loading an instance
            task = (FutureTask<V>) v;
        } else {
            task = new FutureTask<V>(new Callable<V>() {
                public V call() throws Exception {
                    return loader.apply(key);
                }
            });
            Object prevTask = map.putIfAbsent(cacheKey, task);
            if (prevTask == null) {
                // creator does the load
                creator = true;
                task.run();
            } else if (prevTask instanceof FutureTask) {
                task = (FutureTask<V>) prevTask;
            } else {
                return (V) prevTask;
            }
        }

        V result;
        try {
            result = task.get();
        } catch (InterruptedException e) {
            throw new IllegalStateException("Interrupted while loading cache item", e);
        } catch (ExecutionException e) {
            Throwable cause = e.getCause();
            if (cause instanceof RuntimeException) {
                throw ((RuntimeException) cause);
            }
            throw new IllegalStateException("Unable to load cache item", cause);
        }
        if (creator) {
            map.put(cacheKey, result);
        }
        return result;
    }
```

# Enhancer
`Enhancer`是CGLib中的一个字节码增强器，一般我们都用这个来进行生成cglib代理类。

```java
private Class[] interfaces;
private CallbackFilter filter;
private Callback[] callbacks;
// 回调逻辑的类型，包括 MethodInterceptor|NoOp|LazyLoader|Dispatcher|InvocationHandler|FixedValue
private Type[] callbackTypes;
private boolean validateCallbackTypes;
// create()是否只生成代理类,而不是返回一个对象,如果只生成代理类，callback不能设置，会报错
private boolean classOnly;
private Class superclass;
private Class[] argumentTypes;
private Object[] arguments;
// 是否使用工厂类
private boolean useFactory = true;
private Long serialVersionUID;
private boolean interceptDuringConstruction = true;
```
下面是创建逻辑
```java
    private Object createHelper() {
        // 这里进行一些配置校验，比如设置了多个Callback但是没有设置filter
        preValidate();
        // 这里KEY_FACTORY是KeyFactory实例
        Object key = KEY_FACTORY.newInstance((superclass != null) ? superclass.getName() : null,
                ReflectUtils.getNames(interfaces),
                filter == ALL_ZERO ? null : new WeakCacheKey<CallbackFilter>(filter),
                callbackTypes,
                useFactory,
                interceptDuringConstruction,
                serialVersionUID);
        this.currentKey = key;
        Object result = super.create(key);
        return result;
    }
```
`Object result = super.create(key);`又是跳到上面提到过的`net.sf.cglib.core.AbstractClassGenerator#create`
```java
    protected Object create(Object key) {
        try {
            ClassLoader loader = getClassLoader();
            Map<ClassLoader, ClassLoaderData> cache = CACHE;
            ClassLoaderData data = cache.get(loader);
            if (data == null) {
                synchronized (AbstractClassGenerator.class) {
                    cache = CACHE;
                    data = cache.get(loader);
                    if (data == null) {
                        Map<ClassLoader, ClassLoaderData> newCache = new WeakHashMap<ClassLoader, ClassLoaderData>(cache);
                        data = new ClassLoaderData(loader);
                        newCache.put(loader, data);
                        CACHE = newCache;
                    }
                }
            }
            this.key = key;
            Object obj = data.get(this, getUseCache());
            if (obj instanceof Class) {
                return firstInstance((Class) obj);
            }
            return nextInstance(obj);
        } catch (...) {
            ...
        }
    }
```
如上一小节提到，这里面是有缓存的。

## 代理主流程

`net.sf.cglib.proxy.Enhancer#generateClass`方法负责生成代理，主要是通过`CallbackFilter`为不同的`Method`提供不同的`Callback`

```java
public void generateClass(ClassVisitor v) throws Exception {
    Class sc = (superclass == null) ? Object.class : superclass;

    if (TypeUtils.isFinal(sc.getModifiers()))
        throw new IllegalArgumentException("Cannot subclass final class " + sc.getName());
    List constructors = new ArrayList(Arrays.asList(sc.getDeclaredConstructors()));
    filterConstructors(sc, constructors);

    // Order is very important: must add superclass, then
    // its superclass chain, then each interface and
    // its superinterfaces.
    List actualMethods = new ArrayList();
    List interfaceMethods = new ArrayList();
    final Set forcePublic = new HashSet();
    
    // 看下面代码
    getMethods(sc, interfaces, actualMethods, interfaceMethods, forcePublic);
    // 这里把actualMethods中，非abstract,非native,非synchronized方法的修饰符全部变成final，将转化后的方法信息MethodInfo列表 记录在methods中
    List methods = CollectionUtils.transform(actualMethods, new Transformer() {
        public Object transform(Object value) {
            Method method = (Method)value;
            int modifiers = Constants.ACC_FINAL
                | (method.getModifiers()
                   & ~Constants.ACC_ABSTRACT
                   & ~Constants.ACC_NATIVE
                   & ~Constants.ACC_SYNCHRONIZED);
            if (forcePublic.contains(MethodWrapper.create(method))) {
                modifiers = (modifiers & ~Constants.ACC_PROTECTED) | Constants.ACC_PUBLIC;
            }
            return ReflectUtils.getMethodInfo(method, modifiers);
        }
    });

    ClassEmitter e = new ClassEmitter(v);
    // 这个currentData不知道是干嘛的
    if (currentData == null) {
    e.begin_class(Constants.V1_2,
                  Constants.ACC_PUBLIC,
                  getClassName(),
                  Type.getType(sc),
                  (useFactory ?
                   TypeUtils.add(TypeUtils.getTypes(interfaces), FACTORY) :
                   TypeUtils.getTypes(interfaces)),
                  Constants.SOURCE_FILE);
    } else {
        e.begin_class(Constants.V1_2,
                Constants.ACC_PUBLIC,
                getClassName(),
                null,
                new Type[]{FACTORY},
                Constants.SOURCE_FILE);
    }
    List constructorInfo = CollectionUtils.transform(constructors, MethodInfoTransformer.getInstance());

    e.declare_field(Constants.ACC_PRIVATE, BOUND_FIELD, Type.BOOLEAN_TYPE, null);
    e.declare_field(Constants.ACC_PUBLIC | Constants.ACC_STATIC, FACTORY_DATA_FIELD, OBJECT_TYPE, null);
    if (!interceptDuringConstruction) {
        e.declare_field(Constants.ACC_PRIVATE, CONSTRUCTED_FIELD, Type.BOOLEAN_TYPE, null);
    }
    e.declare_field(Constants.PRIVATE_FINAL_STATIC, THREAD_CALLBACKS_FIELD, THREAD_LOCAL, null);
    e.declare_field(Constants.PRIVATE_FINAL_STATIC, STATIC_CALLBACKS_FIELD, CALLBACK_ARRAY, null);
    if (serialVersionUID != null) {
        e.declare_field(Constants.PRIVATE_FINAL_STATIC, Constants.SUID_FIELD_NAME, Type.LONG_TYPE, serialVersionUID);
    }

    // 根据callbackTypes增加属性，名字为CGLIB$CALLBACK_xx(xx是序号)
    for (int i = 0; i < callbackTypes.length; i++) {
        e.declare_field(Constants.ACC_PRIVATE, getCallbackField(i), callbackTypes[i], null);
    }
    // This is declared private to avoid "public field" pollution
    e.declare_field(Constants.ACC_PRIVATE | Constants.ACC_STATIC, CALLBACK_FILTER_FIELD, OBJECT_TYPE, null);

    if (currentData == null) {
        // 为目标方法配置Callback
        emitMethods(e, methods, actualMethods);
        emitConstructors(e, constructorInfo);
    } else {
        emitDefaultConstructor(e);
    }
    emitSetThreadCallbacks(e);
    emitSetStaticCallbacks(e);
    emitBindCallbacks(e);

    if (useFactory || currentData != null) {
        int[] keys = getCallbackKeys();
        emitNewInstanceCallbacks(e);
        emitNewInstanceCallback(e);
        emitNewInstanceMultiarg(e, constructorInfo);
        emitGetCallback(e, keys);
        emitSetCallback(e, keys);
        emitGetCallbacks(e);
        emitSetCallbacks(e);
    }

    e.end_class();
}
```
## 获取要代理的方法
```java
private static void getMethods(Class superclass, Class[] interfaces, List methods, List interfaceMethods, Set forcePublic)
{
    // 下面这一坨是把目标类的方法，接口方法的信息（类型是 MethodInfo）都加入到methods列表里
    ReflectUtils.addAllMethods(superclass, methods);
    List target = (interfaceMethods != null) ? interfaceMethods : methods;
    if (interfaces != null) {
        for (int i = 0; i < interfaces.length; i++) {
            if (interfaces[i] != Factory.class) {
                ReflectUtils.addAllMethods(interfaces[i], target);
            }
        }
    }
    if (interfaceMethods != null) {
        if (forcePublic != null) {
            forcePublic.addAll(MethodWrapper.createSet(interfaceMethods));
        }
        methods.addAll(interfaceMethods);
    }
    // 过滤static方法
    CollectionUtils.filter(methods, new RejectModifierPredicate(Constants.ACC_STATIC));
    // 根据布尔值决定是否过滤protected的方法，过滤private方法
    CollectionUtils.filter(methods, new VisibilityPredicate(superclass, true));
    // 过滤重复
    CollectionUtils.filter(methods, new DuplicatesPredicate(methods));
    // 过滤final方法
    CollectionUtils.filter(methods, new RejectModifierPredicate(Constants.ACC_FINAL));
}
```
针对demo的`RunnerDefault`，获取到的最终方法为

![upload successful](/images/pasted-170.png)

## 为目标方法配置Callback
```java
    // methods的类型是MethodInfo，主要是改了原方法的modifiers
    // actualMethods的类型Method
    private void emitMethods(final ClassEmitter ce, List methods, List actualMethods) {
        CallbackGenerator[] generators = CallbackInfo.getGenerators(callbackTypes);

        Map groups = new HashMap();
        final Map indexes = new HashMap();
        final Map originalModifiers = new HashMap();
        final Map positions = CollectionUtils.getIndexMap(methods);
        final Map declToBridge = new HashMap();

        Iterator it1 = methods.iterator();
        Iterator it2 = (actualMethods != null) ? actualMethods.iterator() : null;

        while (it1.hasNext()) {
            MethodInfo method = (MethodInfo)it1.next();
            Method actualMethod = (it2 != null) ? (Method)it2.next() : null;
            // 获取 index
            int index = filter.accept(actualMethod);
            if (index >= callbackTypes.length) {
                throw new IllegalArgumentException("Callback filter returned an index that is too large: " + index);
            }
            originalModifiers.put(method, new Integer((actualMethod != null) ? actualMethod.getModifiers() : method.getModifiers()));
            // 把index放入map
            indexes.put(method, new Integer(index));
            List group = (List)groups.get(generators[index]);
            if (group == null) {
                groups.put(generators[index], group = new ArrayList(methods.size()));
            }
            group.add(method);
            
            // Optimization: build up a map of Class -> bridge methods in class
            // so that we can look up all the bridge methods in one pass for a class.
            if (TypeUtils.isBridge(actualMethod.getModifiers())) {
            	Set bridges = (Set)declToBridge.get(actualMethod.getDeclaringClass());
            	if (bridges == null) {
            	    bridges = new HashSet();
            	    declToBridge.put(actualMethod.getDeclaringClass(), bridges);
            	}
            	bridges.add(method.getSignature());            	
            }
        }
        
        final Map bridgeToTarget = new BridgeMethodResolver(declToBridge, getClassLoader()).resolveAll();

        Set seenGen = new HashSet();
        CodeEmitter se = ce.getStaticHook();
        se.new_instance(THREAD_LOCAL);
        se.dup();
        se.invoke_constructor(THREAD_LOCAL, CSTRUCT_NULL);
        se.putfield(THREAD_CALLBACKS_FIELD);

        final Object[] state = new Object[1];
        CallbackGenerator.Context context = new CallbackGenerator.Context() {
            public ClassLoader getClassLoader() {
                return Enhancer.this.getClassLoader();
            }
            public int getOriginalModifiers(MethodInfo method) {
                return ((Integer)originalModifiers.get(method)).intValue();
            }
            public int getIndex(MethodInfo method) {
                return ((Integer)indexes.get(method)).intValue();
            }
            // 根据index获取对应的Callback（从DeclaredField获取）
            public void emitCallback(CodeEmitter e, int index) {
                emitCurrentCallback(e, index);
            }
            public Signature getImplSignature(MethodInfo method) {
                return rename(method.getSignature(), ((Integer)positions.get(method)).intValue());
            }
            public void emitLoadArgsAndInvoke(CodeEmitter e, MethodInfo method) {
                // If this is a bridge and we know the target was called from invokespecial,
                // then we need to invoke_virtual w/ the bridge target instead of doing
                // a super, because super may itself be using super, which would bypass
                // any proxies on the target.
                Signature bridgeTarget = (Signature)bridgeToTarget.get(method.getSignature());
                if (bridgeTarget != null) {
                    // checkcast each argument against the target's argument types
                    for (int i = 0; i < bridgeTarget.getArgumentTypes().length; i++) {
                        e.load_arg(i);
                        Type target = bridgeTarget.getArgumentTypes()[i];
                        if (!target.equals(method.getSignature().getArgumentTypes()[i])) {
                            e.checkcast(target);
                        }
                    }

                    e.invoke_virtual_this(bridgeTarget);
                    
                    Type retType = method.getSignature().getReturnType();                    
                    // Not necessary to cast if the target & bridge have
                    // the same return type. 
                    // (This conveniently includes void and primitive types,
                    // which would fail if casted.  It's not possible to 
                    // covariant from boxed to unbox (or vice versa), so no having
                    // to box/unbox for bridges).
                    // TODO: It also isn't necessary to checkcast if the return is
                    // assignable from the target.  (This would happen if a subclass
                    // used covariant returns to narrow the return type within a bridge
                    // method.)
                    if (!retType.equals(bridgeTarget.getReturnType())) {
                        e.checkcast(retType);
                    }
                } else {
                    e.load_args();
                    e.super_invoke(method.getSignature());
                }
            }
            public CodeEmitter beginMethod(ClassEmitter ce, MethodInfo method) {
                CodeEmitter e = EmitUtils.begin_method(ce, method);
                if (!interceptDuringConstruction &&
                    !TypeUtils.isAbstract(method.getModifiers())) {
                    Label constructed = e.make_label();
                    e.load_this();
                    e.getfield(CONSTRUCTED_FIELD);
                    e.if_jump(e.NE, constructed);
                    e.load_this();
                    e.load_args();
                    e.super_invoke();
                    e.return_value();
                    e.mark(constructed);
                }
                return e;
            }
        };
        for (int i = 0; i < callbackTypes.length; i++) {
            CallbackGenerator gen = generators[i];
            if (!seenGen.contains(gen)) {
                seenGen.add(gen);
                final List fmethods = (List)groups.get(gen);
                if (fmethods != null) {
                    try {
                        gen.generate(ce, context, fmethods);
                        gen.generateStatic(se, context, fmethods);
                    } catch (RuntimeException x) {
                        throw x;
                    } catch (Exception x) {
                        throw new CodeGenerationException(x);
                    }
                }
            }
        }
        se.return_value();
        se.end_method();
    }
```
这里代码没有看的很细，了解了大概。

> 一个方法只会有一个`Callback`。

这一节，最后看一下代理出来的类的代码
```java
    public final void run(String var1) {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            var10000.intercept(this, CGLIB$run$0$Method, new Object[]{var1}, CGLIB$run$0$Proxy);
        } else {
            super.run(var1);
        }
    }
```
其中
```java
// CGLIB$run$0$Method 
Method CGLIB$run$0$Method = ReflectUtils.findMethods(new String[]{"run", "(Ljava/lang/String;)V"}, (var1 = Class.forName("com.htc.learning.api.impl.RunnerDefault")).getDeclaredMethods())[0];

// CGLIB$run$0$Proxy
MethodProxy CGLIB$run$0$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/String;)V", "run", "CGLIB$run$0");
```
所以逻辑是这样的，代理类将调用转发给一个`Callback`，在`Callback`里，如果要执行目标类的目标方法，即调用`net.sf.cglib.proxy.MethodProxy#invokeSuper`

等等，`MethodProxy`是什么

此外从生成的类里，我们可以看除了Enhancer和KeyFactory的增强类之外，还生成了三个类

![upload successful](/images/pasted-171.png)
第二个是我们的代理类，那么其余两个是干嘛的，`FastClass`又是什么?

# MethodProxy与Fastclass
## 为什么要有`MethodProxy`、`Fastclass`
我们配置代理的时候，并没有传入一个目标类实例，而是传入目标类的class，这时我们要去调用目标方法的时候，如果每次都靠反射，那就没有直接调用一个对象来的快。

> 没错每次都靠反射说的就是你jdk代理。

## MethodProxy
`MethodProxy`表明了一个方法到另一个方法的映射，我们看一下代理类run方法的代码
```java
    public final void run(String var1) {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            // CGLIB$run$0$Proxy 即是 MethodProxy
            var10000.intercept(this, CGLIB$run$0$Method, new Object[]{var1}, CGLIB$run$0$Proxy);
        } else {
            super.run(var1);
        }
    }
```
其中,
```java
Class var0 = Class.forName("com.htc.learning.api.impl.RunnerDefault$$EnhancerByCGLIB$$5b557d48");  
Class var1 = Class.forName("com.htc.learning.api.impl.RunnerDefault")
CGLIB$run$0$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/String;)V", "run", "CGLIB$run$0");
```
再其中，`"CGLIB$run$0"`是Enhancer代理类里的一个已经生成的方法，
```java
final void CGLIB$run$0(String var1) {
    super.run(var1);
}
```
`MethodProxy`的`create`方法，
```java
public static MethodProxy create(Class c1, Class c2, String desc, String name1, String name2) {
    MethodProxy proxy = new MethodProxy();
    proxy.sig1 = new Signature(name1, desc);
    proxy.sig2 = new Signature(name2, desc);
    proxy.createInfo = new MethodProxy.CreateInfo(c1, c2);
    return proxy;
}
```
这一段就是说`c1`的方法`name1`,对应的代理方法是实现类`c2`的方法`name2`。再具体一点，`RunnerDefault`的`run`方法，对应的就是`com.htc.learning.api.impl.RunnerDefault$$EnhancerByCGLIB$$5b557d48`的`CGLIB$run$0`方法，这两个签名没有依赖，MethodProxy利用这两个签名，提供两种不同的目标方法调用，

## FastClass
上面只是创建了一个关联关系，接下来看`net.sf.cglib.proxy.MethodProxy#invokeSuper`

```java
public Object invokeSuper(Object obj, Object[] args) throws Throwable {
    try {
        this.init();
        MethodProxy.FastClassInfo fci = this.fastClassInfo;
        // f2是Enhancer代理类，i2是配置好的可以调用到目标方法的索引，invoke根据索引，使用switch块直接调用方法，而不是利用反射
        return fci.f2.invoke(fci.i2, obj, args);
    } catch (InvocationTargetException var4) {
        throw var4.getTargetException();
    }
}

private void init() {
    // 单例模式，防止重复创建
    if (this.fastClassInfo == null) {
        synchronized(this.initLock) {
            if (this.fastClassInfo == null) {
                MethodProxy.CreateInfo ci = this.createInfo;
                MethodProxy.FastClassInfo fci = new MethodProxy.FastClassInfo();
                fci.f1 = helper(ci, ci.c1);
                fci.f2 = helper(ci, ci.c2);
                fci.i1 = fci.f1.getIndex(this.sig1);
                fci.i2 = fci.f2.getIndex(this.sig2);
                this.fastClassInfo = fci;
                this.createInfo = null;
            }
        }
    }

}

private static FastClass helper(MethodProxy.CreateInfo ci, Class type) {
    Generator g = new Generator();
    g.setType(type);
    g.setClassLoader(ci.c2.getClassLoader());
    g.setNamingPolicy(ci.namingPolicy);
    g.setStrategy(ci.strategy);
    g.setAttemptLoad(ci.attemptLoad);
    // 进去代码后可以看到使用了缓存，所以不会重复生成FastClass
    return g.create();
}
```
```java
private static class FastClassInfo {
    FastClass f1;
    FastClass f2;
    int i1;
    int i2;

    private FastClassInfo() {
    }
}
```
这个参数命令实在有点难懂，梳理一下，针对`run`方法，

|参数|说明|
|---|---|
|c1 |`RunnerDefault`|
|f1|`RunnerDefault$$FastClassByCGLIB$$a60a67a3`|
|i1|目标类`run`的索引|
|c2|`RunnerDefault$$EnhancerByCGLIB$$5b557d48` (`Enhancer`代理类)|
|f2|`RunnerDefault$$EnhancerByCGLIB$$5b557d48$$FastClassByCGLIB$$9f176e41`（`Enhancer`代理类的一个快速查找类）|
|i2|`Enhancer`代理类调用`run`的索引|

下面是`RunnerDefault$$FastClassByCGLIB$$a60a67a3`的部分代码(如果是`RunnerDefault$$EnhancerByCGLIB$$5b557d48$$FastClassByCGLIB$$9f176e41`switch块会更大，因为`RunnerDefault$$EnhancerByCGLIB$$5b557d48`的方法更多)
```java
    public int getIndex(Signature var1) {
        String var10000 = var1.toString();
        switch(var10000.hashCode()) {
        case -1717138348:
            if (var10000.equals("run(Ljava/lang/String;)V")) {
                return 0;
            }
            break;
        case 1826985398:
            if (var10000.equals("equals(Ljava/lang/Object;)Z")) {
                return 1;
            }
            break;
        case 1913648695:
            if (var10000.equals("toString()Ljava/lang/String;")) {
                return 2;
            }
            break;
        case 1984935277:
            if (var10000.equals("hashCode()I")) {
                return 3;
            }
        }

        return -1;
    }
```
```java
    // 注意这的var2是Enhancer代理类
    public Object invoke(int var1, Object var2, Object[] var3) throws InvocationTargetException {
        RunnerDefault var10000 = (RunnerDefault)var2;
        int var10001 = var1;

        try {
            switch(var10001) {
            case 0:
                var10000.run((String)var3[0]);
                return null;
            case 1:
                return new Boolean(var10000.equals(var3[0]));
            case 2:
                return var10000.toString();
            case 3:
                return new Integer(var10000.hashCode());
            }
        } catch (Throwable var4) {
            throw new InvocationTargetException(var4);
        }

        throw new IllegalArgumentException("Cannot find matching method/constructor");
    }
```
## StackOverflowError
如果我们写Callback的时候，把`invokeSuper`写成`invoke`会怎么样，答案是：**栈溢出**。
`MethodProxy`的`invoke`方法是这样的，
```java
    public Object invoke(Object obj, Object[] args) throws Throwable {
        try {
            this.init();
            MethodProxy.FastClassInfo fci = this.fastClassInfo;
            return fci.f1.invoke(fci.i1, obj, args);
        } catch (...) {
            ...
        }
    }
```
这里的`obj`是Enhancer代理类，而`f1`是`RunnerDefault$$FastClassByCGLIB$$a60a67a3
`，所以又会索引到`Enhancer代理类`的代理`run`方法，接着又执行上面的`invoke`,balabala...陷入死循环。

## invoke与invokeSuper
那是不是`invoke`不能被调用了？不是，上面说到**MethodProxy利用这两个签名，提供两种不同的目标方法调用，**所以，`invoke`是另一种调用目标方法的姿势。

写`Callback`的时候，
```java
@Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        objects[0] = "cglib " + objects[0];
        return methodProxy.invokeSuper(o, objects);
    }
```
传入的`Object o`是`Enhancer`代理类，而我们不能执行`methodProxy.invoke(o, objects)`陷入死循环，所以我们在`Callback`需要保存一个目标类实例的引用`target`，然后`methodProxy.invoke(target, objects)`。

## 总结
`MethodProxy`与`Fastclass`提供了一个  **Signature -> index -> invoke**的机制。

# 缺陷
如果理解了`FastClass`，那么很容猜测`cglib`的性能瓶颈在于，当目标类的方法很多的时候，`switch`块就是一个很慢的查找，这个查找是有优化空间的。此外，`cglib`代理的创建时间会比`jdk`代理的创建更耗时间，不过我觉得这都不是事。

# 参考
1. [cglib demo以及Enhancer源码解析](https://www.jianshu.com/p/20203286ccd9)