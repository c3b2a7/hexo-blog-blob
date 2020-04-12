---
title: Jsp中include指令与include动作
categories: 正常的文章
date: 2018-10-18 16:19:05
tags: [Java,JSP]
---

## 前言
`＜%@ include file=" "%＞`为指令元素
`＜jsp:include page=" " flush="true"/＞`为动作元素

1. include指令通过file属性指定被包含的文件，并且file属性不支持任何表达式；`<jsp:include>`标识通过page属性指定被包含的文件，而page属性支持JSP表达式
2. 使用include指令，被包含的文件被原封不动的插入到包含页面中使用该指令的位置，然后JSP编译器再对这个合成的文件进行编译，所以在一个JSP页面中使用include指令来包含另一个JSP页面，最终编译后的文件只有一个。（静态包含）
3. 使用`<jsp:include>`动作包含文件时，当该动作标识执行后，JSP程序会将请求转发到（注意不是重定向）被包含页面，并将执行结果输出到浏览器中，然后返回页面继续执行后面的代码，因为服务器执行的是多个文件，所以JSP编译器会分别对这些文件进行编译。（动态包含）
4. 在应用include指令包含文件时，由于被包含的文件最终会生成一个文件，所以在被包含文件、包含的文件中不能有重名的变量或方法；而在应用`<jsp:include>`动作标识包含文件时，由于每个文件是单独编译的，所以在被包含文件和包含文件中重名的变量和方法是不相冲突的。

## 示例
```jsp
<%@ page language="java" contentType="text/html;charset=UTF-8" %>
<html>
<head>
    <meta charset="utf-8">
    <title>JSPinclude动作实例</title>
</head>
<body>

<%@ include file="Static.txt" %>
<jsp:include page="Dyamic.jsp" flush="true"/>

</body>
</html>
```

```txt Static.txt
<%@ page language="java" contentType="text/html;charset=UTF-8" %>
<form action="JSPIncludeActiveDemo.jsp" method="post">
    用户名： <input type="text" name="name"><br/>
    密码： <input type="password" name="password"><br/>
    <input type="submit" value="登录">
</form>
```

```jsp Dyamic.jsp
<%@ page language="java" contentType="text/html;charset=UTF-8" %>
<br/>
用户名：<%=request.getParameter("name") %>
<br/>
密码：<%=request.getParameter("password") %>
<br/>
```

![示例](https://lolico.griouges.cn/images/qh4y.png)
