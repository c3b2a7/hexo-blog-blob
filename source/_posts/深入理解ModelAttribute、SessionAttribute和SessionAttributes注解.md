---
title: 深入理解@ModelAttribute、@SessionAttribute和@SessionAttributes注解
categories: 正常的文章
date: 2019-10-19 12:45:57
tags: [Spring,Web]
---

## @ModelAttribute

如果希望将方法入参对象添加到模型中，则仅需要在相应入参前使用`@ModelAttribute`注解。来看一个具体的实例：

```java
    @GetMapping("/h")
    public User user(@ModelAttribute("user") User user) {
        user.setNickName("lolico li");
        return user;
    }
```

SpringMVC将请求消息绑定到`User对象`，然后以`user`为键将`User对象`放到模型中，在准备对视图进行渲染前，SpringMVC还会进一步将模型中的数据转储到视图上下文中并暴露给视图对象。对于JSP来说，SpringMVC将模型数据转储到`ServletRequest`的属性列表中（通过`ServletRequest#setAttribute(String name, Object o)`方法，作用域为请求域)，这也是为什么`@ModelAttribute`注解的属性只能在一次相同的请求中可见的原因。

除了可以在方法入参上使用`@ModelAttribute`注解外 ，还可以在方法定义上使用。SpringMVC在调用目标方法前，会先逐个调用在方法层级上标注了该注解的方法，并将这些方法的返回值添加到模型中。下面一个具体的实例：

```java
    @ModelAttribute("user")
    public User initUserModal() { // 1
        User user = new User();
        user.setNickName("init user");
        return user;
    }

    @GetMapping("/h")
    public User handle(@ModelAttribute("user") User user) { // 2
        user.setNickName("lolico li");
        return user;
    }
```
在访问`UserController`中的任何一个请求处理方法前，都会先执行`initUserModal`方法，并将其返回值以`user`为键添加到模型中。
由于`handle`方法使用入参级的`@ModelAttribute`注解，且属性名和①处方法上的`@ModelAttribute`的属性名相同。这时SpringMVC会从模型中取出①处获得的模型属性，赋给②处的user对象，**然后**再根据请求消息对usr进行属性**填充覆盖**，得到一个整合版本的user对象。

> *注意：*处理方法的入参最多只能使用一个SpringMVC注解。如`handle`方法的user入参使用了`@ModelAttribute`，就不能再使用`@RequestParam`或`@CookieValue`,`@RequestHeader`等注解。如果使用了两个。将抛出异常。


## @SessionAttribute

可以用于获取`HttpSession`中的属性，在入参方法上标注`@SessionAttribute`注解，SpringMVC会将会话中对应的属性绑定到入参，下面是一个具体的实例：

```java
    @PostMapping("/login")
    public User login(@SessionAttribute("user") User user) {
        // do something
        return user;
    }
```

但是很可惜，当向`/login`发送请求时，SpringMVC会抛出异常：

> ServletRequestBindingException: Missing session attribute 'user' of type User

不同于`@ModelAttribute`，如果在`HttpSession`中没有对应的属性，则会抛出异常。那这样看来的话，这个注解是只能用于获取`HttpSession`中的属性吗？当然不是这样的，类似`@ModelAttribute`注解，`@SessionAttribute`还会在方法执行完毕后，将标注该注解的入参对象**再放回**到`HttpSession`中，利用这一特性我们可以在方法中对某个会话属性进行“更新”。

那该如何向会话域中添加属性呢？

- 使用Servlet原生API中的`HttpSession`作为入参时，SpringMVC会将请求会话绑定到入参，然后我们可以操作这个对象去设置会话属性。

- 使用SpringMVC提供的原生Servlet API的代理类`WebRequest`作为入参，使用`WebRequest#setAttribute(String name, Object value, int scope)`方法可同时支持向*请求域*或是*会话域*中添加属性。


## @SessionAttributes

如果希望在多个请求之间共享某个*模型属性数据*，则可以在控制器类上标注一个`@SessionAttributes`，SpringMVC会将**模型中**对应的属性**暂存**到`HttpSession`中。为什么这么说呢？后面会慢慢分析，先来看一个`@SessionAttributes`使用实例：

```java UserController.java
/**
 * @author Lolico li
 */
@Controller
@RequestMapping("/user")
@SessionAttributes("user") // 1 会将2处的模型属性透明的保存到HttpSession中
public class UserController {

    @GetMapping("/h1") // 2
    public String handle1(@ModelAttribute("user") User user) {
        user.setNickName("lolico li");
        return "redirect:/user/h2";
    }

    @ResponseBody
    @GetMapping("/h2")
    public ResponseEntity<Object> handle2(ModelMap modelMap, SessionStatus sessionStatus) {
        User user = (User) modelMap.get("user");
        if (user != null) {
            user.setNickName("griouges");
            sessionStatus.setComplete();
        }
        return ResponseEntity.ok().build();
    }
}
```
在①处标注的`@SessionAttributes("user")`会自动将本处理器的任何处理方法中属性名为user的模型属性透明地存储到`HttpSession`中。在②处，`handler1`方法的user入参会添加到隐含模型中，于是这个模型属性在`handle1`方法执行时，SpringMVC会将其透明的保存到`HttpSession`中。

`handle1`返回的逻辑视图名为*redirect:/user/h2*，重定向发送另一个请求。而这个请求由`handle2`方法负责处理。两个处理方法位于不同的请求上下文中，之所以`handle2`可以获取名为user的模型属性，就是因为`@SessionAttributes("user")`透明的将`handle1`的user模型属性存储到`HttpSession`中，而`handler2`的隐含模型又自动从`HttpSession`中获取到这个模型属性。（PS:一般这种情况我们使用**forWard**转发，这里重定向是为了演示`@SessionAttributes`注解在不同请求上下文中共享**模型属性**的作用）

`handle2`方法还包含一个`SessionStatus`对象，当调用`SessionStatus#setComplete`方法，SpringMVC会清除控制器类的所有会话属性；否则这个会话属性会一直保存在`HttpSession`中，这也是为什么说`@SessionAttributes`是将模型属性**暂存**到`HttpSession`中的原因。

> 我们也可以通过`HttpSession#removeAttribute(String name)`方法手动删除会话属性，但是需要提供属性名，硬编码是不提倡且不方便的。

很可惜，当向`/h1`发送请求时，SpringMVC会抛出异常：

> HttpSessionRequiredException: Session attribute 'user' required - not found in session

这个异常很奇怪，因为Spring仅宣传`@SessionAttributes`的作用是将处理方法对应的模型属性透明的保存到`HttpSession`中，并没有要求`HttpSession`中必须事先拥有对应的模型属性。通过研究SpringMVC的源码，才找到了问题的答案。

原来SpringMVC对`@ModelAttribute`、`@SessionAttributes`的处理遵循一个流程，当流程条件不满足时就会报错。处理流程简单说明如下：

1. SpringMVC再调用处理方法前，在请求的线程中**自动创建**一个隐含的模型对象。

2. 调用所用标注了`@ModelAttribute`的方法，并将方法返回值添加到隐含对象。

3. 查看`Session`中是否存在`@SessionAttributes("xxx")`所指定的xxx属性，如果有，则将其**添加**到隐含模型中。如果模型中已有xxx属性，则覆盖已有的。

4. 对标注了`@ModelAttribute("xxx")`处理方法的入参按以下流程处理。
    
    1. 如果隐含模型拥有名为xxx的属性，则将其赋给该入参，再用请求消息填充该入参对象直接返回，否则转到4.2。

    2. 如果xxx是会话属性，即在处理类上标注了`@SessionAttributes("xxx")`，则尝试从会话中获取该属性，并将其赋给该入参，然后再用请求消息填充该入参对象。**如果在会话中找不到对应的属性，则抛出`HttpSessionRequiredException`异常**。否则转到4.3。

    3. 如果隐含模型中不存在xxx属性，且xxx也不是会话属性，则创建入参的对象实例，然后再用请求消息填充该入参。

分析上面的代码，由于在处理器类上标注了`@SessionAttributes("user")`，所以user为会话属性，在对`handle1`进行处理时会先在隐含模型中查找对应属性，如果没有，则继续在会话中查找这个属性。由于会话中也不存在，因此抛出`HttpSessionRequiredException`异常，也就是说走到了上面的流程的4.2中。

解决这个异常的方法很简单，添加一个标注了`@ModelAttribute("user")`的方法，使得在访问处理方法前先向隐含模型中添加user属性，这样4.1步就会执行，而4.2中也就能在隐含模型中找到对应属性，不会报错。

## 总结

- `@ModelAttribute`注解在方法入参上时就算模型中没有也不会抛出异常（参考上述流程的第1步），在方法执行完毕后透明的将其放入模型中，作用域为请求域，类似于`ServletRequest#setAttribute`；注解在方法上时，处理方法执行前会先执行这个方法，并将返回值放入模型中。

- `@SessionAttribute`只能注解在方法参数，用于访问`Session`中的属性，方法执行完毕后会再将其放入`HttpSession`中，要求`HttpSession`中必须事先拥有对应属性，否则抛出`ServletRequestBindingException：Missing session attribute`异常，可设置`require=false`，作用域为会话域，类似于`HttpSession#setAttribute`

- `@SessionAttributes`可以将指定的模型属性暂存到`HttpSession`中，还有一个作用是将`Session`中的属性填充至模型属性中（参考上述流程的第2步），使得方法入参级的`@ModelAttribute`可以访问到。

- 注意区分`@SessionAttribute`和`@SessionAttributes`的不同
