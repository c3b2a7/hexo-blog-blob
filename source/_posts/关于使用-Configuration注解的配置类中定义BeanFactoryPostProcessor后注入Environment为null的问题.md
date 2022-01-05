---
title: 关于使用@Configuration注解的配置类中定义BeanFactoryPostProcessor后注入Environment为null的问题
categories: 正常的文章
date: 2019-12-17 17:42:22
tags: [Java,Spring]
---

## 问题由来

**异常：**
```java
org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'empDao': Unsatisfied dependency expressed through field 'jdbcTemplate'; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'jdbcTemplate' defined in class path resource [me/lolico/App.class]: Unsatisfied dependency expressed through method 'jdbcTemplate' parameter 0; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dataSource' defined in class path resource [me/lolico/App.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [javax.sql.DataSource]: Factory method 'dataSource' threw exception; nested exception is java.lang.NullPointerException

	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject(AutowiredAnnotationBeanPostProcessor.java:643)
	at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:116)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.postProcessProperties(AutowiredAnnotationBeanPostProcessor.java:399)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean(AbstractAutowireCapableBeanFactory.java:1422)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:594)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:517)
	at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:323)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:222)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:321)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:202)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:879)
	at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:878)
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:550)
	at org.springframework.context.support.ClassPathXmlApplicationContext.<init>(ClassPathXmlApplicationContext.java:144)
	at org.springframework.context.support.ClassPathXmlApplicationContext.<init>(ClassPathXmlApplicationContext.java:85)
	at me.lolico.AppTest.IOC(AppTest.java:24)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.junit.runners.model.FrameworkMethod$1.runReflectiveCall(FrameworkMethod.java:50)
	at org.junit.internal.runners.model.ReflectiveCallable.run(ReflectiveCallable.java:12)
	at org.junit.runners.model.FrameworkMethod.invokeExplosively(FrameworkMethod.java:47)
	at org.junit.internal.runners.statements.InvokeMethod.evaluate(InvokeMethod.java:17)
	at org.junit.runners.ParentRunner.runLeaf(ParentRunner.java:325)
	at org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:78)
	at org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:57)
	at org.junit.runners.ParentRunner$3.run(ParentRunner.java:290)
	at org.junit.runners.ParentRunner$1.schedule(ParentRunner.java:71)
	at org.junit.runners.ParentRunner.runChildren(ParentRunner.java:288)
	at org.junit.runners.ParentRunner.access$000(ParentRunner.java:58)
	at org.junit.runners.ParentRunner$2.evaluate(ParentRunner.java:268)
	at org.junit.runners.ParentRunner.run(ParentRunner.java:363)
	at org.junit.runner.JUnitCore.run(JUnitCore.java:137)
	at com.intellij.junit4.JUnit4IdeaTestRunner.startRunnerWithArgs(JUnit4IdeaTestRunner.java:68)
	at com.intellij.rt.junit.IdeaTestRunner$Repeater.startRunnerWithArgs(IdeaTestRunner.java:33)
	at com.intellij.rt.junit.JUnitStarter.prepareStreamsAndStart(JUnitStarter.java:230)
	at com.intellij.rt.junit.JUnitStarter.main(JUnitStarter.java:58)
Caused by: org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'jdbcTemplate' defined in class path resource [me/lolico/App.class]: Unsatisfied dependency expressed through method 'jdbcTemplate' parameter 0; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dataSource' defined in class path resource [me/lolico/App.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [javax.sql.DataSource]: Factory method 'dataSource' threw exception; nested exception is java.lang.NullPointerException
	at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:798)
	at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:539)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateUsingFactoryMethod(AbstractAutowireCapableBeanFactory.java:1338)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1177)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:557)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:517)
	at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:323)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:222)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:321)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:202)
	at org.springframework.beans.factory.config.DependencyDescriptor.resolveCandidate(DependencyDescriptor.java:276)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1287)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1207)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject(AutowiredAnnotationBeanPostProcessor.java:640)
	... 37 more
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dataSource' defined in class path resource [me/lolico/App.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [javax.sql.DataSource]: Factory method 'dataSource' threw exception; nested exception is java.lang.NullPointerException
	at org.springframework.beans.factory.support.ConstructorResolver.instantiate(ConstructorResolver.java:656)
	at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:484)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateUsingFactoryMethod(AbstractAutowireCapableBeanFactory.java:1338)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1177)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:557)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:517)
	at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:323)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:222)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:321)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:202)
	at org.springframework.beans.factory.config.DependencyDescriptor.resolveCandidate(DependencyDescriptor.java:276)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1287)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1207)
	at org.springframework.beans.factory.support.ConstructorResolver.resolveAutowiredArgument(ConstructorResolver.java:885)
	at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:789)
	... 50 more
Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [javax.sql.DataSource]: Factory method 'dataSource' threw exception; nested exception is java.lang.NullPointerException
	at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:185)
	at org.springframework.beans.factory.support.ConstructorResolver.instantiate(ConstructorResolver.java:651)
	... 64 more
Caused by: java.lang.NullPointerException
	at me.lolico.App.dataSource(App.java:30)
	at me.lolico.App$$EnhancerBySpringCGLIB$$5fe7f88f.CGLIB$dataSource$0(<generated>)
	at me.lolico.App$$EnhancerBySpringCGLIB$$5fe7f88f$$FastClassBySpringCGLIB$$c81ae23e.invoke(<generated>)
	at org.springframework.cglib.proxy.MethodProxy.invokeSuper(MethodProxy.java:244)
	at org.springframework.context.annotation.ConfigurationClassEnhancer$BeanMethodInterceptor.intercept(ConfigurationClassEnhancer.java:363)
	at me.lolico.App$$EnhancerBySpringCGLIB$$5fe7f88f.dataSource(<generated>)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:154)
	... 65 more
```

## 分析
通过错误堆栈信息不难看出异常是在spring容器启动后实例化bean时抛出的。
查看堆栈信息可以发现容器启动后：
> 创建EmpDao->发现其依赖jdbcTemplate
> 创建jdbcTemplate->发现其依赖dataSource
> 创建dataSource->抛出异常

堆底异常：

```java
Caused by: java.lang.NullPointerException
	at me.lolico.App.dataSource(App.java:30)
	at me.lolico.App$$EnhancerBySpringCGLIB$$5fe7f88f.CGLIB$dataSource$0(<generated>)
	at me.lolico.App$$EnhancerBySpringCGLIB$$5fe7f88f$$FastClassBySpringCGLIB$$c81ae23e.invoke(<generated>)
	at org.springframework.cglib.proxy.MethodProxy.invokeSuper(MethodProxy.java:244)
	at org.springframework.context.annotation.ConfigurationClassEnhancer$BeanMethodInterceptor.intercept(ConfigurationClassEnhancer.java:363)
	at me.lolico.App$$EnhancerBySpringCGLIB$$5fe7f88f.dataSource(<generated>)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:154)
	... 65 more
```

那么好，现在知道问题是怎么发生的，那怎么解决呢？遂去查看配置类中dataSource定义是不是有问题：
```java
@PropertySource("classpath:jdbc.properties")
@ComponentScan(basePackageClasses = App.class)
@Configuration
public class App {
    @Autowired
    private Environment env;
    
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
    
    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName(Objects.requireNonNull(env.getProperty("mysql.driver")));
        dataSource.setUrl(env.getProperty("mysql.url"));
        dataSource.setUsername(env.getProperty("mysql.username"));
        dataSource.setPassword(env.getProperty("mysql.password"));
        return dataSource;
    }
    
    @Bean
    public TransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
    
    @Bean
    public BeanFactoryPostProcessor beanFactoryPostProcessor() {
        return new TestBeanFactoryPostProcessor();
    }
    
}
```
没有问题，那为什么创建的dataSource为`null`呢？想起之前跑测试case都没错，出错前在配置类中定义了一个`BeanFactoryPostProcessor`,就是这个东西:
```java
    @Bean
    public BeanFactoryPostProcessor beanFactoryPostProcessor() {
        return new TestBeanFactoryPostProcessor();
    }
```
但是异常和定义的这个`BeanFactoryPostProcessor`扯不上关系啊，而且TestBeanFactoryPostProcessor中也没有逻辑。遂查看框架日志。
## 找到原因所在
```log
2019-12-17 14:20:49,658 [main] [DEBUG] - Refreshing org.springframework.context.support.ClassPathXmlApplicationContext@7e0babb1
2019-12-17 14:20:49,932 [main] [DEBUG] - Loaded 15 bean definitions from class path resource [applicationContext.xml]
2019-12-17 14:20:49,960 [main] [DEBUG] - Creating shared instance of singleton bean 'org.springframework.context.annotation.internalConfigurationAnnotationProcessor'
2019-12-17 14:20:50,103 [main] [DEBUG] - Identified candidate component class: file [D:\ws_idea\springmvc\target\classes\me\lolico\dao\EmpDao.class]
2019-12-17 14:20:50,228 [main] [DEBUG] - Creating shared instance of singleton bean 'org.springframework.context.event.internalEventListenerProcessor'
2019-12-17 14:20:50,230 [main] [DEBUG] - Creating shared instance of singleton bean 'beanFactoryPostProcessor'
2019-12-17 14:20:50,231 [main] [DEBUG] - Creating shared instance of singleton bean 'me.lolico.App#0'
2019-12-17 14:20:50,234 [main] [INFO] - @Bean method App.beanFactoryPostProcessor is non-static and returns an object assignable to Spring's BeanFactoryPostProcessor interface. This will result in a failure to process annotations such as @Autowired, @Resource and @PostConstruct within the method's declaring @Configuration class. Add the 'static' modifier to this method to avoid these container lifecycle issues; see @Bean javadoc for complete details.
2019-12-17 14:20:50,248 [main] [DEBUG] - Creating shared instance of singleton bean 'org.springframework.context.event.internalEventListenerFactory'
2019-12-17 14:20:50,249 [main] [DEBUG] - Creating shared instance of singleton bean 'org.springframework.context.annotation.internalAutowiredAnnotationProcessor'
2019-12-17 14:20:50,251 [main] [DEBUG] - Creating shared instance of singleton bean 'org.springframework.context.annotation.internalCommonAnnotationProcessor'
2019-12-17 14:20:50,254 [main] [DEBUG] - Creating shared instance of singleton bean 'org.springframework.context.annotation.internalPersistenceAnnotationProcessor'
2019-12-17 14:20:50,262 [main] [DEBUG] - Creating shared instance of singleton bean 'org.springframework.web.servlet.resource.DefaultServletHttpRequestHandler#0'
2019-12-17 14:20:50,268 [main] [DEBUG] - Creating shared instance of singleton bean 'org.springframework.web.servlet.handler.SimpleUrlHandlerMapping#0'
2019-12-17 14:20:50,330 [main] [DEBUG] - Patterns [/**] in 'org.springframework.web.servlet.handler.SimpleUrlHandlerMapping#0'
2019-12-17 14:20:50,330 [main] [DEBUG] - Creating shared instance of singleton bean 'mvcCorsConfigurations'
2019-12-17 14:20:50,331 [main] [DEBUG] - Creating shared instance of singleton bean 'org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping'
2019-12-17 14:20:50,337 [main] [DEBUG] - Creating shared instance of singleton bean 'org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter'
2019-12-17 14:20:50,338 [main] [DEBUG] - Creating shared instance of singleton bean 'org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter'
2019-12-17 14:20:50,339 [main] [DEBUG] - Creating shared instance of singleton bean 'org.springframework.web.servlet.view.InternalResourceViewResolver#0'
2019-12-17 14:20:50,366 [main] [DEBUG] - Creating shared instance of singleton bean 'empDao'
2019-12-17 14:20:50,378 [main] [DEBUG] - Creating shared instance of singleton bean 'jdbcTemplate'
2019-12-17 14:20:50,383 [main] [DEBUG] - Creating shared instance of singleton bean 'dataSource'
2019-12-17 14:20:50,386 [main] [WARN] - Exception encountered during context initialization - cancelling refresh attempt: org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'empDao': Unsatisfied dependency expressed through field 'jdbcTemplate'; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'jdbcTemplate' defined in class path resource [me/lolico/App.class]: Unsatisfied dependency expressed through method 'jdbcTemplate' parameter 0; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dataSource' defined in class path resource [me/lolico/App.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [javax.sql.DataSource]: Factory method 'dataSource' threw exception; nested exception is java.lang.NullPointerException
```
注意第六条日志记录及以后几条记录：
```log
2019-12-17 14:20:50,230 [main] [DEBUG] - Creating shared instance of singleton bean 'beanFactoryPostProcessor'
2019-12-17 14:20:50,231 [main] [DEBUG] - Creating shared instance of singleton bean 'me.lolico.App#0'
2019-12-17 14:20:50,234 [main] [INFO] - @Bean method App.beanFactoryPostProcessor is non-static and returns an object assignable to Spring's BeanFactoryPostProcessor interface. This will result in a failure to process annotations such as @Autowired, @Resource and @PostConstruct within the method's declaring @Configuration class. Add the 'static' modifier to this method to avoid these container lifecycle issues; see @Bean javadoc for complete details.
```

Spring容器启动：创建'beanFactoryPostProcessor'Bean，再创建'me.lolico.App#0'Bean。因为beanFactoryPostProcessor定义在App这个配置类中即依赖于'me.lolico.App#0'Bean，那么Spring容器创建'beanFactoryPostProcessor'Bean导致提前实例化'me.lolico.App#0'Bean，而此时Spring容器处于还未实例化所有Bean（只是加载了BeanDefinition）的生命周期，那么在创建'me.lolico.App#0'Bean的时候自然不会去注入Environment属性，只是先实例化出来并未注入属性，参考Spring实例化Bean的三级缓存机制（提前暴露Bean以解决循环依赖的问题)。所以env为null，而后续实例化'dataSource'Bean，使用为null的env去get属性值当然就空指针异常了。

事实上，通过断点调试也验证了这个结论:
![断点调试](https://lolico.griouges.cn/images/38Ri.png)

而且仔细观察日志，Spring也告诉了我们这一点:
> 2019-12-17 14:20:50,234 [main] [INFO] - @Bean method App.beanFactoryPostProcessor is non-static and returns an object assignable to Spring's BeanFactoryPostProcessor interface. This will result in a failure to process annotations such as @Autowired, @Resource and @PostConstruct within the method's declaring @Configuration class. Add the 'static' modifier to this method to avoid these container lifecycle issues; see @Bean javadoc for complete details.

@Bean方法App.beanFactoryPostProcessor是非静态的，并返回可分配给Spring的BeanFactoryPostProcessor接口的对象。这将导致无法在方法的声明@Configuration类中处理诸如@ Autowired，@ Resource和@PostConstruct之类的注释。在此方法中添加“静态”修饰符可避免这些容器生命周期问题；有关完整的详细信息，请参见@Bean javadoc。

## 解决
第一想到的是：既然创建'me.lolico.App#0'Bean的时候不会注入env属性，那么改为构造器强制注入,应该就可以了吧：
```java
    public App(Environment env) {
        this.env = env;
    }
```
事实上是不行的：

`java.lang.NoSuchMethodException: me.lolico.App$$EnhancerBySpringCGLIB$$e5ab47a1.<init>()`

spring在使用cglib动态代理生成App实例的时候调用默认的无参构造器。但cglib现在也并不要求代理类一定要有默认无参构造器了，可能是Spring底层实现的原因？

**所以真正的解决办法**

### App配置类实现EnvironmentAware接口
Spring会通过回调接口传入一个Environment对象
```java
public class App implements EnvironmentAware {
    private Environment env;

    ......

    @Override
    public void setEnvironment(Environment environment) {
        this.env = environment;
    }
}
```
### 将BeanFactoryPostProcessor定义的方法改为静态方法
Spring可以通过类名.方法名调用获取BeanFactoryPostProcessor的静态方法。

### 将BeanFactoryPostProcessor定义在其他配置类中或者定义在xml配置文件中
原因不解释~



## 小结
通过分析这个问题，还能得出一些信息：将BeanFactoryPostProcessor定义在配置类中并且并不以静态方法定义的话，会导致配置类提前实例化（BeanFactoryPostProcessor在Spring生命周期中创建的很早，要早于BeanPostProcessor和Bean参考AbstractApplicationContext#refresh)，从而出现一些生命周期的问题，如果不了解，真的很难发现。可能你这样定义没有问题，那原因之一可能是因为你的这个配置类中没有用到类似Environment这种需要注入进来的属性。



---------- 
*参考：*

^ [*BeanFactoryPostProcessor and @Bean problems #4711*](https://github.com/spring-projects/spring-boot/issues/4711)
