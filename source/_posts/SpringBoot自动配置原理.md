---
title: SpringBoot自动配置原理
categories: 正常的文章
date: 2019-12-19 18:06:00
tags: [Java,SpringBoot]
---

## 前言
不论在工作中，亦或是求职面试，Spring Boot已经成为我们必知必会的技能项。除了某些老旧的政府项目或金融项目持有观望态度外，如今的各行各业都在飞速的拥抱这个已经不是很新的Spring启动框架。

当然，作为Spring Boot的精髓，自动配置原理的工作过程往往只有在“面试”的时候才能用得上，但是如果在工作中你能够深入的理解Spring Boot的自动配置原理，将无往不利。

Spring Boot的出现，得益于“约定大于配置”的理念，没有繁琐的配置、难以集成的内容（大多数流行第三方技术都被集成），这是基于Spring 4.x提供的按条件配置Bean的能力。

## Spring Boot的配置文件
初识Spring Boot时我们就知道，Spring Boot有一个全局配置文件：`application.properties`或`application.yml`

我们的各种属性都可以在这个文件中进行配置，最常配置的比如：`server.port`、`logging.level.*` 等等，然而我们实际用到的往往只是很少的一部分，那么这些属性是否有据可依呢？答案当然是肯定的，这些属性都可以在官方文档中查找到：[common-application-properties](https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/reference/htmlsingle/#common-application-properties)

<<<<<<< HEAD
![](http://lolico.test.upcdn.net/images/3zSS.png)
=======
![](https://i.loli.net/2020/03/09/aZo7kgqtiCfz6ds.png)
>>>>>>> 075af6abe4384f66a943568fca56258b1f49111d
*（所以，话又说回来，找资料还得是官方文档，百度出来一大堆，还是稍显业余了一些）*
除了官方文档为我们提供了大量的属性解释，我们也可以使用IDE的相关提示功能，比如IDEA的自动提示。
 以上，是Spring Boot的配置文件的大致使用方法，其实都是些题外话。

那么问题来了：**这些配置是如何在Spring Boot项目中生效的呢？**
接下来，就需要聚焦本篇博客的主题：自动配置工作原理或者叫实现方式。

## 工作原理剖析
Spring Boot关于自动配置的源码在spring-boot-autoconfigure-x.x.x.x.jar中：
<<<<<<< HEAD
![](http://lolico.test.upcdn.net/images/3BR5.png)
当然，自动配置原理的相关描述，官方文档貌似是没有提及。不过我们不难猜出，Spring Boot的启动类上有一个`@SpringBootApplication`注解，这个注解是Spring Boot项目必不可少的注解。那么自动配置原理一定和这个注解有着千丝万缕的联系！
### @EnableAutoConfiguration
![](http://lolico.test.upcdn.net/images/3EHR.png)
@SpringBootApplication是一个复合注解或派生注解，在@SpringBootApplication中有一个注解`@EnableAutoConfiguration`，翻译成人话就是**开启自动配置**，其定义如下：
![](http://lolico.test.upcdn.net/images/3aNl.png)
而这个注解也是一个派生注解，其中的关键功能由`@Import`提供，其导入的`AutoConfigurationImportSelector#selectImports`方法通过`SpringFactoriesLoader#loadFactoryNames`扫描所有具有META-INF/spring.factories的jar包。spring-boot-autoconfigure-x.x.x.x.jar里就有一个这样的spring.factories文件。

这个spring.factories文件也是一组一组的key=value的形式，其中一个key是`EnableAutoConfiguration`类的全类名，而它的value是一个xxxAutoConfiguration的类名的列表，这些类名以逗号分隔，如下图所示：
![](http://lolico.test.upcdn.net/images/3kiM.png)
=======
![](https://i.loli.net/2020/03/09/LcaPEdMDorqTCIB.png)
当然，自动配置原理的相关描述，官方文档貌似是没有提及。不过我们不难猜出，Spring Boot的启动类上有一个`@SpringBootApplication`注解，这个注解是Spring Boot项目必不可少的注解。那么自动配置原理一定和这个注解有着千丝万缕的联系！
### @EnableAutoConfiguration
![](https://i.loli.net/2020/03/09/3K2ZLRs5tgrzPBD.png)
@SpringBootApplication是一个复合注解或派生注解，在@SpringBootApplication中有一个注解`@EnableAutoConfiguration`，翻译成人话就是**开启自动配置**，其定义如下：
![](https://i.loli.net/2020/03/09/lD1NuVUaZSjJOyB.png)
而这个注解也是一个派生注解，其中的关键功能由`@Import`提供，其导入的`AutoConfigurationImportSelector#selectImports`方法通过`SpringFactoriesLoader#loadFactoryNames`扫描所有具有META-INF/spring.factories的jar包。spring-boot-autoconfigure-x.x.x.x.jar里就有一个这样的spring.factories文件。

这个spring.factories文件也是一组一组的key=value的形式，其中一个key是`EnableAutoConfiguration`类的全类名，而它的value是一个xxxAutoConfiguration的类名的列表，这些类名以逗号分隔，如下图所示：
![](https://i.loli.net/2020/03/09/wz7Lal9YVJSvfOq.png)
>>>>>>> 075af6abe4384f66a943568fca56258b1f49111d
这个`@EnableAutoConfiguration`注解通过`@SpringBootApplication`被间接的标记在了Spring Boot的启动类上。在`SpringApplication.run(...)`的内部就会执行`selectImports()`方法，找到所有JavaConfig自动配置类的全限定名对应的class，然后将所有自动配置类加载到Spring容器中。
## 自动配置生效
每一个**xxxAutoConfiguration**自动配置类都是在某些条件之下才会生效的，这些条件的限制在Spring Boot中以注解的形式体现，常见的**条件注解**有如下几项：

- `@ConditionalOnBean`：当容器里有指定的bean的条件下。
- `@ConditionalOnMissingBean`：当容器里不存在指定bean的条件下。
- `@ConditionalOnClass`：当类路径下有指定类的条件下。
- `@ConditionalOnMissingClass`：当类路径下不存在指定类的条件下。
- `@ConditionalOnProperty`：指定的属性是否有指定的值，比如`@ConditionalOnProperties(prefix=”xxx.xxx”, value=”enable”, matchIfMissing=true)`，代表当xxx.xxx为enable时条件的布尔值为true，如果没有设置的情况下也为true。

以`ServletWebServerFactoryAutoConfiguration`配置类为例，解释一下全局配置文件中的属性如何生效，比如：server.port=8081，是如何生效的（当然不配置也会有默认值，这个默认值来自于org.apache.catalina.startup.Tomcat）。
<<<<<<< HEAD
![](http://lolico.test.upcdn.net/images/3rEA.png)
在`ServletWebServerFactoryAutoConfiguration`类上，有一个`@EnableConfigurationProperties`注解：**开启配置属性**，而它后面的参数是一个`ServerProperties`类，这就是**约定大于配置**的最终落地点。
![](http://lolico.test.upcdn.net/images/3HOt.png)
=======
![](https://i.loli.net/2020/03/09/Dft3IGbcpzCJyZR.png)
在`ServletWebServerFactoryAutoConfiguration`类上，有一个`@EnableConfigurationProperties`注解：**开启配置属性**，而它后面的参数是一个`ServerProperties`类，这就是**约定大于配置**的最终落地点。
![](https://i.loli.net/2020/03/09/QX2wJVoe87Cqv9i.png)
>>>>>>> 075af6abe4384f66a943568fca56258b1f49111d
在这个类上，我们看到了一个非常熟悉的注解：`@ConfigurationProperties`，它的作用就是从配置文件中绑定属性到对应的bean上，而`@EnableConfigurationProperties`负责导入这个已经绑定了属性的bean到spring容器中（见上面截图）。那么所有其他的和这个类相关的属性都可以在全局配置文件中定义，也就是说，真正“限制”我们可以在全局配置文件中配置哪些属性的类就是这些**xxxProperties**类，它与配置文件中定义的`prefix`关键字开头的一组属性是唯一对应的。

至此，我们大致可以了解。在全局配置的属性如：server.port等，通过`@ConfigurationProperties`注解，绑定到对应的`xxxProperties`配置实体类上封装为一个bean，然后再通过`@EnableConfigurationProperties`注解导入到Spring容器中。

而诸多的`xxxAutoConfiguration`自动配置类，就是Spring容器的JavaConfig形式，作用就是为Spring 容器导入bean，而所有导入的bean所需要的属性都通过`xxxProperties`的bean来获得。

可能到目前为止还是有所疑惑，但面试的时候，其实远远不需要回答的这么具体，你只需要这样回答：

> Spring Boot启动的时候会通过@EnableAutoConfiguration注解找到META-INF/spring.factories配置文件中的所有自动配置类，并对其进行加载，而这些自动配置类都是以AutoConfiguration结尾来命名的，它实际上就是一个JavaConfig形式的Spring容器配置类，它能通过以Properties结尾命名的类中取得在全局配置文件中配置的属性如：server.port，而XxxxProperties类是通过@ConfigurationProperties注解与全局配置文件中对应的属性进行绑定的。

通过一张图标来理解一下这一繁复的流程：
<<<<<<< HEAD
![](http://lolico.test.upcdn.net/images/3KfL.png)
=======
![](https://i.loli.net/2020/03/09/DEleGi5g6WbxrcJ.png)
>>>>>>> 075af6abe4384f66a943568fca56258b1f49111d

## 总结
综上是对自动配置原理的讲解。当然，在浏览源码的时候一定要记得不要太过拘泥与代码的实现，而是应该抓住重点脉络。

一定要记得xxxProperties类的含义是：**封装配置文件中相关属性**；xxxAutoConfiguration类的含义是：**自动配置类，目的是给容器中添加组件。**

而其他的主方法启动，则是为了加载这些五花八门的**xxxAutoConfiguration**类。
