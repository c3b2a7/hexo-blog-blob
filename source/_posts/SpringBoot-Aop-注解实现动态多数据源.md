---
title: SpringBoot-Aop+注解实现动态多数据源
categories: 正常的文章
date: 2020-01-17 18:43:19
tags: [Java,Spring,SpringBoot]
---

## 前言
Spring提供了`AbstractRoutingDataSource`类以方便开发者实现多数据源，看一下`AbstractRoutingDataSource#getConnection()`的源码：
```java
	@Override
	public Connection getConnection() throws SQLException {
		return determineTargetDataSource().getConnection();
	}
```
可以看到在`getConnection()`方法中是通过调用`determineTargetDataSource().getConnection();`获取一个连接，继续追踪到`determineTargetDataSource()`方法：
```java
	protected DataSource determineTargetDataSource() {
		Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
        //获取key,模板方法模式
		Object lookupKey = determineCurrentLookupKey();
        //根据key获取DataSource
		DataSource dataSource = this.resolvedDataSources.get(lookupKey);
        //如果dataSource为空并且启用获取默认dataSource或lookupKey为空时取默认的DataSource
        //（lenientFallback默认为true）
		if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
			dataSource = this.resolvedDefaultDataSource;
		}
		if (dataSource == null) {//如果dataSource为空抛出异常
			throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
		}
		return dataSource;
	}
```
可以看到`determineTargetDataSource()`方法先调用`determineCurrentLookupKey()`获取lookupKey，再通过`this.resolvedDataSources.get(lookupKey);`取出DataSource，我们看一下`determineCurrentLookupKey()`方法，发现这个方法是个abstract方法，也就是说子类必须要实现，再看下`this.resolvedDataSources`是个什么东西：
```java
	private Map<Object, DataSource> resolvedDataSources;
```
所以说`AbstractRoutingDataSource`中维护了一个Map，在`getConnection`的时候先获取key再从resolvedDataSources中get一个DataSource返回（如果为空时判断是否启用获取默认dataSource再决定是否用默认的dataSource）而获取key的方法`determineCurrentLookupKey()`由子类实现（模板方法模式）。  
**所以实现动态多数据源的思路就十分明确了**
1. 定义一个`DynamicDataSourceContextHolder`用于保存当前线程需要使用的dataSource对应的key
2. 重写`determineCurrentLookupKey()`方法将这个key返回  

实际上完成上述两个步骤其实就可以实现多数据源了，只要在getConnection前调用`DynamicDataSourceContextHolder#setKey`方法设置需要使用的dataSource对应的key就可以了。
但通常来说，我们并不希望设置使用那个数据库的代码侵入到我们的业务代码中，所以我们可以利用aop实现：定义一个注解`@DataSource`和注解切面`DataSourceAspect`，然后就可以在需要切换数据库的方法上使用注解进行设置。

## 代码

### DataSourceConfiguration
```java
/**
 * @author lolicom
 */
@Slf4j
@Configuration
public class DataSourceConfiguration implements ApplicationContextAware {
    
    private ApplicationContext applicationContext;
    
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
    
    /* configuration */
    
    @Bean
    @Qualifier("defaultDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.first")
    public DataSource dataSource1() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.second")
    public DataSource dataSource2() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @Primary
    public DataSource dataSource(Map<String, DataSource> dataSourceMap, @Qualifier("defaultDataSource") DataSource defaultDataSource) {
        AbstractRoutingDataSource dataSource = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                return DynamicDataSourceContextHolder.getKey();
            }
            
            @Override
            public void setTargetDataSources(Map<Object, Object> targetDataSources) {
                super.setTargetDataSources(targetDataSources);
                DynamicDataSourceContextHolder.setDataSourceMap(targetDataSources);
            }
            
            @Override
            public void setDefaultTargetDataSource(Object defaultTargetDataSource) {
                if (defaultTargetDataSource instanceof String) {
                    super.setDefaultTargetDataSource(dataSourceMap.get(defaultTargetDataSource));
                    DynamicDataSourceContextHolder.setDefaultKey((String) defaultTargetDataSource);
                } else if (defaultTargetDataSource instanceof DataSource) {
                    super.setDefaultTargetDataSource(defaultTargetDataSource);
                    DynamicDataSourceContextHolder.setDefaultKey(resolveSpecifiedLookupKey((DataSource) defaultTargetDataSource));
                } else {
                    log.info("Why am i here?");
                }
            }
            
            private String resolveSpecifiedLookupKey(DataSource defaultTargetDataSource) {
                String[] beanDefinitionNames = applicationContext.getBeanNamesForType(defaultTargetDataSource.getClass());
                for (String beanDefinitionName : beanDefinitionNames) {
                    if (applicationContext.getBean(beanDefinitionName) == defaultTargetDataSource) {
                        return beanDefinitionName;
                    }
                }
                return null;
            }
        };
        dataSource.setTargetDataSources(Collections.unmodifiableMap(dataSourceMap));
        dataSource.setDefaultTargetDataSource(defaultDataSource);
        return dataSource;
    }
    
}
```
### DynamicDataSourceContextHolder
```java
@Slf4j
public class DynamicDataSourceContextHolder {
    private final static ThreadLocal<String> KEY = new ThreadLocal<>();
    private static Map<Object, Object> targetDataSourceMap;
    private static String defaultKey;
    
    public static void setDataSourceMap(Map<Object, Object> targetDataSourceMap) {
        DynamicDataSourceContextHolder.targetDataSourceMap = targetDataSourceMap;
    }
    
    public static String getKey() {
        return Optional.ofNullable(DynamicDataSourceContextHolder.KEY.get())
                .orElseGet(() -> DynamicDataSourceContextHolder.defaultKey);
    }
    
    public static void setKey(String key) {
        DynamicDataSourceContextHolder.KEY.set(targetDataSourceMap.containsKey(key) ? key : DynamicDataSourceContextHolder.defaultKey);
    }
    
    public static void remove() {
        DynamicDataSourceContextHolder.KEY.remove();
    }
    
    public static void setDefaultKey(String defaultKey) {
        DynamicDataSourceContextHolder.defaultKey = defaultKey;
        log.debug("设置defaultKey:[{}]", defaultKey);
    }
}
```
### DataSource注解
```java
/**
 * @author lolicom
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface DataSource {
    String value() default "";
}
```
### DataSourceAspect切面
```java
@Slf4j
@Component
@Aspect
@Order(Ordered.LOWEST_PRECEDENCE - 1)
public class DataSourceAspect {
    
    @Pointcut(value = "@annotation(dataSource)", argNames = "dataSource")
    public void pointcut(DataSource dataSource) {
    }
    
    @Before(value = "pointcut(dataSource)", argNames = "dataSource")
    public void before(DataSource dataSource) {
        String value = dataSource.value();
        DynamicDataSourceContextHolder.setKey(value);
        log.debug("使用数据库{}", value);
    }
    
    @After("@annotation(cn.griouges.learnspringboot.common.annotation.DataSource)")
    public void after() {
        DynamicDataSourceContextHolder.remove();
    }
}
```
测试发现如果切面优先级为`Ordered.LOWEST_PRECEDENCE`时，每次都会在getConnection之后再拦截进行设置key，不符合我们的需求，添加`@Order(Ordered.LOWEST_PRECEDENCE - 1)`设置切面优先级，在获取数据库连接前进行设置数据库key。  
### 配置完毕
>后续还有数据源，只要注册bean到容器中就可以自动添加到`AbstractRoutingDataSource`的`targetDataSources`中，key为bean的name。

-----

## 测试
application.properties中添加配置
```properties
spring.datasource.first.jdbc-url=jdbc:mysql://localhost:3306/mysite?useUnicode=true&characterEncoding=utf-8
spring.datasource.first.username=root
spring.datasource.first.password=leisure.
spring.datasource.first.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.second.jdbc-url=jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=utf-8
spring.datasource.second.username=root
spring.datasource.second.password=leisure.
spring.datasource.second.driver-class-name=com.mysql.cj.jdbc.Driver
```
service层的方法上添加注解
```java
    @DataSource("dataSource2")
    public User findUserForLogin(String username, String password) {
        return userRepository.findByUsernameAndPassword(username, password);
    }
```
测试发现在Dao层的接口进行注解时，拦截会在获取连接后执行，导致失效。具体原因没去深追，后续有时间再断点调试查看是什么原因，orm使用的是Sprng Data Jpa，使用mybatis貌似不会出现这种情况。  
controler添加测试方法
```java
    @PostMapping("/test")
    public AjaxResponseVO test(String username,String password) {
        User userForLogin = service.findUserForLogin(username, password);
        if (userForLogin != null) {
            return AjaxResponseVO.success("登录成功");
        }
        return AjaxResponseVO.fail("用户名或密码错误");
    }
```
post测试：
<<<<<<< HEAD
![](http://lolico.test.upcdn.net/images/NVWy.png)
测试通过，打印日志:
![](http://lolico.test.upcdn.net/images/N7a2.png)
=======
![](https://i.loli.net/2020/03/09/ophckmj4yKtbXSH.png)
测试通过，打印日志:
![](https://i.loli.net/2020/03/09/KfLyP9hvY6M2nNk.png)
>>>>>>> 075af6abe4384f66a943568fca56258b1f49111d
