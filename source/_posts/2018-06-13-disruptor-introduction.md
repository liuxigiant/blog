---
layout: post
title: "Disruptor框架简介"
description: Disruptor是由LMAX公司开源的一个基于内存的在线程间传递数据的组件，以无锁化的方式实现数据传递的低延迟、高吞吐
date: 2018-06-13 15:01:34 +0800
catalog: true
categories:
- Disruptor
tags:
- Disruptor
- Ringbuffer
---

 Disruptor是由LMAX公司开源的一个基于内存的、在线程间传递数据的组件，以无锁化的方式实现数据传递的低延迟、高吞吐  

> 上面的事件可以是消息、事件等，下面以事件这个名词进行继续说明，即线程间的事件传递  

从上面的一句话总结可摘取出以下几个特点：  

- **基于内存**：即事件不持久化  
- **线程间事件传递**：即基于JVM的（并不是分布式）生成者/消费者模型  
- **无锁化**：多生产者/多消费者对临界资源的共享不以锁的方式实现  

# 1 核心概念

## 1.1 生产者/消费者模型

![](/images/cust/disruptor/concept.png)  

在了解Disruptor之前，可先看看官方给出的图例  

我们从**生产者/消费者模型**来看上图，即只需要关注`Producer` `*Consumer` `RingBuffer`，其余的都是Disruptor的实现细节  

- `Producer`: 事件生产者  
- `*Consumer`：事件消费者
- `RingBuffer`：事件存储结构（基于内存）  

## 1.2 核心概念

通过**生产者/消费者模型**对Disruptor有个大致的了解后，下面介绍下图例中Disruptor涉及的一些核心概念：  

- **Product** : 调用Disruptor发送事件的应用程序，在Disruptor框架中并没有对应的实体类  

- **Ring Buffer** : Disruptor框架的核心，事件存储的数据结构（其实底层就是个数组，下面会介绍），在框架中对应的实体类就是`RingBuffer`  

- **EventProcessor** : 事件处理器，即消费者。在框架中是一个接口，其实现类`BatchEventProcessor`支持批量消费。  

- **EventHandler** : 在框架中是一个接口，应用程序实现这个接口，完成事件的处理逻辑。所以`EventHandler`是真正的事件处理实现，而上面的`EventProcessor`其实是封装了事件的获取、调度执行和异常处理等。  

- **Event** : 事件，生产者和消费者直接传递的数据，在Disruptor框架中没有对应的实体类，由应用程序自己定义  

下面几个概念是Disruptor实现无锁化的核心：  

多生产者/多消费者并发往`RingBuffer`（数组）中写/读数据，其实是通过生成者/消费者各自维护了当前处理的数据的一个序列，通过各自维护的序列来实现无锁化的并发控制  

- **Sequence** : 上面提到的序列（对于生产者来说，序列的名称为cursor）。`Sequence`这个类的功能特性和`AtomicLong`类似。其最大不同是`Sequence`通过填充缓存行来优化伪共享问题  

- **Sequencer** : 生产者通过`Sequencer`来与`RingBuffer`交互，这个交互指控制生产者的序列`cursor`。其两个实现类`SingleProducerSequencer`和`MultiProducerSequencer`分别支持Disruptor框架的单生产者和多生产者模型  

- **Sequence Barrier:**  消费者`EventProcessor`通过`SequenceBarrier`与`RingBuffer`交互，同理，这个交互指控制消费者的序列  

- **Wait Strategy** : 等待策略，指定消费者在无事件可消费的情况下的等待策略  

# 2 RingBuffer

在简单了解了Disruptor框架的核心概念之后，下面来了解下Disruptor框架的核心`RingBuffer`  

## 2.1 特点

`RingBuffer`有以下一些特点：  

- 是一个首尾相连的环，可用在不同的上线文（线程）间传递数据  

- 底层是一个数组  

- 数组大小一般设置为**2的N次方**  
	这样生产者/消费者序列映射到数组的下标可通过位操作获取  sequence & （array length－1） = array index  
	
	> 这里有个关键点，即序列是从0开始逐渐递增的long型数值，而数组长度是固定的。  
	> 生产者在写入数组最后一个数据后，下次会从数组下标为0处开始写入。  
	> 这样即形成了RingBuffer的**环形覆盖**写入。  
	> 同时写入的时候会判断消费者是否已经处理完了被覆盖的数据，否则写入会阻塞等待 -- **覆盖判定**  


- RingBuffer拥有一个序号cursor，指向数组中写入并提交的最后一个数据，通过这个序列可获取下一个可写入的序列号  
	在Disruptor的2.x之后的版本中，生产者访问RingBuffer的Sequencer被整合到RingBuffer类中了，即RingBuffer拥有序列cursor。  
	且多个生产者是共享一个序列cursor的（而多消费者有自己的序列），这也是多生产这并发控制的核心  

## 2.2 读

消费者（事件处理器EventProcessor）是一个从RingBuffer里读取数据的线程，消费者维护了自己当前处理的事件的序列，然后通过 sequenceBarrier 获取RingBuffer当前可处理的数据的序列。  

若RingBuffer中可处理数据的序列大于当前消费者维护的序列，则获取事件处理；否则，根据指定的消费者等待策略`WaitStrategy`等待生产者写入事件。  

> 关于消费者依赖问题，也是通过序列来控制的。  
> 例如：消费者A和消费者B都从RingBuffer中消费数据，若A依赖于B，则A的序列必须小于等于B

下面看下BatchEventProcessor处理事件的核心方法`BatchEventProcessor#processEvents`的部分源码  

```java processEvents
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
```  

下面分析下上面的源码  

- 该方法的调用链可以从`Disruptor#handleEventsWith`方法和`Disruptor#start`方法分析，前者封装EventProcessor，后者启动Disruptor调度线程执行EventProcessor  
- `processEvents` 方法由`BatchEventProcessor`的`run`调用（EventProcessor实现Runable接口）  
- `nextSequence` 是当前消费者希望获取的事件的序列（当前序列加1 `sequence.get() + 1L`）  
- `sequenceBarrier.waitFor(nextSequence)`  获取`RingBuffer`可处理的最大序列，包含等待策略以及消费者依赖的处理  
- `dataProvider`即`RingBuffer`，负责提供事件数据  
- `BatchEventProcessor`的批处理体现在若获取的序列比当前序列大于1，则处理完事件后，不用再次获取可用序列  

## 2.3 写

生产者通过 `Sequencer` 来写入`RingBuffer`，通过**2PC-两阶段提交**（申请与提交）的方式，保证数据的不重叠  

### 2.3.1 申请

由 `Sequencer` 负责所有的交互细节来从 `RingBuffer` 中找到下一个节点，然后才允许生产者向它写入数据。  

下面看下`Sequencer`多生产者模式的实现类`MultiProducerSequencer`申请节点的方法`MultiProducerSequencer#next(int)`的部分源码  

```java next(int)
do
{
    current = cursor.get();
    next = current + n;

    long wrapPoint = next - bufferSize;
    long cachedGatingSequence = gatingSequenceCache.get();

    if (wrapPoint > cachedGatingSequence || cachedGatingSequence > current)
    {
        long gatingSequence = Util.getMinimumSequence(gatingSequences, current);

        if (wrapPoint > gatingSequence)
        {
            LockSupport.parkNanos(1); // TODO, should we spin based on the wait strategy?
            continue;
        }

        gatingSequenceCache.set(gatingSequence);
    }
    else if (cursor.compareAndSet(current, next))
    {
        break;
    }
}
while (true);
```

下面分析下上面的源码  

- 该方法的调用链可以从`Disruptor#publishEvent`方法开始分析  
- `next(n)`方法表示向RingBuffer申请n个事件，即支持事件的单条以及批量发送  
- `wrapPoint`等于申请的序列`next`减去数组大小`bufferSize`  
- `wrapPoint`与消费者序列对比`wrapPoint > cachedGatingSequence`是为了判断生产者申请的序列是否要覆盖消费者未处理的数据，若出现这种情况，则生产者需要**自旋等待**  
- `gatingSequences`维护了消费者的序列，`cachedGatingSequence`是缓存下来的最小消费者序列  


从上面分析可知：  

生产者通过`Sequencer`申请节点，且`Sequencer`维护了所有消费者的序列。  
若申请序列要覆盖未消费的数据，则生产者自旋。结束自旋后继续申请。  
申请校验成功后，则使用`CAS`设置`RingBuffer`序列，成功则退出；否则继续进行申请校验。  

### 2.3.2 提交

生产者完成了事件数据的处理后，提交事件，其实就是提交事件对应的序列，由`Sequencer`负责提交  

可参照`MultiProducerSequencer#publish`源码，主要完成以下两个动作  

- 设置当前序列可用  
- 唤醒等待的消费者  

### 2.3.1 多生产者写入

现在来回顾下，多生产者情况下，并发写入RingBuffer情况，其实就是考虑多生产者情况下的并发申请和并发提交的问题  

**并发申请**  

这是Disruptor框架中涉及到的一个并发问题，即并发申请下一节点  

由于多生产者都是向同一个`Sequencer`申请节点，所以并发的处理由`Sequencer`控制，即其实现类`MultiProducerSequencer`控制。  

从上面**2.3.1**章节源码可看出来，其实是由`cursor.compareAndSet`来实现CAS控制并发  

**并发提交**  

由于各个生产者申请节点后，对应的序列不相同，其实此处不存在并发问题  

> 网上有些老版本Disruptor的文章，说提交的时候有先后顺序。  
> 若生产者A申请的序列为1，生产者B申请的序列为2，则A需要先提交。  
> 但是Disruptor 3.4.2发现提交时候并没有这个限制，而是提交时候设置各自的序列为可用。  
> Disruptor 3.4.2是处理后面的序列先提交的问题，是通过消费者获取最大可用序列来控制的，即1不提交，2不能被消费  

## 2.4 特点

- 多生产者的写入，由一个`Sequencer`控制并发问题  

- 生产者、消费者之间的协调，由各自的序列来协调控制  

- 支持多生产者和多消费者，且无需加锁  

# 3 消费者等待策略

在`RingBuffer`中无可处理的事件时候，消费者有多种等待策略(busy wait策略，可能回造成CPU使用率过高)  

- **BlockingWaitStrategy**  
	
	阻塞策略 （默认的等待策略）  
	
	**实现方式**：  
	使用可重入锁 Lock 和 Condition 变量处理线程的阻塞和唤起   
	
	**优缺点**：  
	阻塞策略最慢，但是CPU使用率最低（不会一直占用CPU）  
	
	**适用场景**：  
	吞吐量不高（不需要一直占用CPU来提高事件处理效率）  


- **SleepingWaitStrategy**  

	休眠策略  
	
	和阻塞策略一样，休眠策略也相对来说对CPU使用率不是很高  
	
	**实现方式**：  
	前100次重试 -- 直接计数器递减  
	第101次到200次  -- 计数器递减且让出CPU  Thread.yield()  
	后续重试  --  休眠线程 LockSupport.parkNanos(sleepTimeNs)  
	
	**优缺点**：  
	对生产线程影响最小  -- 生产线程无需考虑增加计数器，也不需要通知Lock Condition  
	事件传递（生产者到事件处理器）事件更长  
	
	**适用场景**：  
	不要求低延迟、高吞吐，期望对生产者线程影响较小（异步日志）  

- **YieldingWaitStrategy**  

	两种用于低延迟、高吞吐场景的等待策略之一  
	
	**实现方式**：  
	前100次重试 -- 直接计数器递减  
	后续重试  -- 让出CPU  Thread.yield()  （隐含一个问题，让出CPU其实是可能会被CPU再次调度）  
	
	**优缺点**：  
	CPU占用率高  （Thread.yield让出CPU，其实是可能会被CPU再次调度）  
	提高性能和吞吐量  
	
	**适用场景**：  
	要求低延迟、高吞吐、高性能（事件处理器线程小于逻辑CPU核数）  


- **BusySpinWaitStrategy**
	
	性能最高的等待策略  
	
	**实现方式**：  
	ThreadHints.onSpinWait()  -- 调用java.lang.Thread.onSpinWait() 或者 什么也不做  
	当Thread.onSpinWait不可调用时候，等待策略相当于没有，一直while循环  
	
	**优缺点**：  
	CPU占用率高   
	提高性能和吞吐量  
	
	**适用场景**：  
	要求低延迟、高吞吐、高性能（事件处理器线程小于物理CPU核数）  

# 4 参考

本文部分内容翻译自官网：  

[Introduction](https://github.com/LMAX-Exchange/disruptor/wiki/Introduction)  
[Getting-Started](https://github.com/LMAX-Exchange/disruptor/wiki/Getting-Started)  


Disruptor一些翻译的文章链接(版本比较老)：  

[ifeve disruptor](http://ifeve.com/disruptor/)


文章中Disruptor源码版本 3.4.2



