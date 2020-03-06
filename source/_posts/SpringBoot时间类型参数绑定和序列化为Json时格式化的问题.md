---
title: SpringBoot时间类型参数绑定和序列化为Json时格式化的问题
categories: 正常的文章
date: 2020-01-15 18:34:15
tags: [Java,Spring,SpringBoot,Web]
---

## 前言
在使用Spring进行web开发时经常会遇到前后台互相传值的问题，大致分无非就是下面两种情况：

- 将参数以及值直接放在request的请求体（POST）或者url（GET）中。
- 将参数以及值以JSON的形式发送（POST或者GET）到服务端。

Spring其实对前后台传值时参数的绑定提供了支持，像我们平时接触的转换器`Converter`以及消息转换器`HttpMessageConverter`的工作就是将‘值’进行类型转化或格式化后绑定到我们的方法参数中。对于简单的基础类型，我们一般并不需要进行处理，而是转换成对应的类型后直接拿原始的值就行了；但是对于某些类型，比如时间，我们通常会考虑进行格式化后传到前台或者要指定传来的格式才能绑定到方法中，所以就需要指定一个标准。这往往让人很困惑，因为可能传来的是request中的也可能是requestBody中的，而在发送给前台时也可能是以JSON或者其他的形式，在网上查了一番资料发现实在是太乱了，经过找资料和测试，现在来总结一下。

## Controller中接受request中的参数
接受`Date`、`LocalDateTime`、`LocalTime`等参数并绑定到方法入参中常用的是下面两种方法：  

1. 使用`@DateTimeFormat`注解  
这种方式比较简单，但缺点是不够灵活。
```java
    //1.只能将Get请求中格式为yyyy-MM-dd HH:mm:ss的date参数的值绑定到方法的date参数,接受其他类型的请求可以将GetMapping改为RequestMapping
    //2.参数类型可以是Date或者jdk8中新的时间类型，如：@DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate date
    @GetMapping("/date")
    public LocalDateTime getDate(@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss") LocalDateTime date) {
        System.out.println(date);
        return date;
    }
```
2. 自定义转换器`Converter`  
实现Converter接口，自定义转换规则，比较灵活。

## Controller中接受requestBody中的参数（反序列化）和序列化为json时的格式

1. `@JsonFormat`注解配合`@JsonDeserialize`或`@JsonSerialize`使用
```java
@Data
public class TestVO {
    private String msg;
    //使用JsonFormat注解指定序列化和反序列化的格式
    //使用JsonDeserialize和JsonSerialize指定使用的序列化器
    //jackson-datatype-jsr310包下是常用的反序列化器和序列化器，spring-boot-starter-json包已经包含该jsr310扩展包
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    @JsonDeserialize(using = LocalDateTimeDeserializer.class)
    @JsonSerialize(using = LocalDateTimeSerializer.class)
    private LocalDateTime localDateTime;
    
}
```
2. 配置ObjectMapper
```java
    @Bean
    @Primary
    public ObjectMapper objectMapper() {
        ObjectMapper objectMapper = new ObjectMapper();
        
        JavaTimeModule javaTimeModule = new JavaTimeModule();
        javaTimeModule.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
        javaTimeModule.addSerializer(LocalDate.class, new LocalDateSerializer(DateTimeFormatter.ofPattern("yyyy-MM-dd")));
        javaTimeModule.addSerializer(LocalTime.class, new LocalTimeSerializer(DateTimeFormatter.ofPattern("HH:mm:ss")));
        
        javaTimeModule.addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
        javaTimeModule.addDeserializer(LocalDate.class, new LocalDateDeserializer(DateTimeFormatter.ofPattern("yyyy-MM-dd")));
        javaTimeModule.addDeserializer(LocalTime.class, new LocalTimeDeserializer(DateTimeFormatter.ofPattern("HH:mm:ss")));
        objectMapper.registerModule(javaTimeModule);
        return objectMapper;
    }
```
