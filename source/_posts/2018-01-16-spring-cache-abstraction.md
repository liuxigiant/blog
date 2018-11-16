---
layout: post
title: "Spring Cache Abstraction简介"
description: Spring Cache Abstraction简介，包含注解的介绍及支持的底层缓存
date: 2018-01-16 14:55:28 +0800
catalog: true
categories:
- Spring Cache Abstraction
tags:
- Spring Cache
---

`Spring Cache Abstraction`是对通过注解的方式对缓存逻辑的抽象与封装，而不涉及底层缓存存储的读写（仅提供几种默认的底层缓存存储的读写集成）。  

**场景**：  
一个典型的场景是获取数据库数据的需求，在读取数据库之前，先读取缓存，缓存有数据，则直接返回；没有数据则读取数据库数据后，回写缓存。  `Spring Cache Abstraction`提供的`@Cacheable`即可完成上述逻辑，只需要在一个读取数据库数据的方法上添加`@Cacheable`注解即可  

那么怎么理解上面说的不涉及底层缓存存储的读写呢？  
`Spring Cache Abstraction`提供缓存逻辑的抽象封装，但是不管你底层是`Redis`还是java的`HashMap`,用户可以根据自己的需求，根据`Spring Cache Abstraction`规范，实现相应的接口，并完成配置，即可集成任意的缓存存储。  
同时，`Spring Cache Abstraction`也提供了默认的开箱即用的第三方缓存存储的实现，例如：Ehcache  

了解`Spring Cache Abstraction` ，重点需要了解 **缓存注解** 和 **底层缓存存储配置**  

> 本文翻译自[官网文档 `Cache Abstraction`](https://docs.spring.io/spring/docs/4.3.13.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#cache)章节(并不是完全直译)，目录组织结构整体也和官网一致

# 1 简介

`Spring`从 3.1 版开始支持缓存的操作，在 4.1 版后， 又对`Cache Abstraction`有极大的改善（包括支持JCache注解、提供更多的自定义支持、类级别配置注解`@CacheConfig`、异常处理等）。  

> `Cache` **VS** `Buffer`  
> 
> `Buffer`即缓冲区，通常用于一个较快的存储介质和一个较慢的存储介质之间进行数据交换。缓冲区是为了平衡双方的数据读写效率，临时存储数据。双方都可以从缓冲区中读写一次交换的所有数据，而不是按数据块的方式交换，这是缓存区的作用。  
> 
> `Cache`即缓存，也是用于存储数据，而缓存是通过提高多次读取数据的速度来提高数据读取的效率。  

## 1.1 `Cache Abstraction`理解

**`Cache Abstraction`到底是什么呢？**  

和Spring的其他服务一样，`Cache Abstraction`是对缓存的操作的一个抽象，而不涉及对底层缓存存储操作的实现。  

> `Cache Abstraction`提供了几种默认的缓存存储操作的实现（JDK的ConcurrentMap、Ehcache、 Guava Cache等），当然我们也可以自己集成我们所需的缓存存储，定制个人化缓存存储的实现  

我们可以将`Cache Abstraction`理解为一套通用的缓存集成解决方案，而`Cache Abstraction`在集成了第三方底层缓存存储后，以注解的方式提供对缓存的操作，从而将开发人员从缓存逻辑代码中解放出来，而只需要配置第三方存储即可。  

这就是开篇说的两个方面： **缓存注解** 和 **底层缓存存储配置**  

下面说下`Cache Abstraction`支持的缓存操作

## 1.2 `Cache Abstraction`缓存操作

`Cache Abstraction`提供以下几种能力：  

- 缓存的击穿回写 get-if-not-found-then- proceed-and-put-eventually  
	在JAVA方法上使用`Cache Abstraction`能够将方法的执行结果缓存下来，下次再执行方法的时候，即可直接返回结果，而无需执行方法  
	**注意：**这里的方法，对于相同的入参，必须返回相同的结果，不然缓存下来的数据也没什么用  

- 缓存的更新和清除  

> 这些缓存操作对应的具体的注解，下面会介绍  

## 1.3 `Cache Abstraction`的多线程、多进程场景

上面说到，`Cache Abstraction`是对缓存逻辑的抽象，且并没有对多线程、多进程场景做特殊处理。  

对于多进程的分布式缓存场景下，缓存数据更新后，多个缓存节点的数据同步问题，`Cache Abstraction`并没有处理，需要由分布式缓存自己处理。  

对于多线程场景下，击穿回写或者缓存更新，在多线程并发的时候，都有可能获取到脏数据，`Cache Abstraction`也没有对这种场景做处理。  

# 2 缓存注解

下面来了解下缓存注解，`Cache Abstraction`提供五种注解：  

- `@Cacheable` : 触发数据缓存
- `@CacheEvict` : 触发缓存数据清除
- `@CachePut`: 触犯缓存数据更新，而并不会影响方法的执行
- `@Caching` : 对多种缓存操作组合使用的支持
- `@CacheConfig` : 类级别的基础配置（相当于方法级别的缓存注解的公共配置）

## 2.1 `@Cacheable`注解

`@Cacheable`注解用在方法上，实现的功能就是典型的击穿回写`get-if-not-found-then- proceed-and-put-eventually`  

- 验证缓存是否存在数据，存在则直接返回，**不执行方法**  
- 不存在缓存数据，则执行方法，然后回写缓存  

每个缓存，需要提供一个缓存的名字，如下方法`findBook`就和名字是`books`的缓存关联起来了：  

```java
@Cacheable("books")
public Book findBook(ISBN isbn) {...}
```

一个方法也能关联多个缓存名字，如下：  

```java
@Cacheable({"books", "isbns"})
public Book findBook(ISBN isbn) {...}
```

当多个名称对应的缓存中，只要在方法执行前，校验缓存数据是否存在的时候，有一个名字对应的缓存被击中，则方法不执行，直接返回，而其他名称对应的缓存值不同，则会被更新  

> 可能上面的内容有人会疑惑，同一个方法上多个缓存名字，为什么击中缓存不是全部击中呢？  
> 
> 这里有个需要了解下，在`Cache Abstraction`，方法可以关联特定名字的缓存，而这个缓存，是包含key和value的，所以击中缓存，是相同的key才会击中，而上面例子中，没有指定key，其实是有默认的策略生成key  

### 2.1.1 默认缓存key生成策略

`Cache Abstraction`中缓存的存储结构是 `key-value` 结构的，在方法被调用时候，需要根据一定的策略，生成缓存对应的 `key`

默认的缓存`key`生成处理类是`org.springframework.cache.interceptor.SimpleKeyGenerator`，策略如下（具体可查看源码）：  

- 若没有入参，则返回`SimpleKey.EMPTY`
- 若只有一个入参，则返回当前入参
- 若有多个入参，则返回一个包含所有入参的`SimpleKey`实例  

这种默认策略满足大部分场景，只要方法的入参实现了`hashCode`和`equals`方法（若没有，则需要修改key生成策略）。  

> 在Spring 4.0版本之后，默认的key生成处理类才修改为`SimpleKeyGenerator`。
> 
> 以前版本的默认处理类是`DefaultKeyGenerator`，在多入参的场景，只考虑了入参的`hashCode`方法，而没有考虑`equals`方法，这会引起缓存key碰撞问题（查看问题背景 [SPR-10237](https://jira.spring.io/browse/SPR-10237)）。  
> 如果想使用老版本的key生成策略，则可配置为`DefaultKeyGenerator`（此类已被弃用）,或者自定义一个基于hash的缓存key生成处理类。  

### 2.1.2 自定义缓存key生成策略

大多数情况下，使用默认的缓存key生成策略能满足数据的缓存需求。  

然而，随着程序的扩展，也有一些特定的需求，需要定制缓存key的生成策略。比如：多入参的方法，只需要使用其中的某个参数生成缓存key。  

`Cache Abstraction`在默认的缓存key生成策略的基础上，也提供了以下两种自定义缓存key的生成策略：  

- 使用SpEL表达式指定缓存key  
- 自定义缓存key生成处理类`KeyGenerator`  
 
#### 2.1.2.1 SpEL表达式指定缓存key

**使用方式**：在注解的`key`属性上配置SpEL表达式  

通过SpEL表达式指定缓存key，可以获取参数值、参数对象的属性值，甚至支持方法调用，示例如下：  

> SpEL表达式用法详见Spring官方文档[Chapter 10, Spring Expression Language (SpEL)](https://docs.spring.io/spring/docs/4.3.13.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#expressions)  

```java
@Cacheable(cacheNames="books", key="#isbn")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)

@Cacheable(cacheNames="books", key="#isbn.rawNumber")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)

@Cacheable(cacheNames="books", key="T(someType).hash(#isbn)")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
```  

#### 2.1.2.2 自定义缓存key生成处理类`KeyGenerator`

**使用方式**：在注解的`keyGenerator`属性上配置**自定义Bean的名称**（即Spring Bean的name）

自定义缓存key生成处理类`KeyGenerator`的方式很简单，只需要实现`org.springframework.cache.interceptor.KeyGenerator`接口即可  
下面示例中，`myKeyGenerator`就是自定义的`KeyGenerator`:  

```java
@Cacheable(cacheNames="books", keyGenerator="myKeyGenerator")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
```

#### 2.1.2.3 互斥性

以上两种自定义缓存key策略是互斥的，即注解的`key`属性和`keyGenerator`属性不可同时配置  

#### 2.1.2.4 源码

下面我们从源码角度来看下，这两种自定义缓存key生成策略的实现方式。  

在`Cache Abstraction`代理拦截器（原理下篇会详细说）调用时候，会生成方法对应的缓存key，调用方法是`org.springframework.cache.interceptor.CacheAspectSupport.CacheOperationContext#generateKey`,源码如下：  

```java
private final CacheOperationExpressionEvaluator evaluator = new CacheOperationExpressionEvaluator();
protected Object generateKey(@Nullable Object result) {
	if (StringUtils.hasText(this.metadata.operation.getKey())) {
		EvaluationContext evaluationContext = createEvaluationContext(result);
		return evaluator.key(this.metadata.operation.getKey(), this.methodCacheKey, evaluationContext);
	}
	return this.metadata.keyGenerator.generate(this.target, this.metadata.method, this.args);
}
```

从上面源码中可以很明显的看出：  

- `if`判断直接中配置了`key`属性，则通过`evaluator`获取`key`配置的表达式的值。  
	`evaluator`即为默认的`CacheOperationExpressionEvaluator`,负责SpEL表达式的处理。  
- 若没有配置`key`属性，则根据`keyGenerator`来生成key  

那么，从前面内容可看出，`keyGenerator`分为默认的`SimpleKeyGenerator`和自定义的`KeyGenerator`，且自定义的`KeyGenerator`是在注解上`keyGenerator`属性配置的自定义Bean的名称，从上面源码看出，`this.metadata.keyGenerator`显然是一个特定的实例了。  

下面看看`KeyGenerator`实例是如何获取的，处理方法参见源码`org.springframework.cache.interceptor.CacheAspectSupport#getCacheOperationMetadata`（下面只贴出了获取`KeyGenerator`实例的源码，其余的省略）  

```java
if (StringUtils.hasText(operation.getKeyGenerator())) {
	operationKeyGenerator = getBean(operation.getKeyGenerator(), KeyGenerator.class);
}
else {
	operationKeyGenerator = getKeyGenerator();
}
```

从源码可以很明显的看出，注解的`keyGenerator`属性值需要配置自定义的Bean的名称，而没有配置这个属性，则通过`getKeyGenerator()`获取的就是默认的`SimpleKeyGenerator`  

### 2.1.3 Cache解析

`Cache Abstraction`中包含两个核心概念 `org.springframework.cache.Cache` 和 `org.springframework.cache.CacheManager`  

- `Cache`： 前面有提过，一个方法上声明缓存注解，指定名称后，就关联一个`Cache`。而`Cache Abstraction`声明`Cache`接口，针对缓存的读写操作。而在集成底层的缓存存储之后，实现`Cache`接口来完成缓存的读写操作。  
- `CacheManager` : 针对`Cache`的管理，包括初始化、解析获取等。    

**提示：**  

- `Cache Abstraction`的源码是在`spring-context`下  
- 前面有提到的，默认集成了一些第三方的底层缓存存储（如：Ehcache）是在`spring-contxt-support`包下，这个后面也会再说  


回到正题，`Cache`解析由`CacheResolver`完成，将方法上声明的注解解析成方法关联的`Cache`  

#### 2.1.3.1 默认的`CacheResolver`

默认的`Cache`解析实现类是`org.springframework.cache.interceptor.SimpleCacheResolver`  

#### 2.1.3.2 自定义`CacheResolver`

自定义Cache解析器需要实现`CacheResolver`接口，使用方式和前面自定义`KeyGenerator`类似，即在注解属性`cacheResolver`配置自定义Bean名称  

```java
@Cacheable(cacheResolver="runtimeCacheResolver")
public Book findBook(ISBN isbn) {...}
```

#### 2.1.3.3 源码

和前面的自定义`KeyGenerator`实现原理类似，在`Cache Abstraction`代理拦截器调用时候，需要将注解元数据解析封装成对应的`Cache`，封装缓存代理上线文，并在后续执行流程中，根据注解的逻辑，使用`Cache`执行缓存操作。  

先看下，在封装缓存调用上线文时候，确认`Cache`解析器的源码方法`org.springframework.cache.interceptor.CacheAspectSupport#getCacheOperationMetadata`(下面只贴出核心部分代码，其余省略)  

```java
CacheResolver operationCacheResolver;
if (StringUtils.hasText(operation.getCacheResolver())) {
	operationCacheResolver = getBean(operation.getCacheResolver(), CacheResolver.class);
}
else if (StringUtils.hasText(operation.getCacheManager())) {
	CacheManager cacheManager = getBean(operation.getCacheManager(), CacheManager.class);
	operationCacheResolver = new SimpleCacheResolver(cacheManager);
}
else {
	operationCacheResolver = getCacheResolver();
	Assert.state(operationCacheResolver != null, "No CacheResolver/CacheManager set");
}
```

从上述源码可看出，首先是根据注解属性`cacheResolver`配置的自定义Bean的名称取Spring容器中获取实例；若没配置，则使用默认的`SimpleCacheResolver`  

下面再来看下，在封装缓存调用上线文时候，使用上面获取的`CacheResolver`实例来解析`Cache`  

源码见`org.springframework.cache.interceptor.CacheAspectSupport.CacheOperationContext#CacheOperationContext`  

```java
public CacheOperationContext(CacheOperationMetadata metadata, Object[] args, Object target) {
	this.metadata = metadata;
	this.args = extractArgs(metadata.method, args);
	this.target = target;
	this.caches = CacheAspectSupport.this.getCaches(this, metadata.cacheResolver);
	this.cacheNames = createCacheNames(this.caches);
	this.methodCacheKey = new AnnotatedElementKey(metadata.method, metadata.targetClass);
}
protected Collection<? extends Cache> getCaches(
			CacheOperationInvocationContext<CacheOperation> context, CacheResolver cacheResolver) {
	Collection<? extends Cache> caches = cacheResolver.resolveCaches(context);
	if (caches.isEmpty()) {
		throw new IllegalStateException("No cache could be resolved for '" +
				context.getOperation() + "' using resolver '" + cacheResolver +
				"'. At least one cache should be provided per cache operation.");
	}
	return caches;
}
```

从上述源码可看出，根据`CacheResolver`实例解析`Cache`并封装带上下文对象中（注意`CacheOperationContext`是内部类）  

### 2.1.3 缓存条件

`Cache Abstraction`提供了两个属性配置，用于条件校验，是否需要缓存数据  

- `condition`属性：配置SpEL表达式，当条件为true时，缓存数据；当条件为false时，不缓存数据。每次方法调用都会判断该属性值。  

- `unless`属性：配置SpEL表达式，表示匹配该表达式，则数据不缓存。每次方法调用之后，会判断该属性值。  

示例如下：

```java
@Cacheable(cacheNames="book", condition="#name.length() < 32", unless="#result.hardback")
public Book findBook(String name)
```

### 2.1.4 SpEL表达式上下文

前面有说到，注解的配置属性`key` 和 `conditional` 都支持SpEL表达式，`Cache Abstraction`为每一个SpEL表达式提供了特定的表达式上下文，使得SpEL表达式中能够访问当前表达式上线文中的元数据（例如：SpEL能通过元数据获取参数名称）。  

如下表展示了表达式上线文的元数据：  


| 元数据变量名 | 元数据对象 | 描述 | 使用示例 |
| :---: | :----: | :---: | :---: |
| methodName | root object | 被调用的方法名称 | #root.methodName |
| method | root object | 被调用的方法 | #root.method.name |
| target | root object | 被调用的目标对象 | #root.target |
| targetClass | root object | 被调用的目标对象的class | #root.targetClass |
| args | root object | 被调用的目标对象的参数列表数组 | #root.args[0] |
| caches | root object | 当前被执行方法关联的缓存Cache | #root.caches[0].name |
| *argument name* | 表达式上下文 | 被调用方法的参数名称 | #iban 或者 #a0 |
| result | 表达式上下文 | 方法的返回值 | #result |

**注意：** 

- *argument name* ： 当不能明确知道参数名称，参数名也可根据规则获取  

	如示例中，`iban`是明确的参数名；`#a0` `#p0` `#p<0>` 也可用于指定参数名 （0都是参数下标）  

- result ： 只能在三种场景下使用  

	1. `unless`表达式配置  
	2. `@CachePut`注解的`key`属性配置  
	3. `@CacheEvict`注解表达式，且`beforeInvocation`属性为`false`  

## 2.2 `@CachePut`注解

`@CachePut`注解用在方法上，实现的功能就是更新缓存（不影响方法执行），即方法执行完之后，将结果放入缓存  

> `@CachePut`注解的配置通上面`@Cacheable`注解  

示例如下：  

```java
@CachePut(cacheNames="book", key="#isbn")
public Book updateBook(ISBN isbn, BookDescriptor descriptor)
```

> `@CachePut`注解和`@Cacheable`注解注解不建议同时使用，因为二者代表不同的场景，前者需要执行方法，并缓存结果。而后者，当缓存数据存在，是不需要执行方法的。  

## 2.3 `@CacheEvict`注解

`@CacheEvict`注解用在方法上，实现的功能就是删除缓存数据  

**特殊属性：**  

- `allEntries`  
	当配置为true时候，删除cache关联的所有缓存数据，key属性会被忽略；  
	配置为false时候，根据key删除缓存数据  

- `beforeInvocation`  
	当配置为true的时候，在方法执行之前操作缓存；  
	当配置为false的时候，在方法成功执行之后操作缓存（当方法未执行或者抛出异常，则不会操作缓存）

示例如下：  

```java
@CacheEvict(cacheNames="books", allEntries=true)
public void loadBooks(InputStream batch)
```

> `@CacheEvict`注解可用在无返回值`void`的方法上，此时该方法用于触发缓存操作。  
> `@Cacheable`和`@CachePut`注解则需要返回值，因为缓存的数据就是方法执行的返回值  

## 2.4 `@Caching`注解

`@Caching`注解用在方法上，当一个方法上需要声明**同一类型**的**多个**注解，则需要使用`@Caching`注解将多个注解组合起来  

`@Caching`注解可用于组合的注解有： `@Cacheable` `@CachePut` `@CacheEvict`

如下例，根据不同的条件删除缓存，则可以用`@Caching`注解组合多个`@CacheEvict`注解：  

```java
@Caching(evict = { @CacheEvict("primary"), @CacheEvict(cacheNames="secondary", key="#p0") })
public Book importBooks(String deposit, Date date)
```

## 2.5 `@CacheConfig`注解

对于同一个类中的多个方法，如果都有使用缓存注解的需求，那么，每个注解上的多个个性化配置显然比较麻烦。  

`@CacheConfig`注解正是为了解决上述问题，提供了类级别的公共配置，且不会触发缓存操作  

如下例，`@CacheConfig`注解在类上配置了缓存名称，类中的方法就能公用缓存名称这个配置：  

```java
@CacheConfig("books")
public class BookRepositoryImpl implements BookRepository {
    @Cacheable
    public Book findBook(ISBN isbn) {...}
}
```

> 若方法上配置属性与类上配置属性相同，则方法上的配置会覆盖类上属性的配置  

## 2.6 启用缓存注解

仅仅在方法上使用缓存注解，并不能使注解生效，还需要配置**启用缓存注解**。   

**启用缓存注解**有以下两种方式：  

- 代码注解配置方式  

在配置类上声明`@EnableCaching`，示例如下：  

```java
@Configuration
@EnableCaching
public class AppConfig {
}
```

- xml配置方式  

在xml中声明`cache:annotation-driven `，示例如下：  

```xml
<cache:annotation-driven />
```

了解上述配置方式后，就会发现与事务注解`@Transactional`的启用方式类似  

其实实现原理也是一样的，都是通过启用注解配置，自动扫描缓存注解，使用AOP的方式生成代理类，实现缓存操作（具体的实现原理下篇再说）。  

同样的，和事务注解`@Transactional`一样，缓存注解的启用，也提供了很多针对AOP的配置参数。  

另外的一些配置则和缓存特性相关，详见如下说明：  

- **cache-manager**  
	XML属性名：cache-manager  
	注解属性名： N/A（请查看CachingConfigurer的javadoc）  
	默认值：cacheManager  
	描述：缓存注解中使用的cache manager的Bean名称  

- **cache-resolver**  
	XML属性名：cache-resolver  
	注解属性名： N/A（请查看CachingConfigurer的javadoc）  
	默认值：使用上面的cache-manager配置的CacheManager创建SimpleCacheResolver实例  
	描述：缓存解析器的bean名称，此属性不是必须配置  

- **key-generator**  
	XML属性名：key-generator  
	注解属性名： N/A（请查看CachingConfigurer的javadoc）  
	默认值：SimpleKeyGenerator  
	描述：自定义KeyGenerator的Bean名称  

- **error-handler**  
	XML属性名：error-handler  
	注解属性名： N/A（请查看CachingConfigurer的javadoc）  
	默认值：SimpleCacheErrorHandler  
	描述：自定义缓存异常处理类的Bean名称。默认情况下，缓存操作抛出的异常会被抛给调用方  

- **mode**  
	XML属性名：mode  
	注解属性名： mode  
	默认值：proxy  
	描述：默认值是proxy，使用Spring AOP代理。可配置为aspectj，通过修改类字节码，实现编译器或者载入期增强    

- **proxy-target-class**  
	XML属性名：proxy-target-class  
	注解属性名： proxyTargetClass  
	默认值：false  
	描述：当上面的mode属性配置为proxy是才生效。配置为true的时候，使用基于类的Spring AOP代理，即CGLIB。配置为false，则使用基于接口的Spring AOP代理方式，即JDK动态代理。  

- **order**  
	XML属性名：order  
	注解属性名： order  
	默认值：Ordered.LOWEST_PRECEDENCE  
	描述：上面说到过，缓存注解的实现原理是通过代理。一个类上的多层代理的执行顺序则通过order属性指定。  


了解完配置属性后，下面归纳下两类问题： **缓存属性优先级问题** 和 **AOP问题**   

- **缓存属性优先级问题**  

	1. 配置优先级  
		从上面了解可知，缓存的配置存在于三个地方：启用注解的配置属性、类级别的配置属性、方法级别的配置属性。从前到后，优先级越来越高，即后面的会覆盖前面的配置  
	
	2. 启用注解作用范围  
		对于常规的SpringMVC + Spring 的项目开发模式，存在父子容器问题，启用注解的配置，一定要与缓存注解所在类在同一个Spring Bean容器中。  
		详见官方文档[Section 22.2, “The DispatcherServlet”](https://docs.spring.io/spring/docs/4.3.13.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#mvc-servlet)

- **AOP问题**  
	AOP问题其实是Spring基于注解，通过AOP增强实现的一些特性所共有的问题，包括本文的缓存注解，以及事务注解`@Transactional`  

	1. 外部调用  
		当modoe属性为proxy，代表使用Spring AOP代理增强，那么缓存注解方法的调用，必须是外部类调用才能生效，实现缓存操作。类内部的方法调用是不生效的，因为内部调用不走AOP代理。要是内部调用也生效，则需要配置modoe属性为aspectj，通过修改类字节码，实现编译器或者载入期增强。  
	
	2. 方法可见性  
		当modoe属性为proxy，代表使用Spring AOP代理增强，那么缓存注解方法必须是public的才能生效，实现缓存操作。若方法是protected、private或者包可见的，则不生效，也不会抛出异常。要想解决这个问题也可以通过配置modoe属性为aspectj，通过修改类字节码，实现编译器或者载入期增强。  
	
	3. 注解方法范围  
		推荐在具体的类的方法上使用注解，不要配置到接口的方法上。因为当modoe属性为proxy，代表使用Spring AOP代理增强，而Spring AOP分为基于类的和基于接口的（查看上面proxy-target-class属性配置），注解使用在接口上，基于类的AOP则不能正常从接口上将注解继承到类上。通用也可以通过配置modoe属性为aspectj，通过修改类字节码，实现编译器或者载入期增强。  
	
	4. 多层AOP代理执行顺序问题  
		一个类上可使用多层注解，执行顺序，则可以通上面的order属性执行  
		详见官方文档[Advice ordering](https://docs.spring.io/spring/docs/4.3.13.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#aop-ataspectj-advice-ordering)
	

## 2.7 自定义注解

`caching abstraction`支持自定义注解，触发缓存操作。自定义注解定义支持使用`@Cacheable` `@CachePut` `@CacheEvict` `@CacheConfig` 作为元注解（可在其他注解上声明的注解）。  

如下例，自定义注解`SlowService`使用`Cacheable`作为元注解：  

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
@Cacheable(cacheNames="books", key="#isbn")
public @interface SlowService {
}
```

那么，以下两种注解方式是等价的  

```java
@Cacheable(cacheNames="books", key="#isbn")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
@SlowService
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
```

**启用了缓存注解**，就能识别示例中的自定义注解，并实现缓存操作（经测有效）  

> 在我们理解中，自定义注解一般需要我们自己编写处理类来解析和处理注解。但上面的自定义注解能生效，是因为自定义注解的定义里面使用了缓存注解作为元注解，`caching abstraction`能处理这部分自定义注解。  


# 3 JCache注解

从Spring 4.1版本之后，`Cache Abstraction`开始支持 JCache 标准注解  

> 详见官网[JCache (JSR-107) annotations](https://docs.spring.io/spring/docs/4.3.13.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#cache-jsr-107)，本文暂略  

# 4 基于XML的缓存声明

前面主要介绍了`Cache Abstraction`中支持的注解，这些注解一般在源代码的方法上声明。  

同样，`Cache Abstraction`也支持XML声明的方式，从而无需侵入源代码，示例如下：  

```xml
<bean id="bookService" class="x.y.service.DefaultBookService"/>
<cache:advice id="cacheAdvice" cache-manager="cacheManager">
    <cache:caching cache="books">
        <cache:cacheable method="findBook" key="#isbn"/>
        <cache:cache-evict method="loadBooks" all-entries="true"/>
    </cache:caching>
</cache:advice>
<!-- apply the cacheable behavior to all BookService interfaces -->
<aop:config>
    <aop:advisor advice-ref="cacheAdvice" pointcut="execution(* x.y.BookService.*(..))"/>
</aop:config>
<!-- cache manager definition omitted -->
```

# 5 底层缓存存储配置

如开篇所述，`Cache Abstraction`主要包含 **缓存注解** 和 **底层缓存存储配置** 两部分。  

对于  **底层缓存存储配置** , 前面有提到过，主要是对第三方底层缓存存储的集成，这种集成方式分两方面：定制第三方的对`Cache Abstraction` **实现** 以及集成 **配置** 。  

`Cache Abstraction`提供了几种常见的第三方的缓存集成的实现（分为对`CacheManager` 和 `Cache` 的实现），即我们在使用这些第三方缓存作为`Cache Abstraction`底层缓存存储的时候，只需要进行集成配置即可。  
  
> 以下几种方案，个人没有做验证，仅仅了解罗列一下，感兴趣的可以参照官网文档和源码深入了解。  
> 本人主要为了研究集成我司分布式缓存，后面文章会说到。    

## 5.1 JDK ConcurrentMap

### 5.1.1 实现

`Cache Abstraction`提供了默认实现，在`spring-context` 模块的 `org.springframework.cache.concurrent` 包下  

这种方式使用 JDK `ConcurrentHashMap ` 作为底层缓存存储。  

- CacheManager 实现： `org.springframework.cache.support.SimpleCacheManager`  
- Cache 实现 ： `org.springframework.cache.concurrent.ConcurrentMapCache`  

### 5.1.2 配置

示例如下：  

```java
<bean id="cacheManager" class="org.springframework.cache.support.SimpleCacheManager">
    <property name="caches">
        <set>
            <bean class="org.springframework.cache.concurrent.ConcurrentMapCacheFactoryBean" p:name="default"/>
            <bean class="org.springframework.cache.concurrent.ConcurrentMapCacheFactoryBean" p:name="books"/>
        </set>
    </property>
</bean>
```

## 5.2 Ehcache

### 5.2.1 实现

`Cache Abstraction`提供了默认实现，在`spring-context-support` 模块的 `org.springframework.cache.ehcache` 包下  

这种方式使用 `Ehcache` 作为底层缓存存储。  

- CacheManager 实现： `org.springframework.cache.ehcache.EhCacheCacheManager`  
- Cache 实现 ： `org.springframework.cache.ehcache.EhCacheCache`  

### 5.2.2 配置

示例如下：  

```java
<bean id="cacheManager"
      class="org.springframework.cache.ehcache.EhCacheCacheManager" p:cache-manager-ref="ehcache"/>
<bean id="ehcache"
      class="org.springframework.cache.ehcache.EhCacheManagerFactoryBean" p:config-location="ehcache.xml"/>
```


## 5.3 Caffeine Cache

### 5.3.1 实现

`Cache Abstraction`提供了默认实现，在`spring-context-support` 模块的 `org.springframework.cache.caffeine` 包下  

这种方式使用 `Caffeine Cache` 作为底层缓存存储。  

- CacheManager 实现： `org.springframework.cache.caffeine.CaffeineCacheManager`  
- Cache 实现 ： `org.springframework.cache.caffeine.CaffeineCache`  

### 5.3.2 配置

示例如下：  

```java
<bean id="cacheManager" class="org.springframework.cache.caffeine.CaffeineCacheManager">
    <property name="caches">
        <set>
            <value>default</value>
            <value>books</value>
        </set>
    </property>
</bean>
```

## 5.4 Guava Cache

### 5.4.1 实现

`Cache Abstraction`提供了默认实现，在`spring-context-support` 模块的 `org.springframework.cache.guava` 包下  

这种方式使用 `Guava Cache` 作为底层缓存存储。  

- CacheManager 实现： `org.springframework.cache.guava.GuavaCacheManager`  
- Cache 实现 ： `org.springframework.cache.guava.GuavaCache`  

### 5.4.2 配置

示例如下：  

```java
<bean id="cacheManager" class="org.springframework.cache.guava.GuavaCacheManager">
    <property name="caches">
        <set>
            <value>default</value>
            <value>books</value>
        </set>
    </property>
</bean>
```

## 5.5 Guava Cache

使用方式参照Spring官方文档[Spring Data GemFire Reference Guide](https://docs.spring.io/spring-gemfire/docs/current/reference/html/)  

## 5.6 JSR-107 Cache

### 5.6.1 实现

`Cache Abstraction`提供了默认实现，在`spring-context-support` 模块的 `org.springframework.cache.jcache` 包下  

这种方式使用 `JCache` 作为底层缓存存储。  

- CacheManager 实现： `oorg.springframework.cache.jcache.JCacheCacheManager`  
- Cache 实现 ： `org.springframework.cache.jcache.JCacheCache`  

### 5.6.2 配置

示例如下：  

```java
<bean id="cacheManager" class="org.springframework.cache.jcache.JCacheCacheManager" p:cache-manager-ref="jCacheManager"/>
```

## 5.7 无底层缓存存储配置方案

当需要验证方法的执行或者切换缓存环境等没有底层缓存存储支持的场景下，通常我们需要去掉缓存的声明。否则，运行时会抛异常。  

同时，`Cache Abstraction`提供了一种更优雅的方案，当底层缓存存储不可用时候，忽略缓存声明，每次调用都直接执行方法。  

配置示例如下：  

```java
<bean id="cacheManager" class="org.springframework.cache.support.CompositeCacheManager">
    <property name="cacheManagers">
        <list>
            <ref bean="jdkCache"/>
            <ref bean="gemfireCache"/>
        </list>
    </property>
    <property name="fallbackToNoOpCache" value="true"/>
</bean>
```

如上，`CompositeCacheManager`组合多个`CacheManager`。  
而`fallbackToNoOpCache`参数配置为`true`，则当通过配置的`CacheManager`获取`Cache`的时候，则会直接执行方法。  

# 6 底层缓存存储集成

> **底层缓存存储集成** 与上面的 **底层缓存配置**  有何异同点呢？  
> 
> 前面有提到 `Cache Abstraction` 是对缓存逻辑的抽象与封装，而不涉及对各个第三方底层缓存存储的实现。  
> 
> 然而Spring为了达到开箱即用的目的，提供了几种默认的第三方缓存存储实现，**底层缓存配置**即只需将这些默认的实现进行配置，即可使用。  
> 
> **底层缓存存储集成**指那些尚未有默认实现的第三方底层缓存存储，为了集成这些缓存，需要先按照`Cache Abstraction`框架设计实现缓存管理与操作，然后进行配置。  

**底层缓存存储集成**主要包含 **自定义缓存管理和操作** 和 **自定义实现的配置**  


- 自定义缓存管理和操作  

	既然是缓存管理和操作，其实就是要实现前面提到的`CacheManager` 和 `Cache`  

	这里没有一个既有的实现标准，我们可以参照Spring提供的默认集成方案，这里不细说，可自行查看源码，后面也有篇文章是关于缓存集成的  

	对于常用的缓存`Redis`，`Spring-context-support`没有提供集成实现方案，但是`spring-data-redis`已经支持`Cache Abstraction`，还支持超时时间设置，感兴趣的可以看下源码，使用起来也很简单  

- 自定义实现的配置

	可参照前面一节，请注意，前面一节其实是XML配置方式，如果是使用注解方式，XML配置会更加简单，参见前面的**启用缓存注解**章节。  

# 7 关于缓存的超时策略等特性

`Cache Abstraction` 作为对缓存逻辑的抽象与封装，并不涉及底层的缓存存储。  

对于缓存超时策略等问题，其实是需要底层缓存的支持的，所以，`Cache Abstraction` 并没有在抽象方案中提供这种个性化特性的支持。  
我们在集成第三方缓存时候，可以考虑下自己的使用场景，合理的定制对各种特性的支持。  

# 8 总结

`Cache Abstraction`总结主要有一下几点：  

- 能有效减少缓存逻辑的模板方法（如：击穿回写、更新清除等）  
- 提供的注解使用以及注解驱动配置比较简单  
- 即使是某些大厂使用的分布式缓存并没有默认的实现方案，集成起来也不是特别麻烦，而且有很多默认集成方案源码可参考  
- 最后注意下，`Cache Abstraction`并不能满足所有的缓存使用场景，完美主义者想所有的缓存操作注解化，可能也很费劲  
- 当然，如果你觉得`Cache Abstraction`只能解决你的一部分问题，或者觉得这玩意没啥用，你有更优的方案解决问题，多了解一种Spring基于AOP和注解的特性也是好的，套路是一样的，大多数人是会用事务注解的吧 -_-    

