---
layout: post
title: "Spring Cache Abstraction集成Redis"
description: Spring Cache Abstraction实现了几种默认的第三方存储集成，实现了缓存注解的开箱即用，而本篇要探讨的是自定义实现其他的第三方存储。
date: 2018-02-07 10:37:41 +0800
catalog: true
categories:
- Spring Cache Abstraction
tags:
- Spring Cache
---

`Spring Cache Abstraction`提供了几种默认的第三方存储集成，实现了缓存注解的开箱即用。  

而本篇要探讨的是自定义实现其他的第三方缓存的集成，特别是现在广泛使用的`Redis`或者某些公司自己实现的**类Redis**缓存。  

`spring_cache_abstraction`作为一种缓存逻辑的抽象封装，实现方式上就是屏蔽底层缓存存储的细节，抽象了缓存管理和操作的接口，便于不同的第三方缓存的扩展和集成。  

> `spring_cache_abstraction`并未提供一套明确的扩展标准，我们可以参考`Spring Cache Abstraction`提供的几种默认的第三方存储集成的实现方式(spring-context-support的cache包下)。  


# 1 背景

`Redis`和某些公司自己实现的**类Redis**缓存使用广泛，且`Spring cache`本身并没有对`Redie`提供默认的支持实现。  

为了集成`Redis`，我们需要自定义`Cache`和`CacheManager`（前面有提到自定义的方式），参照`Spring cache`的默认实现研究一番后，偶然发现`Spring data redis`已经支持了`Spring cache`。  

本着不重复造轮子的思想，集成`Redis`就基于`Spring data redis`对`Spring cache`来做。  

> 那么问题来了，既然`Spring data redis`已经支持了`Spring cache`，为啥还要自己来集成呢？  
> 开篇说到**广泛使用的Redis或者某些公司自己实现的类Redis缓存**，很明显，我司属于后者  

## 1.1 类库版本

Spring版本：5.0.0.RELEASE  

Spring Data版本：2.0.0.RELEASE  

JDK版本：JDK8  

## 1.2 Spring Data Redis 对 Spring Cache的支持

既然是基于`Spring data redis`对`Spring cache`的支持来集成类redis缓存，那么我们先来了解下`Spring Data Redis`对`Spring cache`的实现。  

如下图是Spring Data项目下cache包下面的类文件：  

![](/images/cust/cache/spring-data-cache.png)  

可以看出，对`Spring cache`中`Cache`和`CacheManager`的实现分别是`RedisCache`和`RedisCacheManager`。  

另外，`Spring data redis`的`Spring cache`还有以下两个特性：  

- 支持cache配置化：每个`RedisCache`包含一个配置属性成员变量`RedisCacheConfiguration`（给缓存加失效时间我们就是基于这个配置做的）。  
- 抽象缓存操作接口：cache并不负责缓存的真正操作，主要负责的是与`Spring cache`的集成，即实现`cache`。缓存真正操作由`RedisCacheWriter`来完成（集成类redis缓存，我们只需要实现这个接口即可）。  


# 2 集成

从上述说明可知，基于`Spring Data Redis`来集成类redis缓存，从而支持`Spring cache`，主要做两件事：  

- 初始化cache对应的配置：实现缓存失效时间等特性  
- 实现缓存操作接口：实现类redis缓存的集成   

下图是一个集成类redis缓存的示例：  

![](/images/cust/cache/cache-redis-cust.png)  

重点是`service`包和`springcache`包：  

- `springcache`包：集成类redis缓存  
- `service`包：集成完成后，项目中的使用方式  

> 为了说明方便，这两个包放在了同一个工程下。但是通常，`springcache`包可以放在一个单独的工程中，项目中使用时候依赖这个jar包即可。  

## 2.1 `service`包

为何先说项目中使用方式的`service`包?  
因为这里需要先申明下，`Spring cache`集成类redis缓存的一些设计思路，是基于当前项目中的使用方式来设计、考虑和实现的  

先看下，一般项目中，缓存key会单独放在一个枚举类中：  

```java
public enum CacheConfigEnum {
    TEST_KEY_1(CacheNameConstant.TEST_CACHE_NAME, "test.key.%s", 3600);
    private String cacheName;
    private String format;
    private int exp;

    private CacheConfigEnum(String cacheName, String format, int exp) {
        this.cacheName = cacheName;
        this.format = format;
        this.exp = exp;
    }
    //getter and setter
	public static CacheConfigEnum getConfigByCacheName(String cacheName){
        if (StringUtils.isEmpty(cacheName)){
            return null;
        }
        for (CacheConfigEnum cacheConfigEnum : CacheConfigEnum.values()){
            if (cacheName.equals(cacheConfigEnum.getCacheName())){
                return cacheConfigEnum;
            }
        }
        return null;
    }
}
```

由于在`Spring cache`中，没有缓存key的概念，所以上述枚举类中加了一个`cacheName`属性，将缓存key和cache这个概念关联起来  

```java
public class CacheNameConstant {
	public static final String TEST_CACHE_NAME = "test.cache.name";
}
```

如上述章节所述，基于`Spring Data Redis`来集成类redis缓存，从而支持`Spring cache`,会**初始化cache对应的配置**。基于项目中的使用方式，cache的配置即上面的枚举类声明（主要包含缓存key和失效时间），在项目中，需要将枚举类转换成一个元数据对象，如下：  

```java
@Component
public class CacheConfigReaderImpl implements CacheConfigReader {
	@Override
	public boolean isValidCache(String cacheName) {
		return null != CacheConfigEnum.getConfigByCacheName(cacheName);
	}
	@Override
	public List<CacheConfig> readCache() {
		return Arrays.stream(CacheConfigEnum.values()).map(c -> {
							CacheConfig cacheConfig = new CacheConfig();
							cacheConfig.setCacheName(c.getCacheName());
							cacheConfig.setCacheKey(c.getFormat());
							cacheConfig.setExpire(c.getExp());
							return cacheConfig;
						}).collect(Collectors.toList());
	}
}
```

最后，就是项目中业务层对`Spring cache`注解的使用  

```java
@Service
public class CacheServiceImpl {
	@Cacheable(cacheNames = CacheNameConstant.TEST_CACHE_NAME, keyGenerator = "stringFormat")
	public String getDataByType(int type){
		System.out.println("input is --- " + type);
		String result = type + " RS. ";
		System.out.println("result is --- " + result);
		return result;
	}
}
```

## 2.2 `springcache`包

`springcache`包负责集成redis缓存，如本章节开头所说，，基于`Spring Data Redis`来集成类redis缓存，从而支持`Spring cache`，主要做两件事：  

- 初始化cache对应的配置：实现缓存失效时间等特性  
- 实现缓存操作接口：实现类redis缓存的集成   

### 2.2.1 初始化cache配置

初始化cache配置，主要是声明配置读取接口（读取业务系统中缓存的配置,封装成元数据对象`CacheConfig`），接口声明如下：  

```java
public interface CacheConfigReader {
	boolean isValidCache(String cacheName);
	List<CacheConfig> readCache();
}
```

具体的读取逻辑，由业务系统根据场景自己实现（即上一章节的实现类`CacheConfigReaderImpl`）  

### 2.2.2 实现缓存操作接口

前面说到，`Spring cahce redis`对`Spring cache`的实现，将缓存操作声明了单独的接口，从`Cache`中剥离出来，即声明`RedisCacheWriter`  

要集成自己的类redis缓存，实现`RedisCacheWriter`接口即可，如下：

```java
@Component
public class CustRedisCacheWriter implements RedisCacheWriter {
	@Autowired
	private CacheConfigReader reader;

	@Resource(name = "redisTemplate")
	private RedisTemplate<byte[], byte[]> redisTemplate;

	@Override
	public void put(String name, byte[] key, byte[] value, @Nullable Duration ttl) {
		checkCacheName(name);
		if (shouldExpireWithin(ttl)){
			redisTemplate.boundValueOps(key).set(value, ttl.toMillis(), TimeUnit.MILLISECONDS);
			return;
		}
		redisTemplate.boundValueOps(key).set(value);
	}

	private static boolean shouldExpireWithin(@Nullable Duration ttl) {
		return ttl != null && !ttl.isZero() && !ttl.isNegative();
	}

	@Nullable
	@Override
	public byte[] get(String name, byte[] key) {
		checkCacheName(name);
		return redisTemplate.boundValueOps(key).get();
	}

	@Nullable
	@Override
	public byte[] putIfAbsent(String name, byte[] key, byte[] value, @Nullable Duration ttl) {
//		if (cacheClient.setNX(key, value)){
//			if (shouldExpireWithin(ttl)){
//				cacheClient.pExpire(key, ttl.toMillis(), TimeUnit.MILLISECONDS);
//				return null;
//			}
//		}
//		return cacheClient.get(key);
		throw new UnsupportedCacheOperationException("Unsupported method putIfAbsent, cache name is " + name);
	}

	@Override
	public void remove(String name, byte[] key) {
		checkCacheName(name);
		redisTemplate.delete(key);

	}

	@Override
	public void clean(String name, byte[] pattern) {
//		byte[][] keys = Optional.ofNullable(cacheClient.keys(pattern)).orElse(Collections.emptySet())
//				.toArray(new byte[0][]);
//
//		if (keys.length > 0) {
//			cacheClient.del(keys);
//		}

		throw new UnsupportedCacheOperationException("Unsupported method clean, cache name is " + name);
	}

	private void checkCacheName(String cacheName){
		if (null == reader){
			throw new InvalidCacheException("Check cache name Error: ConfigReader is null!");
		}

		if (!reader.isValidCache(cacheName)){
			throw new InvalidCacheException(String.format("Invalid cache name [%s]", cacheName));
		}
	}

	public CacheConfigReader getReader() {
		return reader;
	}

	public void setReader(CacheConfigReader reader) {
		this.reader = reader;
	}

	public RedisTemplate<byte[], byte[]> getRedisTemplate() {
		return redisTemplate;
	}

	public void setRedisTemplate(RedisTemplate<byte[], byte[]> redisTemplate) {
		this.redisTemplate = redisTemplate;
	}
}
```

**上面类中的RedisTemplate替换成需要集成的类redis缓存的操作客户端，即可实现类redis缓存的集成。本工程为了说明和运行方便，直接使用了spring data redis的RedisTemplate作为缓存客户端。**  


spring data redis的RedisTemplate的xml声明如下：  

```xml
<bean id="jedisConnFactory"
     class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory"
     p:use-pool="true"/>

<bean id="keySerializer"
       class="name.lx.springcache.writer.CustRedisKeySerializer"/>

<bean id="redisTemplate"
     class="org.springframework.data.redis.core.RedisTemplate"
     p:connection-factory-ref="jedisConnFactory"
     p:keySerializer-ref="keySerializer"/>
```

### 2.2.3 配置与缓存操作的集成

最后看下，缓存配置初始化与缓存操作的集成，其实都是封装到`Spring data redis`的`RedisCacheManager`中的，如下：  

```java
@Configuration
public class CacheManagerConfiguration {
	@Autowired
	private RedisCacheWriter jimDBCacheWriter;

	@Autowired
	private CacheConfigReader reader;

	@Bean(name = "custCacheManager")
	public CacheManager cacheManager(){
		return RedisCacheManager.RedisCacheManagerBuilder.fromCacheWriter(jimDBCacheWriter).withInitialCacheConfigurations(initCacheConfig()).build();
	}

	private Map<String, RedisCacheConfiguration> initCacheConfig(){
		if (null == reader){
			throw new CacheInitException("Initialization Error: ConfigReader is null!");
		}

		Map<String, RedisCacheConfiguration> cacheConfigurationMap = new HashMap<>();

		RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig().disableKeyPrefix();

		List<CacheConfig> configList = reader.readCache();
		if(CollectionUtils.isEmpty(configList)){
			throw new CacheInitException("Initialization Error: can not read any caches!");
		}
		configList.forEach(c ->
				cacheConfigurationMap.put(c.getCacheName(), defaultConfig.entryTtl(Duration.ofSeconds(c.getExpire())))
		);
		if (cacheConfigurationMap.isEmpty()){
			throw new CacheInitException("Initialization Error: No caches!");
		}
		return cacheConfigurationMap;
	}
}
```

至此，集成完成后，需要配置xml文件，启用`Spring cache`时候指定上面配置的`RedisCacheManager`  

```xml
<context:component-scan base-package="name.lx"/>
<cache:annotation-driven cache-manager="custCacheManager"/>
```

# 3 总结

- 本篇并非一个完整的对`Spring cache`的实现，而是基于`Spring data redis`对`Spring cache`实现的基础上的一个个性化定制
- 个性化定制基于当前的项目现状考虑：配置的初始化基于当前项目中缓存key枚举的使用；缓存操作接口的实现基于当前项目使用的是类redis缓存客户端  





