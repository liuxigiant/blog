---
layout: post
title: "MyBatis-Spring源码之基于xml的自动扫描"
description: mybatis-spring基于xml的自动扫描并生成实现代理类
date: 2017-08-17 19:45:23 +0800
catalog: true
categories:
- MyBatis Spring
tags:
- MyBatis Spring
---

在首篇**MyBatis-Spring简介与用法**中介绍的用法三，需要配置扫描工具类`MapperScannerConfigurer`,指定扫描的包路径，自动扫描接口，并生成代理类。在用法四种介绍的，通过扩展Spring schema，使用`mybatis:scan`实现自动扫描，屏蔽底层实现细节，从而使coder从需要记忆指定扫描类`MapperScannerConfigurer`中解脱出来  -_- 

# 1 扩展Spring Schema  

Mybatis Spring通过扩展Spring Schema的方式，定义mybatis的xml配置命名空间，配置`scan`元素，实现自动扫描，从而屏蔽底层实现细节。  

下面看下Spring配置文件配置方式：  

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:mybatis="http://mybatis.org/schema/mybatis-spring"
       xsi:schemaLocation="
	   http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
	   http://mybatis.org/schema/mybatis-spring http://mybatis.org/schema/mybatis-spring.xsd">

       <mybatis:scan base-package="name.liux.share.dao" />
</beans>
```

# 2 自动扫描  

如上配置可看出，扩展Spring schema，通过`mybatis:scan`实现自动扫描，而这个自定义的xml节点的处理类是`NamespaceHandler`  

> 查看扩展Spring schema的处理类的方法，细节后面再讲，`mybatis:scan`节点处理类可查看mybatis-spring jar包的 **META-INFO/Resource Bundle 'Spring'/spring.handlers** 文件  

下面看下`NamespaceHandler`的实现原理：  

```java
public void init() {
	//注册scan节点的处理类MapperScannerBeanDefinitionParser
    registerBeanDefinitionParser("scan", new MapperScannerBeanDefinitionParser());
}
```

`MapperScannerBeanDefinitionParser`自动扫描实现如下：  

```java
public synchronized BeanDefinition parse(Element element, ParserContext parserContext) {
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(parserContext.getRegistry());
    ClassLoader classLoader = scanner.getResourceLoader().getClassLoader();
    XmlReaderContext readerContext = parserContext.getReaderContext();
    scanner.setResourceLoader(readerContext.getResourceLoader());
    try {
      String annotationClassName = element.getAttribute(ATTRIBUTE_ANNOTATION);
      if (StringUtils.hasText(annotationClassName)) {
        @SuppressWarnings("unchecked")
        Class<? extends Annotation> markerInterface = (Class<? extends Annotation>) classLoader.loadClass(annotationClassName);
        scanner.setAnnotationClass(markerInterface);
      }
      String markerInterfaceClassName = element.getAttribute(ATTRIBUTE_MARKER_INTERFACE);
      if (StringUtils.hasText(markerInterfaceClassName)) {
        Class<?> markerInterface = classLoader.loadClass(markerInterfaceClassName);
        scanner.setMarkerInterface(markerInterface);
      }
      String nameGeneratorClassName = element.getAttribute(ATTRIBUTE_NAME_GENERATOR);
      if (StringUtils.hasText(nameGeneratorClassName)) {
        Class<?> nameGeneratorClass = classLoader.loadClass(nameGeneratorClassName);
        BeanNameGenerator nameGenerator = BeanUtils.instantiateClass((Class<?>) nameGeneratorClass, BeanNameGenerator.class);
        scanner.setBeanNameGenerator(nameGenerator);
      }
    } catch (Exception ex) {
      readerContext.error(ex.getMessage(), readerContext.extractSource(element), ex.getCause());
    }
    String sqlSessionTemplateBeanName = element.getAttribute(ATTRIBUTE_TEMPLATE_REF);
    scanner.setSqlSessionTemplateBeanName(sqlSessionTemplateBeanName);
    String sqlSessionFactoryBeanName = element.getAttribute(ATTRIBUTE_FACTORY_REF);
    scanner.setSqlSessionFactoryBeanName(sqlSessionFactoryBeanName);
    scanner.registerFilters();
    String basePackage = element.getAttribute(ATTRIBUTE_BASE_PACKAGE);

	//核心还是ClassPathMapperScanner来扫面指定包名下的接口，生成代理类
    scanner.scan(StringUtils.tokenizeToStringArray(basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
    return null;
}
```

如上源码可看出，核心功能和`MapperScannerConfigurer`的实现是一致的，只是换成了使用xml节点配置的方式配置