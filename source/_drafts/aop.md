---
title: aop详解
tags: [Spring,AOP,AspectJ]
---

## 什么是AOP？

AOP（`Aspect-oriented Programming`），称为面向切面编程，作为面向对象的一种补充，用于处理系统中分布于各个模块的横切关注点，比如权限控制、事务管理、日志、缓存等等。

AOP代理主要分为`静态代理`和`动态代理`，静态代理的代表为AspectJ；而动态代理则以Spring AOP为代表。静态代理一般在编译期或者加载期实现，动态代理是运行期实现。

### 静态代理

AspectJ的静态代理，不是基于代理的。它使用修改字节码的技术将通知织入到目标类，所以，在运行时的类已经不是原本的那个类，是经过修改的了。通常来说，你写的AOP相关的代码，和对象的结合过程叫做织入(weave)。AspectJ的织入过程，有可能发生在三个阶段：

- 编译时织入(Compile-time weaving) 使用AspectJ的编译器ajc，在项目编译的阶段就将代码织入目标类
- 编译后织入(Post-compile weaving) 使用AspectJ的编译器ajc，向javac编译出来的的class织入代码
- 加载时织入(Load-time weaving) 使用特定的类加载器在类加载时期修改类字节码后加载进JVM

从上面可以看出，在程序的运行阶段，我们使用的类都是已经织入了通知代码的（并不会创建出多余的对象），而不像Spring AOP，需要在运行的时候去生成动态代理或者生成类。所以说，AspectJ的运行效率相比动态代理生成的类要高一点。

### 动态代理

动态代理则不会修改字节码，而是在内存中生成一个代理对象，这个AOP对象包含了目标对象的全部方法，并且在特定的切点做了增强处理，并回调原对象的方法。所以动态代理是在方法层级上进行代理，代理细度上不如静态代理。

## 生成AOP代理的入口

Spring所有的代理`AopProxy`的创建最后都是`ProxyCreatorSupport#createAopProxy`这个方法，而这个方法如下：

```java
	protected final synchronized AopProxy createAopProxy() {
		if (!this.active) {
			activate();
		}
		return getAopProxyFactory().createAopProxy(this);
	}
```

显然它又是调用了`AopProxyFactory#createAopProxy`方法，它的唯一实现为`DefaultAopProxyFactory`。它做了一个简单的逻辑判断：若实现类接口，使用`JdkDynamicAopProxy`去创建，否则使用`ObjenesisCglibAopProxy`。

最终拿到`AopProxy`后，调用`AopProxy#getProxy()`就可以拿到代理对象，从而进行相应的工作了。

我们基本有一共共识就是：默认情况下，若我们实现了接口，就实用JDK动态代理，若没有就实用CGLIB。那么接下来，就来看看代理对象的创建、执行的具体过程是如何的。

## AopProxy

它是一个AOP代理的抽象接口。提供了两个方法，让我们可以获取对应 配置的AOP对象的代理：

```java
public interface AopProxy {
	Object getProxy();
	Object getProxy(@Nullable ClassLoader classLoader);
}
```

它的实现类只有三个：

- `JdkDynamicAopProxy` 基于jdk动态代理生成代理对象
- `CglibAopProxy` 基于cglib生成一个代理子类
- `ObjenesisCglibAopProxy` 基于Objenesis扩展`CglibAopProxy`，提供不调用类的构造函数来创建代理对象。

### JdkDynamicAopProxy

```java
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {

	private static final long serialVersionUID = 5531744639992436476L;
	private static final Log logger = LogFactory.getLog(JdkDynamicAopProxy.class);

	// 这里就保存这个AOP代理所有的配置信息，包括所有的增强器等等
	private final AdvisedSupport advised;
    // 用于记录代理接口中是否定义了equals和hashCode方法
	private boolean equalsDefined;
	private boolean hashCodeDefined;

	public JdkDynamicAopProxy(AdvisedSupport config) throws AopConfigException {
		Assert.notNull(config, "AdvisedSupport must not be null");
        // 至少需要一个增强器和目标实例才行
		if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
			throw new AopConfigException("No advisors and no TargetSource specified");
		}
		this.advised = config;
	}

	@Override
	public Object getProxy() {
		return getProxy(ClassUtils.getDefaultClassLoader());
	}

    // 创建代理对象
	@Override
	public Object getProxy(@Nullable ClassLoader classLoader) {
		if (logger.isTraceEnabled()) {
			logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
		}
        // 这部很重要，就是去找接口 我们看到最终代理的接口就是这里返回的所有接口们（除了我们自己的接口，还有Spring默认的一些接口）  大致过程如下：
		//1、获取目标对象自己实现的接口(最终肯定都会被代理的)
		//2、是否添加`SpringProxy`这个接口：目标对象实现对就不添加了，没实现过就添加
		//3、是否新增`Adviced`接口，注意不是Advice通知接口。 实现过就不实现了，没实现过并且advised.isOpaque()为false就添加（默认会添加的）
		//4、是否新增DecoratingProxy接口。传入的第二个参数decoratingProxy为true，并且没实现过就添加（显然这里，首次进来是会添加的）
		//5、代理类的接口一共是目标对象的接口+上面三个接口SpringProxy、Advised、DecoratingProxy（SpringProxy是个标记接口而已，其余的接口都有对应的方法的）
		//DecoratingProxy 这个接口Spring4.3后才提供
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
        // 创建jdk动态代理对象，InvocationHandler传的this
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}

	// 找找看接口里有没有自己定义equals方法和hashCode方法，然后标记一下
	// 注意此处用的是getDeclaredMethods，只会找当前类中的
	private void findDefinedEqualsAndHashCodeMethods(Class<?>[] proxiedInterfaces) {
		for (Class<?> proxiedInterface : proxiedInterfaces) {
			Method[] methods = proxiedInterface.getDeclaredMethods();
			for (Method method : methods) {
				if (AopUtils.isEqualsMethod(method)) {
					this.equalsDefined = true;
				}
				if (AopUtils.isHashCodeMethod(method)) {
					this.hashCodeDefined = true;
				}
				if (this.equalsDefined && this.hashCodeDefined) {
					return;
				}
			}
		}
	}

	@Override
	@Nullable
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		Object oldProxy = null;
		boolean setProxyContext = false;

		TargetSource targetSource = this.advised.targetSource;
		Object target = null;

		try {
			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
				// The target does not implement the equals(Object) method itself.
				return equals(args[0]);
			}
			else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
				// The target does not implement the hashCode() method itself.
				return hashCode();
			}
			else if (method.getDeclaringClass() == DecoratingProxy.class) {
				// There is only getDecoratedClass() declared -> dispatch to proxy config.
				return AopProxyUtils.ultimateTargetClass(this.advised);
			}
			else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
				// Service invocations on ProxyConfig with the proxy config...
				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
			}

			Object retVal;

			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}

			// Get as late as possible to minimize the time we "own" the target,
			// in case it comes from a pool.
			target = targetSource.getTarget();
			Class<?> targetClass = (target != null ? target.getClass() : null);

			// Get the interception chain for this method.
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

			// Check whether we have any advice. If we don't, we can fallback on direct
			// reflective invocation of the target, and avoid creating a MethodInvocation.
			if (chain.isEmpty()) {
				// We can skip creating a MethodInvocation: just invoke the target directly
				// Note that the final invoker must be an InvokerInterceptor so we know it does
				// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
			}
			else {
				// We need to create a method invocation...
				MethodInvocation invocation =
						new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				// Proceed to the joinpoint through the interceptor chain.
				retVal = invocation.proceed();
			}

			// Massage return value if necessary.
			Class<?> returnType = method.getReturnType();
			if (retVal != null && retVal == target &&
					returnType != Object.class && returnType.isInstance(proxy) &&
					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
				// Special case: it returned "this" and the return type of the method
				// is type-compatible. Note that we can't help if the target sets
				// a reference to itself in another returned object.
				retVal = proxy;
			}
			else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
				throw new AopInvocationException(
						"Null return value from advice does not match primitive return type for: " + method);
			}
			return retVal;
		}
		finally {
			if (target != null && !targetSource.isStatic()) {
				// Must have come from TargetSource.
				targetSource.releaseTarget(target);
			}
			if (setProxyContext) {
				// Restore old proxy.
				AopContext.setCurrentProxy(oldProxy);
			}
		}
	}

	@Override
	public boolean equals(@Nullable Object other) {
		if (other == this) {
			return true;
		}
		if (other == null) {
			return false;
		}

		JdkDynamicAopProxy otherProxy;
		if (other instanceof JdkDynamicAopProxy) {
			otherProxy = (JdkDynamicAopProxy) other;
		}
		else if (Proxy.isProxyClass(other.getClass())) {
			InvocationHandler ih = Proxy.getInvocationHandler(other);
			if (!(ih instanceof JdkDynamicAopProxy)) {
				return false;
			}
			otherProxy = (JdkDynamicAopProxy) ih;
		}
		else {
			// Not a valid comparison...
			return false;
		}

		// If we get here, otherProxy is the other AopProxy.
		return AopProxyUtils.equalsInProxy(this.advised, otherProxy.advised);
	}

	@Override
	public int hashCode() {
		return JdkDynamicAopProxy.class.hashCode() * 13 + this.advised.getTargetSource().hashCode();
	}

}
```

### CglibAopProxy