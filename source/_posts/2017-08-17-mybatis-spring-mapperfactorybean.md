---
layout: post
title: "MyBatis-Spring源码之MapperFactoryBean"
description: mybatis-spring使用MapperFactoryBean生成Dao层接口的实现代理类
date: 2017-08-17 16:06:08 +0800
catalog: true
categories:
- MyBatis Spring
tags:
- MyBatis Spring
---

在首篇**MyBatis-Spring简介与用法**中介绍的用法一，需要对各个Dao层接口，编写实现类，实现类中方法一般都是直接调用`SqlSessionTemplate`的相应方法。在用法二中介绍的，Mybatis Spring根据Dao层接口生成实现代理类，从而使coder从机械、繁琐的Dao层实现类编码中解脱出来。  

# 1 配置  

首先回顾下，生成Dao层接口实现类的配置文件的使用方式，如下：  

```xml
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean" >
      <property name="mapperInterface" value="name.liux.share.dao.UserDao" />
      <property name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
```

对于每个Dao层接口，都需要配置一个工厂类`MapperFactoryBean`(老套路，Spring中*FactoryBean，看getObject方法)，需要注入两个参数，一个是被代理的接口（即Dao层接口声明），一个是`sqlSessionFactory`。  

# 2 初始化  

关于`MapperFactoryBean`初始化，需要注意以下几点：  

- **入参mapperInterface**：mapper接口声明，即Dao层接口声明，用于生成代理类（JDK动态代理）
- **入参sqlSessionFactory**：用于初始化`SqlSessionDaoSupport`（也可配置`sqlSessionTemplate`，配置`sqlSessionFactory`最终也是生成`sqlSessionTemplate`，首篇文章2.2有说到过）
- **初始化回调**：`SqlSessionDaoSupport`继承自`DaoSupport`,`DaoSupport`实现了`InitializingBean`,`MapperFactoryBean`初始化回调方法为`DaoSupport#afterPropertiesSet`

## 2.1 回调方法checkDaoConfig  

上面说到`MapperFactoryBean`初始化回调方法为`DaoSupport#afterPropertiesSet`，从源码（略）中可看出，`MapperFactoryBean`重写了父类的回调方法`checkDaoConfig`,源码如下：  

```java
protected void checkDaoConfig() {
    super.checkDaoConfig();

    notNull(this.mapperInterface, "Property 'mapperInterface' is required");

	//获取配置元数据对象Configuration
    Configuration configuration = getSqlSession().getConfiguration();

	//判断当前mapper声明是否在Configuration中注册（mapperRegistry -- Configuration中又一个关键成员变量，用于维护mapper接口声明和代理类的映射）
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
      try {

		//当前mapper声明未在Configuration中注册，则执行注册逻辑
        configuration.addMapper(this.mapperInterface);
      } catch (Throwable t) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", t);
        throw new IllegalArgumentException(t);
      } finally {
        ErrorContext.instance().reset();
      }
    }
  }
```

## 2.2 mapper接口声明注册  

mapper接口声明注册`configuration.addMapper`,其实是调用mapper接口注册器`MapperRegistry`的`addMapper`方法，源码如下：  

```java
public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {

		//再次判断是否已注册，已注册抛异常
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {

		//维护mapper接口声明和代理类的映射，代理类的工厂类为MapperProxyFactory（工厂类的初始化只是设置接口，代理类的生成后面再看）
        knownMappers.put(type, new MapperProxyFactory<T>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.

		//以下是使用注解的方式声明sql的解析（目前没使用这种方式，暂不关注）
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }
```

# 3 代理类生成  

代理工厂类`MapperFactoryBean`生成代理类的源码如下(老套路，Spring中*FactoryBean，看getObject方法)：  

```java
public T getObject() throws Exception {
	//getSqlSession返回为SqlSessionTemplate，详见SqlSessionDaoSupport类
	return getSqlSession().getMapper(this.mapperInterface);
}
```

## 3.1 生成代理工厂类

从上面源码跟踪可看出，最终是通过获取配置元数据对象`Configuration`中的mapper接口注册器`MapperRegistry`来获取代理类的，下面看看源码：  

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
	//获取代理类的工厂类，在初始化中设置的
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null)
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    try {

		//获取代理类实例
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

## 3.2 获取代理

代理工厂类获取实例如下：  

```java
public T newInstance(SqlSession sqlSession) {
	//代理类的拦截器，重点关注拦截器的回调方法MapperProxy#invoke
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
}
protected T newInstance(MapperProxy<T> mapperProxy) {
	//生成JDK动态代理
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}
```

## 3.3 代理拦截器回调方法  

下面看下代理类`MapperProxy`的回调方法`invoke`  

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

	//目标对象和方法声明对象是同一个，说明不是自动生成的代理类，直接调用目标方法
    if (Object.class.equals(method.getDeclaringClass())) {
      return method.invoke(this, args);
    }

	//解析、封装被调用方法，并缓存
    final MapperMethod mapperMethod = cachedMapperMethod(method);

	//执行方法
    return mapperMethod.execute(sqlSession, args);
}
private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
}
```

**MapperMethod**：根据被调用的方法，获取配置元数据对象`Configuration`中配置的信息，决定调用对`session`的方法调用  

**疑问**：  
了解完`MapperMethod`解析过程，来理解一下下面这个问题  
Q：Dao接口全限定类名和Mapper XML文件命名空间是否需要一致

A：需要保持一致（MapperMethod在解析SqlCommand时候是根据接口名加方法名去元数据对象Configur中获取声明的sql语句的）  