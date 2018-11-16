---
layout: post
title: "Java注解基础"
description: Java注解基础
date: 2016-12-26 16:23:49 +0800
catalog: true
categories:
- Java Annotation
tags:
- Java Annotation
---

**Java 注解（Annotations）**是程序的一种**元数据**形式（可理解为程序的描述信息），而不是程序本身。注解对被注解的代码没有直接的影响。  

下面是一个使用注解的简单例子，MyClass类上有一个Author注解，Author注解包含两个元素，name和date  

> 若注解中只包含一个元素，且元素名称是value，则在使用注解的时候可以省略元素名称，直接声明元素值  

```java
@Author(
   name = "Benjamin Franklin",
   date = "3/27/2003"
)
class MyClass() { ... }
```

# 1 基础  

## 1.1 注解的用处  
注解在程序从编译到运行的各个时期都能提供不同的作用，主要包含以下几个方面：  

- 通知编译器：编译器可以通过注解做错误检查（后面提到的checker框架）和忽略告警（SuppressWarnings注解）  
- 编译期和部署期处理：通过注解生成代码、XML文件等  
- 运行期处理：程序运行期通过注解信息，执行特定逻辑处理（自定义注解与Spring结合的处理）  

## 1.2 注解的使用方式  
注解可被用在类、成员变量、成员方法等程序元素的声明上（类、成员变量、方法等声明）。  

## 1.3 类型注解 Type Annotation  

Java SE 8之前，注解仅能被使用在程序元素的声明上。Java SE 8后，注解可以被使用在类型（这里的类型指类的使用，而非类的声明）之前，这种注解称之为**类型注解**。例如：实例化对象new表达式、类型转换、实现接口语句和throws语句  

```java
- Class instance creation expression:
    new @Interned MyObject();
- Type cast:
    myString = (@NonNull String) str;
- implements clause:
    class UnmodifiableList<T> implements
    @Readonly List<@Readonly T> { ... }
- Thrown exception declaration:
    void monitorTemperature() throws
    @Critical TemperatureException { ... }
```

## 1.4 类型系统组件  Pluggable Type Systems

**类型注解**的引入是为了提高对Java程序的分析，以确保增强对代码的类型检查。Java SE 8没有提供类型检查框架，但是可以在编译器上集成类型检查插件。下面的例子使用了一个类型检查的注解，以及简要说了何为类型检查的插件。  

例如：在程序中，一个变量被声明，且在程序逻辑上不可能为`null`，为了避免程序运行过程中的`NullPointerException`，可以在变量前声明特定**类型检查注解**，并实现对注解的校验。此时对类型检查注解声明的元素的校验程序即为**类型检查插件**。声明方式如下：  
```java
@NonNull String str;
```
通过上面变量的注解和检查插件，在编译过程中，若编译器检测变量在某些地方可能为空，则会输出警告信息。通过排查警告并修复，就能避免运行时的`NullPointerException`。

通过集成各种不同类型的类型检查插件模块，可以避免更多运行时错误，从而构建更稳当的系统。  

> 华盛顿大学开发了第三方的类型检查框架，提供了`NonNull`模块，同时包含了常规表达式模块、互斥锁模块等，更多信息，详见 [Checker Framework](http://types.cs.washington.edu/checker-framework/)

# 2. 注解分类  

## 2.1 Java内置注解  

Java的内置注解包含：`@Deprecated`、 `@Override` 和 `@SuppressWarnings`  

**@Deprecated注解**  
`@Deprecated注解`表示被注解的元素是过时的，不应该再被使用。若程序元素被声明了`@Deprecated注解`,编译器会在使用了该程序元素的地方生成一条告警信息。当一个元素被声明为过时的，不仅仅需要加上`@Deprecated注解`,同时需要在Javadoc中使用`@deprecated标签`,如下：  
```java
	// Javadoc comment follows
    /**
     * @deprecated
     * explanation of why it was deprecated
     */
    @Deprecated
    static void deprecatedMethod() { }
```

**@Override注解**  
`@Override注解`用于告知编译器被注解元素是对父类的重写。`@Override注解`不是必须的，更多的是用于防止重写错误。若使用`@Override注解`注解，在重写错误的情况下，编译器会生成错误信息。  

**@SuppressWarnings注解**  
`@SuppressWarnings注解`用于告知编译器忽略特定类型的告警信息。下面的例子中，使用了一个过时的方法，编译器会生成告警信息，然而，在使用了`@SuppressWarnings注解`后，可以使编译器忽略这个过时告警  
```java
	// use a deprecated method and tell 
   // compiler not to generate a warning
   @SuppressWarnings("deprecation")
    void useDeprecatedMethod() {
        // deprecation warning
        // - suppressed
        objectOne.deprecatedMethod();
    }
```
每个编译器告警都归属于一种告警类型，Java包含的告警类型有`deprecation`和`unchecked`。如果要使编译器忽略多种类型的告警，可以使用如下语法：  
```java
@SuppressWarnings({"unchecked", "deprecation"})
```

## 2.2 元注解  

元注解是指被用于定义其他注解的注解（可理解为：元注解用于定义自定义注解）。元注解被定义在`java.lang.annotation`包下。  

**@Retention注解**  
`@Retention注解`指定了注解信息如何被存储，有以下三种选项：  

- RetentionPolicy.SOURCE 注解信息仅在源码级别保留，会被编译器忽略
- RetentionPolicy.CLASS 注解信息在编译期间被编译器保留，但是会被JVM忽略
- RetentionPolicy.RUNTIME 注解信息在运行时被JVM保留

**@Documented注解**  
`@Documented注解`用于声明注解信息是否需要被Javadoc工具生成的文档包含（默认情况下，Javadoc不包含注解信息）。  

**@Target注解**  
`@Target注解`指定自定义注解可以被用在哪些程序元素上，有以下几种选项：  

- ElementType.ANNOTATION_TYPE 可用于注解类型
- ElementType.CONSTRUCTOR 可用于构造器
- ElementType.FIELD 可用于成员变量
- ElementType.LOCAL_VARIABLE 可用于局部变量
- ElementType.METHOD 可用于方法上
- ElementType.PACKAGE 可用于包声明上
- ElementType.PARAMETER 可用于方法的参数上
- ElementType.TYPE 可用于类的任何元素上

**@Inherited注解**  
`@Inherited注解`用于指定注解是否可以从父类继承（默认是否）。当在类A上搜索指定的注解（该注解的`@Inherited`声明为`true`），若在类A上未搜到到注解，则会去父类中搜索。  

> `@Inherited注解`是被标注过的class的子类所继承。类并不从它所实现的接口继承annotation，方法并不从它所重载的方法继承annotation  

**@Repeatable注解**  
`@Repeatable注解`在Java SE 8被引入，用于指定注解是否可以声明多次。  


----  
**本文翻译整理自java官网：**  
[Lesson: Annotations](https://docs.oracle.com/javase/tutorial/java/annotations/index.html)  
[Annotations Basics](https://docs.oracle.com/javase/tutorial/java/annotations/basics.html)  
[Predefined Annotation Types](https://docs.oracle.com/javase/tutorial/java/annotations/predefined.html)  
[Type Annotations and Pluggable Type Systems](https://docs.oracle.com/javase/tutorial/java/annotations/type_annotations.html)  


