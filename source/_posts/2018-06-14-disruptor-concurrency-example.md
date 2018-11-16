---
layout: post
title: "Disruptor并发消费示例"
description: Disruptor框架处理并发消费的一个应用示例
date: 2018-06-14 16:24:33 +0800
catalog: true
categories:
- Disruptor
tags:
- Disruptor
- Ringbuffer
---

在[Rxjava并发示例](/blog/20180524/rxjava-concurrency-example.html)中以一个实际的场景，介绍了`RxJava`并发消费的实现方式，下面根据同样的场景，实现`Disruptor`的并发消费。

先来回顾下场景，**场景如下**：

现需要根据一个**商家列表**获取各个商家下面的商品，然后根据每个商品，获取商品的价格

包含如下两个接口：  

- 获取商品接口 ： 入参是商家ID，出参是商品列表
- 获取商品价格接口 ： 入参是商品ID，出参是价格

# 1 期望效果

期望效果和商品一致：期望获取商品和获取价格都能并发执行

即多个商家并发获取商品，获取后的商品列表能并发获取价格，二者不相互影响，即先查询到的商品可以先去获取价格，而不必等待所有商家商品都获取成功后执行

# 2 实现方案

环境信息：

- JDK8
- Disruptor 3.4.2

## 2.1 两个Disruptor

创建两个`Disruptor`

- `venderDisruptor`根据商家同步商品，获取的商品发布到第二个`Disruptor`中
- `productDisruptor`获取商品的价格

> 创建两个`Disruptor`的目的是让获取商品和获取价格两个任务都能并发执行  
> 而且两个`Disruptor`之间有依赖关系，我们可以通过调整两者的`RingBuffer`数组大小，来控制并发执行的节奏

代码如下：

```java Disruptor创建
public static void main(String[] args) throws Exception{
	SyncProduct syncProduct = new SyncProduct();
	syncProduct.doSync();
}

public void doSync() {
   try{
	   List<String> venderList = getVenderList();
	   List<Product> productList = new ArrayList<>();

	   //-------------product init & handle-----------------
	   ThreadFactory productThreadFactory = generateThreadFactory("productPool-");
	   Disruptor<ProductPriceEvent> productDisruptor =
			   new Disruptor<>(ProductPriceEvent::new, 16, productThreadFactory,
					   ProducerType.MULTI, new BlockingWaitStrategy());
	   productDisruptor.handleEventsWithWorkerPool(getProductWorkHandler())
			   			.then(((event, sequence, endOfBatch) -> {
			   				if (event.getProduct().getPrice() == null){
			   					return;
							}
							productList.add(event.getProduct());
							System.out.println(String.format("add product [%s] in [%s]", event.getProduct().getName(), Thread.currentThread().getName()));
			   			}));

	   productDisruptor.setDefaultExceptionHandler(getProductExceptionHandler());
	   productDisruptor.start();


	   //-------------vender init & handle-----------------
	   ThreadFactory venderThreadFactory = generateThreadFactory("venderPool-");
	   Disruptor<ProductEvent> venderDisruptor =
			   new Disruptor<>(ProductEvent::new, 4, venderThreadFactory,
					   			ProducerType.SINGLE, new BlockingWaitStrategy());
	   venderDisruptor.handleEventsWithWorkerPool(getVenderWorkHandler(productDisruptor));

	   venderDisruptor.setDefaultExceptionHandler(getVenderExceptionHandler());
	   venderDisruptor.start();

	   for(String v :  venderList){
		   venderDisruptor.publishEvent(((event, sequence) -> {
			   event.setVenderId(v);
			   System.out.println(String.format("publish vender [%s] in [%s]", v, Thread.currentThread().getName()));
		   }));
	   }

	   //wating...
	   while ((venderDisruptor.getRingBuffer().getMinimumGatingSequence() + 1) < venderList.size()){
		   Thread.sleep(100);
		   System.out.println(String.format("wait vender [%s]--[%s]------------", venderDisruptor.getRingBuffer().getMinimumGatingSequence() + 1, venderList.size()));
	   }

	   System.out.println("get product for vender done");
	   System.out.println(String.format("venderDisruptor  [%s]--[%s]------------", venderDisruptor.getRingBuffer().getMinimumGatingSequence() + 1, venderList.size()));


	   while (productDisruptor.getRingBuffer().getMinimumGatingSequence() < productDisruptor.getRingBuffer().getCursor()){
		   Thread.sleep(100);
		   System.out.println(String.format("wait product [%s]--[%s]------------", productDisruptor.getRingBuffer().getMinimumGatingSequence(), productDisruptor.getRingBuffer().getCursor()));
	   }

	   System.out.println("get price for product done");
	   System.out.println(String.format("productDisruptor  [%s]--[%s]------------", productDisruptor.getRingBuffer().getMinimumGatingSequence(),  productDisruptor.getRingBuffer().getCursor()));

	   System.out.println("===================" + Thread.currentThread().getName());

	   System.out.println(JSON.toJSON(productList));

	   venderDisruptor.shutdown();
	   productDisruptor.shutdown();
   } catch (Exception e){
	   System.out.println("Exception main " + Thread.currentThread().getName());
	   System.out.println("Exception main " + e.getMessage());
	   throw new RuntimeException(e);
   }
}
private ThreadFactory generateThreadFactory(String threadNamePrefix) {
	return new ThreadFactory() {
		private volatile AtomicInteger counter = new AtomicInteger(0);
		public Thread newThread(Runnable r) {
			return  new Thread(r,  threadNamePrefix + counter.getAndIncrement());
		}
	};
}
```

## 2.2 判断完成

`Disruptor`框架实现并发消费的场景在判断任务处理完成上有天然的优势，因为`RingBuffer`中的序列就是处理的任务数量，而无需我们自己记录任务总数

针对两个`Disruptor`，判断完成方案如下：

- `venderDisruptor`: 由于`venderDisruptor`的发送事件是在当前线程，所以执行完所有的事件发送之后，只需要判断消费者消费完成即可
- `productDisruptor`： 由于`productDisruptor`依赖于`venderDisruptor`，在`venderDisruptor`消费完成之后，说明`productDisruptor`发送事件已经完成，所以这步也只需要判断消费完成即可

> 可看出两个`Disruptor`的任务处理是否完成，都是通过`RingBuffer`序列和消费者序列之间的关系来判断的

## 2.3 并发消费

针对两个`Disruptor`，实现并发的方式都一样，都是将一个逻辑复制成多份（这里的分数决定了并发执行线程数），然后通过`handleEventsWithWorkerPool`并发执行

代码如下：

```java Disruptor并发消费
private static final int VENDER_THREAD_POOL_SIZE = 3;
private static final int PRODUCT_THREAD_POOL_SIZE = 8;
private WorkHandler<ProductEvent>[] getVenderWorkHandler(Disruptor<ProductPriceEvent> productDisruptor) {
	WorkHandler<ProductEvent>[] handlerArr = new WorkHandler[VENDER_THREAD_POOL_SIZE];
	IntStream.range(0, VENDER_THREAD_POOL_SIZE)
				.forEach(i -> handlerArr[i] = event -> handleVender(productDisruptor, event));
	return handlerArr;
}

private void handleVender(Disruptor<ProductPriceEvent> productDisruptor, ProductEvent event) {
	List<Product> productListForVender = getProductListByVender(event.getVenderId());
	productListForVender.forEach(p -> {
		productDisruptor.publishEvent(((event1, sequence1) -> {
			event1.setProduct(p);
			System.out.println(String.format("publish product [%s] in [%s]", p.getName(), Thread.currentThread().getName()));
		}));
	});
	System.out.println(String.format("handle vender [%s] in [%s]", event.getVenderId(), Thread.currentThread().getName()));
}

private WorkHandler<ProductPriceEvent>[] getProductWorkHandler() {
	WorkHandler<ProductPriceEvent>[] handlerArr = new WorkHandler[PRODUCT_THREAD_POOL_SIZE];
	IntStream.range(0, PRODUCT_THREAD_POOL_SIZE)
				.forEach(i -> handlerArr[i] = event -> handleProduct(event));
	return handlerArr;
}

private void handleProduct(ProductPriceEvent event) {
	Product p = event.getProduct();
	getProductPrice(p);
	System.out.println(String.format("handle product [%s] in [%s]", p.getName(), Thread.currentThread().getName()));
}

```

## 2.4 异常处理

针对两个`Disruptor`，异常处理都是通过自定义异常，而且两个处理消费异常的方式都是忽略当前异常，丢弃事件

> 这种异常处理方式是根据实际业务场景来觉得的，例如根据某一个商家获取商品，若失败，则不获取该商家商品，丢弃该商家；根据某个商品获取价格失败，则丢弃这个商品。

代码如下：

```java Disruptor异常处理
private ExceptionHandler<ProductEvent> getVenderExceptionHandler() {
	return new ExceptionHandler<ProductEvent>() {
		@Override
		public void handleEventException(Throwable ex, long sequence, ProductEvent event) {
			System.out.println(String.format("get product for vender error, vender is %s", event.getVenderId()));
		}

		@Override
		public void handleOnStartException(Throwable ex) {
			System.out.println("disruptor start exception : " + ex.getMessage());
		}

		@Override
		public void handleOnShutdownException(Throwable ex) {
			System.out.println("disruptor shout down exception : " + ex.getMessage());
		}
	};
}
private ExceptionHandler<ProductPriceEvent> getProductExceptionHandler() {
	return new ExceptionHandler<ProductPriceEvent>() {
		@Override
		public void handleEventException(Throwable ex, long sequence, ProductPriceEvent event) {
			System.out.println(String.format("get product price error, vender is %s, product is %s", event.getVenderId(), event.getProduct().getName()));
		}

		@Override
		public void handleOnStartException(Throwable ex) {
			System.out.println("disruptor start exception : " + ex.getMessage());
		}

		@Override
		public void handleOnShutdownException(Throwable ex) {
			System.out.println("disruptor shout down exception : " + ex.getMessage());
		}
	};
}
```

## 2.5 外部接口模拟

下面是获取商家、商品和价格的辅助代码

```java 外部接口模拟
private List<String> getVenderList() {
   List<String> venderList = new ArrayList<>();
   for (int i = 1; i < 15; i++){
	   venderList.add("v" + i);
   }
   return venderList;
}

private void getProductPrice(Product p) {
   Random random = new Random();

//	   if (p.getVender().equals("v2")){
//		   throw new RuntimeException("error price");
//	   }

   p.setPrice((Long.parseLong(p.getVender().substring(1)) * Long.parseLong(p.getName().replace(p.getVender(), "").substring(3))));
//		p.setPrice((long)(random.nextInt(5000)));



   netWorkCost(random.nextInt(1000));

}

private List<Product> getProductListByVender(String vender) {
   System.out.println(String.format("call %s in [%s] ", vender, Thread.currentThread().getName()));
   List<Product> venderProductList = new ArrayList<>();
   Random random = new Random();
//		netWorkCost(random.nextInt(5000));
//		netWorkCost(2000);

   for (int i = 1; i < 13; i++){
	   Product p = new Product();
	   p.setVender(vender);
	   p.setName(vender + "--" + "p" + i);
	   venderProductList.add(p);
   }

	if (vender.equals("v2") ){
		throw new RuntimeException("error product");
	}
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

> `netWorkCost`方法是为了模拟真实场景中调用外部服务的网络损耗
