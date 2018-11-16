---
layout: post
title: "Spring Cache Abstraction实现原理"
description: 从源码的角度，分析Spring Cache Abstraction实现原理
date: 2018-02-06 14:36:09 +0800
catalog: true
categories:
- Spring Cache Abstraction
tags:
- Spring Cache
---

上篇文章 [Spring Cache Abstraction简介](/blog/20180116/spring-cache-abstraction.html) 中介绍了 `Spring Cache Abstraction`支持的特性以及使用方式。  

本文从源码的角度，分析下`Spring Cache Abstraction`注解方式的实现原理，主要包含以下几个方面：  

- 注解驱动
- 代理类生成
- 注解处理  

# 1 注解驱动

`Spring Cache Abstraction`支持注解的方式，但是要是注解生效，需要在 spring xml 文件中配置注解驱动 `<cache:annotation-driven/>`，来识别并处理缓存注解  

注解驱动主要完成对注解的识别和处理，即以下两方面：  

- 识别注解：注册xml元素解析器
- 处理注解：注册自动代理生成器（针对注解自动生成代理所需要的一系列Bean）

熟悉spring xml schema解析规则的话，可以在 `spring-context` 这个jar包的`META_INF/spring.handler`文件中找到cache元素的处理类`CacheNamespaceHandler`  

> spring xml schema解析规则可参考前文 [Spring Schema 扩展](/blog/20170818/spring-schema.html)  

## 1.1 注册xml元素解析器

从`CacheNamespaceHandler`类源码`init()`方法中可看出，在spring xml解析体系中注册了`annotation-driven`元素的解析处理类`AnnotationDrivenCacheBeanDefinitionParser`  

> 至于Spring xml的解析原理不是本文的讨论内容  

## 1.2 注册自动代理生成器

`AnnotationDrivenCacheBeanDefinitionParser`解析xml元素方法调用图如下：  

![](/images/cust/cache/cache-anno.png)

上图中第一个bean和第二个bean是对`Spring cache`的处理，第三个bean则是对AOP的处理  

- `AopNamespaceUtils#registerAutoProxyCreatorIfNecessary`: 用于注册一个自动代理的后处理器  
	
- `JCacheCachingConfigurer#registerCacheAdvisor`: 用于注册缓存注解处理的切面类
	其实就是向Spring容器中注册了几个bean  
	- 切面处理类`BeanFactoryCacheOperationSourceAdvisor`（**advisor = advice/interceptor + pointcut**）
	- 缓存注解解析类`AnnotationCacheOperationSource`（包装成pointcut）
	- 缓存注解代理类的回调处理器`CacheInterceptor`

# 2 代理类生成

上面基于注解以及xml配置驱动注解，完成了各种缓存注解的基础类的注册，下面需要使用注册的这些类，生成缓存代理，并在方法调用时候执行响应的缓存操作  

代理类生成方法调用图如下：  

![](/images/cust/cache/cache-proxy.png)

下面总结下缓存代理生成的步骤：  

- 配置xml注解驱动`cache:annotation-driven`,会注册自动代理的BeanPostProcessor `InfrastructureAdvisorAutoProxyCreator` 和缓存advisor `BeanFactoryCacheOperationSourceAdvisor`  
- Spring IOC容器初始化过程中：扫描所有的BeanPostProcessor（可查看`AbstractApplicationContext#refresh方法中的registerBeanPostProcessors`）  
- Spring IOC容器初始化过程中：在实例化单例Bean后调用BeanPostProcessor的回调函数postProcessAfterInitialization（就会调用到上面注册的`InfrastructureAdvisorAutoProxyCreator`）  
- BeanPostProcessor的回调扫描所有的advisor（就会扫描到上面注册的事务advisor `BeanFactoryCacheOperationSourceAdvisor`）  
- 过滤合法的advisor：若目标类的方法上存在缓存注解，那么`BeanFactoryCacheOperationSourceAdvisor`就是合法的advisor  
- `ProxyFactory`生成代理：根据目标类和`BeanFactoryCacheOperationSourceAdvisor`（回调链中的一个）生成代理  
- 代理类被调用，触发回调函数，会调用`BeanFactoryCacheOperationSourceAdvisor`中advice（CacheInterceptor）的回调函数  
- CacheInterceptor完成缓存操作  

> 熟悉spring 事务注解的话，可发现，事务注解代理类的生成原理和缓存注解代理类生成原理基本一致，可参见 [Spring基于注解事务代理类的生成](/blog/20171111/spring-transaction-proxy.html)  

# 3 注解处理

从上面可知，被缓存注解声明的目标方法，会生成当前类的代理类，并织入回调拦截器，在调用目标方法时，通过回调拦截器，完成缓存功能的织入操作  

缓存代理回调拦截器`CacheInterceptor`方法执行图如下：  

![](/images/cust/cache/cache-interceptor.png) 

`CacheInterceptor.invoke`是代理类目标方法被调用的回调执行入口（Spring AOP的内容，本文不详述）


重点是`CacheAspectSupport`中两次`execute`方法的调用  

## 3.1 第一个`execute`

- **方法签名**  

`org.springframework.cache.interceptor.CacheAspectSupport#execute(org.springframework.cache.interceptor.CacheOperationInvoker, java.lang.Object, java.lang.reflect.Method, java.lang.Object[])`

- **核心功能**  

完成注解信息解析，封装成缓存操作元数据`CacheOperationContexts`  

在封装缓存操作元数据`CacheOperationContexts`时候，重点是将根据缓存注解解析出对应的`Cache`,并将不同的缓存封装成不同的操作（例如：`CacheableOperation`、`CacheEvictOperation`等）  

> 在这过程中上篇文章中的`CacheManager`、`KeyGenerator`等配置就派上用场，可参照上篇文章 [Spring Cache Abstraction简介](/blog/20180116/spring-cache-abstraction.html)  

## 3.2 第二个`execute`

- **方法签名**  

`org.springframework.cache.interceptor.CacheAspectSupport#execute(org.springframework.cache.interceptor.CacheOperationInvoker, java.lang.reflect.Method, org.springframework.cache.interceptor.CacheAspectSupport.CacheOperationContexts)`

- **核心功能**  

完成缓存操作

可自行参照方法源码，封装的缓存操作逻辑很清晰，重点是关注下，缓存操作，是最终由解析出来的`Cache`执行的。由此，`Cache`缓存操作方法的调用，就由底层缓存实现来完成。