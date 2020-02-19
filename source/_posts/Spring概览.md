---
title: Spring概览
date: 2017-02-20 17:14:00
tags: ["Java", "Spring"]
---
在Java领域[Spring框架](https://spring.io/ "Spring框架")可谓是大名鼎鼎，最开始Spring的目标是简化企业级应用的开发，如今看来Spring早已实现了自己的"野心",且发展成了一个大家族。当前Spring最新稳定版本为4.3.6，并仍在持续发展。学习一门技术时对其有个大概全面的认识有助于后面的学习，因此本文主要从宏观上介绍Spring，首先看一下[官方文档](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/ "官方文档")中给出的Spring简述：

>Spring是构建企业级应用的一个轻量级的解决方案和一个潜在的“一站式商店”。然而，Spring是模块化的，允许你只使用你所需要的模块，而不用引入剩下的其他模块。你可以在任何框架之上去使用IoC(控制反转)容器，但你也可以只使用Hibernate集成代码或JDBC抽象层。Spring框架支持声明式事务，RMI(远程方法调用），web服务，还有各种各样持久化数据的选项。它提供了一个全功能的MVC框架，并且使你能够透明地集成AOP(面向切面编程)到你的软件中去。Spring框架被设计成非侵入式的，这意味着你的业务逻辑代码不用依赖于框架本身。

以上，是官方文档给出的Spring介绍。接下来，我们再来了解一下Spring是如何简化Java开发的。
<!--more-->
## 简化Java开发

Spring是为了解决企业级应用开发的复杂度而创建的，使用Spring可以让简单的JavaBean实现之前只有EJB(企业级JavaBean)才能完成的事情，但Spring不仅仅局限于服务器端开发，任何Java应用都能在简单性、可测试性、和松耦合等方面从Spring中受益。

为了降低Java开发的复杂性，Spring采取了以下4种关键策略：

+ 基于POJO的轻量级和最小侵入式编程

+ 通过依赖注入和面向接口实现松耦合

+ 基于切面和惯例进行声明式编程

+ 通过切面和模板减少样板式代码

### 激发POJO的潜能

很多Java框架通过强迫应用继承它们的类或实现它们的接口从而导致应用与框架绑死，比如早期的EJB、Struts、WebWork等等。而Spring坚持最小侵入式编程，不会强迫你实现Spring规范的接口或继承Spring规范的类。最坏的场景是，一个类或许会使用Spring注解，但它仍是POJO。

请看下面的HelloWorldBean类：
 
```java
public class HelloWorldBean {
	public String sayHello(){
		return "Hello World!";
	}
}
```
可以看到，这是一个简单的POJO类。没有任何地方表明它是一个Spring组件。Spring的非侵入式编程模型意味着这个类在Spring应用和非Spring应用中都可以发挥同样的作用。

### DI和IoC

控制反转（Inversion of Control，缩写为IoC），是面向对象编程中的一种设计原则，可以用来降低计算机代码之间的耦合度。而依赖注入（Dependency Injection）则是IoC的一种特殊实现方式。那么究竟什么是IoC呢？我们都知道，对于Java这种面向对象的语言来说，应用程序是由一个个对象组成的，对象之间互相协作，最终实现系统的业务功能。因此对象之间有依赖关系是相当正常的，毕竟耦合度为0的系统基本上是不存在的。

那么问题来了，假设A类的对象依赖于B类的对象，我们往往会在A类的某个方法中通过new显式调用B类的构造函数来获得B类对象，当系统很小时也许或者B类对象创建很简单时，也许看不出这种方式有什么不妥。但是考虑在一个大系统中，某一个类可能依赖几十个类，又或者某个类的对象创建十分复杂时，这时再由该类负责创建其他类的对象则显得十分不优雅了。这也违反了单一职责原则(不要存在多于一个导致类变更的原因，即一个类只负责一项职责)。最终会导致高度耦合和代码难以测试。那有没有什么好的解决方案呢？

稍微了解设计模式的同学也许立马会想到工厂模式，是的，工厂模式通过引入工厂类来负责目标类对象的创建，很好地降低了调用者类与目标类的耦合度，这在很多时候都是一种良好的解决方案。

但是，使用工厂模式等设计模式需要我们自己在应用程序中来实现。除此之外还要考虑的问题有：

- 对于不同类型的目标类，都要通过引入一个对应的工厂类来解耦吗？

- 有没有什么方法可以降低对象之间的耦合度，又不用引入那么多的工厂类呢？

- 有没有什么方法能管理所有类的实例以及它们之间的依赖关系和生命周期呢？

- ...

是的，没错，IoC就是一个好的解决方案。

IoC通过引入一个巨大的工厂(称为IoC容器)，所有类的实例（称为bean)都由IoC容器统一管理，IoC容器不仅负责管理对象的创建，还负责管理对象之间的依赖关系以及对象的生命周期等。

可以理解为只需要将B类在工厂中注册，之后当A类需要B类的实例时，只需要向IoC容器索取而不用自己手动创建，由IoC容器负责将A类对B类实例的依赖"注入"到A类中。不用再专注于对象的创建过程，从而降低了对象之间的耦合度。

关于依赖注入和控制反转，可参考Martin Fowler的[这篇文章](https://martinfowler.com/articles/injection.html)。

总之，通过DI,对象的依赖关系将由系统中负责协调对象的第三方组件在创建对象的时候进行设定。对象无需自行创建或管理它们的依赖关系。DI所带来的最大收益是松耦合。如果一个对象只通过接口（而不是具体实现或初始化过程）来表明依赖关系，那么这种依赖就能够在对象本身毫不知情的情况下，用不同的具体实现进行替换。

上面所介绍的IoC是Spring核心。除此之外，Spring还有众多激动人心的模块。那么，Spring究竟有哪些模块呢？

##  Spring模块

话不多说，直接上图：

![Spring模块](http://upload-images.jianshu.io/upload_images/4778432-60943f29e97e33b6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Spring框架的功能被组织成了20来个模块。这些模块分成 Core Container, Data Access/Integration,Web,AOP(Aspect Oriented Programming，面向切面编程),Instrumentation,Messaging,和Test。前面说过，Spring是模块化的，你可以只选择你需要的模块。下面介绍几个常用模块：

### 核心容器

容器是Spring最核心的部分，它管理着Spring应用中bean的创建、配置和管理。Core Container 由 spring-core, spring-beans, spring-context, spring-context-support, 和spring-expression (Spring表达式语言) 模块组成。

其中spring-core 和 spring-beans 提供框架的基础部分，包括 IoC 和 Dependency Injection 功能。

spring-context有多种实现，每一种都提供了配置Spring的不同方式。

### AOP模块

在AOP模块中，Spring对面向切面编程提供了丰富的支持。同DI一样，AOP可以帮助应用对象解耦。借助于AOP，可以将遍布系统的关注点（例如事务、安全、日志等）从它们所应用的对象中解耦出来。

### 数据访问/集成

Data Access/Integration 层由 JDBC, ORM, OXM, JMS, 和 Transaction 模块组成。

我们都知道，传统的JDBC编程通常会导致大量的样板式代码，例如获得数据库连接、创建语句、处理结果集到最后关闭数据库连接。spring-jdbc 模块提供了不需要编写冗长的JDBC代码和解析数据库厂商特有的错误代码的JDBC-抽象层。

spring-tx 模块的支持可编程和声明式事务管理，用于实现了特殊的接口的和你所有的POJO类（ Plain Old Java Objects）

spring-orm 模块提供了流行的 object-relational mapping（ 对象-关系映射） API集成层，其包含JPA，JDO，Hibernate。使用ORM包，你可以使用所有的 O/R 映射框架结合所有Spring

提供的特性，比如前面提到的简单声明式事务管理功能。

### Web

Web层由spring-web, spring-webmvc, spring-websocket, 和 spring-webmvc-portlet 组成。

spring-web 模块提供了基本的面向 web 开发的集成功能，例如多方文件上传、使用 Servlet

listeners 和 Web 开发应用程序上下文初始化 IoC 容器。它也包含 HTTP 客户端以及 Spring

远程访问的支持的 web 相关的部分。

spring-webmvc 模块（ 也被称为 Web Servlet 模块） 包含 Spring 的model-view-controller

（ 模型-视图-控制器（ MVC） 和 REST Web Services 实现的Web应用程序。Spring 的 MVC

框架提供了domain model（ 领域模型） 代码和 web form (网页) 之间的完全分离，并且集成了

Spring Framework 所有的其他功能。

值得一提的是，SpringMVC框架是当前Java最流行的MVC框架。

### Test

spring-test 模块支持通过组合 JUnit 或 TestNG 来进行单元测试和集成测试 。它提供了连续的

加载 ApplicationContext 并且缓存这些上下文。它还提供了 mock object（ 模仿对象） ，您可

以使用隔离测试你的代码。

## Spring项目

事实上，Spring远不是Spring框架所下载的那些。作为一个Java领域的龙头老大，Spring提供了一系列[项目](https://spring.io/projects "Spring Projects")。下面介绍几个热门项目：

### Spring Boot

[Spring Boot](http://projects.spring.io/spring-boot/ "Spring Boot")支持约定大于配置，可以让开发人员快速搭建Spring项目，而不用进行传统繁杂的配置。

大部分Spring Boot项目只需要非常少的Spring配置，Spring Boot有如下特点：

- 创建独立的Spring应用程序

- 内置Tomcat,Jetty或Undertow（无需部署WAR包)

- 提供了'starter' POMs，简化Maven配置

- 尽可能地自动化配置Spring

- 绝对没有代码生成和XML配置要求

### Spring Cloud

[Spring Cloud](http://projects.spring.io/spring-cloud/ "Spring Cloud")是基于Spring Boot构建的，旨在为开发人员在分布式系统中构建一些常见的模式提供了一些工具（例如：配置管理，服务发现，断路器、智能路由，微代理，控制总线，全局锁等）。

### Spring Data

[Spring Data](http://projects.spring.io/spring-data/ "Spring Data")的目标是为数据访问提供熟悉的，一致的，基于Spring的编程模型，同时保留底层数据存储的特点。

它使得使用数据访问技术，关系型和非关系型数据库，map-reduce框架，基于云的数据服务变得简单。Spring Data是一个伞型结构的项目，包含了许多子项目，每一个都针对一个给定的数据库。

当前发布版本包含以下模块：

+ Spring Data Commons

+ Spring Data JPA

+ Spring Data KeyValue

+ Spring Data LDAP

+ Spring Data MongoDB

+ Spring Data Gemfire

+ Spring Data REST

+ Spring Data Redis

+ ...

### Spring Security

[Spring Security](http://projects.spring.io/spring-security/ "Spring Security")是一个强大的和高度可定制的身份验证和访问控制框架，它是确保基于Spring的应用程序安全的约定俗成的标准。

## 总结

Spring在Java开发领域具有不可替代的作用，从各个方面大大简化了Java应用程序的开发，每个Java开发人员都必须多多少少地了解Spring。Spring背后的设计思想更是一笔宝贵的财富，然而，想要精通Spring并非易事。最后，感谢Spring！

### 参考资料

- [《Spring实战》第4版](https://book.douban.com/subject/26767354/)

- [Spring官方文档](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/)