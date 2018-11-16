---
layout: post
title: "RxJava并发"
description: Rxjava是一个响应式编程框架，支持数据发送和接收的异步化处理，也能满足并发执行的逻辑。
date: 2018-05-24 14:37:55 +0800
catalog: true
categories:
- Rxjava
tags:
- Rxjava
- Concurrency
---

RxJava 在 [GitHub 主页](https://github.com/ReactiveX/RxJava/wiki)上的介绍是:  
 "a library for composing asynchronous and event-based programs using observable sequences for the Java VM"  
一个在 Java VM 上使用可观测的序列来组成异步的、基于事件的程序的库  

本篇主要探讨RxJava实现异步并发，对于RxJava的基础只做简要的概述  

# 1 Rxjava简介

## 1.1 基本概念

Rxjava是一个响应式编程的框架，包含 `Observeable` 和 `Observer` 两个核心概念。  

- `Observeable` ： 被观察者  
- `Observer`： 观察者  

观察者`Observer`订阅被观察者`Observeable`是通过调用`subscribe`方法实现的，即`Observeable.subscribe(Observer)`  

> 看到`Observeable.subscribe(Observer)`这个方法调用，可能会比较疑惑，被观察者定于观察者？  
> 其实这么设计是为了链式调用API设计的方便  

被观察者`Observeable`通过`onNext`方法向观察者`Observer`发送数据。

> `onNext`方法并不是`Observeable`，而是在创建`Observeable`对象时候，创建的数据发射对象`ObservableEmitter`的方法  


## 1.2 线程切换、异步执行

**默认情况下**，被观察者`Observeable`发送数据和观察者`Observer`处理数据是在**同一个线程**执行的。  

同时Rxjava也支持异步发送数据和处理数据，但是**异步执行并不意味着并发执行**。 

- `Observeable.subscribeOn`： 切换被观察者`Observeable`发送数据的执行线程  
- `Observeable.observerOn`：切换观察者`Observer`处理数据的执行线程  


# 2 Rxjava并发

虽然线程切换能实现异步执行，但是很多场景下，我们希望的是**异步并发**执行。  

显然，异步并发执行涉及到两部分：被观察者`Observeable`发送数据和观察者`Observer`处理数据两部分。  

首先我们了解下Rxjava的设计理念： `onNext`(即数据发送)需要被按照特定的顺序**依次执行**，即使在多线程执行的情况下,`onNext`也不能同时被调用。  

```
A common question about RxJava is how to achieve parallelization, or emitting multiple items concurrently from an Observable.
Of course, this definition breaks the Observable Contract which states that onNext() must be called sequentially and never concurrently by more than one thread at a time.
```

## 2.1 被观察者`Observeable`异步并发

首先分析下被观察者`Observeable`异步并发执行，其实是指我们能够实现多线程并发的准备数据，而并不是要多线程并发执行`onNext`方法。  

那么现在问题就转换成，如何实现被观察者`Observeable`异步并发执行准备数据逻辑，方案如下：  

在源`Observeable`上执行`flatMap`，然后在`flatMap`中创建多个`Observeable`，多个`Observeable`通过`subscribeOn`切换线程，并发执行。  

其实就是，由于单个`Observeable`能实现异步执行，不能实现并发执行，我们寻找一种方式来创建多个`Observeable`，每个`Observeable`占用一个线程执行，来达到**整体并发执行**的效果。  

```
So how do we make calculations happen on more than one computation thread? And do it without breaking the Observable contract?
The secret is to catch each Integer in a flatMap(), create an Observable off it, do a subscribeOn() to the computation scheduler, and then perform the process all within the flatMap().
But how is this not breaking the Observable contract you ask?
Remember that you cannot have concurrent onNext() calls on the same Observable.
We have created an independent Observable off each integer value and scheduled them on separate computational threads, making their concurrency legitimate.
```

那么问题来了，通过多个`Observeable`实现**整体并发**，如何保证`onNext`在多线程情况下不会被并发执行呢？  

其实就是当线程A想执行`onNext`，若发现线程B正在执行`onNext`，此时线程A会将自己要发送的数据转移给B，从而消除了二者的并发执行`onNext`的问题。  

> 这里隐含了一个问题：单个线程`Observeable`发送数据（执行`onNext`）是按照**特性的顺序**的，但是多线程`Observeable`发送数据的**顺序是不能保证**的。  

```
Now you may also be asking "Well... why is the Subscriber receiving emissions from multiple threads then? That sounds an awful lot like concurrent onNext() calls are happening and that breaks the contract."
Actually, there are no concurrent onNext() calls happening. The flatMap() has to merge emissions from multiple Observables happening on multiple threads, but it cannot allow concurrent onNext() calls to happen down the chain including the Subscriber. It will not block and synchronize either because that would undermine the benefits of RxJava. Instead of blocking, it will re-use the thread currently emitting something out of the flatMap(). If other threads want to emit items while another thread is emitting out the flatMap(),   the other threads will leave their emissions for the occupying thread to take ownership of.
```

## 2.2 观察者`Observer`异步并发

前面有提到`onNext`即使是在多线程并发执的情况下，也是依次执行的，也就是不会有并发的情况。  


所以，`Observer`并不会并发收到数据来处理，`Observer`的并发执行其实是没有意义的，虽然也能实现异步执行的效果，但是和异步并发的效果其实是不一致的。  

> 后面一篇文章中会给出一个例子，可以将`Observer`的并发处理数据提升到`Observeable`中  

---

**本文参考自下面一篇文章，英文部分是从下面这篇文章中摘录出来的**  
[http://tomstechnicalblog.blogspot.hk/2015/11/rxjava-achieving-parallelization.html]()