---
title: response.getWriter()和内置对象out的区别
categories: 正常的文章
date: 2018-11-03 16:25:16
tags: [Java,JSP,Web]
---

**首先我们需要知道的是:**

 1. response对象和out对象都是jsp内置的隐式对象。
 2. out是JSP九大内置对象之一，是JspWriter的一个对象，JspWriter继承了java.io.Writer类。
 3. response.getWriter()返回的是PrintWriter的一个对象，PrintWriter同样继承了java.io.Writer类。

----------
**两者的主要区别**

 1. 两者的类型不一样：
    内置对象out的类型是JspWriter，response.getWrite()返回的类型是PrintWriter。out和response.getWriter()的类不一样，一个是JspWriter，另一个是PrintWriter,但这两个类都继承自java.io.Writer，不同的是JspWriter是抽象类。
 2. 获取方式不同：
    JspWriter是JSP的内置对象，可以直接使用，对象名out是保留字，也只能通过out来调用其相关方法。此外还可以通过内置对象pageContext调用方法getOut()获得；PrintWriter则是在用的时候需要通过内置对象response调用getWriter()获得。
 3. JspWriter的print()方法会抛出IOException，而PrintWriter则不会。
 4. out为jsp的内置对象，刷新jsp页面，自动初始化获得out对象，所以使用out对象是需要刷新页面的，而response.getWriter()响应信息通过out对象输出到网页上，当响应结束时它自动被关闭，与jsp页面无关，无需刷新页面。
 5. out的print()方法和println()方法在缓冲区溢出并且没有自动刷新时候会产生IOException，而response.getWrite()方法的print和println中都是抑制ioexception异常的，不会有IOException。out.println("")并不能使页面布局换行，只能令html代码换行，要实现页面布局换行可以out.println("\</br>")
 6. 执行原理不同:
*示例：*
```jsp
<%="aaaa"%>
bbbb
<%out.write("cccc");%>
<%response.getWriter().write("dddd");%>
```
注意：response.getWriter().print()实际上调用的还是write方法，PrintWriter中print(String)源码:
```java
public void print(String s) {
    if (s == null) {
        s = "null";
    }
    write(s);
}
```
运行上面的示例会在页面输出:
dddd aaaa bbbb cccc

顺序为什么是这样的呢?

> 示例中的前三个方式，在jsp被翻译为servlet时，都会被翻译为out.write()方法，当我们向页面输出内容时，tomcat服务器会默认先刷新response缓冲区中的内容到网页中。而out对象本身也有个out缓冲区，前3个方法执行后要输出的内容先被存到out缓冲区内，然后再转移到response缓冲区中被tomcat服务器提取，因为JspWriter相当于一个带缓存功能的printWriter，它不是直接将数据输出到页面，而是将数据刷新到response的缓冲区后再输出。所以response.getWriter().print(String)直接输出数据，out.print(String)只能在其后输出。

![](https://raw-1257226137.file.myqcloud.com/images/qiLZ.png)
当然我们也可以让1、2、3的内容直接存到response缓冲区中。这是因为，out缓冲区可以通过指令buffer来设置它的缓存区大小，一般默认的是8kb，当我们设置为buffer=“0kb”时，就让out缓冲区存储空间为0，这样1、2、3方法输出的内容就会直接存到response缓冲区中。
