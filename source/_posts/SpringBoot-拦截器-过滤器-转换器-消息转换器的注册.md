---
title: SpringBoot 拦截器，过滤器，转换器，消息转换器的注册
categories: 正常的文章
date: 2019-12-26 18:16:57
tags: [Java,Spring,SpringBoot,Web]
---

环境：SpringBoot 2.2.2
## 拦截器  
**方式一**：写一个配置类，实现`WebMvcConfigurer`接口并实现`addInterceptors`方法  

## 过滤器  
**方式一**：使用`@Component`或`@Bean`配合进行注册  
**方式二**：当使用嵌入式web服务器时使用`@ServletComponentScan`配置扫描，同时可以用来注册`filter`、`servlet`和`linstener`  

## 转换器  
**方式一**：使用`@Component`或`@Bean`注册  
**方式二**：写一个配置类，实现`WebMvcConfigurer`接口并实现`addFormatters`方法  

## 消息转换器  
**方式一**：使用`@Component`或`@Bean`注册  
**方式二**：写一个配置类，实现`WebMvcConfigurer`接口并实现`configureMessageConverters`方法  
**方式三**：写一个配置类，实现`WebMvcConfigurer`接口并实现`extendMessageConverters`方法  

## 总结  
得益于SpringBoot的自动配置，通常我们只需要通过注册Bean这样简单的方式就可以向容器中的组件注入配置。