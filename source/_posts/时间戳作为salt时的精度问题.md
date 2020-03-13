---
title: 时间戳作为salt时的精度问题
categories: 正常的文章
date: 2020-01-23 18:46:22
tags: [Java,密码加盐]
---

## 前言
在web项目中采用用户注册时的时间戳作为密码加密的salt：
```java
    public String getSalt(User user) {
        return String.valueOf(user.getRegistrationTime().getTime()/1000L);
    }
```
数据库中保存注册时间戳的字段类型使用Timestamp(0)即10位精确到的秒时间戳  
注册用户逻辑：
```java
    public User registerAnAccount(User user) {
        if (isExist(user.getName(), user.getEmail())) {
            return null;
        }
        HttpServletRequest request = ((ServletRequestAttributes) Objects.requireNonNull(RequestContextHolder.getRequestAttributes())).getRequest();
        
        user.setRegistrationIp(request.getRemoteAddr());
        user.setRegistrationTime(Timestamp.from(Instant.now()));
        user.setStatus(User.Status.WAITING_CONFIRMATION);
        user.setPassword(HashUtils.hash(user.getPassword(), getSalt(user)));
        
        return repository.save(user);
    }
```
获取时间戳后设置注册用户的属性，使用时间戳作为salt给密码加密保存到数据库。`getSalt(User)`方法直接返回`timestamp.getTime()/1000L`的值，是一个10位精确到秒的时间戳。  
在用户登录时发现部分用户登录密码错误。可以保证的是业务代码的逻辑是没有问题的，那么可以肯定应该就是数据存储到数据库时发生了问题。

## 原因
模拟注册操作，打上断点：
![](https://lolico.griouges.cn/images/NBbK.png)
**salt**：1579764044  
**registrationTime**：2020-01-23 15:20:44.986（可以看到nanos字段为986000000，精确到毫秒）  
模拟登录操作，采用Shiro框架实现认证和鉴权，在返回`SimpleAuthenticationInfo`时打上断点：
![](https://lolico.griouges.cn/images/NaeN.png)
查看从数据库查出来的用户信息和salt值：
![](https://lolico.griouges.cn/images/NL2o.png)
**salt**：1579764045  
**registrationTime**：2020-01-23 15:20:45.0（nanons字段为0因为数据库字段设置为精确到秒）  
在这个测试用例之前还模拟了一些用户注册和登录，发现用户在登陆时的salt值和注册时的是一样的，可以登录成功，而这个在登陆时的却比注册时所用的salt大1，观察到能登陆成功的用户信息注册时的时间戳字段毫秒部分都小于500，而这个用例的毫秒部分为986，猜测在保存时间戳到数据库，丢失精度（毫秒部分）时会进行四舍五入，毫秒部分大于500进1，所以导致的就是在注册时如果毫秒部分大于500，注册get的盐值会比登录时get的盐值要小1，也就是上面图片中的情况。google并且大量测试证实了猜想。

## 解决
1. 数据库Timestamp类型长度设置为3即精确到毫秒（可以保证在保存时不会发生丢失精度导致的四舍五入情况）
2. 修改`getSalt(User)`方法：
```java
    public String getSalt(User user) {
        int nanos = user.getRegistrationTime().getNanos();
        if (nanos > 500000000) {
            return String.valueOf(user.getRegistrationTime().getTime() / 1000L + 1);
        }
        return String.valueOf(user.getRegistrationTime().getTime() / 1000L);
    }
```
3. 注册时设置registrationTime
```
    //user.setRegistrationTime(Timestamp.from(Instant.now()));
    //改为下面这种，截断毫秒部分
    user.setRegistrationTime(new Timestamp(Instant.now().getEpochSecond()*1000));
```