---
layout: post
title: "MyBatis-Spring源码之SqlSessionFactory"
description: mybatis-spring通过读取配置文件信息，由对象工厂类创建SqlSessionFactory
date: 2017-08-02 10:06:33 +0800
catalog: true
categories:
- MyBatis Spring
tags:
- MyBatis Spring
---

MyBatis-Spring 封装了`SqlSessionFactory`的创建过程，通过读取、解析Mybatis XML配置文件，并将配置信息的元数据封装到`Configuration`对象中，最后通过对象工厂类`SqlSessionFactoryBean`创建`SqlSessionFactory`。  

> Mybatis XML配置文件元数据封装成`Configuration`和Spring XML配置文件封装成`BeanDefinition`的设计思路很相似，都是将XML配置文件解析成对象。　　

# 1 关键类说明　　

`SqlSessionFactory`的创建过程涉及主要类说明如下：　　

- **SqlSessionFactory**：MyBatis操作数据库的核心类
- **SqlSessionFactoryBean**：创建`SqlSessionFactory`的对象工厂类（看见Spring工厂类，很自然的应该找`getObject`方法）
- **Configuration**：MyBatis XML配置文件解析后的元数据对象，是后续完成操作数据库的基础  
- **TypeHandlerRegistry**：MyBatis XML配置文件解析过程中，注册并持有MyBatis类型转换类TypeHandler
- **TypeAliasRegistry**： MyBatis XML配置文件解析过程中，注册并持有MyBatis别名的声明
- **MappedStatement** : MyBatis XML配置文件解析过程中，封装Mapper配置文件中sql语句
- **ResultMap** : MyBatis XML配置文件解析过程中，封装Mapper配置文件中声明的数据库查询结果映射  

> MyBatis XML中还有很多其他配置元素，上面说明针对常用用法涉及的核心类做说明，下面关注源码，也只关注着部分  

各核心类相互关系如下图：
![](http://i.imgur.com/88FlOgn.png)

# 2 源码分析  

`SqlSessionFactory`是由对象工厂类`SqlSessionFactoryBean`创建的，配置方式如下：  

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
      <property name="dataSource" ref="dataSource" />
      <property name="configLocation" value="classpath:mybatis-configure.xml" />
</bean>
```

## 2.1 SqlSessionFactory对象创建

上面有说到看见Spring工厂类，很自然的应该找`getObject`方法，`SqlSessionFactoryBean`的`getObject`方法如下：  
```java
public SqlSessionFactory getObject() throws Exception {
	if (this.sqlSessionFactory == null) {
	  afterPropertiesSet();
	}
	
	return this.sqlSessionFactory;
}
```

很显然，核心方法变成`afterPropertiesSet`方法，`afterPropertiesSet`方法是通过调用`buildSqlSessionFactory`方法完成对象创建
> `SqlSessionFactoryBean`实现了Spring的`InitializingBean`接口，在Spring完成Bean初始化后，会触发回调函数`afterPropertiesSet`。`SqlSessionFactoryBean`中的`getObject`方法只是在`SqlSessionFactory`为null情况下再显示调用一次  

`buildSqlSessionFactory`完成XML文件解析和`sqlSessionFactory`的创建，以下源码摘选重要部分  
```java
Configuration configuration;

XMLConfigBuilder xmlConfigBuilder = null;

//configLocation为XML文件中配置的MyBatis配置文件路径
if (this.configLocation != null) {

  //初始化XML解析工具类，真正的解析在下面调用
  xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);

  //这里的Configuration是创建解析工具类时候，new出来的一个默认的对象
  configuration = xmlConfigBuilder.getConfiguration();
} else {
  if (logger.isDebugEnabled()) {
    logger.debug("Property 'configLocation' not specified, using default MyBatis Configuration");
  }
  configuration = new Configuration();
  configuration.setVariables(this.configurationProperties);
}


//.....省略


if (xmlConfigBuilder != null) {
  try {

	//解析XML文件，并封装到Configuration对象中
    xmlConfigBuilder.parse();

    if (logger.isDebugEnabled()) {
      logger.debug("Parsed configuration file: '" + this.configLocation + "'");
    }
  } catch (Exception ex) {
    throw new NestedIOException("Failed to parse config resource: " + this.configLocation, ex);
  } finally {
    ErrorContext.instance().reset();
  }
}



//.....省略


//根据封装完成的Configuration，创建并返回SqlSessionFactory（new DefaultSqlSessionFactory）
return this.sqlSessionFactoryBuilder.build(configuration);
```


## 2.2 XMLConfigBuilder初始化  


**XMLConfigBuilder**：将XML解析后的对象解析封装到元数据对象`Configuration`中  

**XPathParser**：解析XML文件成Docment对象，并使用xpath方式读取XML文件

下面看下`XMLConfigBuilder`的初始化过程：  

```java
public XMLConfigBuilder(InputStream inputStream, String environment, Properties props) {
	//根据配置文件输入流，创建一个XPathParser，用于XML解析
    this(new XPathParser(inputStream, true, props, new XMLMapperEntityResolver()), environment, props);
}
private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
	//初始化配置元数据对象
	super(new Configuration());
	ErrorContext.instance().resource("SQL Mapper Configuration");
	this.configuration.setVariables(props);
	this.parsed = false;
	this.environment = environment;
	//持有XML文件解析工具类
	this.parser = parser;
}
```


初始化过程重点是以下两点：  

-  初始化`XPathParser`： 通过输入流创建XML文件解析工具（可查看`XPathParser`构造函数源码，其实就是讲XML输入流转换成`Document`对象，并生成一个`XPth`来读取XML文件）  
-  初始化配置元数据对象`Configuration`：直接创建配置对象（可查看`Configuration`构造函数源码，在创建对象时会注册一些默认的别名）  

## 2.3 配置文件解析  

`XMLConfigBuilder`对象初始化完成后，就具备了解析封装配置元数据的基础，在如上所说的`SqlSessionFactoryBean#buildSqlSessionFactory`方法中，调用`xmlConfigBuilder.parse()`，完成XML配置信息到`Configuration`元数据对象的封装。  

下面我们先看看`XMLConfigBuilder#parse`方法，源码如下：    

```java
//已经解析过，则抛出异常
if (parsed) {
  throw new BuilderException("Each XMLConfigBuilder can only be used once.");
}
parsed = true;
//从根节点configuration开始递归解析
parseConfiguration(parser.evalNode("/configuration"));
return configuration;
```

`XPathParser#evalNode`根据`XPath`读取`Document`对象，不做说明，重点看`parseConfiguration`方法，源码如下：  

```java
try {
  propertiesElement(root.evalNode("properties")); //issue #117 read properties first
  //解析XML配置文件typeAliases节点，注册别名
  typeAliasesElement(root.evalNode("typeAliases"));
  pluginElement(root.evalNode("plugins"));
  objectFactoryElement(root.evalNode("objectFactory"));
  objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
  settingsElement(root.evalNode("settings"));
  environmentsElement(root.evalNode("environments")); // read it after objectFactory and objectWrapperFactory issue #631
  databaseIdProviderElement(root.evalNode("databaseIdProvider"));
  //解析XML配置文件typeHandlers节点，注册类型转换器
  typeHandlerElement(root.evalNode("typeHandlers"));
  //解析XML配置文件mappers节点，mappers节点下一般配置多个mapper文件路径
  mapperElement(root.evalNode("mappers"));
} catch (Exception e) {
  throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
}
```

如上源码，`parseConfiguration`方法清晰的罗列出了XML配置文件各个节点解析封装的流程，按照一般使用方式，重点关注别名解析、类型转换器解析、Mapper文件解析。  

## 2.4 别名注册  

**别名注册**主要是解析`typeAliases`节点下的`typeAlias`节点，并注册封装到元数据对象`Configuration`中。开篇类图中可看出，`Configuration`持有一个`TypeAliasRegistry`,`TypeAliasRegistry`持有一个集合`Map<String, Class<?>>`维护别名与类型的映射。  

别名解析注册源码如下：  

```java
private void typeAliasesElement(XNode parent) {
    if (parent != null) {
      //循环遍历解析子节点typeAlias
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String typeAliasPackage = child.getStringAttribute("name");
          configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
        } else {
          //获取配置的别名属性alias
          String alias = child.getStringAttribute("alias");
          //获取配置的类型属性type
          String type = child.getStringAttribute("type");
          try {
            Class<?> clazz = Resources.classForName(type);
            if (alias == null) {
              //未指定alias属性的直接根据类型注册
              typeAliasRegistry.registerAlias(clazz);
            } else {
			  //指定别名和类型注册
              typeAliasRegistry.registerAlias(alias, clazz);
            }
          } catch (ClassNotFoundException e) {
            throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
          }
        }
      }
    }
  }
```

- 别名注册其实就将别名和类型维护到Map中，方便在Mapper中直接使用别名来代替类型
- 别名注册`TypeAliasRegistry`实例化时候会注册一些基本的类型对应的别名
- 只根据类型注册，别名会默认为类的简单名称，若使用`Alias`注解，别名则为注解指定的名称（可查看源码`registerAlias(Class<?> type)`）  

## 2.5 类型转换器注册  

**类型转换器注册**主要是解析`typeHandlers`节点下的`typeHandler`子节点，并注册封装到元数据对象`Configuration`中。。开篇类图中可看出，`Configuration`持有一个`TypeHandlerRegistry`,`TypeHandlerRegistry`持有一个集合`Map<Type, Map<JdbcType, TypeHandler<?>>>`,外层Map的键是java类型，内层集合的键是JDBC类型，其实就是java类型与JDBC类型的转换关系对应的类型转换工具类`TypeHandler`。   


类型转换器注册源码如下：  

```java
private void typeHandlerElement(XNode parent) throws Exception {
	if (parent != null) {
      ////循环遍历解析子节点typeHandler
	  for (XNode child : parent.getChildren()) {
	    if ("package".equals(child.getName())) {
	      String typeHandlerPackage = child.getStringAttribute("name");
	      typeHandlerRegistry.register(typeHandlerPackage);
	    } else {
          //获取配置的java类型属性javaType
	      String javaTypeName = child.getStringAttribute("javaType");
          //获取配置的JDBC类型属性jdbcType
	      String jdbcTypeName = child.getStringAttribute("jdbcType");
          //获取配置的类型转换属性名称
	      String handlerTypeName = child.getStringAttribute("handler");
	      Class<?> javaTypeClass = resolveClass(javaTypeName);
	      JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
	      Class<?> typeHandlerClass = resolveClass(handlerTypeName);
	      if (javaTypeClass != null) {
	        if (jdbcType == null) {
              //根据java类型注册转换器
	          typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
	        } else {
			  //根据java类型和JDBC类型注册转换器
	          typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
	        }
	      } else {
            //直接注册转换器
	        typeHandlerRegistry.register(typeHandlerClass);
	      }
	    }
	  }
	}
}
```

- 类型转换器注册类`TypeHandlerRegistry`实例化时候会注册一些基本的类型转换
- 默认情况下注册类型转换器到集合`Map<Type, Map<JdbcType, TypeHandler<?>>>`中需要java类型、jdbc类型。从`typeHandlerRegistry.register`的多个重载方法可看出：若没有java类型，则在转换器类上查找`MappedTypes`注解或者实现`TypeReference`接口，否则为null；若没有jdbc类型，则在类型转换器类上查找`MappedJdbcTypes`注解，否则为null。
- `TypeHandlerRegistry`中还持有一个集合`Map<JdbcType, TypeHandler<?>>`，维护JDBC类型与转换器类的映射，在`TypeHandlerRegistry`实例化注册时添加  

## 2.6 Mapper文件解析  

Mybatis配置文件解析中，核心就是**Mapper文件解析**。Mapper文件中一般包含结果集映射、公用sql块、sql语句的声明。  

常用的配置方式是以`resource`方式配置，`mappers`节点下可声明多个`mapper`文件，配置如下：  

```xml
<mappers>
      <mapper resource="mappers/Order.xml"/>
      <mapper resource="mappers/User.xml"/>
</mappers>
```

下面看看`mappers`节点解析的源码实现方式，重点只关注`resource`方式配置的解析方式：  

```java
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {

      //循环mappers节点子节点集合，依次解析mapper节点
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {

          //获取mapper文件resource配置路径
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);

            //获取mapper配置文件输入流
            InputStream inputStream = Resources.getResourceAsStream(resource);

            //XMLMapperBuilder用于解析mapper文件配置，并封装到元数据对象Configuration(解析实现方式与XMLConfigBuilder类似)
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());

			//和XMLConfigBuilder类似，parse方法才是解析并封装数据的核心
            mapperParser.parse();
          } 


			//.....省略


        }
      }
    }
  }
```

XMLMapperBuilder#parse方法源码如下：  

```java
public void parse() {
    
    //判断mapper文件是否已经解析过
	if (!configuration.isResourceLoaded(resource)) {
      
      //从根节点mapper开始解析
	  configurationElement(parser.evalNode("/mapper"));

      //完成mapper文件解析
	  configuration.addLoadedResource(resource);
	  bindMapperForNamespace();
	}
	
	parsePendingResultMaps();
	parsePendingChacheRefs();
	parsePendingStatements();
}
```

mapper文件解析入口，重点关注常用节点（结果集映射、公用sql块、sql语句）的解析：  

```java
private void configurationElement(XNode context) {
	try {
	  String namespace = context.getStringAttribute("namespace");
	  if (namespace.equals("")) {
		  throw new BuilderException("Mapper's namespace cannot be empty");
	  }
	  builderAssistant.setCurrentNamespace(namespace);
	  cacheRefElement(context.evalNode("cache-ref"));
	  cacheElement(context.evalNode("cache"));
	  parameterMapElement(context.evalNodes("/mapper/parameterMap"));

      //结果映射解析
	  resultMapElements(context.evalNodes("/mapper/resultMap"));

      //公用sql块解析
	  sqlElement(context.evalNodes("/mapper/sql"));

      //sql语句解析
	  buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
	} catch (Exception e) {
	  throw new BuilderException("Error parsing Mapper XML. Cause: " + e, e);
	}
}
```


### 2.6.1 结果集映射resultMap解析  

若有多个resultMap声明，则循环依次解析：  

```java
private void resultMapElements(List<XNode> list) throws Exception {
	for (XNode resultMapNode : list) {
	  try {
		resultMapElement(resultMapNode);
	  } catch (IncompleteElementException e) {
		// ignore, it will be retried
	  }
	}
}
```

resultMap解析过程如下（可对照一个配置文件理解解析过程）：  

```java
private ResultMap resultMapElement(XNode resultMapNode) throws Exception {
    return resultMapElement(resultMapNode, Collections.<ResultMapping> emptyList());
  }

  private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings) throws Exception {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());

    //获取结果集映射resultMap的id
    String id = resultMapNode.getStringAttribute("id",
        resultMapNode.getValueBasedIdentifier());

    //获取结果集映射resultMap的类型
    String type = resultMapNode.getStringAttribute("type",
        resultMapNode.getStringAttribute("ofType",
            resultMapNode.getStringAttribute("resultType",
                resultMapNode.getStringAttribute("javaType"))));

    String extend = resultMapNode.getStringAttribute("extends");
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
    Class<?> typeClass = resolveClass(type);
    Discriminator discriminator = null;

	//mapper文件中结果集resultMap下的子节点（一般是result）解析后的元数据集合
    List<ResultMapping> resultMappings = new ArrayList<ResultMapping>();
    resultMappings.addAll(additionalResultMappings);
    List<XNode> resultChildren = resultMapNode.getChildren();

    //循环解析各个字段的映射关系，其实就是mapper文件中结果集resultMap下的子节点（一般是result）
    for (XNode resultChild : resultChildren) {
      if ("constructor".equals(resultChild.getName())) {
        processConstructorElement(resultChild, typeClass, resultMappings);
      } else if ("discriminator".equals(resultChild.getName())) {
        discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
      } else {
        ArrayList<ResultFlag> flags = new ArrayList<ResultFlag>();
        if ("id".equals(resultChild.getName())) {
          flags.add(ResultFlag.ID);
        }

        //mapper文件中结果集resultMap下的子节点（一般是result）解析、封装元数据对象
        resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
      }
    }
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator, resultMappings, autoMapping);
    try {

      //封装resultMap节点解析后的元数据对象ResultMap，并添加到Mybatis配置文件元数据对象Configuration中
      return resultMapResolver.resolve();
    } catch (IncompleteElementException  e) {
      configuration.addIncompleteResultMap(resultMapResolver);
      throw e;
    }
  }

//mapper文件中结果集resultMap下的子节点（一般是result）解析
private ResultMapping buildResultMappingFromContext(XNode context, Class<?> resultType, ArrayList<ResultFlag> flags) throws Exception {

    //获取result节点配置的各个属性
    String property = context.getStringAttribute("property");
    String column = context.getStringAttribute("column");
   
	//.....省略


    //将获取的result节点的各个属性声明封装到元数据对象ResultMapping中
    return builderAssistant.buildResultMapping(resultType, property, column, javaTypeClass, jdbcTypeEnum, nestedSelect, nestedResultMap, notNullColumn, columnPrefix, typeHandlerClass, flags, resulSet, foreignColumn);
  }
```

### 2.6.2 公用sql代码块解析  

  
公用sql代码块节点解析与其他节点解析有点不太，sql代码块节点的解析，并没有将配置信息封装成元数据添加到元数据对象Configura中，而是将节点的XML信息放到元数据对象Configura中，供下面解析sql语句使用。  

```java
private void sqlElement(List<XNode> list, String requiredDatabaseId) throws Exception {
	
    //循环解析多个sql节点
    for (XNode context : list) {
      String databaseId = context.getStringAttribute("databaseId");
   
      //获取sql节点的id属性
      String id = context.getStringAttribute("id");
      id = builderAssistant.applyCurrentNamespace(id, false);

      //将sql对应的XML NODE信息与id关联，暂存到mapper配置文件解析辅助类XMLMapperBuilder中
      if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) sqlFragments.put(id, context);
    }
  }
```

### 2.6.3 sql语句解析  

Mybatis配置文件解析的重点是各个mapper配置文件的解析，mapper配置文件解析的重点是各个sql语句的解析。  


请注意，sql语句解析，是解析配置文件中的`select|insert|update|delete`节点，源码如下：  

```java
private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {

    //循环遍历多个sql语句
    for (XNode context : list) {

      //第三个XML*Builder出现，可见sql语句重要程度
      final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
      try {

        //和前两个XML*Builder套路一致，创建对象是为了持有必要成员变量，重点解析还是在 parse 方法
        statementParser.parseStatementNode();
      } catch (IncompleteElementException e) {
        configuration.addIncompleteStatement(statementParser);
      }
    }
  }
```

下面是具体的sql解析过程：  

```java
public void parseStatementNode() {

    //获取各种属性信息
    String id = context.getStringAttribute("id");
    
	//.....省略


	//根据节点名称，获取sql类型
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    
    // Include Fragments before parsing
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);

	//遍历sql语句节点下的include节点，根据include的id，获取元数据对象中保存的公用sql的XML node，并替换include
    includeParser.applyIncludes(context.getNode());

    //.....省略


    //SqlSource中封装了纯文本的sql语句
    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    
	//.....省略

	
	//封装sql解析后的元数据对象MappedStatement，并添加到Mybatis配置文件元数据对象Configuration中
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered, 
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
  }
```

**疑问**：  
了解完sql语句的解析过程，来理解一下下面这个问题  
Q：dao层调用sql时候，需要指定sql语句id还是需要命名空间加上id呢？  

A：最好使用命名空间加id的方式  

`Configuration`中的sql语句元数据是以map维护的，`StrictMap`是一个自定义的map  

`Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection")`

map的键是sql语句的id加上命名空间（applyCurrentNamespace），StrictMap在put时候，先根据整个字符串put一遍，再截取最后一个**.**之后的字符串作为键，再put一遍。  

例如：命名空间是 **com.a.b**, sql的id是 **select**，那么在解析完这个sql后，`Configuration`的mappedStatements这个map中会存在两条数据，一个key为**com.a.b.select**，一个为**select**