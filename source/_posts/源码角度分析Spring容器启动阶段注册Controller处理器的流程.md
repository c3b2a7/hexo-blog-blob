---
title: 源码角度分析Spring容器启动阶段注册Controller处理器的流程
categories: 正常的文章
date: 2020-03-09 16:09:08
tags: [Java,Spring,Web]
---

## 前言

我们都知道，在一个请求被前端控制器`DispatchServlet`捕获后会经历下面几个流程：

1. `DispatherServlet`根据请求URL解析获取请求URI，调用`HandlerMapping#getHandler`方法获取`HandlerExecutionChain`
2. 获取返回的`HandlerExecutionChain`处理器执行链（包括处理器对象和拦截器对象）
3. 根据处理器执行链获取一个处理器适配器`HandlerAdapter`，如果成功获取，则开始执行拦截器）
4. 处理器适配器根据请求的`Handler`（一般来说是`HandlerMethod`）适配并根据配置的`HttpMessageConveter`将请求消息解析为模型数据，填充 `Handler`入参，开始执行处理器逻辑。
5. 处理器执行完毕，返回`ModelAndView`，处理器适配器接收到后返回给`DispatherServlet`
6. `DispatherServlet`根据模型和视图请求对应的视图解析器
7. 视图解析器解析模型数据获取对应的视图，渲染视图后返回给`DispatherServlet`
8. `DispatherServlet`将渲染后的视图相应给用户或客户端

那么Spring容器启动后是如何自动发现处理器（我在这称之为**自动发现机制**）并进行注册的呢？

> 在web环境下Spring容器启动时会注册`HandlerMapping`，具体在SpringBoot中，`WebMvcAutoConfiguration`会通过`WebMvcConfigurationSupport#requestMappingHandlerMapping`方法向容器中注册一个`HandlerMapping`的实现`RequestMappingHandlerMapping`

## 自动发现机制的实现

Spring容器启动时会注册`HandlerMapping`，在这里我们就以`RequestMappingHandlerMapping`实现类为例进行分析

### RequestMappingHandlerMapping

`RequestMappingHandlerMapping#afterPropertiesSet`方法实际上调用了父类的`afterPropertiesSet`的方法：

```java RequestMappingHandlerMapping#afterPropertiesSet
	public void afterPropertiesSet() {
        ...
		super.afterPropertiesSet();
	}
```

而在父类的这个方法中进行初始化处理器逻辑：

```java AbstractHandlerMethodMapping#afterPropertiesSet
	/**
	 * Detects handler methods at initialization.
	 * @see #initHandlerMethods
	 */
	@Override
	public void afterPropertiesSet() {
		initHandlerMethods();
	}
```

在`initHandlerMethods`方法中先从容器中获取所有的候选bean，调用`processCandidateBean`方法：

```java AbstractHandlerMethodMapping#initHandlerMethods
	/**
	 * Scan beans in the ApplicationContext, detect and register handler methods.
	 * @see #getCandidateBeanNames()
	 * @see #processCandidateBean
	 * @see #handlerMethodsInitialized
	 */
	protected void initHandlerMethods() {
		for (String beanName : getCandidateBeanNames()) {
			if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
				processCandidateBean(beanName);
			}
		}
		handlerMethodsInitialized(getHandlerMethods());
	}
```

在`processCandidateBean`方法中根据bean的类型调用由子类实现的`isHandler`方法判断是否是处理器，然后调用`detectHandlerMethods`方法寻找该处理器中的处理方法并注册。

来看下`RequestMappingHandlerMapping#isHandler`

```java
	protected boolean isHandler(Class<?> beanType) {
		return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
				AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
	}
```

对应的`AnnotatedElementUtils#hasAnnotation`方法，最终会调用到`AnnotatedElementUtils#searchWithFindSemantics`方法，代码片段如下

```java
  else if (element instanceof Class) {
    Class<?> clazz = (Class<?>) element;
    if (!Annotation.class.isAssignableFrom(clazz)) {
        // Search on interfaces 在实现接口中查找
        for (Class<?> ifc : clazz.getInterfaces()) {
            T result = searchWithFindSemantics(ifc, annotationTypes, annotationName,
                    containerType, processor, visited, metaDepth);
            if (result != null) {
                return result;
            }
        }
        // Search on superclass 在父类中查找
        Class<?> superclass = clazz.getSuperclass();
        if (superclass != null && superclass != Object.class) {
            T result = searchWithFindSemantics(superclass, annotationTypes, annotationName,
                    containerType, processor, visited, metaDepth);
            if (result != null) {
                return result;
            }
        }
    }
}
```

发现这个地方查找是否有指定注解时，如果继承的类或实现的接口有相应的注解也是可以的，这个特性在某些情况下是很有用的，比如在自动配置类中可以利用这一特点，使用`@Bean`注解再配合`@Conditionalxxx`等注解实现按需注入Controller。我们只要定义一个*标记类/接口*，再在该类上注解`@Controller`或`@RequestMapping`，然后让需要按需注入的Controller继承这个*标记类/接口*即可。

>  注意，在一般情况下使用`@Bean`或`@Component`注解是不能将一个Bean注册为Controller的

#### AbstractHandlerMethodMapping#detectHandlerMethods

现在我们来看下这个自动发现机制中最关键的方法：

```java AbstractHandlerMethodMapping#detectHandlerMethods
	protected void detectHandlerMethods(Object handler) {
		Class<?> handlerType = (handler instanceof String ?
				obtainApplicationContext().getType((String) handler) : handler.getClass());

		if (handlerType != null) {
            // 获取用户定义的类，防止在代理模式下不能获取到方法以及上面的注解
			Class<?> userType = ClassUtils.getUserClass(handlerType);
            // 返回方法及其相关元数据的一个map，元数据是回调lambda获取的，可以看下MetadataLookup#inspect函数式接口
			Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
					(MethodIntrospector.MetadataLookup<T>) method -> {
						try {
                            // 由子类实现的抽象方法
                            // 在RequestMappingHandlerMapping中是RequestMappingInfo
							return getMappingForMethod(method, userType);
						}
						catch (Throwable ex) {
							throw new IllegalStateException("Invalid mapping on handler class [" +
									userType.getName() + "]: " + method, ex);
						}
					});
			if (logger.isTraceEnabled()) {
				logger.trace(formatMappings(userType, methods));
			}
			methods.forEach((method, mapping) -> {
                // 获取该方法的可调用方法
				Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
                // 注册
				registerHandlerMethod(handler, invocableMethod, mapping);
			});
		}
	}
```

而在`RequestMappingHandlerMapping#getMappingForMethod`中先检查是否注解了`@RequestMapping`，如果有则获取注解中的请求的路径，请求的方式（GET,POST,...)等等信息，封装到`RequestMappingInfo`中返回，具体的代码就不放了，还是很好理解的。

再来看下最后这个`AbstractHandlerMethodMapping#registerHandlerMethod`方法

```java AbstractHandlerMethodMapping#registerHandlerMethod
	protected void registerHandlerMethod(Object handler, Method method, T mapping) {
		this.mappingRegistry.register(mapping, handler, method);
	}
```

很好理解，调用`mappingRegistry#register`方法进行注册，那这个`mappingRegistry`是什么东西呢？

>  `MappingRegistry`实际上就是`AbstractHandlerMethodMapping`类中的一个内部类，看名字很好理解：映射注册表，这个类中维护了存放映射信息的map

### MappingRegistry

`MappingRegistry`是`AbstractHandlerMethodMapping`类中的一个内部类，在其中维护了五个用于存放映射信息的Map：

```java MappingRegistry类字段
		// 真正意义上的注册表，以RequestMappingInfo为key
		// MappingRegistration中存放请求映射信息RequestMappingInfo、处理器方法HandlerMethod（即标注了@RequestMapping的处理方法）、映射路径url、处理器name
		private final Map<T, MappingRegistration<T>> registry = new HashMap<>();
		// RequestMappingInfo和HandlerMethod的map
		private final Map<T, HandlerMethod> mappingLookup = new LinkedHashMap<>();
		// url(请求路径)和RequestMappingInfo的map
		private final MultiValueMap<String, T> urlLookup = new LinkedMultiValueMap<>();
		// 处理器name和处理器方法的map
		private final Map<String, List<HandlerMethod>> nameLookup = new ConcurrentHashMap<>();
		// 处理器方法和跨域配置的map
		private final Map<HandlerMethod, CorsConfiguration> corsLookup = new ConcurrentHashMap<>();
```

#### MappingRegistry#register

```java  MappingRegistry#register
		public void register(T mapping, Object handler, Method method) {
			// Assert that the handler method is not a suspending one.
			if (KotlinDetector.isKotlinType(method.getDeclaringClass()) && KotlinDelegate.isSuspend(method)) {
				throw new IllegalStateException("Unsupported suspending handler method detected: " + method);
			}
			this.readWriteLock.writeLock().lock();
			try {
                // 创建处理器方法
				HandlerMethod handlerMethod = createHandlerMethod(handler, method);
				validateMethodMapping(handlerMethod, mapping);
                // 将映射信息和处理器方法放到map中
				this.mappingLookup.put(mapping, handlerMethod);
				// 获取映射的请求路径数组，在RequestMapping注解中可对一个方法指定多个映射路径
				List<String> directUrls = getDirectUrls(mapping);
				for (String url : directUrls) {
                    // 放入map
					this.urlLookup.add(url, mapping);
				}
                // 如果name存储策略不为空，将处理器name和处理器方法放到nameLookup map中
				String name = null;
				if (getNamingStrategy() != null) {
					name = getNamingStrategy().getName(handlerMethod, mapping);
                    // 将处理器name和处理器方法放到nameLookup map中
					addMappingName(name, handlerMethod);
				}
                // 跨域配置
				CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
				if (corsConfig != null) {
					this.corsLookup.put(handlerMethod, corsConfig);
				}
                // 将映射信息和MappingRegistration放到注册表Map
				this.registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod, directUrls, name));
			}
			finally {
				this.readWriteLock.writeLock().unlock();
			}
		}
```

这个方法说白了，就是把处理器，处理器方法，请求映射信息等等信息放到Map中，看到这里可能会对`RequestMappingInfo`和 `MappingRegistration`这两个东西感到疑惑，其实很简单，前者是Spring容器启动时控制器自动发现机制根据方法上`@RequestMapping`注解中的信息封装的对象，**决定了什么样的请求能命中那个处理器方法**，这也是`MappingRegistry#registry`注册表中以`RequestMappingInfo`为key的原因，而`MappingRegistration`中封装了处理器`Handler`,处理器方法`HandlerMethod`信息。

</br>

所以现在能知道一个请求到来时是如何**找到处理器方法**并调用的了：

> 根据请求的url到`MappingRegistry#urlLookup`字段中匹配，如果匹配上，则取出对应的`RequestMappingInfo`，再到`MappingRegistry#registry`字段中取出`MappingRegistration`，再取出`MappingRegistration`中的处理器`Handler`和方法`HandlerMethod`，反射调用完成请求。

当然，这只是大概的流程。



## 后语

Spring容器启动，注册处理器的流程分析完了。其实说白了，Spring容器注册控制器就是扫描容器中的bean然后检查是否是控制器Controller，然后将其中注解有`@RequestMapping`的方法注册为处理器方法。

其实`HandlerMapping`的实现类并不是只有`RequestMappingHandlerMapping`一个，但是是`AbstractHandlerMethodMapping`提供的**主要的**实现逻辑，而实现类只是提供了基础的判断：是否是处理器（`isHandler`)，获取请求映射信息`getMappingForMethod`等抽象方法，所以说如果在某些场景下需要实现自定义的`HandlerMapping`时我们可以通过继承`RequestMappingInfoHandlerMapping`然后重写`isHandler`和`getMappingForMethod`方法即可。（`RequestMappingHandlerMapping`继承自`RequestMappingInfoHandlerMapping`）



至于案例，以后有空了再补上