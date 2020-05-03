---
title: SpringBoot 2.x AOP默认使用CGLIB代理
tags:
  - SpringBoot
  - Spring
  - AOP
categories: 正常的文章
date: 2020-05-03 19:18:32
---

在一次偶然情况下发现SpringBoot开发的应用在没有使用`@EnableAspectJAutoProxy`注解的情况下，AOP还是可以正常工作。不经引起了我的注意，随后便猜测SpringBoot中是否存在一个自动配置类在不使用注解的情况下完成自动配置并开启了AOP。

<!-- more -->

在Idea中尝试搜索`AopAutoConfiguration`，果然不出所料，有这么一个自动配置类：

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)
public class AopAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass(Advice.class)
    static class AspectJAutoProxyingConfiguration {

        @Configuration(proxyBeanMethods = false)
        @EnableAspectJAutoProxy(proxyTargetClass = false)
        @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false",
                matchIfMissing = false)
        static class jdkDynamicAutoProxyConfiguration {

        }

        @Configuration(proxyBeanMethods = false)
        @EnableAspectJAutoProxy(proxyTargetClass = true)
        @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true",
                matchIfMissing = true)
        static class CglibAutoProxyConfiguration {

        }

    }
    // ...
}
```

可以看到`@ConditionalOnProperty`注解中`matchIfMissing`属性的设置，也正是这个属性的设置使得我们在不使用`@EnableAspectJAutoProxy`注解开启AOP的情况下，AOP也可以正常工作。

从这个自动配置类中还能够看出AOP默认使用Cglib代理。那么如果我们使用`@EnableAspectJAutoProxy`注解去指定默认不使用Cglib代理，有没有用呢？

经过测试发现，使用`@EnableAspectJAutoProxy`注解显示指定是没有用的。那么该如何设置默认使用jdk动态代理呢？根据自动配置类能知道，我们只需要在`application.properties`中指定`spring.aop.proxy-target-class=false`或者设置`spring.aop.auto=false`关闭aop自动配置后自行使用`@EnableAspectJAutoProxy`注解进行设置。

并且如果查看SpringBoot1.5.x中的这个自动配置类会发现默认是使用jdk动态代理，那么为什么在SpringBoot2.x中改为默认使用Cglib了呢？

> 使用jdk动态代理会存在一个代理问题：目标对象一定要实现接口。因为jdk动态代理是基于接口的，并且意味着当使用`@Autowired`注入一个使用jdk代理生成的对象时必须使用接口。

就像这样：

```java
@Autowired
UserService userService;    
```

一旦使用实现类的方式进行注入就会失败：

```java
@Autowired
UserServiceImpl userService;
```

使用Cglib代理就不存在这个问题，原因就在于Cglib是使用子类的方式进行代理。

当查看类似`@EnableCaching`、`@EnableAsync`、`@EnableTransactionManagement`注解中`proxyTargetClass`属性的注释，我们还会发现注释中说到，这个属性设置为true后会影响Spring管理的所有Bean使用的代理方式。但是我经过测试发现其他注解的这个属性设置为true时并不会影响`@EnableAsync`的代理方式。但是`@EnableAsync`的这个属性设置为true后会影响其他注解下对象的代理方式。(SpringBoot2.2.2下测试)

注意：`proxyTargetClass`属性默认为false时并不意味着如果目标对象没有实现接口，生成代理类就会失败，这种情况下Spring会使用Cglib代理。



---
*参考：*

^ [*Use @EnableTransactionManagement(proxyTargetClass = true)*](https://github.com/spring-projects/spring-boot/issues/5423)
^ [*@EnableTransactionManagement proxyTargetClass not control by spring.aop.proxyTargetClass*](https://github.com/spring-projects/spring-boot/issues/8434)
