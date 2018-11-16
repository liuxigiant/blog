---
layout: post
title: "Java注解处理之反射API"
description: Java注解处理反射API
date: 2016-12-27 15:41:23 +0800
catalog: true
categories:
- Java Annotation
tags:
- Java Annotation
---

## 1. AnnotatedElement接口简介
java.lang.reflect 包下主要包含一些实现反射功能的工具类，实际上，java.lang.reflect 包所有提供的反射API扩充了读取运行时Annotation信息的能力。当一个Annotation类型被定义为运行时的Annotation后，该注解才能是运行时可见，当class文件被装载时被保存在class文件中的Annotation才会被虚拟机读取。  

> AnnotatedElement 接口是所有程序元素（Class、Method和Constructor）的父接口，所以程序通过反射获取了某个类的AnnotatedElement对象之后，程序就可以调用该对象的方法来访问Annotation信息。  

## 2. API方法
- `boolean isAnnotationPresent(Class<?extends Annotation> annotationClass)`  
判断该程序元素上是否包含指定类型的注解，存在则返回true，否则返回false。注意：此方法会忽略注解对应的注解容器。  

- `<T extends Annotation> T getAnnotation(Class<T> annotationClass)` 
返回该程序元素上存在的、指定类型的注解，如果该类型注解不存在，则返回null。  

- `Annotation[] getAnnotations()`  
返回该程序元素上存在的所有注解，若没有注解，返回长度为0的数组。  

- `<T extends Annotation> T[] getAnnotationsByType(Class<T> annotationClass)`  
返回该程序元素上存在的、指定类型的注解数组。没有注解对应类型的注解时，返回长度为0的数组。该方法的调用者可以随意修改返回的数组，而不会对其他调用者返回的数组产生任何影响。**方法4 getAnnotationsByType方法与方法2 getAnnotation的区别在于，方法4 getAnnotationsByType会检测注解对应的重复注解容器。若程序元素为类，当前类上找不到注解，且该注解为可继承的，则会去父类上检测对应的注解。**  

- `<T extends Annotation> T getDeclaredAnnotation(Class<T> annotationClass)`  
返回直接存在于此元素上的所有注解。与此接口中的其他方法不同，该方法将**忽略继承的注释**。如果没有注释直接存在于此元素上，则返回null  

- `<T extends Annotation> T[] getDeclaredAnnotationsByType(Class<T> annotationClass)`  
返回直接存在于此元素上的所有注解。与此接口中的其他方法不同，该方法将忽略继承的注释  

- `Annotation[] getDeclaredAnnotations()`  
返回直接存在于此元素上的所有注解及注解对应的重复注解容器。与此接口中的其他方法不同，该方法将忽略继承的注解。如果没有注释直接存在于此元素上，则返回长度为零的一个数组。该方法的调用者可以随意修改返回的数组，而不会对其他调用者返回的数组产生任何影响。  

## 3. 总结
- AnnotatedElement代表运行时JVM中的一个被注解的元素  
- AnnotatedElement使元素上的注解可通过反射API读取、解析  
- AnnotatedElement接口中返回数组的方法，返回的数据可被调用者修改，而不会影响其他调用者返回的数组  
- AnnotatedElement接口中方法名称包含`Declared`的会忽略注解继承，方法名称包含`ByType`的会搜索注解类型对应的重复注解容器  
- AnnotatedElement接口中方法在检索注解的时候，会将注解分为直接声明（directly present）、间接声明（indirectly present）、声明（present）和相关联（associated ）四种类型。元素指注解使用的地方，包括类、方法、参数、成员变量等。  
	
	> **直接声明（directly present）**：  
	> 注解在元素上直接声明  
	> 
	> **间接声明（indirectly present）**：  
	> 重复注解在元素上直接声明  
	> 
	> **声明（present）**：  
	> 注解在元素上直接声明；  
	> 注解没有在元素上直接声明，而被注解元素为类，注解未可继承，注解在父类上声明  
	> 
	> **相关联（associated ）**：  
	> 包含直接声明和间接声明  
	> 注解没有在元素上直接声明或者间接声明，而被注解元素为类，注解未可继承，注解在父类上被关联  

- 下面总结下各个方法在检索注解时候与注解声明的关系（标对勾的为方法可以检索到的注解声明位置）  

| 方法 | 直接声明 | 间接声明 | 声明 | 相关联 |
| :-- | :--: | :--: | :--: | :--: |
|getAnnotation| - | - | √ | - |
|getAnnotations| - | - | √ | - |
|getAnnotationsByType| - | - | - | √ |
|getDeclaredAnnotation| √ | - | - | - |
|getDeclaredAnnotations| √ | - | - | - |
|getDeclaredAnnotationsByType| √ | √ | - | - |


----  
**本文翻译整理自AnnotatedElement Javadoc：**  
[Interface AnnotatedElement Javadoc](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/AnnotatedElement.html)