---
layout: post
title: "Java自定义注解"
description: Java自定义注解及自定义注解的处理
date: 2016-12-27 10:35:55 +0800
catalog: true
categories:
- Java Annotation
tags:
- Java Annotation
---

通过上篇文章中对注解基础知识的说明，自定义注解其实就是用Java提供的**元注解**声明的一种注解类型。  

下面以一个定时任务注解的例子说明**自定义注解的声明、使用以及处理**。  

# 1. 自定义注解  

## 1.1 自定义注解的声明  

下面定义了一个定时任务的自定义注解`@Schedule`，包含三个元素`scheduleName` `cron` `desc`
> 自定义注解时候，若给元素设置有默认值，则使用时候可不指定其值（如下例中的`desc`元素）。  

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Documented
public @interface Schedule {
    /**
     * Schedule名称
     */
    String scheduleName();

    /**
     * Schedule执行表达式
     */
    String cron();

    /**
     * Schedule描述
     */
    String desc() default "";
}
```

## 1.2 自定义注解的使用  

下面定义了一个业务处理类`CustAnnotationsService`，包含一个`handler`方法，该方法需要被定时触发执行，使用上面自定义的注解`@Schedule`  
```java
public class CustAnnotationsService {

    @Schedule(scheduleName = "sc1", cron = "0 0/30 * * * ?")
    public void handler(){
        System.out.println("do something ...");
    }
}
```

## 1.3 自定义注解的处理  

在上篇文章中提到过，自定义注解对代码的本身的执行没有什么影响，那么要想通过注解实现特定逻辑，就需要对自定义注解进行处理。**自定义注解的处理，主要是通过反射 API实现的。**  

> 注解处理的反射API详见下篇

以下是读取注解的处理类示例（根据`@Schedule`自定义注解生成Quartz定时任务，下面不做扩展）  

```java
public class TestHandlerAnnotations {
    public static void main(String[] args) {
        Method[] methods = CustAnnotationsService.class.getDeclaredMethods();
        for (Method m : methods){
            Schedule[] scheduleArr = m.getAnnotationsByType(Schedule.class); //兼容注解和重复注解容器的获取
            for (Schedule schedule : scheduleArr){
                System.out.println(String.format("method name = %s; schedule : scheduleName = %s, cron = %s, desc = %s",
                        m.getName(), schedule.scheduleName(), schedule.cron(), schedule.desc()));
            }
        }
    }
}
```
输出结果如下：
```
method name = handler; schedule : scheduleName = sc1, cron = 0 0/30 * * * ?, desc = 
```

# 2 重复注解  

## 2.1 何为重复注解  

在某些场景下，需要在同一个程序元素上声明多个注解，对于一个注解声明多次，即是**重复注解**。`重复注解在Java SE 8才被引入。`  

## 2.2 重复注解容器  

重复注解容器是一个特殊的自定义注解，重复使用的自定义注解被存储在重复注解容器中，这个动作由编译器自动完成。重复注解容器声明的元注解使用与普通自定义注解一致，其包含一个`value`元素，`value`元素的类型为`自定义注解数组`。  

还是以上面的定时任务注解为例，若我们需要为定时执行的方法定义两种不同维度的执行触发条件，则可以在方法上声明多个自定义注解。这种情况下我们为自定义注解`@Schedule`声明一个重复注解容器为`@Schedules`，如下：  
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Documented
public @interface Schedules {
    Schedule[] value();
}
```

## 2.3 自定义注解声明可重复  

自定义的注解声明可重复是通过在自定义注解的声明上添加`@Repeatable`元注解（Java SE 8被引入），且值为重复注解容器类型

下面我们对例子中的自定义注解`@Schedule`进行改造，使其变为可重复注解。  
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Documented
@Repeatable(Schedules.class)
public @interface Schedule {
    /**
     * Schedule名称
     */
    String scheduleName();

    /**
     * Schedule执行表达式
     */
    String cron();

    /**
     * Schedule描述
     */
    String desc() default "";
}
```

## 2.4 重复注解的使用  

在业务处理类`CustAnnotationsService`中，新增一个`handler2`方法，该方法上使用了两个自定义的注解`@Schedule`  
```java
public class CustAnnotationsService {

    @Schedule(scheduleName = "sc1", cron = "0 0/30 * * * ?")
    public void handler(){
        System.out.println("do something ...");
    }

    @Schedule(scheduleName = "sc21", cron = "0 0 23 * * ?", desc = "每天23点执行一次")
    @Schedule(scheduleName = "sc22", cron = "0 0 1 * * ?", desc = "每天凌晨1点执行一次")
    public void handler2(){
        System.out.println("do something ...");
    }
}
```

## 2.5 重复注解处理  

上面例子中自定义注解处理类`TestHandlerAnnotations`在使用反射API处理自定义注解时候，使用的是兼容获取重复注解容器的API方法`getAnnotationsByType`。因此无需修改注解处理类`TestHandlerAnnotations`,再次执行，输出结果如下：  
```java
method name = handler; schedule : scheduleName = sc1, cron = 0 0/30 * * * ?, desc = 
method name = handler2; schedule : scheduleName = sc21, cron = 0 0 23 * * ?, desc = 每天23点执行一次
method name = handler2; schedule : scheduleName = sc22, cron = 0 0 1 * * ?, desc = 每天凌晨1点执行一次
```


----  
**本文翻译整理自java官网：**  
[Declaring an Annotation Type](https://docs.oracle.com/javase/tutorial/java/annotations/declaring.html)  
[Repeating Annotations](https://docs.oracle.com/javase/tutorial/java/annotations/repeating.html)