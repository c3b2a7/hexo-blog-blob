---
categories: 正常的文章
date: 2022-03-22 22:57:05
title: Java知识小记
tags:
---


1、一个字符的`String.length()`是多少？

Java中字符采用Unicode（UTF16）编码，即16位（2字节）一个单元，UTF16可以包含一个单元或两个单元，对于`U+0000-U+FFFF`范围内的字符采用2字节编码，大于`U+FFFF`的采用四字节编码。例如一些emoji或者生僻字，采用四字节编码及两单元，`String#length()`将返回2，如需返回字符数可使用`String#codePointCount(0,length)`。

2、SpringSecurity使用了什么模式？

`构建者模式`、`责任链模式`。

3、SpringSecurity使用构建者模式最后构建的是一个什么？

是一个`Filter`、而且是`FilterChainProxy`，其内部包含多个`SecurityFilterChain`，根据请求匹配一个`SecurityFilterChain`并应用其中的`Filter`。

![image-20220330234452227](https://s2.loli.net/2022/03/30/rGf8lqhwI5tpYA9.png)

4、简单说一下`Synchronized`和`AQS`有什么不同？

	- 前者是JVM层面实现、后者是语言层面实现。
	- 前者根据不同锁实现方式不一样、后者是基于队列+CAS实现。
	- 后者相比前者提供了更灵活的API、比如是否响应中断、超时时间设置、可绑定多个条件队列等等
	- 前者只能是非公平锁、后者可实现公平锁，即先等待的线程先获取到锁。

5、偏向锁、轻量级锁、重量级锁的升级过程简单说一下？

在不存在锁竞争时是偏向锁、此时只使用了一次CAS设置对象头中的偏向线程ID；存在资源竞争时、在全局安全点撤销偏向锁升级为轻量级锁，轻量级锁使用CAS来修改对象头中持有锁的线程ID；在轻量级锁CAS竞争锁失败一定次数（默认10）后升级为重量级锁；重量级锁基于操作系统的`Mutex Lock `实现。

6、三种锁是怎样确保线程安全的？

使用`Synchronized`关键字时，会在字节码中插入`monitorenter`和`monitorexit`指令，当执行 `monitorenter` 指令时，线程试图获取锁也就是获取 **对象监视器 `monitor`** 的持有权，如果获取失败则根据锁类型自旋或者阻塞，从而确保线程安全。

7、AQS的原理简单说一下？内部使用了什么数据结构与设计模式？



8、ThreadLocal存不存在内存泄漏问题?

`ThreadLocalMap` 中使用的 key 为 `ThreadLocal` 的弱引用,而 value 是强引用。所以，如果 `ThreadLocal` 没有被外部强引用的情况下，在垃圾回收的时候，key 会被清理掉，而 value 不会被清理掉。这样一来，`ThreadLocalMap` 中就会出现 key 为 null 的 Entry。假如我们不做任何措施的话，value 永远无法被 GC 回收，这个时候就可能会产生内存泄露。ThreadLocalMap 实现中已经考虑了这种情况，在调用 `set()`、`get()`、`remove()` 方法的时候，会清理掉 key 为 null 的记录。但在使用完 `ThreadLocal`方法后 最好手动调用`remove()`方法以便及时回收资源。