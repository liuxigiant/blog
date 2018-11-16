---
layout: post
title: "Spring Schema 扩展"
description: 通过自定义xml schema及处理类，扩展标准spring xml配置文件声明方式
date: 2017-08-18 11:34:08 +0800
catalog: true
categories: 
- Spring Schema
tags:
- Spring Schema
---

在使用Spring时候，一般会采用基于xml的配置方式。Spring基于xml schema提供了丰富的xml配置方式，以支持Spring的各种特性（例如：事物相关的tx schema，任务调度相关的task schema等）。  


Spring在官方提供的xml schema配置方式外（可参考官方文档X[ML Schema-based configuration](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#xsd-configuration)），还提供了自定义方式来扩展Spring schema。下面来说说如何扩展Spring Schema（翻译自官方文档[Extensible XML authoring](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#xml-custom)）

> spring版本：4.3.10.RELEASE

# 42 可扩展的xml schema

## 42.1 介绍  

Spring从2.0版开始，在官方的XML配置的基础上，提供了以xml schema的方式来扩展Spring XML配置，从而扩展了Spring Bean的定义和配置方式。本部分详细说明基于XML schema的自定义Spring Bean的配置、解析以及与Spring IOC容器的集成。  

在了解扩展Spring Schema前，可先了解下Spring官方提供的基础的schema声明[ML Schema-based configuration](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#xsd-configuration)  

扩展Spring schema分为以下几步：  

- **自定义xml scheme**：用于描述需要配置的元素（xml schema用于定义和描述xml节点）
- **自定义NamespaceHandler**：自定义NamespaceHandler实现类，用于处理自定义的xml schema
- **自定义BeanDefinitionParser**：自定义BeanDefinitionParser实现类，用于将xml声明解析成Spring bean元数据对象BeanDefinition
- **注册**：注册自定义信息，与Spring框架集成

下面通过一个实践示例来逐一解释各个步骤内容，本示例采用扩展Spring schema的方式，更方便地声明`java.text.SimpleDateFormat`类的实例。完成了本示例的扩展Spring schema实践后，声明bean的方式如下：  

```xml
<myns:dateformat id="dateFormat"
    pattern="yyyy-MM-dd HH:mm"
    lenient="true"/>
```

> xml相关的一些概念：
> 
> - 格式规范的XML：符合XML的声明结构（例如：元素需要有开始标签、结束标签；元素需要被尖括号包含）
> - 合法的XML：合法的XML需要满足一定的约束（这个约束一般会定义一个元素下可以包含那些属性、那些子元素，属性的值的范围等，约束定义了xml元素的含义与逻辑），xml文件的约束有两种方式DTD（.dtd文件）和Schema（.xsd文件 Spring使用的schema）
> - xml schema：定义xml文件的一种方式，即xsd文件的内容
> - xml命名空间：一个xml schema一般就是一个命名空间，多个schema下可以有相同的元素，但是schema不同，含义和格式可能不同

## 42.2 自定义xml schema  

扩展spring xml的配置，与spring IOC集成，首先需要自定义配置bean的xml 元素定义，即自定义xml schema。本例中配置`SimpleDateFormat`对象，xml schema定义如下：  

```xml
<!-- myns.xsd (inside package org/springframework/samples/xml) -->

<?xml version="1.0" encoding="UTF-8"?>
<xsd:schema xmlns="http://www.mycompany.com/schema/myns"
        xmlns:xsd="http://www.w3.org/2001/XMLSchema"
        xmlns:beans="http://www.springframework.org/schema/beans"
        targetNamespace="http://www.mycompany.com/schema/myns"
        elementFormDefault="qualified"
        attributeFormDefault="unqualified">

    <xsd:import namespace="http://www.springframework.org/schema/beans"/>

    <xsd:element name="dateformat">
        <xsd:complexType>
            <xsd:complexContent>
                <xsd:extension base="beans:identifiedType">
                    <xsd:attribute name="lenient" type="xsd:boolean"/>
                    <xsd:attribute name="pattern" type="xsd:string" use="required"/>
                </xsd:extension>
            </xsd:complexContent>
        </xsd:complexType>
    </xsd:element>
</xsd:schema>
```

上面schema定义中引入了spring beans的schema，是为了让自定义的配置bean继承`beans:identifiedType`（`identifiedType`定义了一个id属性，作为被声明对象在spring IOC容器中的标识）。  

上面定义的schema，在spring xml配置文件中，只需要配置`<myns:dateformat/>`元素即可声明`SimpleDateFormat`对象，如下：  

```xml
<myns:dateformat id="dateFormat"
    pattern="yyyy-MM-dd HH:mm"
    lenient="true"/>
```

通过上面的方式扩展Spring schema，与Spring提供的bean配置方式本质上是一致的，都能达到在Spring IOC容器中声明对象的目的，被声明对象标识为`dateFormat`，类型为`SimpleDateFormat`，同时给属性赋值。Spring提供的bean配置方式如下：  

```xml
<bean id="dateFormat" class="java.text.SimpleDateFormat">
    <constructor-arg value="yyyy-HH-dd HH:mm"/>
    <property name="lenient" value="true"/>
</bean>
```

## 42.3 实现NamespaceHandler接口  

除了自定义xml schema，在Spring解析xml配置文件，解析自定义元素，需要自定义`NamespaceHandler`接口的实现类完成。对于本例，需要解析的元素是`myns:dateformat`。  

`NamespaceHandler`接口包含以下三个方法：  

- `init()`：用于初始化，在使用该接口前由Spring触发调用
- `BeanDefinition parse(Element, ParserContext)`：当Spring解析到一个顶级元素（不包含嵌套声明或者在其他命名空间元素中声明）时候被调用。这个方法可实现注册和返回bean definitions 
- `BeanDefinitionHolder decorate(Node, BeanDefinitionHolder, ParserContext)`：当Spring解析到属性或其他命名空间下的元素时被调用。本例中没用到这个方法，下面有一个例子进行说明。  

编写一个自定义的`NamespaceHandler`接口实现类来解析整个命名空间下的元素看起来可能是比较完美的，通常Spring xml配置文件中一个顶级元素对应声明一个对象（本例中`<myns:dateformat/>`元素对应声明了`SimpleDateFormat`对象）。Spring提供了一些类来支持xml元素的解析，本利中使用`NamespaceHandlerSupport`类。  

```java
package org.springframework.samples.xml;
import org.springframework.beans.factory.xml.NamespaceHandlerSupport;
public class MyNamespaceHandler extends NamespaceHandlerSupport {
    public void init() {
        registerBeanDefinitionParser("dateformat", new SimpleDateFormatBeanDefinitionParser());
    }
}
```

从上面代码中可发现，并没有解析元素的详细逻辑。其实，解析逻辑由`NamespaceHandlerSupport`辅助类代理完成，通过支持注册多个`BeanDefinitionParser`,在解析到元素时候，由对应的`BeanDefinitionParser`代理完成真正的解析逻辑。这种注册与解析的分离，可让`NamespaceHandler`接口的实现类关注当前命名空间需要解析的所有元素，而真正的解析逻辑由对应的`BeanDefinitionParser`完成，意思就是一个`BeanDefinitionParser`只需要关注一个元素的解析即可，下面章节会做说明。  

## 42.4 实现BeanDefinitionParser接口  

从上面章节的注册逻辑可看出，在自定义的xml schema中，一个解析器`BeanDefinitionParser`只负责解析一个顶级元素（本例中是`dateformat`）。在解析器`BeanDefinitionParser`中可访问xml元素（包括子元素），从而实现解析逻辑。如下：  

```java
package org.springframework.samples.xml;
import org.springframework.beans.factory.support.BeanDefinitionBuilder;
import org.springframework.beans.factory.xml.AbstractSingleBeanDefinitionParser;
import org.springframework.util.StringUtils;
import org.w3c.dom.Element;
import java.text.SimpleDateFormat;
public class SimpleDateFormatBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {
    protected Class getBeanClass(Element element) {
        return SimpleDateFormat.class;
    }
    protected void doParse(Element element, BeanDefinitionBuilder bean) {
        // this will never be null since the schema explicitly requires that a value be supplied
        String pattern = element.getAttribute("pattern");
        bean.addConstructorArg(pattern);
        // this however is an optional property
        String lenient = element.getAttribute("lenient");
        if (StringUtils.hasText(lenient)) {
            bean.addPropertyValue("lenient", Boolean.valueOf(lenient));
        }
    }
}
```

- 通过继承`AbstractSingleBeanDefinitionParser`完成创建`BeanDefinition`（xml bean定义的元数据封装对象）的一些基本逻辑
- `getBeanClass`方法用于声明当前bean的类型  

如上代码，即可完成自定义元素的解析。被声明元素的元数据封装对象`BeanDefinition`由父类`AbstractSingleBeanDefinitionParser`完成。同时，父类也会获取和设置bean的id。  

## 42.5 handler和schema的注册  

至此，代码部分已经完成，接下来需要将自定义的`namespaceHandler`和 XSD 文件注册到特定的properties属性文件中，从而是Spring的XML解析框架能够识别自定义扩展的Spring schema及其处理类。这些特性的properties属性文件都需要在应用（当前应用或者依赖的JAR包）的`META-INF`目录下。Spring的XML解析框架会自动读取这些特定的配置文件，这些配置文件格式描述如下：  

### 42.5.1 META-INF/spring.handlers  

`spring.handlers`这个属性配置文件中声明 **XML schema URL** 与 **schema处理类** 的映射关系。对于本例，需要配置的信息如下：  

```
http\://www.mycompany.com/schema/myns=org.springframework.samples.xml.MyNamespaceHandler
```

> 如上配置中 
> 
> - **:**是java属性配置文件的保留字符，需要进行转义  
> - 配置属性key是扩展spring schema的命名空间URL，需要与自定义的xml schema文件（即XSD文件)中的`targetNamespace`属性的值相同

### 42.5.2 META-INF/spring.schemas  

`spring.schemas`这个属性配置文件中声明 **XML schema XSD路径**（在Spring xml中使用自定义schema时，需要引入自定义schema，本属性在`xsi:schemaLocation`中声明） 与 **XSD文件的类路径** 的映射关系。这个配置文件可以让Spring在读取自定义schema的XSD文件时候，从本地类路径中获取，而避免了从远程获取。对于本例来说，自定义schema的XSD文件`myns.xsd`在`org.springframework.samples.xml`包下，配置信息如下：  

```
http\://www.mycompany.com/schema/myns/myns.xsd=org/springframework/samples/xml/myns.xsd
```

## 42.6 扩展Spring schema的使用  

以上，完成了Spring schema的扩展，使用自定义的schema和使用Spring提供的schema其实一样。  

本例扩展的schema配置使用如下：  

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:myns="http://www.mycompany.com/schema/myns"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.mycompany.com/schema/myns http://www.mycompany.com/schema/myns/myns.xsd">

    <!-- as a top-level bean -->
    <myns:dateformat id="defaultDateFormat" pattern="yyyy-MM-dd HH:mm" lenient="true"/>

    <bean id="jobDetailTemplate" abstract="true">
        <property name="dateFormat">
            <!-- as an inner bean -->
            <myns:dateformat pattern="HH:mm MM-dd-yyyy"/>
        </property>
    </bean>

</beans>
```

## 42.7 进阶版示例  

下面有几个进阶版的扩展Spring schema的示例  

### 42.7.1 自定义元素的嵌套  

自定义元素的嵌套使用，配置示例如下，翻译略 -_-  

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:foo="http://www.foo.com/schema/component"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.foo.com/schema/component http://www.foo.com/schema/component/component.xsd">

    <foo:component id="bionic-family" name="Bionic-1">
        <foo:component name="Mother-1">
            <foo:component name="Karate-1"/>
            <foo:component name="Sport-1"/>
        </foo:component>
        <foo:component name="Rock-1"/>
    </foo:component>

</beans>
```

### 42.7.2 自定义属性

在正常的Spring声明中使用自定义的元素，配置示例如下，翻译略 -_-  

```xml
<bean id="checkingAccountService" class="com.foo.DefaultCheckingAccountService"
        jcache:cache-name="checking.account">
    <!-- other dependencies here... -->
</bean>
```

## 42.8 参考文档  

下面有一些XML schema的参考文档  

- [XML Schema Part 1: Structures Second Edition](https://www.w3.org/TR/2004/REC-xmlschema-1-20041028/)
- [XML Schema Part 2: Datatypes Second Edition](https://www.w3.org/TR/2004/REC-xmlschema-2-20041028/)