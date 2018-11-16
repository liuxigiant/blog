---
layout: post
title: "Spring Data ElasticSearch"
description: Spring Data ElasticSearch的实践之旅
date: 2016-08-03 10:47:54 +0800
catalog: true
categories:
- ElasticSearch
tags:
- ElasticSearch
- Spring Data
---

随着业务系统数据量的增长，MySQL单表存储保证不了读写的高效性。分库分表和历史数据迁移等解决方案就闪亮登场，但是这两种方案都存在聚合查询的问题。此时，ElasticSearch（下面简称ES）作为一种实时的全文检索引擎，能够快速的查询海量数据。将全量数据写入ES，通过查询ES，解决聚合查询的问题，同时保证查询效率。  

# 1. ES使用方式

## 1.1 简介
ElasticSearch（下面简称ES）是一个开源的、基于Apache Lucene的、分布式的实时分析搜索引擎。其设计理念就是可以从不用的数据源获取数据，进行实时的检索和分析（As the company behind the open source projects — Elasticsearch, Logstash, Kibana, and Beats — designed to take data from any source and search, analyze, and visualize it in real time, Elastic is helping people make sense of data. ）。

## 1.2 使用方式

- Rsetful API  
   基于http协议，使用JSON为数据交换格式，通过9200端口的与Elasticsearch进行通信  
- JAVA API  
   Elasticsearch为Java用户提供了两种内置客户端：  

   **节点客户端(node client)：**  
   节点客户端以无数据节点(none data node)身份加入集群，换言之，它自己不存储任何数据，但是它知道数据在集群中的具体位置，并且能够直接转发请求到对应的节点上。  

   **传输客户端(Transport client)：**  
   这个更轻量的传输客户端能够发送请求到远程集群。它自己不加入集群，只是简单转发请求给集群中的节点。  

   两个Java客户端都通过9300端口与集群交互，使用Elasticsearch传输协议(Elasticsearch Transport Protocol)。集群中的节点之间也通过9300端口进行通信。如果此端口未开放，你的节点将不能组成集群。  

   >使用JAVA API方式，需要我们自己根据业务需求在JAVA API的基础上封装一套交互、查询的模块，便于维护和扩展  
   >ps：学习ES可以参考[`Elasticsearch 权威指南`的中文翻译文档](http://es.xiaoleilu.com/)，以上第1、2点参考自此文档  
- Spring Data ElasticSearch  
   Spring Data ElasticSearch项目是Spring Data和ES的集成，也提供了和JAVA API中一样的两种客户端。Spring Data ElasticSearch封装了与ES交互的实现细节，可以使系统开发者以Spring Data Repository 风格实现与ES的数据交互。  
  
# 2. Spring Data ElasticSearch实践  

## 2.1 maven依赖  

``` xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-elasticsearch</artifactId>
    <version>2.0.2.RELEASE</version>
</dependency>
```
本篇文章中Spring版本使用的是`4.2.3.RELEASE`版本  

>近期项目中接入ES，ES服务端提供的是2.1.1版本，版本比较新，所以Spring Data ElasticSearch版本我选的也是比较新的，兼容的Spring版本也比较高。可惜，运维提供的ES服务版本2.1.1不能变，Spring版本也不让升到4.0+，结果就是我不断调整Spring Data ElasticSearch和Spring版本，经历了各种报错，最后无奈放弃，只能选择JAVA API方式接入ES -_- 

## 2.2 配置文件  
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:elasticsearch="http://www.springframework.org/schema/data/elasticsearch"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                            http://www.springframework.org/schema/data/elasticsearch http://www.springframework.org/schema/data/elasticsearch/spring-elasticsearch.xsd
                            http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <!-- bean管理支持注解 -->
    <context:component-scan base-package="name.liuxi.service"/>

    <!-- Spring data 自动扫描repository接口，生成实现类 -->
    <elasticsearch:repositories base-package="name.liuxi.es" />

    <!-- ip:port换成具体的ip和端口，多个以逗号分隔 -->
    <elasticsearch:transport-client id="client" cluster-nodes="ip:port" cluster-name="elasticsearch" />

    <bean name="elasticsearchTemplate" class="org.springframework.data.elasticsearch.core.ElasticsearchTemplate">
        <constructor-arg ref="client"/>
    </bean>

    <bean name="orderRepositoryImpl" class="name.liuxi.es.impl.OrderRepositoryImpl">
        <property name="elasticsearchTemplate" ref="elasticsearchTemplate"/>
    </bean>
</beans>
```
以上配置包含：spring容器自动扫描、Spring data repository扫描、ES transport client、elasticsearchTemplate、自定义repository  

>以上配置可以使用java注解的方式

## 2.3 实体类映射  
``` java
@Document(indexName = "test_es_order_index", type = "test_es_order_type")
public class Order {
    @Id
    private Long id;
    private String userName;
    private String skuName;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public String getSkuName() {
        return skuName;
    }

    public void setSkuName(String skuName) {
        this.skuName = skuName;
    }

    @Override
    public String toString() {
        return "Order{" +
                "id=" + id +
                ", userName='" + userName + '\'' +
                ", skuName='" + skuName + '\'' +
                '}';
    }
}
```
注意：  
@Document注解指定了Order实体类对应的ES存储时的index（索引）名称和type（类型）名称  
@Id注解指定了主键  

>另外还有其他一些功能很强大的注解  

## 2.4 spring data repository
spring data elsaticsearch提供了三种构建查询模块的方式：  

- 基本的增删改查：继承spring data提供的接口就默认提供  
- 接口中声明方法：无需实现类，spring data根据方法名，自动生成实现类，方法名必须符合一定的规则（这里还扩展出一种忽略方法名，根据注解的方式查询）  
- 自定义repository：在实现类中注入elasticsearchTemplate，实现上面两种方式不易实现的查询（例如：聚合、分组、深度翻页等）  

>上面的第一点和第二点只需要声明接口，无需实现类，spring data会扫描并生成实现类  

**自定义Repository接口**：使用elasticsearchTemplate实现复杂查询  
``` java
public interface OrderEsCommonRepository {
    /**
     * 创建索引
     * @return
     */
    public boolean createOrderIndex();
}

```

**基础的repository接口**：提供基本的增删该查和根据方法名的查询
``` java
/**
 * 基础的repository接口
 *
 * spring data自动生成实现类，此处集成自定义的接口是为了在自动生成的实现类中添加自定义的实现（注意：实现类是这个基础的repository接口加上Impl后缀，这样才能被spring自动扫描到）
 *
 *  ElasticsearchRepository 继承 PagingAndSortingRepository, PagingAndSortingRepository提供了分页和排序的支持
 */
public interface OrderRepository extends ElasticsearchRepository<Order, Long>, OrderEsCommonRepository {

    /**
     * spring data提供的根据方法名称的查询方式
     * @param userName
     * @param skuName
     * @return
     */
    public Order findByUserNameAndSkuName(String userName, String skuName);

    /**
     * 使用Query注解指定查询语句
     * @param userName
     * @param skuName
     * @return
     */
    //双引号和不加引号都可，不能是单引号
//    @Query("{bool : {must : [ {field : {userName : ?}}, {field : {skuName : ?}} ]}}")   . //---   field查询已经废弃，可参考当前查询语法，已换成term查询
    @Query("{\"bool\" : {\"must\" : [ {\"term\" : {\"skuName\" : \"?1\"}}, {\"term\" : {\"userName\" : \"?0\"}} ]}}")
    //注意：需要替换的参数？需要加双引号；需要指定参数下标，从0开始
    public Order findByUserNameAndSkuName2(String userName, String skuName);

    //还有分页、排序等API
}
```
**实现类**：自定义了创建索引的方法，其余基础repository接口中的方法无需实现
``` java
/**
 * 自定义Repository实现类
 *
 * 接口的实现类名字后缀必须为impl才能在扫描包时被找到（可参考spring data elasticsearch自定义repository章节）
 *
 */
public class OrderRepositoryImpl implements OrderEsCommonRepository {

    private ElasticsearchTemplate elasticsearchTemplate;

    /**
     * 创建索引
     * @return
     */
    public boolean createOrderIndex() {
        return elasticsearchTemplate.createIndex(Order.class);
    }


    //自定义实现可以使用ElasticsearchTemplate做复杂的查询，例如：分组、聚合等，
    //增加了spring data elasticsearch的灵活度，使用方法名定义和Query注解实现困难的查询操作，可借助ElasticsearchTemplate实现自定义的查询


    public void setElasticsearchTemplate(ElasticsearchTemplate elasticsearchTemplate) {
        this.elasticsearchTemplate = elasticsearchTemplate;
    }
}
```
>实现类必须是基础查询接口加Impl（后缀Impl可配置）  

## 2.5 总结
Spring Data ElasticSearch提供了基于接口声明的方式实现查询、基于注解方式查询，对于复杂的查询或者其他ES操作，也可以使用自定义的repository实现。提供多种查询方式，可以根据需求选择使用。使用也非常方便，相对于JAVA API方式，能减少很多重复代码。  

**参考资料：**  
[Introduction to Spring Data Elasticsearch](http://www.baeldung.com/spring-data-elasticsearch-tutorial)  
[官方文档](http://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/)：包含spring data repository、自定义repository  
[官方文档中文版](http://es.yemengying.com/)  
[Elasticsearch 权威指南（中文版）](http://es.xiaoleilu.com/)  

>最后附上实例程序下载地址：[http://download.csdn.net/download/liuxigiant/9593796]()