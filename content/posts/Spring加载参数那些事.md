---
title: '[Spring]Spring加载参数那些事'
author: 土川
tags:
  - properties
categories:
  - Spring
slug: 298553133
date: 2018-03-10 15:06:00
---
# 通过&lt;context:property-placeholder location="classpath:conn.properties" &gt;
```xml
 <context:property-placeholder location="classpath:conn.properties"/>
<bean id="dataSource" class="${dataSource}"> <!-- 这些配置Spring在启动时会去conn.properties中找 -->  
    <property name="driverClass" value="${driverClass}" />  
    <property name="jdbcUrl" value="${jdbcUrl}" />  
    <property name="user" value="${user}" />  
    <property name="password" value="${password}" />  
</bean>  
```
# 通过@Value注解
@Value注解有两种Configurer可以使用，一种是PropertiesFactoryBean，另一种是PropertyPlaceholderConfigurer，后者注重于模板${}
```xml
<!-- 第二种方式是使用注解的方式注入，主要用在java代码中使用注解注入properties文件中相应的value值 -->  
<bean id="prop" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">  
<!-- 或者使用<bean id="prop" class="org.springframework.beans.factory.config.PropertiesFactoryBean">  -->
    <property name="locations"><!-- 这里是PropertiesFactoryBean类，它也有个locations属性，也是接收一个数组，跟上面一样  
        <array>  
            <value>classpath:public.properties</value>  
        </array>  
    </property>  
</bean>  
```
然后在bean里使用
@Value("#{beanId[property_name]}")(PropertiesFactoryBean)
或者
@Value("${xxxxx}")(PropertyPlaceholderConfigurer)即可，如public.properties里有

    username=htc  
那么在bean里使用@Value("${username}") String username 注入在方法参数

    注意要有set方法才能被注入进来，注解写在set方法上即可。在setFilePath方法中通过控制台打印filePath是为了在启动tomcat的时候，观察控制台有没有输出来，如果有，说明Spring在启动时，已经将filePath给加载好了，我们看一下控制台的启动信息：



# 重要：第三种
@ConfigurationProperties
```java
    @Bean(value = "dataSource", destroyMethod = "close", initMethod = "init")
    @Primary
    @ConfigurationProperties(prefix = "druid")
    public DataSource getDataSource(@Value("${jdbcUrl}") String jdbcUrl,@Value("${user}") String user,@Value("${password}") String password){
        return DataSourceBuilder.create().username(user).password(password).type(DruidDataSource.class).url(jdbcUrl).build();
    }
```
这里使用了@ConfigurationProperties在使得在处理bean的时候可以自动根据fieldname注入值
然而使用上面的配置是不会注入的，因为读不到！上面的配置是用PropertyPlaceholderConfigurer这个类来存储配置文件，而@ConfigurationProperties是使用这个类

    org.springframework.context.support.PropertySourcesPlaceholderConfigurer
所以要配置的话得这么配置
```xml
<bean id="resourcePropertyConfigurer" class="org.springframework.context.support.PropertySourcesPlaceholderConfigurer">
        <property name="locations">
            <list>
                <value>classpath*:druid.properties</value>
            </list>
        </property>
</bean>
```
> 实际上`PropertySourcesPlaceholderConfigurer`是springframework后来出的，相当于`PropertyPlaceholderConfigurer`的升级版


druid.properties里的内容类似于

    druid.initialSize=10
    druid.minIdle=1
    ...
如上，spring还可以自动根据前缀装配，根据属性名模糊匹配。

spring使用了模糊匹配，对于一个属性可以生成几十个变种，如有个属性叫"serverPort"，可以有“serverPort“、“server-port“、“serverport“等很多变种，是一种松懈的匹配，然后对于生成的几十种变种遍历，逐一和properties匹配，匹配到第一个就返回

以上就是Spring加载properties配置文件的几种方式。实际上，上面基于xml方式中的PropertyPlaceholderConfigurer类和这里基于注解方式的PropertiesFactoryBean类都是继承PropertiesLoaderSupport，都是用来加载properties配置文件的。

> 在spring配置文件中，对于bean的配置有这样一个配置：
&lt;property name="ignoreUnresolvablePlaceholders" value="true" /&gt; 
这个主要是为了解决抛出cannot be resolved的异常。