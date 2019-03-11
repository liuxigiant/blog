---
layout: post
title: "Spring生命周期回调与容器扩展"
description: Spring生命周期回调与容器扩展
date: 2016-12-28 16:17:18 +0800
catalog: true
categories:
- Spring Beans
tags:
- Spring Beans
---

本篇主要总结下Spring容器在初始化实例前后，提供的一些回调方法和可扩展点。利用这些方法和扩展点，可以实现在Spring初始化实例前后做一些特殊逻辑处理。  

下面主要介绍：  

- 类级别的生命周期初始化回调方法`init-method`配置、`InitializingBean`接口和`PostConstruct`注解  
- 容器级别的扩展`BeanPostProcessor`接口和`BeanFactoryPostProcessor`接口  

# 1. 类级别生命周期回调  

## 1.1 init-method  

> **参照：[Spring bean xsd init-method](http://www.springframework.org/schema/beans/spring-beans.xsd)**  

`init-method`是在Spring配置文件中声明bean的时候的一个配置项。`init-method`配置项的值为类中的一个**无参方法**，但可抛出异常。该方法会在Spring容器**实例化对象并设置完属性值之后**被调用。  

> `init-method`能实现的功能与`InitializingBean`接口、`PostConstruct`注解一致  

Spring配置文件及测试类如下：  

```xml
<bean id = "initMethodBeanService" class="name.liuxi.spring.ext.InitMethodBeanService" init-method="init">
      <property name="f2" value="2"/>
</bean>
```

测试类如下：  
```java
public class InitMethodBeanService {
    private static Integer f1;
    private Integer f2;

    static {
        f1 = 1;
        System.out.println("InitMethodBeanService static block execute...");
    }

    public InitMethodBeanService(){
        System.out.println("InitMethodBeanService construct method execute...");
    }

    public void init(){
        System.out.println("InitMethodBeanService init method execute...");
    }

    public Integer getF2() {
        return f2;
    }

    public void setF2(Integer f2) {
        this.f2 = f2;
        System.out.println("InitMethodBeanService setF2 method execute...");
    }
}
```

执行结果打印如下：
```
InitMethodBeanService static block execute...
InitMethodBeanService construct method execute...
InitMethodBeanService setF2 method execute...
InitMethodBeanService init method execute...
test method execute...
```

## 1.2 InitializingBean接口  

> **参照：[Spring官方文档beans-factory-lifecycle-initializingbean](http://docs.spring.io/spring/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#beans-factory-lifecycle-initializingbean)**  

`InitializingBean`接口中声明了一个方法`afterPropertiesSet`，该方法会在Spring容器**实例化对象并设置完属性值之后**被调用。和上面的`init-method`实现的功能一致，因此Spring不推荐使用`InitializingBean`接口。  

例子比较简单，不列出来了  

## 1.3 PostConstruct注解  

> **翻译：[Spring官方文档beans-postconstruct-and-predestroy-annotations](http://docs.spring.io/spring/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#beans-postconstruct-and-predestroy-annotations)**  

`@PostConstruct`注解是和`init-method`、`InitializingBean接口`实现效果一致的生命周期回调方法  

```java
@PostConstruct
public void postConstruct(){
    System.out.println("PostConstructService postConstruct method execute...");
}
```

---  

> **总结下上面三个生命周期回调方法`init-method`、`InitializingBean接口`、`@PostConstruct注解`**   
> 
 - 都是针对单个类的实例化后处理  
 -  执行时间都是在类实例化完成，且成员变量完成注入之后调用的  
 -  对于`init-method`，还可以在Spring配置文件的`beans`元素下配置默认初始化方法，配置项为`default-init-method`  
 -  若以上三种方式配置的初始化方法都不一样，则执行顺序为：`@PostConstruct注解方法` --> `InitializingBean的afterPropertiesSet` --> `init-method方法` ；若三种方式配置的方法一样，则方法只执行一次 （参照：[Spring官方文档beans-factory-lifecycle-combined-effect](http://docs.spring.io/spring/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#beans-factory-lifecycle-combined-effects)）
 -  有初始化回调方法，对应的也有销毁的回调方法。`@PostConstruct注解方法` --> `InitializingBean的afterPropertiesSet` --> `init-method方法` 分别对应 `@PreDestroy注解方法` --> `DisposableBean的destroy` --> `destroy-method方法`  

# 2. 容器级别扩展  

> **翻译：**[Spring官方文档3.8 Container Extension Points](http://docs.spring.io/spring/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#beans-factory-extension)

通常情况下，开发人员无需自定义实现一个`ApplicationContext`的子类去扩展Spring IOC容器，Spring IOC容器通过对外暴露的一些接口，可实现对Spring IOC容器的扩展。

## 2.1 BeanPostProcessor接口  

### 2.1.1 bean实例初始化后处理器及后处理器链  
`BeanPostProcessor`接口定义了两个**容器级别**的回调方法`postProcessBeforeInitialization`和`postProcessAfterInitialization`，用于在初始化实例后的一些逻辑处理，会针对容器中的所有实例进行处理。实现了`BeanPostProcessor`接口的类，称之为**bean实例初始化后处理器** 。  

若在Spring IOC容器中集成了多个实例初始化后处理器，这些后处理器构成的集合称之为**bean实例初始化后处理器链**。

1. `postProcessBeforeInitialization`方法在类实例化且成员变量注入完成之后执行，初始化方法（例如`InitializingBean`的`afterPropertiesSet`方法）**之前**执行  
2. `postProcessAfterInitialization`方法在类实例化且成员变量注入完成之后执行，初始化方法（例如`InitializingBean`的`afterPropertiesSet`方法）**之后**执行  

> - **实例初始化后处理器**多用于对实例的一些代理操作。Spring中一些使用到AOP的特性也是通过后处理器的方式实现的。  
> - **实例初始化后处理器链** 是多个后处理器，就会有执行顺序的问题，可以通过实现`Ordered`接口，指定后处理的执行顺序，`Ordered`接口声明了`getOrder`方法，方法返回值越小，后处理的优先级越高，越早执行。  
> -  在通过实现`BeanPostProcessor`接口自定义实例初始化后处理器的时候，建议也实现`Ordered`接口，指定优先级。  
> - 这些后处理器的作用域是当前的Spring IOC容器，即后处理器被声明的Spring IOC容器。对于有层次结构的Spring IOC容器，**实例初始化后处理器链**不会作用于其他容器所初始化的实例上，即使两个容器在同一层次结构上。  
> - 实例初始化后处理器的实现类只需要和普通的被Spring管理的bean一样声明，Spring IOC容器就会自动检测到，并添加到实例初始化后处理器链中。  
>- 相对于自动检测，我们也可以调用`ConfigurableBeanFactory`的`addBeanPostProcessor`方法，以编程的方式将一个实例初始化后处理器添加到实例初始化后处理器链中。这在需要判定添加条件的场景下比较实用。这种编程式的方式会忽略到实现的`Ordered`接口所指定的顺序，而会作用于所有的被自动检测的实例初始化后处理器之前。  

### 2.1.2 bean实例初始化后处理器与AOP  
`BeanPostProcessor`是一个特殊的接口，实现这个接口的类会被作为Spring管理的bean的实例的后处理器。因此，在Spring应用上下文启动的一个特殊阶段，会直接初始化所有实现了`BeanPostProcessor`接口的实例，以及该实例所引用的类也会被实例化。然后作为后处理器应用于其他普通实例。  

由于AOP的自动代理是以实例化后处理器的方式实现的，所以无论是**bean实例初始化后处理器链**实例还是其引用的实例，都不能被自动代理。因而，不要在这些实例上进行切面织入。（对于这些实例，会产生这样的日志消息：“类foo不能被所有的实例化后处理器链处理，即不能被自动代理”）。  

**注意：**当实例化后处理器以autowiring或`@Resource`的方式引用其他bean，Spring容器在以类型匹配依赖注入的时候，可能会注入非指定的bean（例如：实例化后处理器实现类以`Resource`方式依赖bean，若set方法和被依赖的bean的名称一致或者被依赖bean未声明名称，则依赖注入会以类型匹配的方式注入，此时可能会注入非指定的bean）。这也会导致自动代理或其他方式的实例化后处理器处理失败。

### 2.1.3 bean实例初始化后处理器示例  

```java
public class BeanPostProcessorService implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object o, String s) throws BeansException {
        System.out.println("BeanPostProcessorService postProcessAfterInitialization method execute... ");
        return o;
    }

    @Override
    public Object postProcessBeforeInitialization(Object o, String s) throws BeansException {
        System.out.println("BeanPostProcessorService postProcessBeforeInitialization method execute... ");
        return o;
    }
}

```

## 2.2 BeanFactoryPostProcessor接口  

### 2.2.1 bean factory后处理器  

通过实现`BeanFactoryPostProcessor`接口，可以读取容器所管理的bean的配置元数据，在bean完成实例化之前进行更改，这些bean称之为**bean factory后处理器**。  

>BeanFactoryPostProcessors与BeanPostProcessor接口的异同点：  
>**相同点：**  
>
> - 都是容器级别的后处理器  
> - 都可配置多个后处理器，并通过实现`Ordered`接口，指定执行顺序  
> - 都是针对接口声明的容器中所管理的bean进行处理，在有层级结构的容器中，不能处理其他容器中的bean，即使两个容器是同一层次  
> - 都是只需要在容器中和普通bean一样声明，容器会自动检测到，并注册为后处理器  
> - 会忽略延迟初始化属性配置
>
>**不同点：**  
>
> - `BeanFactoryPostProcessors`接口在bean**实例化前**处理bean的**配置元数据**，`BeanPostProcessor`接口在bean**实例化后**处理bean的**实例**  
> - `BeanFactoryPostProcessors`接口也能通过`BeanFactory.getBean()`方法获取bean的实例，这样会引起bean的实例化。由于`BeanFactoryPostProcessors`后处理器是在所有bean实例化之前执行，通过`BeanFactory.getBean()`方法会导致提前实例化bean，从而打破容器标准的生命周期，这样可能会引起一些负面的影响（例如：提前实例化的bean会忽略bean实例化后处理器的处理）。  

### 2.2.2 Spring内置及自定义bean factory后处理器
Spring内置了一些bean factory后处理器（例如：`PropertyPlaceholderConfigurer`和`PropertyOverrideConfigurer`）。同时也支持实现`BeanFactoryPostProcessor`接口，自定义bean factory后处理器。下面说说Spring内置的两个后处理器和自定义后处理器。  

**PropertyPlaceholderConfigurer**  

Spring为了避免主要的XML定义文件的修改而引起的风险，提供了配置分离，可以将一些可能变更的变量配置到属性配置文件中，并在XML定义文件中以占位符的方式引用。这样，修改配置只需要修改属性配置文件即可。**PropertyPlaceholderConfigurer**用于检测占位符，并替换占位符为配置属性值。示例如下：  

`PropertyPlaceholderConfigurer`通过`jdbc.properties`属性配置文件，在**运行时**，将`dataSource`这个bean中数据库相关信息的属性占位符替换成对应的配置值。  
XML配置如下：  

```java
<bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
	<property name="locations" value="classpath:com/foo/jdbc.properties"/>
</bean>

<bean id="dataSource" destroy-method="close"
		class="org.apache.commons.dbcp.BasicDataSource">
	<property name="driverClassName" value="${jdbc.driverClassName}"/>
	<property name="url" value="${jdbc.url}"/>
	<property name="username" value="${jdbc.username}"/>
	<property name="password" value="${jdbc.password}"/>
</bean>
```

属性配置文件`jdbc.properties`如下：  

```
jdbc.driverClassName=org.hsqldb.jdbcDriver
jdbc.url=jdbc:hsqldb:hsql://production:9002
jdbc.username=sa
jdbc.password=root
```

`PropertyPlaceholderConfigurer`不仅支持属性配置文件的读取，也支持读取系统属性。通过`systemPropertiesMode`属性值可配置读取优先级。各种取值说明如下：  

- 0 ： 不读取系统属性  
- 1 ： 若引用的属性配置文件中未检索到对应占位符的配置，则读取系统属性。默认为1  
- 2 ： 先读取系统属性，再读取引用的属性配置文件。这种配置可能导致系统属性覆盖配置文件。  

**PropertyOverrideConfigurer**  

`PropertyOverrideConfigurer`类可以通过引用属性配置文件，直接给容器中的bean赋值。当一个bean的属性被多个`PropertyOverrideConfigurer`类实例赋值时，最后一个的值会覆盖前面的。  

还是以上面给上面的`dataSource`的bean赋值为例：  

`PropertyOverrideConfigurer`类对属性配置文件的引用使用一个新的方式，如下：  
```
<context:property-override location="classpath:override.properties"/>
```

`override.properties`属性配置文件的属性的命名规则和上面不同（上面例子中需要保证属性名和占位符一致），命名规则是`beanName.property`  

```
dataSource.driverClassName=com.mysql.jdbc.Driver
dataSource.url=jdbc:mysql:mydb
dataSource.username=sa
dataSource.password=root
```

> 支持复合属性的赋值，但是要保证引用被赋值属性的对象非空  
> 例如：foo.fred.bob.sammy=123  

**自定义bean factory后处理器**  

自定义bean factory后处理器就是实现BeanFactoryPostProcessor接口，完成对Spring容器管理的bean的配置元数据进行修改。例如：修改类属性注入的值，示例如下：  

定义一个用户类`UserBean`  

```java
public class UserBean {
    private String userName;

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }
}
```

Spring XML配置文件配置用户类，并给用户名属性`userName`注入值`haha`  

```
<bean class="name.liuxi.spring.ext.BeanFactoryPostProcessorService"/>
<bean id="user" class="name.liuxi.spring.ext.UserBean">
      <property name="userName" value="haha"/>
</bean>
```

下面是自定义的bean factory后处理器，修改属性`userName`的值为`heihei`  

```java
public class BeanFactoryPostProcessorService implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("BeanFactoryPostProcessorService postProcessBeanFactory method execut...");
        BeanDefinition bd = beanFactory.getBeanDefinition("user");
        MutablePropertyValues pv =  bd.getPropertyValues();
        if(pv.contains("userName"))
        {
            pv.addPropertyValue("userName", "heihei");
        }
    }
}
```