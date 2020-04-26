---
title: SpringSecurity 跨域
categories: 正常的文章
date: 2020-04-27 00:01:12
tags: [Spring,SpringBoot,Security,CORS,Web]
---

## 写在前面

在SpringSecurity中配置跨域，我相信所有用过SpringSecurity的人应该都知道，因为实在是太简单了。那我为什么还要写这篇文章呢？写这篇文章的目的当然不是去解释如何配置跨域，而是通过分析Spring对跨域支持的源码来感受设计中的优雅。

先声明一下开发环境：`SpringBoot：2.2.2`

## 正文

既然说到SpringSecurity配置跨域，那么我们就先简单复习一下如何配置跨域。

### 配置跨域

我们都知道集成SpringSecurity后配置跨域我们只需要在继承`WebSecurityConfigurerAdapter`类，重写`configure(HttpSecurity http)`方法，开启`cors`并提供一个跨域配置源即可。下面是一个例子：

```java 开启跨域
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS) // 前后端分离
                .and()
                .csrf().disable() // 禁用csrf
                .cors(); // 跨域
        // ... 省略其他配置
    }
    // 提供一个CorsConfigurationSource
    // 这里直接注册成Bean即可，注意方法名必须是corsConfigurationSource，后面会解释
    // 也可以cors().configurationSource(corsConfigurationSource())指定
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.addAllowedOrigin("*"); // 最好根据实际的需要去设置，这里演示，所以粗暴了一点
        configuration.addAllowedMethod("*"); // 同上
        configuration.addAllowedHeader("*"); // 这里起码需要允许 Access-Control-Allow-Origin
        configuration.setMaxAge(3600L);
        configuration.setAllowCredentials(true);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
```

就如前面所说，只需要开启`cors`再提供一个跨域配置源即可。方法很简单，但这里有个坑需要注意一下。

#### 暴露公共接口时跨域的一个坑

> 如果我们还重写了`configure(WebSecurity web)`方法，使用`web.ignoring().antMatchers(ignorePaths)`去暴露一个公共接口'/pub'那么上面的跨域配置对这个接口来说就没用，也就是说这个接口会出现跨域问题。然而我们原本就是为了提供公共接口'/pub'，但现在却有跨域问题，那怎么能行！！！（一般来说这个方法是对静态资源设置直接放行，而不是公共接口！）

*那这到底是为什么呢？*

**因为SpringSecurity配置跨域支持，是通过`CorsFilter`过滤器来实现的**，我们`web.ignoring()`中设置后对应的接口请求就不会经过`CorsFilter`来处理，这个接口当然就存在跨域问题了！之所以说*这个方法是对静态资源设置直接放行，而不是公共接口*也是这个原因，那正确的方法是什么呢？还是`configure(HttpSecurity http)`方法：

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS) // 前后端分离
                .and()
                .csrf().disable() // 禁用csrf
                .cors() // 跨域
                .and()
                .authorizeRequests()
                .antMatchers("/pub/**").permitAll() // 匿名通过认证
                .anyRequest().authenticated() //剩下的任何请求都需要认证
    }
```

### cors方法

现在我们来看一下`cors()`方法，点进这个方法看看，其实很简单，就是应用了一个`CorsConfigurer`配置类。如果看过`SpringSecurity`自动配置，对形如`xxxConfigurer`的类名应该不陌生。
这个`Configurer`其实就是在"FilterChain"上添加了一个过滤器，即`CorsFilter`

我们都知道`CorsFilter`的构造方法需要一个`CorsConfigurationSource`，在请求到来时，使用`CorsProcessor`根据提供的`CorsConfiguration`去对请求进行处理（在`CorsFilter`中默认是`DefaultCorsProcessor`）而`CorsConfiguration`是通过`CorsConfigurationSource#getCorsConfiguration`方法获得的，所以说怎么获得`CorsConfigurationSource`至关重要。

还记得上面在配置`CorsConfigurationSource`时，我们直接注册Bean而不是通过`configurationSource()`方法指定吗？这种方法为什么是可行的呢？来看一下`CorsConfigurer`是如何获得`CorsConfigurationSource`并构造`CorsFilter`的：

```java CorsConfigurer#getCorsFilter
    private CorsFilter getCorsFilter(ApplicationContext context) {
        //如果指定了CorsConfigurationSource，那么用指定的
        if (this.configurationSource != null) {
            return new CorsFilter(this.configurationSource);
        }
        boolean containsCorsFilter = context.containsBeanDefinition(CORS_FILTER_BEAN_NAME);
        //如果容器中已经有名字是’corsFilter‘的bean，则用已经有的
        if (containsCorsFilter) { 
            return context.getBean(CORS_FILTER_BEAN_NAME, CorsFilter.class);
        }
        boolean containsCorsSource = context.containsBean(CORS_CONFIGURATION_SOURCE_BEAN_NAME);
        //如果既没有指定，容器中也不存在名字是’corsFilter‘的CorsFilter
        //那么看一下容器中有没有名字是’corsConfigurationSource‘的CorsConfigurationSource
        //如果有，取出来作为CorsConfigurationSource
        if (containsCorsSource) {
            CorsConfigurationSource configurationSource = context.getBean(CORS_CONFIGURATION_SOURCE_BEAN_NAME, CorsConfigurationSource.class);
            return new CorsFilter(configurationSource);
        }
        //如果也没有corsConfigurationSource，看看类路径下存不存在HandlerMappingIntrospector这个类
        boolean mvcPresent = ClassUtils.isPresent(HANDLER_MAPPING_INTROSPECTOR,context.getClassLoader());
        if (mvcPresent) { //如果存在
            return MvcCorsFilter.getMvcCorsFilter(context);
        }
        return null;
    }
    static class MvcCorsFilter { //内部类
        private static final String HANDLER_MAPPING_INTROSPECTOR_BEAN_NAME = "mvcHandlerMappingIntrospector";
        private static CorsFilter getMvcCorsFilter(ApplicationContext context) {
            if (!context.containsBean(HANDLER_MAPPING_INTROSPECTOR_BEAN_NAME)) {
                throw new NoSuchBeanDefinitionException(HANDLER_MAPPING_INTROSPECTOR_BEAN_NAME, "A Bean named " + 
                HANDLER_MAPPING_INTROSPECTOR_BEAN_NAME +" of type " + HandlerMappingIntrospector.class.getName()
                + " is required to use MvcRequestMatcher. Please ensure Spring Security & Spring MVC are configured in a shared ApplicationContext.");
            }
            // 从容器中取出HandlerMappingIntrospector作为CorsConfigurationSource
            HandlerMappingIntrospector mappingIntrospector = context.getBean(HANDLER_MAPPING_INTROSPECTOR_BEAN_NAME, HandlerMappingIntrospector.class);
            return new CorsFilter(mappingIntrospector);
        }   
    }
```

获取`CorsConfigurationSource`并构造`CorsFilter`的步骤注释里写的很清楚了，正常来说我们配置跨域配置源不管是直接指定也好，还是注册成Bean也好（注意Bean名字的要求），都是可以被获取到的。一般情况下，我们也的确是这样做的（直接提供一个`CorsConfigurationSource`）。但为什么最后有`MvcCorsFilter.getMvcCorsFilter(context)`这样一个调用？通过这个方法里抛出的异常信息不难猜测到是SpringSecurity为了兼容SpringMVC中配置跨域的方式。

还记得不使用SpringSecurity时如何在SpringMVC中配置支持跨域吗？

**两种方式：**

- 将`@CrossOrigin`注解标注在支持跨域的接口上
- 重写`WebMvcConfigurer#addCorsMappings`方法进行全局配置

看到这里你可能会猜测：是不是`HandlerMappingIntrospector`实现了`CorsConfigurationSource`，并且是根据上面两种方式的配置来返回跨域配置的呢？

事实上，的确是这样的。为了便于理解后面给出的代码，先来看看`CorsFilter`类和`CorsConfigurationSource`接口：

#### CorsConfigurationSource

```java
public interface CorsConfigurationSource {
    @Nullable
    CorsConfiguration getCorsConfiguration(HttpServletRequest request);
}
```

跨域配置源，实现类要实现`getCorsConfiguration`方法返回一个`CorsConfiguration`跨域配置，其中包含允许那些域、请求方法、请求头，是否允许携带凭证，缓存时间是多久，允许携带的头属性等信息。

`CorsConfigurationSource`有五个实现类：

- CorsInterceptor
- HandlerMappingIntrospector
- PreFlightHandler
- ResourceHttpRequestHandler
- UrlBasedCorsConfigurationSource

#### CorsFilter

```java
public class CorsFilter extends OncePerRequestFilter {
    private final CorsConfigurationSource configSource;
    private CorsProcessor processor = new DefaultCorsProcessor();

    public CorsFilter(CorsConfigurationSource configSource) {
        Assert.notNull(configSource, "CorsConfigurationSource must not be null");
        this.configSource = configSource;
    }

    public void setCorsProcessor(CorsProcessor processor) {
        Assert.notNull(processor, "CorsProcessor must not be null");
        this.processor = processor;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {
        // 获取CorsConfiguration
        CorsConfiguration corsConfiguration = this.configSource.getCorsConfiguration(request);
        // 根据CorsConfiguration处理请求
        boolean isValid = this.processor.processRequest(corsConfiguration, request, response);
        if (!isValid || CorsUtils.isPreFlightRequest(request)) {
            return;
        }
        filterChain.doFilter(request, response);
    }
}
```

`DefaultCorsProcessor#processRequest`中根据请求是否跨域，是否是预检请求以及`CorsConfiguration`等信息来对请求进行处理和在响应头中写入一些信息。具体的源码就不分析了，还是比较好理解的。前提是需要对CORS有一定的了解，可以看下[HTTP访问控制（CORS）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)这篇文章。

### HandlerMappingIntrospector

`HandlerMappingIntrospector`比较特别，不要认为这是个拦截器，"Introspector"翻译成中文是*内省者*的意思。

这个类在初始化后会调用`afterPropertiesSet`方法，将容器中所有的`HandlerMapping`添加到该类的`handlerMappings`这个`List`中。

来看一下官方对于这个类的解释：

```txt
Helper class to get information from the HandlerMapping that would serve a specific request.
Provides the following methods:
    - getMatchableHandlerMapping(javax.servlet.http.HttpServletRequest) — obtain a HandlerMapping to check request-matching criteria against.
    - getCorsConfiguration(javax.servlet.http.HttpServletRequest) — obtain the CORS configuration for the request.
```

> 这个类是一个帮助类，用于从`HandlerMapping`中获取请求的特定信息，提供了两个方法。第一个方法用于获取一个`MatchableHandlerMapping`来检查请求匹配条件，第二个方法用于获取适用于这个请求的`CorsConfiguration`跨域配置。

我们重点关注第二个方法：

```java HandlerMappingIntrospector#getCorsConfiguration
    public CorsConfiguration getCorsConfiguration(HttpServletRequest request) {
        Assert.notNull(this.handlerMappings, "Handler mappings not initialized");
        HttpServletRequest wrapper = new RequestAttributeChangeIgnoringWrapper(request);
        for (HandlerMapping handlerMapping : this.handlerMappings) {
            HandlerExecutionChain handler = null;
            try {
                handler = handlerMapping.getHandler(wrapper); // 获取处理执行链
            }
            catch (Exception ex) {
                // Ignore
            }
            if (handler == null) {
                continue;
            }
            if (handler.getInterceptors() != null) {
                //遍历拦截器，如果拦截器同时实现了CorsConfigurationSource则用这个拦截器作为跨域配置源
                for (HandlerInterceptor interceptor : handler.getInterceptors()) {
                    if (interceptor instanceof CorsConfigurationSource) {
                        return ((CorsConfigurationSource) interceptor).getCorsConfiguration(wrapper);
                    }
                }
            }
            //从执行链获取处理器，如果处理器本身也实现了CorsConfigurationSource，则用处理器作为跨域配置源
            if (handler.getHandler() instanceof CorsConfigurationSource) {
                return ((CorsConfigurationSource) handler.getHandler()).getCorsConfiguration(wrapper);
            }
        }
        return null;
    }
```

这个`CorsConfigurationSource`实现类根据请求从`HandlerMapping`中获取获取`HandlerExecutionChain`执行链，再依次从执行链的拦截器和处理器中获取`CorsConfigurationSource`，如果获取到了再调用其`HandlerMappingIntrospector#getCorsConfiguration`方法返回跨域配置。具体来说就是那两个if判断。

所以这么说来的话，`HandlerMappingIntrospector`虽然实现了`CorsConfigurationSource`但其本质有点像一个委托类？它检查请求对应的执行链上的拦截器和处理器有没有实现`CorsConfigurationSource`，如果有，再委托给这个`CorsConfigurationSource`来获取`CorsConfiguration`。所以说如果我们在一个`Controller`的接口上标注了`@CrossOrigin`注解，那么对应的，在拦截器中获取不到`CorsConfiguration`，就会从这个Handler上获取到`CorsConfiguration`，也就是将`@CrossOrigin`注解中提供的信息封装成了`CorsConfiguration`。那为什么还会先检查执行链中的拦截器呢？

因为SpringMVC中还有第二种方法配置跨域支持，也就是上面提到的重写`WebMvcConfigurer#addCorsMappings`方法进行全局配置。那为什么重写这个方法添加跨域配置最后会注册成拦截器呢？（一个实现了`CorsConfigurationSource`的拦截器）

这就要说到SpringBoot在WebMvc的自动配置、`WebMvcConfigurer`和`HandlerMapping`了。

如果你有仔细看过SpringBoot在SpringMVC的自动配置方面的源码，你一定知道`WebMvcConfigurationSupport`这个最主要的配置类在注册`HandlerMapping`的时候会从一个`CorsRegisty`中获取跨域配置：（这里以`RequestMappingHandlerMapping`为例）

```java WebMvcConfigurationSupport.java
@Bean
public RequestMappingHandlerMapping requestMappingHandlerMapping(
        @Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
        @Qualifier("mvcConversionService") FormattingConversionService conversionService,
        @Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {
            
    RequestMappingHandlerMapping mapping = createRequestMappingHandlerMapping();
    mapping.setOrder(0);
    mapping.setInterceptors(getInterceptors(conversionService, resourceUrlProvider));
    mapping.setContentNegotiationManager(contentNegotiationManager);
    mapping.setCorsConfigurations(getCorsConfigurations()); //设置跨域配置
    // ...省略一大段set
    return mapping;
}
protected final Map<String, CorsConfiguration> getCorsConfigurations() {
    if (this.corsConfigurations == null) {
        CorsRegistry registry = new CorsRegistry();
        addCorsMappings(registry);  //向CorsRegistry中添加跨域映射
        this.corsConfigurations = registry.getCorsConfigurations(); //获取跨域配置
    }
    return this.corsConfigurations;
}
```

`addCorsMappings()`方法是个空方法，并且只有`DelegatingWebMvcConfiguration`类重写了这个方法。实际上`WebMvcConfigurationSupport`这个类中用`@Bean`这个可传递的注解标注了很多方法但该类上并没有标注`@Configuration`，那么为什么还会起到配置类的作用呢？其实真正的配置类是`DelegatingWebMvcConfiguration`。

### DelegatingWebMvcConfiguration

在`DelegatingWebMvcConfiguration`这个类上有个`@Configuration`注解，并且继承自`WebMvcConfigurationSupport`，实际上它就是个委托类。

*可以说这个类才是真正的配置类，去看看`DelegatingWebMvcConfiguration`这个类，相信你一定会发现什么！！！*

`DelegatingWebMvcConfiguration`中有`WebMvcConfigurerComposite`这么一个对象，并且将容器中所有`WebMvcConfigurer`注入进来：

```java DelegatingWebMvcConfiguration.java
@Configuration(proxyBeanMethods = false)
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
    // WebMvcConfigurer复合类
    private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();
    // 将容器中所有WebMvcConfigurer添加到WebMvcConfigurerComposite
    @Autowired(required = false)
    public void setConfigurers(List<WebMvcConfigurer> configurers) {
        if (!CollectionUtils.isEmpty(configurers)) {
            this.configurers.addWebMvcConfigurers(configurers);
        }
    }
    // ...省略
}
```

如果你去看了一下这个类的源码，你就会发现`WebMvcConfigurer`中有的方法这个类都有，并且这个委托类仅仅是将请求委托给`configurers`，来看看重写的`addCorsMappings`方法：

```java DelegatingWebMvcConfiguration#addCorsMappings
    @Override
    protected void addCorsMappings(CorsRegistry registry) {
        this.configurers.addCorsMappings(registry);
    }
```

调用`WebMvcConfigurerComposite#addCorsMappings`，显而易见`WebMvcConfigurerComposite`是个复合的`WebMvcConfigurer`，他也实现了`WebMvcConfigurer`并且内部维护了一个`List<WebMvcConfigurer> delegates`列表，实现的所有方法会依次调用列表中`WebMvcConfigurer`对应的方法。（并且你还能发现`WebMvcConfigurer`中的方法都是作为回调方法并且大部分是返回void的）

> 说到这里，不得不说一个题外话。如果看Spring源码比较多的话，就会发现Spring中类的命名都有规律可循并且某些后缀都是有特定意义的，比如`xxxComposite`、`xxxConfigurer`、`Delegatingxxx`、`xxxDelegator`等等，这样我们看到这个类名就立马能猜到它的作用。

我们平时对WebMvc进行一些配置都是实现`WebMvcConfigurer`类，重写其中的方法。下面是一个例子：

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowedHeaders("*")
                .allowedMethods("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
    // ...
}
```

说到这，也就是相当于`WebMvcConfigurationSupport#getCorsConfigurations`方法会回调容器中所有`WebMvcConfigurer`实现类的`addCorsMappings()`方法，向`CorsRegistry`中添加跨域映射，然后再取出`CorsConfiguration`返回：

```java
    protected final Map<String, CorsConfiguration> getCorsConfigurations() {
        if (this.corsConfigurations == null) {
            CorsRegistry registry = new CorsRegistry();
            addCorsMappings(registry); //向CorsRegistry中添加跨域映射
            this.corsConfigurations = registry.getCorsConfigurations(); //获取跨域配置
        }
        return this.corsConfigurations;
    }
```

最后反应到`AbstractHandlerMapping`中的就是使用`CorsConfiguration`注册一个`CorsInterceptor`拦截器，这个拦截器是`AbstractHandlerMapping`中的一个内部类，继承自`HandlerInterceptorAdapter`，并且实现了`CorsConfigurationSource`。

> 看到这里，如果没有了解过`HandlerMapping`，可能会一头雾水，可以看看我的这篇文章{% post_link 源码角度分析Spring容器启动阶段注册Controller处理器的流程 %}，虽然不是讲`HandlerMapping`，但是相信在看完后，会对`HandlerMapping`有一个理解。

### CorsInterceptor

```java
private class CorsInterceptor extends HandlerInterceptorAdapter implements CorsConfigurationSource {
    @Nullable
    private final CorsConfiguration config;
    
    public CorsInterceptor(@Nullable CorsConfiguration config) {
        this.config = config;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)throws Exception {
		return corsProcessor.processRequest(this.config, request, response);
	}

    @Override
    @Nullable
    public CorsConfiguration getCorsConfiguration(HttpServletRequest request) {
        return this.config;
    }
}
```

这个类只重写了拦截器的`preHandle`方法，其他方法都是空方法。而且你能发现这个`preHandle`方法中的内容和`CorsFilter#doFilterInternal`方法基本是一模一样的，都是根据`CorsConfiguration`使用跨域处理器处理请求。

看到这里，现在应该知道关于`HandlerMappingIntrospector`的猜测是没错的，并且知道了`HandlerMappingIntrospector`是如何与SpringMVC两种支持跨域的配置方式联系起来的，这里再次总结一下：

1. 首先获取请求对应的执行链上的拦截器，判断拦截器有没有实现`CorsConfigurationSource`（`CorsInterceptor`类），如果有则调用`getCorsConfiguration`获取`CorsConfiguration`后返回
2. 如果拦截器上获取失败，则判断处理器有没有实现`CorsConfigurationSource`（`PreFlightHandler`类），如果有则调用`getCorsConfiguration`获取`CorsConfiguration`后返回

从而实现了兼容SpringMVC中两种配置跨域的方式。

这其中最关键的几点就在于`CorsConfigurer`获取`CorsConfigurationSource`并且构造`CorsFilter`的步骤、`HandlerMappingIntrospector`获取`CorsConfiguration`的步骤，还有Spring回调`WebMvcConfigurer`对`HandlerMapping`进行设置跨域配置等信息的步骤

其中还涉及到了SpringMVC中`HandlerMapping`、`HandlerExecutionChain`、`Handler`、`Interceptor`等相关知识。

**根据这次的分析，能得到几个结论：**

- SpringMVC支持跨域两种方式一个是基于处理器实现，另一个是基于拦截器实现。
- SpringSecurity跨域是基于过滤器，并且兼容了SpringMVC的两种配置（使用`HandlerMappingIntrospector`“桥接”）。
- SpringSecurity中的`CorsConfigurer`使用`HandlerMappingIntrospector`来兼容SpringMVC跨域两种方式。
- `HandlerMappingIntrospector`获取`CorsConfiguration`时的优先级是先拦截器，再处理器。
- SpringBoot注册`HandlerMapping`或者说通过`WebMvcAutoConfiguration`自动配置来对WebMvc必要的组件进行装配和注入。
- `WebMvcConfigurer`是`DelegatingWebMvcConfiguration`类驱动`WebMvcConfigurerComposite`来进行回调的。

并且经过这次的源码阅读，也是足足感受到Spring设计上的优雅。

## 写在最后

文章写的有点乱，并且有点跳跃。仅仅是跟着文章来看可能不大能看懂，最好在电脑上根据源码来阅读。这篇文章也仅仅是作为我个人在一次踩坑后好奇心大法，阅读源码后的一段总结以及感悟吧，自己能看懂并且以后还能看懂也就满意了。如果这篇文章有幸被你刷到并且你能够看懂我想表达的那我自然是更高兴。其实这个博客存在的理由也仅仅是为了记录自己学习过程中的感悟和总结，便于自己以后回顾，~~毕竟我比较健忘~~。所以需要记录下有必要的，并且在个人看来，这篇文章干货还是足足的，所以说更加有必要记录。其实在写文章之初我也不想写这么一篇文章，因为实在是太难写明白了，并且由于涉及到的东西比较分散很难进行组织，~~也可能我表达能力差的原因吧~~，但最终还是花了一下午加一晚上，在不断修改下产出了这么一篇很长很长很长的文章，可能是写过的字数最多的文章了吧😥。文章中可能有错别字也可能有错误的内容，如果你发现文章有什么错误的地方或者没表述清楚的内容，欢迎在评论中交流。
