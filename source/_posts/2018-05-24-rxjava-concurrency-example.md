---
layout: post
title: "RxJava并发示例"
description: Rxjava是一个响应式编程框架，支持数据发送和接收的异步化处理，也能满足并发执行的逻辑。
date: 2018-05-24 17:15:53 +0800
catalog: true
categories:
- Rxjava
tags:
- Rxjava
- Concurrency
---

上篇文章[Rxjava并发](/blog/20180524/rxjava-concurrency.html)有介绍Rxjava中实现并发的方案，本篇文章主要借一个实际的场景，来实践下并发执行的方案。  

**场景如下**：  

现需要根据一个**商家列表**获取各个商家下面的商品，然后根据每个商品，获取商品的价格  

包含如下两个接口：  

- 获取商品接口 ： 入参是商家ID，出参是商品列表  
- 获取商品价格接口 ： 入参是商品ID，出参是价格  

# 1 方案描述

## 1.1 期望效果

期望获取商品和获取价格都能并发执行  

即多个商家并发获取商品，获取后的商品列表能并发获取价格，二者不相互影响，即先查询到的商品可以先去获取价格，而不必等待所有商家商品都获取成功后执行  

## 1.2 方案调研

首先，不是很想用`java.util.concurrent`包下面的那套submit task、future的方案，因为需要自己控制任务的提交和结果的获取  

其次，想用一些现成的框架来实现。  

正好在了解`Hystrix`框架的时候有了解到`Rxjava`这个框架，且也比较喜欢JDK8 Lambda那种链式调用API  

So，开始使用Rxjava来作为方案写Demo研究实现方式  

废话好多，下面上代码...

# 2 实现方案

环境信息：  

- JDK8  
- RxJava 2.1.12  
- Spring 4.3.7.RELEASE  

## 2.1 实现并发

实现并发的方案在上篇文章[Rxjava并发](/blog/20180524/rxjava-concurrency.html)中有说过：  

通过在源`Observable`上调用`flatMap`来实现并发准备数据，在当前场景中源`Observable`为商家列表，调用`flatMap`来实现并发获取商品  

> 那么在这个场景中遇到一个问题：如何并发获取价格？  
> 一开始想的是在观察者`Observer`中并发获取价格，即`Observable`并发获取商品，`Observer`并发获取价格。  
> 
> 但是在写Demo中发现，并不能并发获取价格，因为商品获取的是list,价格的获取只能是商家维度的并发，并不能达到我想要的整体商品并发获取价格  
> 
> 所以最后把获取价格放到`Observable`中，其实就是双重并发  

核心代码如下：  

```java
public void doSync() {
	try{
		List<String> venderList = getVenderList();
		List<Product> productList = new ArrayList<>();
		CountDownLatch venderLatch = new CountDownLatch(venderList.size());
		ConcurrentHashMap<String, AtomicInteger> venderMap = new ConcurrentHashMap<>();
		Observable.fromArray(venderList.toArray(new String[venderList.size()]))
				.flatMap(s ->
					Observable.just(s).subscribeOn(Schedulers.from(venderPool)).flatMap(ss -> {
						System.out.println(String.format("emit %s in [%s] ", ss, Thread.currentThread().getName()));
						List<Product> venderProductList = getProductListByVender(ss);
						prepareCountLatch(venderMap, venderLatch, ss, venderProductList.size());
						return Observable.fromArray(venderProductList.toArray(new Product[venderProductList.size()]));
					})
				)
				.flatMap(s ->
					Observable.just(s).subscribeOn(Schedulers.from(productPool)).map(ss -> {
						System.out.println(String.format("emit2 %s in [%s] ", ss.getName(), Thread.currentThread().getName()));
						getProductPrice(ss);
						return ss;
					})
				)
				.observeOn(Schedulers.from(subPool))
				.subscribe(p -> {
					System.out.println(String.format("sub %s in [%s] ", p.getName(), Thread.currentThread().getName()));
					productList.add(p);
					if (0 == venderMap.get(p.getVender()).decrementAndGet()){
						venderLatch.countDown();
					}
					System.out.println(String.format("sub %s out [%s] ", p.getName(), Thread.currentThread().getName()));
				}, error -> {
					System.out.println("Exception sub " + Thread.currentThread().getName());
					System.out.println("Exception sub " + error.getMessage());
					releaseLatch(venderLatch, productList);
				});
		venderLatch.await();

		System.out.println("===================" + Thread.currentThread().getName());
		System.out.println(JSON.toJSON(productList));
	} catch (Exception e){
		System.out.println("Exception main " + Thread.currentThread().getName());
		System.out.println("Exception main " + e.getMessage());
		throw new RuntimeException(e);
	}
}
```

- 源`Observable`实现代理商列表的数据发送  
- 第一个`flatMap`实现并发获取商品  
- 第二个`flatMap`实现并发获取价格  
- `Observer`实现聚合所有的商品

下面是获取商家、商品和价格的辅助代码  

> `netWorkCost`方法是为了模拟真实场景中调用外部服务的网络损耗  

```java
private List<String> getVenderList() {
   List<String> venderList = new ArrayList<>();
   for (int i = 1; i < 15; i++){
	   venderList.add("v" + i);
   }
   return venderList;
}

private void getProductPrice(Product p) {
	Random random = new Random();
	p.setPrice((Long.parseLong(p.getVender().substring(1)) * Long.parseLong(p.getName().replace(p.getVender(),"").substring(3))));
//		p.setPrice((long)(random.nextInt(5000)));

//		if (p.getVender().equals("v2")){
//			throw new RuntimeException("error price");
//		}
	netWorkCost(random.nextInt(1000));
}

private List<Product> getProductListByVender(String vender) {
	System.out.println(String.format("call %s in [%s] ", vender, Thread.currentThread().getName()));
	List<Product> venderProductList = new ArrayList<>();
	Random random = new Random();
//		netWorkCost(random.nextInt(5000));
//		netWorkCost(2000);

	for (int i = 1; i < 130; i++){
		Product p = new Product();
		p.setVender(vender);
		p.setName(vender + "--" + "p" + i);
		venderProductList.add(p);
	}

//		if (vender.equals("v2") ){
//			throw new RuntimeException("error product");
//		}
	return venderProductList;
}

private void netWorkCost(long time) {
	try {
		Thread.sleep(time);
	} catch (Exception e){
		throw new RuntimeException(e);
	}
}
```

## 2.2 完善方案

完成并发貌似就实现了上面的场景，但是作为一个要上生产的程序，显然考虑的有点少  

下面说明下针对完成判断、异步线程池和异常处理三个方面来完善实现方案  

### 2.2.1 判断处理完成

由于所有的处理都是异步并发处理的，异步并发是提高效率的手段，结果显然是所有商家的商品，且包含价格  

也就是在当前当前线程如何判断所有的商家的商品都获取完成价格，处理流程描述如下：   

- 根据商家数据创建`CountDownLatch`，并在当前线程中`CountDownLatch.await`，等待所有商家获取完成  
- 当一个商家获取完商品后，写入`map`，`key`为商家ID，`value`为商品数量  
	（这里隐藏一个逻辑，这个商家确实没商品，则不写入`map`，直接`CountDownLatch.countDown`数量减1）  
- 当商家商品获取价格成功后，将`map`中对应的`value`减去1  
- 当`map`中`key`对应的`value`为0，表示该商家获取商品和价格完成，`CountDownLatch.countDown`数量减1  
- `CountDownLatch` 数量为0时，`CountDownLatch.await`等待结束，获取所有商家商品及价格完成  

处理代码如下：  

```java
private void prepareCountLatch(ConcurrentHashMap<String, AtomicInteger> venderMap, CountDownLatch venderLatch, String ss, int size) {
	if (size == 0){
		venderLatch.countDown();
		return;
	}
	venderMap.putIfAbsent(ss, new AtomicInteger(size));
}
private void releaseLatch(CountDownLatch venderLatch, List<Product> productList) {
	while (venderLatch.getCount() > 0){
		venderLatch.countDown();
	}
	productList.clear();
}
```

### 2.2.2 异步线程池

- 获取商品和价格分别为两个固定大小的线程池  
- 需要特别说明下`Observer`的线程池也是个固定大小的线程池，但是线程数是1  
	首先处理逻辑很简单  
	其次重点是防止多线程情况下商家处理完成数量减1操作`venderMap.get(p.getVender()).decrementAndGet()`的并发问题  

线程初始化和销毁主要通过类的初始化和销毁回调方法来执行，代码如下：  

```java
private ExecutorService venderPool;
private ExecutorService productPool;
private ExecutorService subPool;
@PostConstruct
public void init(){
	System.out.println("init------------------------");
	venderPool = Executors.newFixedThreadPool(3, new ThreadFactoryBuilder().setNameFormat("venderPool-%d").build());
	productPool = Executors.newFixedThreadPool(8, new ThreadFactoryBuilder().setNameFormat("productPool-%d").build());
	subPool = Executors.newFixedThreadPool(1, new ThreadFactoryBuilder().setNameFormat("subPool-%d").build());
}
@PreDestroy
public void clean(){
	System.out.println("clean------------------------");
	productPool.shutdown();
	venderPool.shutdown();
	subPool.shutdown();
}
```


### 2.2.3 异常处理

获取商品、价格等这种外部操作，必然需要考虑获取失败的情况，异常的处理就是Rxjava的异常处理  

> 这个例子中，重点是异常处理中，需要释放`CountDownLatch`,不然当前线程会一直等待  





