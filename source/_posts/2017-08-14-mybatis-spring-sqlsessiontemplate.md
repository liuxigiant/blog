---
layout: post
title: "MyBatis-Spring源码之SqlSessionTemplate"
description: mybatis-spring使用SqlSessionTemplate提供数据库操作模板方法
date: 2017-08-14 14:28:44 +0800
catalog: true
categories:
- MyBatisSpring
tags:
- MyBatisSpring
---

在MyBatis中，可以使用`SqlSessionFactory`来创建`SqlSession`。一旦你获得一个 session 之后，你可以使用它来执行映射语句，提交或回滚连接，最后，当不再需要它的时候，你可以关闭 session。  
**那么，为什么还需要使用SqlSessionTemplate呢？**  

`SqlSessionTemplate` 是 MyBatis-Spring 的核心。 这个类负责管理 MyBatis 的 `SqlSession`, 调用 MyBatis 的 SQL 方法, 翻译异常。 `SqlSessionTemplate` 是线程安全的, 可以被多个 DAO 所共享使用。  

当调用 SQL 方法时, 包含从映射器 getMapper()方法返回的方法, SqlSessionTemplate 将会保证使用的 SqlSession 是和当前 **Spring 的事务**相关的。此外，它管理 session 的生命周期，包含必要的关闭，提交或回滚操作。  

> 以上摘选自[Spring-MyBatis中文官方文档](http://www.mybatis.org/spring/zh/sqlsession.html#SqlSessionTemplate)  

下面从源码角度了解下`SqlSessionTemplate`对`SqlSession`的管理以及与Spring事务的集成。  

# 1 配置  

先回顾下`SqlSessionTemplate`在Spring配置中的声明方式，如下：  

```xml
<bean id="sqlSessionTemplate" class="org.mybatis.spring.SqlSessionTemplate">
      <constructor-arg index="0" ref="sqlSessionFactory" />
</bean>
```  

# 2 构造方法  

如上配置，通过构造方法把`sqlSessionFactory`注入到`SqlSessionTemplate`中，下面看看构造方法源码：  

```java
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
	this(sqlSessionFactory, sqlSessionFactory.getConfiguration().getDefaultExecutorType());
}
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType) {
	this(sqlSessionFactory, executorType,
	    new MyBatisExceptionTranslator(
	        sqlSessionFactory.getConfiguration().getEnvironment().getDataSource(), true));
}
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {	
	notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
	notNull(executorType, "Property 'executorType' is required");

	//持有sqlSessionFactory变量
	this.sqlSessionFactory = sqlSessionFactory;
	this.executorType = executorType;
	this.exceptionTranslator = exceptionTranslator;

	//生成SqlSession的代理类，代理类执行方法都需要执行拦截器SqlSessionInterceptor，这个拦截器是实现SqlSession管理的重点
	this.sqlSessionProxy = (SqlSession) newProxyInstance(
	    SqlSessionFactory.class.getClassLoader(),
	    new Class[] { SqlSession.class },
	    new SqlSessionInterceptor());
}
```  

# 3 常用方法  

使用MyBatis框架就是为了方法操作数据库，`SqlSessionTemplate`通过实现`SqlSession`接口，暴露了丰富的数据库操作API，下面看看在`SqlSessionTemplate`类中这些方法的实现方式：  

```java
//查询返回集合
public <E> List<E> selectList(String statement, Object parameter) {

	//由代理类完成数据库操作
	return this.sqlSessionProxy.<E> selectList(statement, parameter);
}
//提交、回滚和关闭连接会抛异常，这些由代理类的回调方法完成（实现与Spring事务管理的集成）
public void commit() {
	throw new UnsupportedOperationException("Manual commit is not allowed over a Spring managed SqlSession");
}
public void rollback() {
	throw new UnsupportedOperationException("Manual rollback is not allowed over a Spring managed SqlSession");
}
public void close() {
	throw new UnsupportedOperationException("Manual close is not allowed over a Spring managed SqlSession");
}
```

# 4 回调拦截器SqlSessionInterceptor  


从`SqlSessionTemplate`的构造方法和数据库操作方法可看出，`SqlSessionTemplate`是Mybatis Spring提供的一个数据库操作**模板类**，具体数据库操作由`SqlSession`**代理类**完成，代理类的创建随`SqlSessionTemplate`对象的创建而创建，而对Mybatis SqlSession的管理，是由作代理类的回调拦截器`SqlSessionInterceptor`完成的。  

代理类的创建：  

```java
this.sqlSessionProxy = (SqlSession) newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class },
        new SqlSessionInterceptor());
```  

代理类的回调拦截器`SqlSessionInterceptor`实现如下：  

```java
//实现InvocationHandler接口，对代理类方法的调用都会执行回调方法invoke
private class SqlSessionInterceptor implements InvocationHandler {
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

	  //获取sqlsession，详见SqlSessionUtils#getSqlSession
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {

        //执行被代理类方法（真正的方法调用，其实就是调用sqlSession的目标方法）
        Object result = method.invoke(sqlSession, args);
		//如果sqlSession没有关联事务，则提交
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {

		//异常转换
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        if (sqlSession != null) {

		  //关闭数据库链接，详见SqlSessionUtils#closeSqlSession
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }
```  

# 5 事务管理  

Mybatis Spring通过集成Spring事务管理的机制来实现事务的管理，真正的事务管理也是交由Spring事务管理器（Spring的`PlatformTransactionManager`）来管理的。在上面拦截器的回调方法中，获取session和关闭session都是需要进行事务判断。  

下面了解下获取和关闭session的源码：  

```java
//获取session
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
    notNull(sessionFactory, "No SqlSessionFactory specified");
    notNull(executorType, "No ExecutorType specified");

	//当前sessionFactory是否有事务绑定的session（SqlSessionHolder是session的持有对象）
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    //存在绑定的session，且session与事务关联
    if (holder != null && holder.isSynchronizedWithTransaction()) {
      if (holder.getExecutorType() != executorType) {
        throw new TransientDataAccessResourceException("Cannot change the ExecutorType when there is an existing transaction");
      }

      //session引用次数加1
      holder.requested();
      if (logger.isDebugEnabled()) {
        logger.debug("Fetched SqlSession [" + holder.getSqlSession() + "] from current transaction");
      }

      //返回session -- 其实就是当前sessionFactory有与事务关联的session，则返回
      return holder.getSqlSession();
    }
    if (logger.isDebugEnabled()) {
      logger.debug("Creating a new SqlSession");
    }

	//从sessionFactory中创建session  -- 这里是session真正被创建的地方
    SqlSession session = sessionFactory.openSession(executorType);
    // Register session holder if synchronization is active (i.e. a Spring TX is active)
    //
    // Note: The DataSource used by the Environment should be synchronized with the
    // transaction either through DataSourceTxMgr or another tx synchronization.
    // Further assume that if an exception is thrown, whatever started the transaction will
    // handle closing / rolling back the Connection associated with the SqlSession.

    //当前线程是否存在事务
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
      Environment environment = sessionFactory.getConfiguration().getEnvironment();
      if (environment.getTransactionFactory() instanceof SpringManagedTransactionFactory) {
        if (logger.isDebugEnabled()) {
          logger.debug("Registering transaction synchronization for SqlSession [" + session + "]");
        }

		//封装持有session对象 
        holder = new SqlSessionHolder(session, executorType, exceptionTranslator);

		//sessionFactory与session绑定
        TransactionSynchronizationManager.bindResource(sessionFactory, holder);

		//注册当前线程事务绑定
        TransactionSynchronizationManager.registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));

		//设置session被事务管理
        holder.setSynchronizedWithTransaction(true);

		//session引用次数加1
        holder.requested();
      } else {
        if (TransactionSynchronizationManager.getResource(environment.getDataSource()) == null) {
          if (logger.isDebugEnabled()) {
            logger.debug("SqlSession [" + session + "] was not registered for synchronization because DataSource is not transactional");
          }
        } else {
          throw new TransientDataAccessResourceException(
              "SqlSessionFactory must be using a SpringManagedTransactionFactory in order to use Spring transaction synchronization");
        }
      }

	//不存在事务，则打印日志
    } else {
      if (logger.isDebugEnabled()) {
        logger.debug("SqlSession [" + session + "] was not registered for synchronization because synchronization is not active");
      }
    }
    return session;
}

//关闭session
public static void closeSqlSession(SqlSession session, SqlSessionFactory sessionFactory) {
	notNull(session, "No SqlSession specified");
	notNull(sessionFactory, "No SqlSessionFactory specified");

	//当前sessionFactory是否有事务绑定的session（SqlSessionHolder是session的持有对象）
	SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
	if ((holder != null) && (holder.getSqlSession() == session)) {
	  if (logger.isDebugEnabled()) {
	    logger.debug("Releasing transactional SqlSession [" + session + "]");
	  }

	  //存在事务，则引用次数减1
	  holder.released();
	} else {
	  if (logger.isDebugEnabled()) {
	    logger.debug("Closing non transactional SqlSession [" + session + "]");
	  }

	  //不存在事务，则关闭session
	  session.close();
	}
}
```

看完上述源码，再来理解下事务管理：  

事务的管理（事务的开启、提交、回滚）是由Spring实现的，而获取和关闭连接（在Mybatis/ibatis中是session）是有具体的持久层框架来实现的.  

Spring实现了多种持久层框架的集成（ibatis，Hibernate等），当然也实现了这些持久层框架集成后事务的管理。  

Mybatis Spring事务管理的思路，就是使用Spring的事务管理机制