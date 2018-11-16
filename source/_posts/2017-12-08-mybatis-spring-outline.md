---
layout: post
title: "MyBatis-Spring用法与源码系列"
description: mybatis Spring用法与源码系列
date: 2017-12-08 11:36:21 +0800
catalog: true
categories:
- MyBatis Spring
tags:
- MyBatis Spring
- MyBatis
---

本系列从Spring集成mybatis的四种方式开始，逐步分析各种集成方式的底层源码实现方式  

- 简介与用法    

介绍`Mybatis Spring`组件及其用法，详见 [MyBatis-Spring简介与用法](/blog/20170718/mybatis-spring-usage.html)

- 源码  

根据各种用法，分析其源码实现，详见如下：  

[MyBatis-Spring源码之SqlSessionFactory](/blog/20170802/mybatis-spring-sessionfactory.html)  
[MyBatis-Spring源码之SqlSessionTemplate](/blog/20170814/mybatis-spring-sqlsessiontemplate.html)  
[MyBatis-Spring源码之MapperFactoryBean](/blog/20170817/mybatis-spring-mapperfactorybean.html)  
[MyBatis-Spring源码之MapperScannerConfigurer](/blog/20170817/mybatis-spring-mapperscannerconfigurer.html)  
[MyBatis-Spring源码之基于xml的自动扫描](/blog/20170817/mybatis-spring-scan.html)  

- Spring Schema 扩展  

通过自动扫描方式使用`Mybatis Spring`，需要在Spring XML配置文件中配置`mybatis:scan`扫描接口类，`mybatis`其实是对Spring XML标准的Schema的扩展  

Spring Schema扩展方法介绍详见  [Spring Schema 扩展](/blog/20170818/spring-schema.html)  

