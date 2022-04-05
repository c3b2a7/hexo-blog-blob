---
title: Spring中ControllerAdvice失效的几种场景
tags:
  - Spring
  - SpringBoot
  - Web
categories: 正常的文章
date: 2020-07-07 23:01:12
---


1. 异步调用的方法中抛出
2. Around切面在调用ProceedingJoinPoint#proceed时catch并且处理了异常