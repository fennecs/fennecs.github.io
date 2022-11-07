---
title: '[Spring]Spring之循环依赖'
author: 土川
tags:
  - Bean
  - IOC
  - ''
categories:
  - Spring
slug: 3244702836
date: 2018-03-20 01:57:00
---
> 面试面过这个问题，整理一下。所谓循环依赖就是多个Bean之间依赖关系形成一个闭环，例如A->B->C->...->A 这种情况，当然，最简单的循环依赖就是2个Bean之间互相依赖：A->B（A依赖B), B->A(B依赖A) 。在Spring中，如果A->B,那么在创建A的过程中会去创建B,在创建B（或B的依赖)的过程中又发现B->A，这个时候就出现了循环依赖的现象。

<!--more-->

# 不是循环调用

循环依赖就是循环引用，就是两个或多个Bean相互之间的持有对方，比如CircleA引用CircleB，CircleB引用CircleC，CircleC引用CircleA，则它们最终反映为一个环。此处不是循环调用，循环调用是方法之间的环调用，如下图：

![upload successful](/images/pasted-54.png)
而循环调用是无法解决的，除非有终结条件，否则就是死循环，最终导致内存溢出错误。

Spring容器循环依赖包括构造器循环依赖和setter循环依赖，那Spring容器如何检测和解决循环依赖呢？

# Spring可以解决的循环依赖
spring中的循环依赖只有当

* Bean是单例
* 通过属性注入的情况

这两个条件满足的情况下是没问题的。但是如果是通过构造器依赖，或者不是单例模式的情况下循环依赖就会抛出异常BeanCurrentlyInCreationException。下面从代码层面上解析一下为什么。

> 至于为什么要有`prototype`类型的bean，我想典型的应用场景就是struts的Action（有点像springmvc的Controller）实例。struts的request参数是绑定在Action对象的成员变量上的，如果Action的bean是单例的就会造成线程不安全。

# Prototype循环依赖抛异常

```java
	protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			.// 日志代码
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
        	// 请看这里，抛异常
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						getBean(dep);
					}
				}

				// Create bean instance.
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							try {
								return createBean(beanName, mbd, args);
							}
							catch (BeansException ex) {
								// Explicitly remove instance from singleton cache: It might have been put there
								// eagerly by the creation process, to allow for circular reference resolution.
								// Also remove any beans that received a temporary reference to the bean.
								destroySingleton(beanName);
								throw ex;
							}
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
							@Override
							public Object getObject() throws BeansException {
								beforePrototypeCreation(beanName);
								try {
									return createBean(beanName, mbd, args);
								}
								finally {
									afterPrototypeCreation(beanName);
								}
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		.// type校验
		return (T) bean;
	}
```
可以看出，该流程中就考虑了Prototype的循环依赖的问题，只要在创建Prototype的Bean中出现循环依赖那么就抛出异常。但是在singleton的情况下，则通过另外的方式来解决。

# Singleton的循环依赖之构造注入
上面的代码有这么一段Singleton的处理

```java
// Create bean instance.
if (mbd.isSingleton()) {
    sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
        @Override
        public Object getObject() throws BeansException {
            try {
                return createBean(beanName, mbd, args);
            }
            catch (BeansException ex) {
                // Explicitly remove instance from singleton cache: It might have been put there
                // eagerly by the creation process, to allow for circular reference resolution.
                // Also remove any beans that received a temporary reference to the bean.
                destroySingleton(beanName);
                throw ex;
            }
        }
    });
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```
这个getSingleton涉及到了ObjectFactory这个接口类，这个接口的功能和FactoryBean类似，但是主要是用来解决循环依赖的。在初始化过程决定返回的Singleton对象。关于单例的对象的创建，又要介绍一下DefaultSingletonBeanRegistry这个类，这个类主要用来帮助创建单例模式，其中主要的属性：
```java
/** 缓存创建的单例对象: bean名字 --> bean对象 */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

/** 缓存单例的factory,就是ObjectFactory这个东西，: bean name --> ObjectFactory */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);

/** 也是缓存创建的单例对象，功能和singletonObjects不一样，
在bean构造成功之后，属性初始化之前会把对象放入到这里，
主要是用于解决属性注入的循环引用: bean name --> bean instance
*/
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);

/** 记录在创建单例对象中循环依赖的问题，还记得Prototype中又记录创建过程中依赖的map吗？
在Prototype中只要出现了循环依赖就抛出异常，而在单例中会尝试解决 */
private final Set<String> singletonsCurrentlyInCreation =
		Collections.newSetFromMap(new ConcurrentHashMap<String, Boolean>(16));
```
现在看`getSingleton(beanName, new ObjectFactory<Object>()`的实现

```java
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "'beanName' must not be null");
		synchronized (this.singletonObjects) {
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
                //日志代码
				...

                //把当前beanName加入到singletonsCurrentlyInCreation中
				beforeSingletonCreation(beanName);
				boolean newSingleton = false;
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<Exception>();
				}
				try {
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				} catch(...){
                	...
                }

                //从singletonsCurrentlyInCreation中删除beanName
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					afterSingletonCreation(beanName);
				}
				if (newSingleton) {
					addSingleton(beanName, singletonObject);
				}
			}
			return (singletonObject != NULL_OBJECT ? singletonObject : null);
		}
	}
```
这段逻辑是不是和Prototype中解决循环类似,这里其实就是调用了ObjectFactory的getObject()获取对象，回过头去看前面代码，ObjectFactory的getObject()方法实际调用的是createBean(beanName, mbd, args)。说到createBean(beanName, mbd, args)又不得不说AbstractAutowireCapableBeanFactory这个类，主要功能就是完成依赖注入的Bean的创建，这个类的createBean方法代码如下,注意注解说明：
```java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
    ...
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    ...
}

protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
            throws BeanCreationException {

    // 实例化bean
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        //如果没实例化则创建新的BeanWrapper
        //如果是通过构造器注入，这里是一个关键点
        /*
        因为在A初始化的时候发现构造函数依赖B，就会去实例化B，
        然后B也会运行到这段逻辑，构造函数中发现依赖A，
        这个时候就会抛出循环依赖的异常
        */
                instanceWrapper = createBeanInstance(beanName, mbd, args);
    }


   //如果当前是单例，并且allowCircularReferences为true(默认就是true，除非我们不希望Spring帮我们解决)
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        /*
        ！！！这里很重要，把构造成功，但属性还没注入的
        的bean加到singletonFactory中，这样再解决A的依赖
        过程中如果依赖A，就把这个半成品返回回去。
        */
        addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                return getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }


    Object exposedObject = bean;
    try {
        //自动注入属性
        populateBean(beanName, mbd, instanceWrapper);
        if (exposedObject != null) {
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
    }
    ...

    return exposedObject;
}
```
# 总结

* AbstractBeanFactory,这个类中包含了Bean创建的主要流程，在doGetBean这个方法中包含了对Prototype循环依赖处理。逻辑很简单，出现了循环依赖则直接抛出异常
* DefaultSingletonBeanRegister 用于管理Singleton的对象的创建，以及解决循环依赖的问题,其中解决循环依赖的关键属性就是了earlySingletonObjects，他会在构造Singleton对象过程中暂时缓存构造成功，但属性还未注入的对象，这样就可以解决循环依赖的问题。
* AbstractAutowireCapableBeanFactory,自动注入的相关逻辑，包自动注入的对象的创建、初始化和注入。但如果在调用构造函数中发现了循环依赖，则抛出异常
* ObjectFactory,这个接口功能和FactoryBean类似，但是为了解决循环依赖，他决定了在获取的getSingleton()是一个完成品还是一个半成品。

# 构造函数和属性注入依赖的循环
看下面的场景
```
@Component
public class BeanA {
    private BeanB beanB;

    @Autowired
    public BeanA(BeanB beanB) {
        this.beanB = beanB;
    }
}

@Component
public class BeanB {
    @Autowired
    private BeanA beanA;
}
```
这种情况会不会报依赖异常？写个demo，报下面的错误信息。

	  ***************************
      APPLICATION FAILED TO START
      ***************************

      Description:

      The dependencies of some of the beans in the application context form a cycle:

      ┌─────┐
      |  beanA defined in file [C:\Users\zack.huang\IdeaProjects\xunhuanbean\target\classes\com\htc\testbean\xunhuanbean\BeanA.class]
      ↑     ↓
      |  beanB (field private com.htc.testbean.xunhuanbean.BeanA com.htc.testbean.xunhuanbean.BeanB.beanC)
      └─────┘

按照类名的字典排序，BeanA是会比BeanB先被扫描到的，那么先构造BeanA，BeanA明显构造需要BeanB依赖，初始化BeanB，BeanA没构造完成无法注入，于是GG。

如果BeanA改名为BeanC，就可以解决问题，不过这种谁先谁后是不可靠的，勿写这种代码。