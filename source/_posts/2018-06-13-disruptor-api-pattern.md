---
layout: post
title: "Disruptor核心API和应用模式"
description: Disruptor类核心API的使用方式以及Disruptor应用的一些模式
date: 2018-06-13 20:30:22 +0800
catalog: true
categories:
- Disruptor
tags:
- Disruptor
- Ringbuffer
---

上篇文章[Disruptor框架简介](/blog/20180613/disruptor-introduction.html)主要介绍了Disruptor框架中的一些概念及实现方式  

本篇文章主要针对`Disruptor`这个类的一些核心API进行说明，包括以下几个方面：  

- 创建  
- 事件处理  
- 启动  
- 事件发送  
- 异常处理  
- 关闭  
 

同时本篇文章也会介绍`Disruptor`框架在实际应用中的一些模式，包括以下几个方面：  

- 生产者端：支持单生产者和多生产者  
- 消费者端：支持多线程并发消费、事件广播  
- 消费者依赖关系构造  

**注：Disruptor版本3.4.2**

<!-- more -->

# 1 核心API

## 1.1 创建

`Disruptor`的创建是通过构造函数来实例化`Disruptor`对象的，构造函数如下：  

```java Disruptor构造函数
public Disruptor(
        final EventFactory<T> eventFactory,
        final int ringBufferSize,
        final ThreadFactory threadFactory,
        final ProducerType producerType,
        final WaitStrategy waitStrategy)
{
    this(
        RingBuffer.create(producerType, eventFactory, ringBufferSize, waitStrategy),
        new BasicExecutor(threadFactory));
}
```

上面的构造函数会初始化`Disruptor`对象包含的两个核心对象：  

- `RingBuffer`：创建`RingBuffer`对象需要指定4个基本属性  
	- `producerType`：生产者类型（单生产者还是多生产者，通过`ProducerType`枚举指定，不同的类型，生产者对应的`Sequencer`不同）  
	- `eventFactory`：生成事件对象的工厂类  
	- `ringBufferSize`：RingBuffer底层数组的大小  
	- `waitStrategy`：消费者等待策略
- `BasicExecutor` ： 通过线程创建工厂`threadFactory`构造线程池，用于执行消费者事件处理逻辑  

## 1.2 事件处理

消费者针对事件的处理，主要有以下两个核心方法：  

- `handleEventsWith(EventProcessor[])`  
	入参是一个EventProcessor数组  

	该方法会将**每个**EventProcessor封装成EventProcessorInfo，存储到Disruptor的消费者事件处理集合ConsumerRepository中  

	注意：事件会被EventProcessor数组中的每一个处理器处理  

- `handleEventsWith(EventHandler[])`  
	入参是一个EventHandler数组  

	该方法会将**每个**EventHandler封装成EventProcessorInfo，存储到Disruptor的消费者事件处理集合ConsumerRepository中  
	与上面的方法不同的是：**这个方法通过EventHandler构造成BatchEventProcessor，从而支持消费者端的批处理**

	注意：事件会被EventHandler数组中的每一个处理器处理

- `handleEventsWithWorkerPool(WorkHandler[])`  
	入参是一个WorkHandler数组  

	该方法会将每一个WorkHandler封装成一个WorkProcessor，然后将所有的WorkProcessor封装到一个WorkerPool里面  
	最后将这个WorkerPool封装成WorkerPoolInfo，存储到Disruptor的消费者事件处理集合ConsumerRepository中  

	注意：事件会被WorkHandler数组中的**一个**WorkHandler处理  

*总结：*  

- 前两个方法适用于一个事件多个不同的逻辑处理器并行处理  
- 第三个方法使用于一个事件只需要被一个事件处理器抢占处理  

> 可类比MQ的消费模式  
> 方法处理逻辑可参照源码  

## 1.3 启动

在上述事件处理中只是将处理器封装成ConsumerRepository存储到Disruptor中，而启动方法才是真正的调度线程，执行这个逻辑处理器  

```java Disruptor启动方法
public RingBuffer<T> start()
{
    checkOnlyStartedOnce();
    for (final ConsumerInfo consumerInfo : consumerRepository)
    {
        consumerInfo.start(executor);
    }

    return ringBuffer;
}
```

从源码可看出`Disruptor`启动，会通过线程池异步执行所有的`ConsumerInfo`，就会启动`EventProcessor`执行线程（`EventProcessor`继承自`Runnable`）  

我们可以通过`EventProcessor`的子类`BatchEventProcessor`和`WorkProcessor`的`run`方法可以看出  

异步线程启动后的处理逻辑就是循环获取`RingBuffer`的可处理序列，若可获取到序列，则处理事件；否则根据等待策略等待。  

## 1.4 事件发送

生产者发送事件主要是调用`publishEvent(EventTranslator)`实现的，该方法封装了`RingBuffer`节点申请、事件初始化和`RingBuffer`序列提交  

`EventTranslator`支持无参`EventTranslator`、一个参数`EventTranslatorOneArg`、两个参数`EventTranslatorTwoArg`、三个参数`EventTranslatorThreeArg`的多种版本  

`EventTranslator`只处理事件对象的加工，而事件对象的创建（创建`Disruptor`对象时候提供的事件工厂类）和获取由`Disruptor`框架封装  

可参照官方[Getting Start -- using-version-3-translators](https://github.com/LMAX-Exchange/disruptor/wiki/Getting-Started#using-version-3-translators)  

## 1.5 异常处理

在`Disruptor`框架中，消费者异步线程处理事件属于异常高发地段，那么Disruptor框架是如何处理和传递异常的呢？消费者事件处理异常是否回造成序列的消费失败，从而卡死RingBuffer环呢？  

**异常处理接口声明ExceptionHandler**  

下面来了解下`Disruptor`处理异常的方式。  

在`Disruptor`中声明了异常处理的接口`ExceptionHandler`,包含三个方法：  

- `handleEventException`：事件消费异常处理（重点关注）  
- `handleOnStartException`: 启动异常处理  
- `handleOnShutdownException` : 关闭异常处理  

**默认的异常处理类FatalExceptionHandler**  

我们再回过头开看看`Disruptor`这个类：  

- `Disruptor`里面包含了一个`ExceptionHandlerWrapper`类型的`exceptionHandler`对象，这是`Disruptor`异常处理的一个代理类   
- 真正的异常处理是由`ExceptionHandlerWrapper`这个代理类的目标对象`delegate`来实现的，其默认值为`FatalExceptionHandler`。  

至此我们知道*`Disruptor`的异常默认是由`FatalExceptionHandler`类实现的*。  

看下`FatalExceptionHandler`的`handleEventException`方法，消费者处理事件异常，异常处理器会抛出运行时异常。


**异常处理的工作机制**  

从上面可知`Disruptor`对象创建的时候就初始化了异常处理代理类`ExceptionHandlerWrapper`。  

下面我们看下，在消费者处理事件的时候，异常处理类是如何生效的  

从`1.2 事件处理`和`1.3 启动`两部分可知，事件处理主要是封装`EventProcessor`，而启动时候是将`EventProcessor`放在线程池中异步执行  

那么还是回到`EventProcessor`线程执行的`run`方法

> 由于EventProcessor继承自Runnable，所以执行时候就看run方法  

下面以`EventProcessor`的子类`BatchEventProcessor`的`run`方法为例来看看异常处理类的作用  

`BatchEventProcessor`的`run`方法调用`processEvents`方法处理事件  

```java  BatchEventProcessor异常处理
private void processEvents()
{
    T event = null;
    long nextSequence = sequence.get() + 1L;

    while (true)
    {
        try
        {
            final long availableSequence = sequenceBarrier.waitFor(nextSequence);
            if (batchStartAware != null)
            {
                batchStartAware.onBatchStart(availableSequence - nextSequence + 1);
            }

            while (nextSequence <= availableSequence)
            {
                event = dataProvider.get(nextSequence);
                eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence);
                nextSequence++;
            }

            sequence.set(availableSequence);
        }
        catch (final TimeoutException e)
        {
            notifyTimeout(sequence.get());
        }
        catch (final AlertException ex)
        {
            if (running.get() != RUNNING)
            {
                break;
            }
        }
        catch (final Throwable ex)
        {
            exceptionHandler.handleEventException(ex, nextSequence, event);
            sequence.set(nextSequence);
            nextSequence++;
        }
    }
}
```

从上面源码可看出，在消费者处理事件的时候，会捕获异常，并交给`exceptionHandler`处理  

> 从源码也能看出，默认的异常处理类`FatalExceptionHandler`在事件消费异常时候，会抛出运行时异常，那么当前`sequence`并不能被正常消费，这可能会造成`RingBuffer`环的卡死（无法消费，那么生产者也无法发送事件）  

**自定义处理异常**  

自定义异常其实就是实现`ExceptionHandler`接口，然后通过`Disruptor`的`setDefaultExceptionHandler`来设置到异常处理代理类`ExceptionHandlerWrapper`中，从而替换掉默认异常处理类  

自定义异常并实现消费异常处理方法`handleEventException`可以根据实际应用场景来处理（打印异常、设置标志位等等），避免`RingBuffer`卡死的情形  

> 自定义异常是为了让使用者掌握主动权，根据自己的实际场景来处理异常，在避免潜在问题的前提下，满足业务场景  

`Disruptor`中提供了三个方法来设置自定义的异常处理类:  

- `setDefaultExceptionHandler`：对所有的`EventHandler`和`WorkerPool`都有效  
- `handleExceptionsFor`：对指定的`EventHandler`设置异常处理类  
- `handleExceptionsWith`: 对该方法被调用之后设置的`EventHandler`有效（这种方式使用起来容易出错，已废弃）  
	可以参照[Disruptor issue](https://github.com/LMAX-Exchange/disruptor/issues/122)  

## 1.6 关闭

`Disruptor`提供了关闭方法`shutdown`，该方法会等待所有的事件都被处理完成之后，再停止所有的事件处理器`EventProcessor.halt`  

注意：  

- 该方法不会关闭消费者执行线程池也不会等待消费者执行线程完毕  
- 该方法被调用的时候，所有的事件发送需要都已经完成  


# 2 应用模式

在开篇提到，`Disruptor`框架在实际应用中有多种模式，包括以下几个方面：  

- 生产者端：支持单生产者和多生产者  
- 消费者端：支持事件广播、多线程并发消费  
- 消费者依赖关系构造  

**生产者端**  

对于生产者来说，`Disruptor`框架支持单生产者也支持多生产者，其不同支持就是在创建`Disruptor`对象的时候，指定不同的`ProducerType`  

**消费者端**  

消费者端主要说明下事件广播和多线程并发消费  

- 事件广播  
	
	*事件广播*其实就是针对一个事件，会有多个处理器来处理，多个处理器完成不同的功能，且互不影响  
	
	从`1.2 事件处理`中可知，这种情形调用`handleEventsWith(EventHandler[])`即可实现  

- 多线程并发消费  

	*多线程并发消费*是指针对一个事件，事件会被提交到多个处理器组成的线程池执行，这个事件只会被一个线程执行。  
	
	从`1.2 事件处理`中可知，这种情形调用`handleEventsWithWorkerPool(WorkHandler[])`即可实现  
	
	> 对于事件处理的一个特定逻辑，我们创建多个`WorkHandler`实例，即多个`WorkHandler`组成线程池，但是逻辑一致，从而实现多线程并发消费  


**消费者依赖关系构造**  

消费者的典型场景就是菱形结构，如下图（图片来自[Dissecting the Disruptor: Wiring up the dependencies](http://mechanitis.blogspot.com/2011/07/dissecting-disruptor-wiring-up.html)）：  

![](/images/cust/disruptor/consumer-dependency.png)  

如上图：一个生产者P1，三个消费者C1、C2、C3  

- C1 和 C2 直接依赖`RingBuffer`,获取事件处理  
- C3也是直接依赖`RingBuffer`，但同时也直接依赖C1、C2先对数据处理完成，再从`RingBuffer`中获取事件处理  
- P1在发送事件的时候，依赖三个消费者的处理进度，即依赖C1、C2、C3三者的序列  
- C1、C2直接依赖生产者的进度来消费数据，即依赖`RingBuffer`的序列  
- C3不仅仅依赖`RingBuffer`的序列，同时还要依赖C1和C2的序列（例如：C1消费到了序列7， C2消费到了9，那么C3只能处理7之前的事件）  

下面是这个接口的一个示例代码：  

```java Disruptor菱形结构
public class ConsumerDependencyMain {
	public static void main(String[] args) {
		ThreadFactory productThreadFactory = generateThreadFactory("pool-");
		Disruptor<LongEvent> disruptor =
				new Disruptor<>(LongEvent::new, 4, productThreadFactory,
						ProducerType.SINGLE, new BlockingWaitStrategy());

		EventHandler<LongEvent> c1 = (event, sequence, endOfBatch) -> System.out.println(String.format("c1 handle sequence %s in %s", sequence, Thread.currentThread().getName()));
		EventHandler<LongEvent> c2 = (event, sequence, endOfBatch) -> System.out.println(String.format("c2 handle sequence %s in %s", sequence, Thread.currentThread().getName()));
		EventHandler<LongEvent> c3 = (event, sequence, endOfBatch) -> System.out.println(String.format("c3 handle sequence %s in %s", sequence, Thread.currentThread().getName()));
		disruptor.handleEventsWith(c1, c2).then(c3);

		disruptor.start();

		IntStream.range(0, 20).forEach( i ->
			disruptor.publishEvent((event, sequence) -> {
				event.setValue(sequence);
				System.out.println(String.format("p1 publish sequence %s in %s", sequence, Thread.currentThread().getName()));
			})
		);

		disruptor.shutdown();
	}

	private static ThreadFactory generateThreadFactory(String threadNamePrefix) {
		return new ThreadFactory() {
			private volatile AtomicInteger counter = new AtomicInteger(0);
			public Thread newThread(Runnable r) {
				return  new Thread(r,  threadNamePrefix + counter.getAndIncrement());
			}
		};
	}
}
```
 
其中自定义事件类`LongEvent`包含一个`value`字段  

> 在多消费者场景下，可能c3会依赖c1和c2的处理结果，而c1和c2可能并发处理同一个事件对象，此时可能会产生并发写的情况  
> 这种情况下，建议不同的消费者回写事件对象的不同字段来规避并发写的情况  





