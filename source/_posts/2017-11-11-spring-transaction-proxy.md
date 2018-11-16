---
layout: post
title: "Spring基于注解事务代理类的生成"
description: 从源码角度分析，Spring基于注解事务代理类的生成
date: 2017-11-11 15:01:24 +0800
catalog: true
categories: 
- Spring Transaction
tags: 
- Spring Transaction
- Spring Proxy
---

在读《Spring技术内幕》一书的过程中，IOC、AOP、事务等都是分模块讲解实现原理和源码。根据个人经验，将各个模块与项目中的使用方式相对应，由使用方式开始分析，顺藤摸瓜，能更加清晰的查看源码实现的脉络。   

例如：  
  
- 常规的web项目会在XML中配置Spring IOC容器初始化的监听器ContextLoaderListener，从ContextLoaderListener源码入手，就能挖掘出Spring IOC容器XmlWebApplicationContext初始化的过程  
- Spring MVC的IOC容器初始化也可以通过查看DispatcherServlet源码的方式查看初始化过程    
- Spring HTTP远端调用则可通过使用方式，从web xml配置开始，捋清请求拦截和调用的服务端源码调用方式   

Spring事务作为常规项目中的一个常用模块，对于事务的使用和配置方式，大家都比较清晰，而《Spring技术内幕》一书中的“Spring事务处理的实现”是从事务代理开始讲解的，从项目中常用的配置方式中，并不能直接找到代理生成的入口。  

现在大多项目使用基于注解的方式配置事务，要是还是使用TransactionTemplate的编程式事务，那么事务处理的脉络还是很清晰的，所以本文讨论的就是**基于注解的方式配置事务代理类的生成**。    

# 1 开篇  

**基于注解事务代理类的生成**：研究这个问题是为了更好的研究Spring事务原理和源码，在项目使用注解事务时候，能知其然使其所以然。  

Spring事务实现基于AOP实现，而AOP的核心是代理和回调（或者称之为拦截器），对于Spring注解事务，manager层的业务类（项目开发过程中事务注解一般在manager层使用）的**代理类的生成**其实是在IOC容器初始化过程中，实例化Bean的时候就已经生成代理和织入事务处理回调。  

综上所述：Spring注解事务的实现其实由Spring IOC、AOP 和 事务 三大模块支撑，下面从源码角度一一道来  

> Spring 源码版本 4.3.0  

# 2 Spring IOC容器初始化  

本节部分重点关注基于注解事务的代理类的生成在IOC容器初始化过程中的方法调用脉络，关于IOC容器初始化的其他核心方法会被忽略    

## 2.1 监听器ContextLoaderListener  

![](/images/tx_listener.png)   

如上方法调用时序图，分析**Spring IOC容器初始化**过程，可以从Spring容器初始化监听器**ContextLoaderListener**（web项目中使用Spring一般需要在web.xml中配置监听器）开始  

从**ContextLoaderListener**的源码（核心方法在其父类ContextLoader中）可看出，默认的IOC容器是**XmlWebApplicationContext**，在实例化完Bean后，调用**refresh**方法初始化IOC容器。  

> 看过《Spring技术内幕》一书IOC部分就知道，IOC容器的初始化流程核心就是refresh方法  

## 2.2 容器初始化  

![](/images/tx_ioc.png)   

如上图展示了Spring IOC容器初始化过程中，基于注解事务代理类生成的方法调用时序关系，下面简单分析下各个方法    

### 2.2.1 refresh  

XmlWebApplicationContext#refresh  

在2.1小节监听器最终调用`XmlWebApplicationContext#refresh`方法，从`XmlWebApplicationContext`类的继承体系可看出，`refresh`方法是在类`AbstractApplicationContext`中实现的。  

该方法是IOC容器初始化的核心流程，在IOC容器 beanFactory创建完成并根据Spring配置文件加载类定义BeanDefinition（obtainFreshBeanFactory方法）后，会实例化单例Bean（非延迟初始化），即调用`AbstractApplicationContext#finishBeanFactoryInitialization`方法  

### 2.2.2 finishBeanFactoryInitialization  

AbstractApplicationContext#finishBeanFactoryInitialization

finishBeanFactoryInitialization方法的核心是调用实例化单例Bean方法`DefaultListableBeanFactory#preInstantiateSingletons`     

### 2.2.3 preInstantiateSingletons  

DefaultListableBeanFactory#preInstantiateSingletons

preInstantiateSingletons会循环获取单例Bean实例  

### 2.2.4 doGetBean  

AbstractBeanFactory#doGetBean

doGetBean获取Bean时候，若当前容器中不包含该Bean，则取父容器中获取。同时会获取所有依赖的Bean。  

重点在创建单例Bean的`DefaultSingletonBeanRegistry#getSingleton`方法的调用（注意此处传了一个接口BeanFactory的实现，作为真正创建Bean的实现）。  

### 2.2.5 getSingleton  

DefaultSingletonBeanRegistry#getSingleton

单例Bean在创建完后会添加到容器中，而创建单例Bean的方法就是前面说的到传入的BeanFactory的getObject方法  

### 2.2.6 createBean&doCreateBean  

AbstractAutowireCapableBeanFactory#createBean&doCreateBean

创建Bean的核心方法是`doCreateBean`，包含以下几个重要的方法：  

- `createBeanInstance`：使用反射创建对象  
- `populateBean`：属性赋值以及实例化回调（实现`InstantiationAwareBeanPostProcessor`接口）  
- `initializeBean`：初始化方法调用以及初始化回调（实现`BeanPostProcessor`接口）    

### 2.2.7 initializeBean    

AbstractAutowireCapableBeanFactory#initializeBean

`initializeBean`方法完成初始化方法和初始化回调处理器链（实现`BeanPostProcessor`接口）的调用，源码如下：    

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {	
	//......      略    ......
	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}

	try {
		invokeInitMethods(beanName, wrappedBean, mbd);
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				(mbd != null ? mbd.getResourceDescription() : null),
				beanName, "Invocation of init method failed", ex);
	}

	if (mbd == null || !mbd.isSynthetic()) {
		//初始化回调生成代理类
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}
	return wrappedBean;
}
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
			//回调处理
			result = beanProcessor.postProcessAfterInitialization(result, beanName);
			if (result == null) {
				return result;
			}
		}
		return result;
	}
```

### 2.2.8 postProcessAfterInitialization  

AbstractAutoProxyCreator#postProcessAfterInitialization

`AbstractAutoProxyCreator`的初始化回调方法`postProcessAfterInitialization`完成了代理类的生成  

>   `AbstractAutoProxyCreator`是一个抽象类，从类名也能看出是自动代理构造器；  
>   此处为了梳理逻辑使用抽象类，至于在项目中使用的具体类，后续再说    

# 3 Spring AOP事务代理生成  

![](/images/tx_aop.png)  

上面IOC容器初始化最后是到`AbstractAutoProxyCreator#postProcessAfterInitialization`方法完成代理类的生成，具体生成流程方法调用时序图如上，下面简单分析下各个方法  

## 3.1 wrapIfNecessary  

AbstractAutoProxyCreator#wrapIfNecessary  

从下面源码能很清晰的看出代理的创建过程：  

- 获取回调拦截器（从下面源码能看出，未获取到回调拦截器则不会生成代理）  
- 生成代理类 

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
	//......      略    ......
	// Create proxy if we have advice.
	//获取回调拦截器（AOP核心：代理和回调拦截器）
	Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
	if (specificInterceptors != DO_NOT_PROXY) {
		this.advisedBeans.put(cacheKey, Boolean.TRUE);
		//生成代理类
		Object proxy = createProxy(
				bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
		this.proxyTypes.put(cacheKey, proxy.getClass());
		return proxy;
	}

	this.advisedBeans.put(cacheKey, Boolean.FALSE);
	return bean;
}
```

## 3.2 获取回调拦截器  

还是根据上面方法调用时序图的方法调用一一介绍  

### 3.2.1 获取合法的advisor  

AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean  
AbstractAdvisorAutoProxyCreator#findEligibleAdvisors  

`findEligibleAdvisors`方法是查找合法的advisor，包含三个主要方法：  

- `findCandidateAdvisors`：查找候选的advisor，从源码可看出就是根据`Advisor.class`类型在IOC容器中查找  
- `findAdvisorsThatCanApply`：遍历候选advisor，判断是否是合法的advisor，此处重点关注事务advisor的判断处理，下面再说  
- `sortAdvisors`：给advisor排序，下面再说  


源码如下：  

```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
	List<Advisor> candidateAdvisors = findCandidateAdvisors();
	List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
	extendAdvisors(eligibleAdvisors);
	if (!eligibleAdvisors.isEmpty()) {
		eligibleAdvisors = sortAdvisors(eligibleAdvisors);
	}
	return eligibleAdvisors;
}
```

至此，对于advisor、回调拦截器、advisor的判断，可能比较迷糊，下面说说我自己的理解：  

在Spring AOP中，有如下几个概念：  

- PointCut：切点  --  横切面的定义（advisor的判断就是根据切点判断的）  
- Advice：增强  -- 在切面前、后需要做的事       Interceptor接口继承Advice接口 （就是上面说的回调拦截器）   
- Advisor：通知器  -- 将切面和增强关联起来   

AOP的核心技术是代理，只有给目标对象生成代理，才能面向切面做增强（ps：可参阅《Spring技术内幕》AOP相关部分）  

下面回到正题，分析下事务advisor的判断逻辑和advisor的排序  

### 3.2.2 事务advisor的判断  

此处为何是事务advisor的判断?  获取advisor时候，会根据类型获取所有的advisor，但是本文只关注事务，所有只需要分析`BeanFactoryTransactionAttributeSourceAdvisor`即可  

> 至于为何是`BeanFactoryTransactionAttributeSourceAdvisor`类，后续再说（遗留两个问题了-_-）  

上面说到，advisor包含pointcut和advice，判断匹配是pointcut的事，从源码可以看出，该advisor包含一个默认的pointcut，即`TransactionAttributeSourcePointcut`，判断方法源码如下：  

> 从`AopUtils#canApply`方法可看出，通过循环目标类的所有方法进行判断的  

```java
public boolean matches(Method method, Class<?> targetClass) {
	if (TransactionalProxy.class.isAssignableFrom(targetClass)) {
		return false;
	}
	//获取TransactionAttributeSource
	TransactionAttributeSource tas = getTransactionAttributeSource();
	//从方法上获取到事务属性（即使用Transactional注解配置的信息），则匹配成功
	return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
}
```  

从上述源码可看出，匹配成功的标注是从目标方法上解析出事务属性，而事务属性的来源可以有多种（例如：xml和注解）  

本文讨论的是基于注解的事务，所以`getTransactionAttributeSource()`方法会返回`AnnotationTransactionAttributeSource`（这个类从何而来，后续再说，遗留三个问题了-_-)  

下面看下`AnnotationTransactionAttributeSource`从目标类的方法上获取事务属性的方法`determineTransactionAttribute`源码：  

```java
protected TransactionAttribute determineTransactionAttribute(AnnotatedElement ae) {
	if (ae.getAnnotations().length > 0) {
		for (TransactionAnnotationParser annotationParser : this.annotationParsers) {
			//解析方法上的事务属性
			TransactionAttribute attr = annotationParser.parseTransactionAnnotation(ae);
			if (attr != null) {
				return attr;
			}
		}
	}
	return null;
}
```

上面解析部分主要是有解析器`SpringTransactionAnnotationParser`负责，解析器是在`AnnotationTransactionAttributeSource`类创建的时候初始化的  

下面看看具体的解析方法`parseTransactionAnnotation`源码:  

```java
public TransactionAttribute parseTransactionAnnotation(AnnotatedElement ae) {
	AnnotationAttributes attributes = AnnotatedElementUtils.getMergedAnnotationAttributes(ae, Transactional.class);
	if (attributes != null) {
		return parseTransactionAnnotation(attributes);
	}
	else {
		return null;
	}
}
```

从上述源码可看出：读取方法上的`Transactional`注解来解析注解属性  

总结：  

- 若目标类的方法上的有`Transactional`注解，则生成目标类的代理类，并将回调拦截器（从上面源码中看出是advisor，包含拦截器advice）织入代理中  
- 目标类方法调用时候触发代理的回调拦截器，实现事务管理  


### 3.2.3 回调拦截器排序  

可查看下`AbstractAdvisorAutoProxyCreator#sortAdvisors`源码，是通过`Order`注解指定顺序的  

> 在自定义拦截器时候，会存在拦截器先后顺序的问题，此时用到的就是Order注解指定顺序  


## 3.3 生成代理类  

AbstractAutoProxyCreator#createProxy  

从如下源码中可看出，最终调用`proxyFactory.getProxy`生成代理类  

> 至此：AOP部分完毕。熟悉Spring AOP实现的话，对`proxyFactory`会比较熟悉  
> 《Spring技术内幕》一书就是从此类讲解Spring AOP源码的（包括对JDK动态代理和CGLIB的封装及选择）        

```java
protected Object createProxy(
		Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

	if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
		AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
	}

	ProxyFactory proxyFactory = new ProxyFactory();
	proxyFactory.copyFrom(this);

	if (!proxyFactory.isProxyTargetClass()) {
		if (shouldProxyTargetClass(beanClass, beanName)) {
			proxyFactory.setProxyTargetClass(true);
		}
		else {
			evaluateProxyInterfaces(beanClass, proxyFactory);
		}
	}

	Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
	for (Advisor advisor : advisors) {
		proxyFactory.addAdvisor(advisor);
	}

	proxyFactory.setTargetSource(targetSource);
	customizeProxyFactory(proxyFactory);

	proxyFactory.setFrozen(this.freezeProxy);
	if (advisorsPreFiltered()) {
		proxyFactory.setPreFiltered(true);
	}

	return proxyFactory.getProxy(getProxyClassLoader());
}
```

# 4 事务  

从前面 `Spring IOC` 和 `Spring AOP` 两部分可看出，对于目标类，若方法上有事务注解，则回生成代理类，并织入事务处理的advisor  


在调用代理类的方法，会触发回调拦截器链的处理，其中就包含上面织入的事务处理的advisor，即`BeanFactoryTransactionAttributeSourceAdvisor`  

在上一节AOP中有提到，advisor包含pointcut和advice，advice就是拦截器，负责拦截目标方法进行AOP增强处理  

`BeanFactoryTransactionAttributeSourceAdvisor`中包含的advice就是`TransactionInterceptor`（这个类从何而来，后续再说，遗留四个问题了-_-)  

看到拦截器`TransactionInterceptor`，就非常熟悉了，《Spring技术内幕》事务原理和源码就是从这个拦截器类开始讲解的     

> 事务管理就是通过`TransactionInterceptor`拦截器对代理类的事务方法进行处理，实现Spring事务的管理    
> 具体实现可参阅《Spring技术内幕》   


# 5 遗留问题  

前面遗留了四个问题，下面分两部分来说，第一个问题是第一部分；第二、三、四个问题是第二部分  

## 5.1 问题二、三、四：注解事务Advisor  

先来回顾下问题二、三、四这三个问题：  

- 3.3.2小节问题二：在根据类型`Advisor.class`查找所有advisor时候，为什么会有事务处理的advisor类`BeanFactoryTransactionAttributeSourceAdvisor`  
- 3.3.2小节问题三：`BeanFactoryTransactionAttributeSourceAdvisor`包含pointcut（直接new的），在校验是否需要拦截`match()`时候，判断事务来源`AnnotationTransactionAttributeSource`是从何而来  
- 4小节问题四：`BeanFactoryTransactionAttributeSourceAdvisor`包含的advice拦截器`TransactionInterceptor`从何而来  

要分析上面三个问题，让我们先看下，Spring基于注解事务在xml文件中的配置：  

```xml
<tx:annotation-driven/>
```

上面xml配置用于启用事务注解`Transactional`,从Spring schema定义方式可查看`tx`到命名空间的处理类为`TxNamespaceHandler`  

> 前面有篇文章有专门说明**Spring schema扩展**  
> Spring事务Schema `tx` 的处理方式和自定义扩展一致，可到jar包的查看命名空间指定的处理类，配置文件路径是`META-INF/spring.handlers`  
> 事务`tx`处理类则可在 `spring-tx-4.3.0.RELEASE.jar/META-INF/spring.handlers`中找到  

下面看下`TxNamespaceHandler`源码是如何处理`tx`配置的  

```java
public void init() {
	registerBeanDefinitionParser("advice", new TxAdviceBeanDefinitionParser());
	registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
	registerBeanDefinitionParser("jta-transaction-manager", new JtaTransactionManagerBeanDefinitionParser());
}
```   

我们使用的是`annotation-driven`元素，所以接下来看下xml元素解析器`AnnotationDrivenBeanDefinitionParser`的源码  

```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
	this.registerTransactionalEventListenerFactory(parserContext);
	String mode = element.getAttribute("mode");
	if ("aspectj".equals(mode)) {
		this.registerTransactionAspect(element, parserContext);
	} else {
		AnnotationDrivenBeanDefinitionParser.AopAutoProxyConfigurer.configureAutoProxyCreator(element, parserContext);
	}

	return null;
}
```  

我们使用`annotation-driven`元素配置时候未指定mode参数，即为默认的`proxy`,所以下面看下`AnnotationDrivenBeanDefinitionParser.AopAutoProxyConfigurer#configureAutoProxyCreator`源码  

```java
public static void configureAutoProxyCreator(Element element, ParserContext parserContext) {
	AopNamespaceUtils.registerAutoProxyCreatorIfNecessary(parserContext, element);
	String txAdvisorBeanName = "org.springframework.transaction.config.internalTransactionAdvisor";
	if (!parserContext.getRegistry().containsBeanDefinition(txAdvisorBeanName)) {
		Object eleSource = parserContext.extractSource(element);
		RootBeanDefinition sourceDef = new RootBeanDefinition("org.springframework.transaction.annotation.AnnotationTransactionAttributeSource");
		sourceDef.setSource(eleSource);
		sourceDef.setRole(2);
		String sourceName = parserContext.getReaderContext().registerWithGeneratedName(sourceDef);
		RootBeanDefinition interceptorDef = new RootBeanDefinition(TransactionInterceptor.class);
		interceptorDef.setSource(eleSource);
		interceptorDef.setRole(2);
		AnnotationDrivenBeanDefinitionParser.registerTransactionManager(element, interceptorDef);
		interceptorDef.getPropertyValues().add("transactionAttributeSource", new RuntimeBeanReference(sourceName));
		String interceptorName = parserContext.getReaderContext().registerWithGeneratedName(interceptorDef);
		RootBeanDefinition advisorDef = new RootBeanDefinition(BeanFactoryTransactionAttributeSourceAdvisor.class);
		advisorDef.setSource(eleSource);
		advisorDef.setRole(2);
		advisorDef.getPropertyValues().add("transactionAttributeSource", new RuntimeBeanReference(sourceName));
		advisorDef.getPropertyValues().add("adviceBeanName", interceptorName);
		if (element.hasAttribute("order")) {
			advisorDef.getPropertyValues().add("order", element.getAttribute("order"));
		}
		parserContext.getRegistry().registerBeanDefinition(txAdvisorBeanName, advisorDef);
		CompositeComponentDefinition compositeDef = new CompositeComponentDefinition(element.getTagName(), eleSource);
		compositeDef.addNestedComponent(new BeanComponentDefinition(sourceDef, sourceName));
		compositeDef.addNestedComponent(new BeanComponentDefinition(interceptorDef, interceptorName));
		compositeDef.addNestedComponent(new BeanComponentDefinition(advisorDef, txAdvisorBeanName));
		parserContext.registerComponent(compositeDef);
	}
}
```  

上述源码会在Spring IOC容器中注册三个bean定义：  

- advisor：  
`RootBeanDefinition advisorDef = new RootBeanDefinition(BeanFactoryTransactionAttributeSourceAdvisor.class)`  

- transactionAttributeSource：  
`RootBeanDefinition sourceDef = new RootBeanDefinition("org.springframework.transaction.annotation.AnnotationTransactionAttributeSource");`  

- advice:  
`RootBeanDefinition interceptorDef = new RootBeanDefinition(TransactionInterceptor.class);`  

从上述源码也能看出，advisor中是包含transactionAttributeSource和advice的  

```java
advisorDef.getPropertyValues().add("transactionAttributeSource", new RuntimeBeanReference(sourceName));
advisorDef.getPropertyValues().add("adviceBeanName", interceptorName);
```  

**总结**：  

我们在使用注解事务，配置xml注解驱动`tx:annotation-driven`时候，会自动在Spring IOC容器中注册Bean定义  
`BeanFactoryTransactionAttributeSourceAdvisor`、`AnnotationTransactionAttributeSource`和`TransactionInterceptor`  

## 5.2 问题一：自动代理构造器BeanPostProcessor  

先回顾下2.2.8小节的问题一，自动代理构造器我们是通过抽象类`AbstractAutoProxyCreator`来分析的，那么具体的实现类是哪个呢？又是如何注册到IOC容器中的呢？  

在理解自动创建代理之前，先来看下，**编程式代理**的使用方式（可查看[官网文档programmatically](https://docs.spring.io/spring/docs/4.3.12.RELEASE/spring-framework-reference/htmlsingle/#aop-prog)）  

```java
ProxyFactory factory = new ProxyFactory(myBusinessInterfaceImpl);
factory.addAdvice(myMethodInterceptor);
factory.addAdvisor(myAdvisor);
MyBusinessInterface tb = (MyBusinessInterface) factory.getProxy();
```  
> ProxyFactory是Spring AOP创建代理的工厂类，《Spring技术内幕》AOP部分就是以这个类为核心说明的  

和事务的使用方式的演变过程一样：编程式事务、XML配置、基于注解  

对于Spring AOP来说：也有编程式、配置方式和自动创建AOP代理  

Spring事务管理中对目标类的代理就是**自动创建代理**的方式（可查看[官方文档auto-proxy](https://docs.spring.io/spring/docs/4.3.12.RELEASE/spring-framework-reference/htmlsingle/#aop-autoproxy)）  

- BeanNameAutoProxyCreator：根据类名匹配（其实就是类名匹配作为pointcut的校验）  
- DefaultAdvisorAutoProxyCreator：Spring提供的一个默认实现类，扫描Advisor（包含pointcut和advice），自动创建代理  
- AbstractAdvisorAutoProxyCreator：Spring提供的一个抽象实现类(扫描Advisor，自动创建代理)，可根据需求扩展  

再回过头来看下：问题1中的`AbstractAutoProxyCreator`其实就是上面第二、三种方式的父类  

从`AbstractAdvisorAutoProxyCreator`的类图可看出，有4个具体的实现类：  

- AnnotationAwareAspectJAutoProxyCreator  
- AspectJAwareAdvisorAutoProxyCreator  
- DefaultAdvisorAutoProxyCreator  
- InfrastructureAdvisorAutoProxyCreator  

Spring注解事务使用的是哪个实现类呢？  

回到5.1小节中`AnnotationDrivenBeanDefinitionParser.AopAutoProxyConfigurer#configureAutoProxyCreator`的源码  

```java
public static void configureAutoProxyCreator(Element element, ParserContext parserContext) {
	AopNamespaceUtils.registerAutoProxyCreatorIfNecessary(parserContext, element);
	// ...... 略 ......
}
```  

第一行就是注册自动代理构建器，最终调用`AopConfigUtils#registerAutoProxyCreatorIfNecessary`方法，注册的是`InfrastructureAdvisorAutoProxyCreator`  

**总结**：  
  
我们在使用注解事务，配置xml注解驱动`tx:annotation-driven`时候，不仅会注册事务相关的advisor，还会注册自动代理构建器`InfrastructureAdvisorAutoProxyCreator`  

> 由于`InfrastructureAdvisorAutoProxyCreator`实现了`BeanPostProcessor`，所以也相当于向Spring IOC容器中添加了一个`BeanPostProcessor`  


# 6 总结  

从上述内容可看出：基于注解事务是IOC、AOP和事务的集大成者  

- 配置xml事务注解驱动`tx:annotation-driven`,会注册自动代理的BeanPostProcessor `InfrastructureAdvisorAutoProxyCreator` 和事务advisor `BeanFactoryTransactionAttributeSourceAdvisor`  
- Spring IOC容器初始化过程中：扫描所有的BeanPostProcessor（可查看`AbstractApplicationContext#refresh方法中的registerBeanPostProcessors`）  
- Spring IOC容器初始化过程中：在实例化单例Bean后调用BeanPostProcessor的回调函数postProcessAfterInitialization（就会调用到上面注册的`InfrastructureAdvisorAutoProxyCreator`）  
- BeanPostProcessor的回调扫描所有的advisor（就会扫描到上面注册的事务advisor `BeanFactoryTransactionAttributeSourceAdvisor`）  
- 过滤合法的advisor：若目标类的方法上存在`Transactional`事务注解，那么`BeanFactoryTransactionAttributeSourceAdvisor`就是合法的advisor  
- `ProxyFactory`生成代理：根据目标类和`BeanFactoryTransactionAttributeSourceAdvisor`（回调链中的一个）生成代理  
- 代理类被调用，触发回调函数，会调用`BeanFactoryTransactionAttributeSourceAdvisor`中advice（TransactionInterceptor）的回调函数  
- TransactionInterceptor完成事务的管理  

