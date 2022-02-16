---
title: SpringBoot时间格式化总结
categories: 正常的文章
date: 2020-01-16 20:18:15
tags: [Java,Spring,SpringBoot,Web]
---

## Json中的时间

对于json形式的请求或响应(content-type=application/json)，格式化时间有如下方法：

- 配置spring.jackson.date-format
- 使用@JsonFormat
- 自定义ObjectMapper
- 通过Jackson2ObjectMapperBuilderCustomizer自定义ObjectMapper
- 自定义com.fasterxml.jackson.databind.Module

> 见org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration

## 非Json中的时间

对于表单请求，格式化时间有如下方法：

- 配置spring.mvc.format.date

- 配置spring.mvc.format.time
- 配置spring.mvc.format.date-time
- 定义Converter
- 定义Formatter
- 使用@DateTimeFormat

> 见org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration.EnableWebMvcConfiguration#mvcConversionService