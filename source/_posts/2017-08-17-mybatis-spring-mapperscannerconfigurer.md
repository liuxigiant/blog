---
layout: post
title: "MyBatis-Spring源码之MapperScannerConfigurer"
description: mybatis-spring使用MapperScannerConfigurer扫描Dao层接口并生成实现代理类
date: 2017-08-17 18:11:32 +0800
catalog: true
categories:
- MyBatis Spring
tags:
- MyBatis Spring
---

在首篇**MyBatis-Spring简介与用法**中介绍的用法二，需要对各个Dao层接口，配置代理工厂类。在用法三中介绍的，Mybatis Spring根据D指定扫描的包路径，自动扫描接口，并生成代理类，从而使coder从大量繁琐的配置中解脱出来  -_-    

# 1 配置  

首先回顾下，指定包路径扫描的配置文件的使用方式，如下：  

```xml
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
      <property name="basePackage" value="name.liux.share.dao" />
</bean>
```

如上配置，只需要配置包路径，Mybatis Spring就会扫描包路径下的接口，并生成代理实现类。  

# 2. 自动扫描   

mapper自动扫描辅助类`MapperScannerConfigurer`实现了spring的`BeanDefinitionRegistryPostProcessor`接口。  

>**BeanDefinitionRegistryPostProcessor**：Spring容器Bean Factory的一个后置处理器，在Spring解析完XML文件并封装完Bean定义`BeanDefinition`后、Bean初始化之前，会执行回调方法`postProcessBeanDefinitionRegistry`。主要用于修改Bean定义`BeanDefinition`。  

从`MapperScannerConfigurer`实现了spring的`BeanDefinitionRegistryPostProcessor`接口能大致能猜出来，Spring Mybatis通过扫描指定包下的接口，生成Spring类定义`BeanDefinition`,下面从回调方法`postProcessBeanDefinitionRegistry`开始看：  

```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
    if (this.processPropertyPlaceHolders) {
      processPropertyPlaceHolders();
    }

	//mapper接口声明扫描工具类，继承自Spring的类路径扫描类ClassPathBeanDefinitionScanner
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.registerFilters();

	//重点在扫描类，可以看出是扫描配置的包路径
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
}
```

下面看下mapper接口声明扫描工具类ClassPathMapperScanner的扫描方法：  

```java
public Set<BeanDefinitionHolder> doScan(String... basePackages) {

	//调用父类的扫描方法（可参考Spring IOC相关源码，此处不做说明），获取Bean属性
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
    } else {

		//循环Bean定义处理
      for (BeanDefinitionHolder holder : beanDefinitions) {
        GenericBeanDefinition definition = (GenericBeanDefinition) holder.getBeanDefinition();

        if (logger.isDebugEnabled()) {
          logger.debug("Creating MapperFactoryBean with name '" + holder.getBeanName() 
              + "' and '" + definition.getBeanClassName() + "' mapperInterface");
        }

        // the mapper interface is the original class of the bean
        // but, the actual class of the bean is MapperFactoryBean

		//给当前Bean添加mapperInterface属性，值为当前Bean名称（即mapper接口名）
        definition.getPropertyValues().add("mapperInterface", definition.getBeanClassName());

		//指定当前类为MapperFactoryBean
        definition.setBeanClass(MapperFactoryBean.class);

        definition.getPropertyValues().add("addToConfig", this.addToConfig);

        boolean explicitFactoryUsed = false;
        if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
          definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
          explicitFactoryUsed = true;
        } else if (this.sqlSessionFactory != null) {
          definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
          explicitFactoryUsed = true;
        }

        if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
          if (explicitFactoryUsed) {
            logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
          }
          definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
          explicitFactoryUsed = true;
        } else if (this.sqlSessionTemplate != null) {
          if (explicitFactoryUsed) {
            logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
          }
          definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
          explicitFactoryUsed = true;
        }

        if (!explicitFactoryUsed) {
          if (logger.isDebugEnabled()) {
            logger.debug("Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
          }

			//根据类型，自动注入
          definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
        }
      }
    }

	//返回修改后的bean定义
    return beanDefinitions;
}
```

从上面可看出自动扫描生成代理，其实是生成代理工厂，流程如下：  

- **扫描**（借助Spring的扫描工具类）：指定包下的mapper接口声明（即Dao层接口），获取Bean定义`BeanDefinition`
- **修改Bean定义BeanDefinition**：
	- 添加属性`mapperInterface`,值为接口名
	- 修改Bean定义的类为`MapperFactoryBean`
	- 设置Bean定义的自动注入（用于自动注入`sqlSessionTemplate`）

**总结：**  
通过Spring扫描指定包下接口声明，获取Bean定义，修改Bean定义为MapperFactoryBean，并设置必要属性