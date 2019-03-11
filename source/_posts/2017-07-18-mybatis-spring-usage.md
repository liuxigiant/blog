---
layout: post
title: "MyBatis-Spring简介与用法"
description: mybatis-spring简介与用法
date: 2017-07-18 10:42:05 +0800
catalog: true
categories:
- MyBatis Spring
tags:
- MyBatis Spring
---

JAVA企业级应用，多以Spring为基础，集成其他开源组件构建。在ORM（Object Relational Mapping）层，Spring提供了对主流ORM工具（Hibernate、iBatis、JPA等）的集成支持。  

> Spring对iBatis的支持只到Spring 3.x版本，Spring 4.x不包含集成iBatis的模块（从[Spring源码](https://github.com/spring-projects/spring-framework)的`spring-orm`模块可清晰看到当前支持集成的ORM工具）。Spring 4.x版本建议使用MyBatis(2010年iBatis更名为MyBatis)作为ORM工具。Spring与MyBatis的集成，需要引入`MyBatis_spring`包。  

# 1 MyBatis-Spring简介  

下面对MyBatis-Spring做简要介绍，内容摘选自MyBatis-Spring[中文官方文档](http://www.mybatis.org/spring/zh/index.html)  
  
> 另：可参考[英文官方文档](http://www.mybatis.org/spring/index.html)，中文官方文档部分内容不是最新  

## 1.1 什么是MyBatis-Spring  

MyBatis-Spring 会帮助你将 MyBatis 代码无缝地整合到 Spring 中。 使用这个类库中的类, Spring 将会加载必要的 MyBatis 工厂类和 session 类。 这个类库也提供一个简单的方式来注入 MyBatis 数据映射器和 SqlSession 到业务层的 bean 中。 而且它也会处理事务, 翻译 MyBatis 的异常到 Spring 的 DataAccessException 异常(数据访问异常,译者注)中。最终,它并 不会依赖于 MyBatis,Spring 或 MyBatis-Spring 来构建应用程序代码。  

## 1.2 为何创建MyBatis-Spring 

正如第二版那样,Spring 3.0 也仅支持 iBatis2。那么,我们就想将 MyBatis3 的支持添加 到 Spring3.0(参考 Spring Jira 中的问题)中。而不幸的是,Spring 3.0 的开发在 MyBatis 3.0 官方发布前就结束了。 因为 Spring 开发团队不想发布一个基于非发布版的 MyBatis 的整合支 持,那么 Spring 官方的支持就不得不继续等待了。要在 Spring 中支持 MyBatis,MyBatis 社 区认为现在应该是自己团结贡献者和有兴趣的人一起来开始将 Spring 的整合作为 MyBatis 社 区的子项目的时候了。  

# 2 MyBatis-Spring用法  

如上所述，在以Sprig为基础构建的JAVA企业级应用中，ORM层选用MyBatis框架，需使用`MyBatis-Spring`将Spring与MyBatis集成。下面总结了四种集成的使用方式，从这四种方式的演进历程，可以清晰的了解`MyBatis-Spring`的改进，简化配置，减少无用功。  

> 背景：在了解使用方法之前，先简要说明下Mybatis的基本用法，即根据MyBatis配置生成SqlSessionFactory，从SqlSessionFactory中获取session，最后使用session完成数据库操作。那么，MyBatis-Spring的作用就是封装这些动作，并把MyBatis集成到Spring中（可回过头看看1.1章节）。  

## 2.1 SqlSessionFactory生成  

如上背景介绍，`SqlSessionFactory`是使用MyBatis的核心入口类，`MyBatis-Spring`是通过工厂类`SqlSessionFactoryBean`来生成`SqlSessionFactory`的。

>熟悉Spring IOC的对以`FactoryBean`结尾的类可能会比较敏感，`FactoryBean`是Spring声明的一个工厂类接口，Spring IOC在从对象工厂中获取实例（getBean，具体实现逻辑可参考Spring IOC源码），会调用`getObject`方法  

如下配置文件`spring-config-datasource.xml`，配置了数据源（数据源使用druid，配置信息简化过）和`SqlSessionFactoryBean`  

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
              http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

       <!-- mybatis配置 -->
       <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
              <property name="dataSource" ref="dataSource" />
              <property name="configLocation" value="classpath:mybatis-configure.xml" />
       </bean>

       <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
              <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
              <property name="url" value="jdbc:mysql://localhost:3306/test?characterEncoding=UTF-8"/>
              <property name="username" value="root"/>
              <property name="password" value="root"/>
              <!--数据源名字-->
              <property name="name" value="ds_master"/>
       </bean>
</beans>
```

## 2.2 用法一：在Dao实现类中注入SqlSessionTemplate  

如下：`OrderDaoImpl`中注入了`sqlSessionTemplate`  

> 注意：  `SqlSessionTemplate`的方法中的第一个参数`statement`，指定sql语句，一般都是对用的Mapper文件的`命名空间.sql语句id`。当然，也可直接用sql语句id，不加命名空间，但不建议这么做，回存在sql覆盖的问题，详细分析可查看下篇`mybatis-spring源码解析之创建SqlSessionFactory`~~

```java
public class OrderDaoImpl implements OrderDao {

	private SqlSessionTemplate sqlSessionTemplate;

	@Override
	public Order searchOrderById(Long id) {
		return sqlSessionTemplate.selectOne("name.liux.share.dao.OrderDao.searchOrderById", id);
	}

	public SqlSessionTemplate getSqlSessionTemplate() {
		return sqlSessionTemplate;
	}

	public void setSqlSessionTemplate(SqlSessionTemplate sqlSessionTemplate) {
		this.sqlSessionTemplate = sqlSessionTemplate;
	}
}
```

配置文件`spring-config-dao-template.xml`中声明了`sqlSessionTemplate`,并将上上面配置的`sqlSessionFactory`注入到`sqlSessionTemplate`中，再将`sqlSessionTemplate`注入到`orderDaoImpl`中  

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
              http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

       <import resource="spring-config-datasource.xml"/>

       <bean id="sqlSessionTemplate" class="org.mybatis.spring.SqlSessionTemplate">
              <constructor-arg index="0" ref="sqlSessionFactory" />
       </bean>

       <bean id="orderDaoImpl" class="name.liux.share.dao2.impl.OrderDaoImpl">
              <property name="sqlSessionTemplate" ref="sqlSessionTemplate"/>
       </bean>

</beans>
```

**总结：**  

1. 需要在每个`*DaoImpl`中注入`SqlSessionTemplate`，然后使用`SqlSessionTemplate`完成数据库操作，配置比较繁琐  
2. 为何需要使用`SqlSessionTemplate`，而不直接使用`sqlSessionFactory`？  -- 因为使用`SqlSessionTemplate`能封装事物操作并使用Spring框架的事物管理器（后面讲解源码会说明）  
3. 还有种异曲同工的用法，让每个`*DaoImpl`继承`SqlSessionDaoSupport`，通过继承的方式，则在`*DaoImpl`需要注入`sqlSessionTemplate`或者`sqlSessionFactory`，二者注入一个即可（若注入的是`sqlSessionFactory`，也会通过`sqlSessionFactory`创建一个`sqlSessionTemplate`，所以最终是要的还是`sqlSessionTemplate`）

## 2.3 用法二：配置MapperFactoryBean，指定生成Dao实现  

下面以`UserDao`为例说明，这只是一个接口，声明了数据库操作，但是没有实现类，我们通过配置`MapperFactoryBean`为`UserDao`生成一个实现类，完成具体的数据库操作，从而去除了大量的`*DaoImpl`的代码。  

```java
public interface UserDao {

	User searchUserById(Long userId);
}
```

通过配置生成对应Dao的实现类  

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
              http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
              http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

       <!-- 自动扫描 -->
       <context:component-scan base-package="name.liux.share" />

       <import resource="spring-config-datasource.xml"/>

       <bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean" >
              <property name="mapperInterface" value="name.liux.share.dao.UserDao" />
              <property name="sqlSessionFactory" ref="sqlSessionFactory" />
       </bean>

</beans>
```

**总结：**  

1. 这种使用方式简化了`*DaoImpl`代码的编写，通过`MapperFactoryBean`（和前面类似，这个也是个工厂类，通过`getObject`生成类实例）生成对用Dao接口的实现类（其实底层就是通过JDK的动态代理实现的，所以配置中需要注入`mapperInterface`，指定代理类代理的接口）
2. 从配置中可以看出来，依然需要注入`sqlSessionTemplate`或者`sqlSessionFactory`(可参照用法一总结第3点)
3. 对于多个`*DaoImpl`，需要配置多个代理工厂类`MapperFactoryBean`，会有很多配置工作  

## 2.4 用法三：配置MapperScannerConfigurer，指定扫描，生成代理  

这种方式相对**用法二**来说，简化了配置多个`*DaoImpl`对应的代理工厂类`MapperFactoryBean`，通过配置包路径，扫描包下面的接口，自动生成接口对应的代理类  

接口声明如上`UserDao`，配置文件如下  

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
              http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
              http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

       <!-- 自动扫描 -->
       <context:component-scan base-package="name.liux.share" />

       <import resource="spring-config-datasource.xml"/>

       <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
              <property name="basePackage" value="name.liux.share.dao" />
       </bean>
</beans>
```

**总结：**  

1. 扫描类需要指定包路径扫描
2. `MapperScannerConfigurer`扫描类通过实现Spring生命周期回调接口`BeanDefinitionRegistryPostProcessor`，在回调方法`postProcessBeanDefinitionRegistry`中扫描指定包下的接口，修改接口的Spring bean声明元数据BeanDefinition，重点是将BeanDefinition中的beanClass属性设置为MapperFactoryBean，从而实现对每个接口生成一个代理类工厂（后续通过分析源码详细说明）
3. 配置`MapperScannerConfigurer`并没有注入`sqlSessionTemplate`或者`sqlSessionFactory`，是通过将第2点中说的BeanDefinitio的autowireMode属性设置为`AUTOWIRE_BY_TYPE`，即根据类型自动注入  

## 2.5 用法四：指定扫描路径，自动扫描  

这种方式其实是**用法三**的一个配置简化版，通过自定义扩展Spring Schema，简化声明方式，实现原理和**用法三**一致

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mybatis="http://mybatis.org/schema/mybatis-spring"
       xsi:schemaLocation="
              http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
              http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
              http://mybatis.org/schema/mybatis-spring http://mybatis.org/schema/mybatis-spring.xsd">

       <!-- 自动扫描 -->
       <context:component-scan base-package="name.liux.share" />

       <import resource="spring-config-datasource.xml"/>

       <mybatis:scan base-package="name.liux.share.dao" />
</beans>
```

## 2.6 总结  

1. **用法一**：通过读取配置生成`sqlSessionFactory`，并封装成`sqlSessionTemplate`，实现数据库操作。其实就是对mybatis使用的一个简单封装，并完成与Spring框架的集成  
2. **用法二**：通过JDK动态代理，由工厂类生成Dao层接口对应的代理实现类，简化Dao层代码（去*DaoImpl类）
3. **用法三**：自动扫描包路径，生成对应包下接口的代理工厂类，简化代理工厂类配置
4. **用法四**：扩展Spring Schema，再次简化配置  

> 后续源码解析按照以上四种使用方法逐一了解