---
title: 源码角度分析Shiro认证流程以及一个前后端分离案例
categories: 正常的文章
date: 2020-02-04 18:50:51
tags: [Shiro,Web,Security]
---

从源码去分析认证流程前，你需要知道Shiro是什么，以及Shiro中的基本组件。在看本篇文章前，我假设你已经知道上述东西，并且后续的分析不会对这些组件是什么进行讲解。

如果你并不了解Shiro可以看下我的这篇博文：{% post_link Shiro简介 %} 对于更详细的分析，大家可以百度、google一下，资料应该是很全的。

## Shiro工作流程简述
Shiro进行认证的本质还是通过过滤器进行拦截，过滤器拦截后判断是否需要进行认证，如果需要，取出"token"并交给`SecurityManager`进行认证，认证通过后放行，如果不需要认证则直接放行。

至于为什么token打上引号，是因为token并不一定是普遍意义上的JWT（json web token）,也可以是基于BASIC HTTP的token，还可以是表单中的用户名和密码...

所以Shiro中就内置了一些常用的Filter，有基于表单认证的`FormAuthenticationFilter`类和基于BASIC HTTP的`BasicHttpAuthenticationFilter`类，还有不需要认证直接放行的`AnonymousFilter`类，也有一些用于检查`Roles`和`Permissions`的过滤器，具体的可以去`DefaultFilter`中看看。

## 过滤器
Shiro中的过滤器其实还是Servlet中的Filter，只不过对其进行了封装。

先来看下Shiro中Filter的类图
![Shiro Filter](https://lolico.griouges.cn/images/NKaB.png)

- `AbstractFilter`：提供了简化的初始化逻辑和access到初始化参数，并且提供了`getInitParam(String paramName)`方法获取。
- `NameableFilter`：为Filter提供`name`属性值，可通过getter和setter方法获取和设置。
- `OncePerRequestFilter`：继承自`NameableFilter`，通过名字看出，一次请求执行一次。

不难看出Shiro中的所有Filter都是通过继承`OncePerRequestFilter`抽象类而来，来看下这个类的`doFilter(...)`方法：

```java
    public final void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        String alreadyFilteredAttributeName = getAlreadyFilteredAttributeName();
        if ( request.getAttribute(alreadyFilteredAttributeName) != null ) {
            filterChain.doFilter(request, response);
        } else if (!isEnabled(request, response) || shouldNotFilter(request) ) {
            filterChain.doFilter(request, response);
        } else {
            request.setAttribute(alreadyFilteredAttributeName, Boolean.TRUE);
            try {
                doFilterInternal(request, response, filterChain);
            } finally {
                request.removeAttribute(alreadyFilteredAttributeName);
            }
        }
    }
    // 默认实现是使用过滤器的name作为属性名，如果为空则用类的全路径名作为属性名
    // 这个属性名一般来说是唯一的
    protected String getAlreadyFilteredAttributeName() {
        String name = getName();
        if (name == null) {
            name = getClass().getName();
        }
        return name + ALREADY_FILTERED_SUFFIX; // ALREADY_FILTERED_SUFFIX = ".FILTERED";
    }
```

可以看出首先调用`getAlreadyFilteredAttributeName()`获取当前过滤器的属性名，该方法的默认实现是使用过滤器的name作为属性名，如果为空则用类的全路径名作为属性名，这个属性名一般来说是唯一的。获取到属性名后看下request中有没有这个属性，如果存在，则放行，继续执行下一个过滤器。如果不存在，则继续判断过滤器是否启用，或者是否直接放行，如果是则也放行，最后上述两种情况都不符合，也就是说需要进行过滤,先将过滤器的属性名设置到这次请求中，再尝试进行具体逻辑`doFilterInternal(...)`方法（该方法是抽象方法，需要子类实现），最后finally中逻辑显而易见，过滤器具体逻辑执行完毕后移除过滤器的属性名。

说的简单点，也就是实现了一次请求过滤一次，然后将过滤器具体的逻辑放在了这个抽象方法中：
```java
    protected abstract void doFilterInternal(ServletRequest request, ServletResponse response, FilterChain chain) throws ServletException, IOException;
```

### `AdviceFilter`类、`PathMatchingFilter`类和`AccessControlFilter`类

**AdviceFilte类**

`AdviceFilter`类继承自`OncePerRequestFilter`，也是个抽象类，但他实现了父类的`doFilterInternal(...)`方法中，将具体逻辑又分为三个步骤：

```java
    public void doFilterInternal(ServletRequest request, ServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        Exception exception = null;
        try {
            //执行前置处理
            boolean continueChain = preHandle(request, response);
            if (log.isTraceEnabled()) {
                log.trace("Invoked preHandle method.  Continuing chain?: [" + continueChain + "]");
            }
            //前置处理如果没问题，执行executeChain(...)方法
            if (continueChain) {
                executeChain(request, response, chain);
            }
            //执行后置处理
            postHandle(request, response);
            if (log.isTraceEnabled()) {
                log.trace("Successfully invoked postHandle method");
            }
        } catch (Exception e) {
            exception = e;
        } finally {
            cleanup(request, response, exception);
        }
    }
```

- `preHandle(...)`方法默认返回`true`，其实所有子类过滤器的逻辑应该放在这里面。
- `executeChain(ServletRequest request, ServletResponse response, FilterChain chain)`方法一定要注意，因为这个方法的逻辑是`chain.doFilter(request, response)`，执行下一个过滤器
- `postHandle(...)`方法默认是一个空方法。

所有过滤器真正的逻辑应该在`preHandle(...)`中而不是`executeChain(...)`中，自定义过滤器重写方法的时候一定要注意。

看到这你可能会疑惑：真正的前置处理在哪里？其实并没有提供真正意义上的前置处理，为什么这么说呢？看下`PathMatchingFilter`和`AccessControlFilter`就知道了。

**PathMatchingFilter类**
继承自`AdviceFilter`的`PathMatchingFilter`中重写的`preHandler`方法会检查请求路径是否匹配配置，匹配的话则执行`isFilterChainContinued()`方法，而这个方法中调用了一个抽象方法`onPreHandle(...)`，源码如下，就不解释了。
```java
    protected boolean preHandle(ServletRequest request, ServletResponse response) throws Exception {

        if (this.appliedPaths == null || this.appliedPaths.isEmpty()) {
            if (log.isTraceEnabled()) {
                log.trace("appliedPaths property is null or empty.  This Filter will passthrough immediately.");
            }
            return true;
        }

        for (String path : this.appliedPaths.keySet()) {
            // If the path does match, then pass on to the subclass implementation for specific checks
            //(first match 'wins'):
            if (pathsMatch(path, request)) {
                log.trace("Current requestURI matches pattern '{}'.  Determining filter chain execution...", path);
                Object config = this.appliedPaths.get(path);
                return isFilterChainContinued(request, response, path, config);
            }
        }

        //no path matched, allow the request to go through:
        return true;
    }

    /**
     * Simple method to abstract out logic from the preHandle implementation - it was getting a bit unruly.
     *
     * @since 1.2
     */
    @SuppressWarnings({"JavaDoc"})
    private boolean isFilterChainContinued(ServletRequest request, ServletResponse response,
                                           String path, Object pathConfig) throws Exception {

        if (isEnabled(request, response, path, pathConfig)) { //isEnabled check added in 1.2
            if (log.isTraceEnabled()) {
                log.trace("Filter '{}' is enabled for the current request under path '{}' with config [{}].  " +
                        "Delegating to subclass implementation for 'onPreHandle' check.",
                        new Object[]{getName(), path, pathConfig});
            }
            //The filter is enabled for this specific request, so delegate to subclass implementations
            //so they can decide if the request should continue through the chain or not:
            return onPreHandle(request, response, pathConfig);
        }

        if (log.isTraceEnabled()) {
            log.trace("Filter '{}' is disabled for the current request under path '{}' with config [{}].  " +
                    "The next element in the FilterChain will be called immediately.",
                    new Object[]{getName(), path, pathConfig});
        }
        //This filter is disabled for this specific request,
        //return 'true' immediately to indicate that the filter will not process the request
        //and let the request/response to continue through the filter chain:
        return true;
    }
```

**AccessControlFilter类**
`AccessControlFilter`抽象类继承自`PathMatchingFilter`  
`AccessControlFilter`实现了`onPreHandle(...)`方法：
```java
    public boolean onPreHandle(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
        return isAccessAllowed(request, response, mappedValue) || onAccessDenied(request, response, mappedValue);
    }
    protected abstract boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception;

    protected boolean onAccessDenied(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
        return onAccessDenied(request, response);
    }

    protected abstract boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception;
```
看到这里也就明朗了，返回`isAccessAllowed(...) || onAccessDenied(...)`，这两个方法都是抽象方法，这个逻辑的意思解释起来就是：先执行`isAccessAllowed`方法看下能不能访问能访问则直接返回也就是true，如果不能访问，则执行`onAccessDenied`方法并返回结果（注意理解‘或’运算符）。

### `AuthenticationFilter`类和`AuthenticatingFilter`类

注意这两个类名的不同。  
`AuthenticationFilter`只实现了`isAccessAllowed(...)`方法：
```java
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        // 返回绑定到当前线程上的主体是否已经认证的结果
        Subject subject = getSubject(request, response);
        return subject.isAuthenticated();
    }
```
而`AuthenticatingFilter`继承自`AuthenticationFilter`类：
```java
    protected boolean executeLogin(ServletRequest request, ServletResponse response) throws Exception {
        AuthenticationToken token = createToken(request, response);
        if (token == null) {
            String msg = "createToken method implementation returned null. A valid non-null AuthenticationToken " +
                    "must be created in order to execute a login attempt.";
            throw new IllegalStateException(msg);
        }
        try {
            Subject subject = getSubject(request, response);
            subject.login(token);
            return onLoginSuccess(token, subject, request, response);
        } catch (AuthenticationException e) {
            return onLoginFailure(token, e, request, response);
        }
    }

    protected abstract AuthenticationToken createToken(ServletRequest request, ServletResponse response) throws Exception;

    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        return super.isAccessAllowed(request, response, mappedValue) ||
                (!isLoginRequest(request, response) && isPermissive(mappedValue));
    }
```

解释一下这个`isAccessAllowed(...)`方法：
> The default implementation returns true if the user is authenticated. Will also return true if the isLoginRequest returns false and the "permissive" flag is set  
如果已经认证则返回true，或者请求不需要认证并且设置了"permissive"，设置"permissive"是什么意思呢：比如说配置过滤器的路由策略时：map.add("/**","authc[permissive]")

所以这个类其实已经提供了一个通用的逻辑，一般来说，也是能够适应大多数场景下的需求的，所以说，一般我们自定义过滤器就是继承`AuthenticatingFilter`类，然后只需要重写`onAccessDenied(...)`方法，在其中调用`executeLogin`方法进行认证，然后提供创建Token的具体逻辑也就是`createToken(..)`方法就可以了。

如果有特殊需求，比如对于Option请求直接放行，那么可以重写`isAccessAllowed(...)`方法，判断是否是option请求，一般来说我们还是会调用一下父类的通用判断方法：`super.isAccessAllowed(...)`，后续的案例就用了这种方法实现跨域请求直接放行。

并且可以看到`executeLogin(..)`方法中获取`Subject`并使用`createToken`方法返回的Token去调用`login`进行认证，对于认证成功和认证失败还提供了回调处理，我们可以重写这两个方法以满足某些场景下的需求，比如jwt认证成功后判断是否需要刷新token。

## SecurityManager
当我们使用Subject去调用`login(AuthenticationToken)`方法时，实际上是委托给`DelegatingSubject`去处理,而这个类又会从`SecurityManager`中获取信息再进行处理:  

**`DelegatingSubject#login`方法**

```java
    public void login(AuthenticationToken token) throws AuthenticationException {
        ...
        Subject subject = securityManager.login(this, token);
        ...
    }
```

**`SecurityManager#login`方法**
```java
    public Subject login(Subject subject, AuthenticationToken token) throws AuthenticationException {
        AuthenticationInfo info;
        try {
            //调用authenticate方法
            info = authenticate(token);
        } catch (AuthenticationException ae) {
            try {
                onFailedLogin(token, ae, subject);
            } catch (Exception e) {
                if (log.isInfoEnabled()) {
                    log.info("onFailedLogin method threw an " +
                            "exception.  Logging and propagating original AuthenticationException.", e);
                }
            }
            throw ae; //propagate
        }
        Subject loggedIn = createSubject(token, info, subject);
        onSuccessfulLogin(token, info, loggedIn);
        return loggedIn;
    }
```
`AbstractSecurityManager`中有一个`Authenticator`认证器，在`SecurityManager`进行认证的时候就是委托它进行认证的：
```java
    public AuthenticationInfo authenticate(AuthenticationToken token) throws AuthenticationException {
        return this.authenticator.authenticate(token);
    }
```
`Authenticator`是一个接口，并且只有一个认证方法，该方法返回认证信息。
```java
public interface Authenticator {
    public AuthenticationInfo authenticate(AuthenticationToken authenticationToken)
            throws AuthenticationException;
}
```
**`AbstractAuthenticator#authenticate`**方法：
```java
    public final AuthenticationInfo authenticate(AuthenticationToken token) throws AuthenticationException {

        if (token == null) {
            throw new IllegalArgumentException("Method argument (authentication token) cannot be null.");
        }

        log.trace("Authentication attempt received for token [{}]", token);

        AuthenticationInfo info;
        try {
            //调用`doAuthenticate(AuthenticationToken)`抽象方法
            info = doAuthenticate(token);
            if (info == null) {
                String msg = "No account information found for authentication token [" + token + "] by this " +
                        "Authenticator instance.  Please check that it is configured correctly.";
                throw new AuthenticationException(msg);
            }
        } catch (Throwable t) {
            AuthenticationException ae = null;
            if (t instanceof AuthenticationException) {
                ae = (AuthenticationException) t;
            }
            if (ae == null) {
                //Exception thrown was not an expected AuthenticationException.  Therefore it is probably a little more
                //severe or unexpected.  So, wrap in an AuthenticationException, log to warn, and propagate:
                String msg = "Authentication failed for token submission [" + token + "].  Possible unexpected " +
                        "error? (Typical or expected login exceptions should extend from AuthenticationException).";
                ae = new AuthenticationException(msg, t);
                if (log.isWarnEnabled())
                    log.warn(msg, t);
            }
            try {
                notifyFailure(token, ae);
            } catch (Throwable t2) {
                if (log.isWarnEnabled()) {
                    String msg = "Unable to send notification for failed authentication attempt - listener error?.  " +
                            "Please check your AuthenticationListener implementation(s).  Logging sending exception " +
                            "and propagating original AuthenticationException instead...";
                    log.warn(msg, t2);
                }
            }


            throw ae;
        }

        log.debug("Authentication successful for token [{}].  Returned account [{}]", token, info);

        notifySuccess(token, info);

        return info;
    }
```
该方法调用`doAuthenticate(AuthenticationToken)`抽象方法，返回认证信息。  
Shiro中的认证器只有一个实现类：`ModularRealmAuthenticator`

**`ModularRealmAuthenticator#doAuthenticate`方法**
```java
    protected AuthenticationInfo doAuthenticate(AuthenticationToken authenticationToken) throws AuthenticationException {
        assertRealmsConfigured();
        Collection<Realm> realms = getRealms();
        if (realms.size() == 1) {
            return doSingleRealmAuthentication(realms.iterator().next(), authenticationToken);
        } else {
            return doMultiRealmAuthentication(realms, authenticationToken);
        }
    }
```
如果只有一个`Realm`，执行`doSingleRealmAuthentication(..)`方法  
否则执行`doMultiRealmAuthentication(..)`方法

这两个方法有什么区别呢？

区别在于第二个方法多了一个认证策略`AuthenticationStrategy`，有三个实现：
- `AllSuccessfulStrategy`：需要所有的Realm认证通过
- `AtLeastOneSuccessfulStrategy`：至少需要一个Realm认证通过
- `FirstSuccessfulStrategy`：只需要一个Realm认证通过

最后两个认证策略有什么不同呢？前者在第一个`Realm`认证成功后还会继续认证剩下的`Realm`，并把所有的认证信息`AuthenticationInfo`进行合并。后者在第一个`Realm`认证成功后直接返回认证信息。默认使用的是`AtLeastOneSuccessfulStrategy`

来看下`doSingleRealmAuthentication(..)`和`doMultiRealmAuthentication(..)`这两个方法：

```java
    protected AuthenticationInfo doSingleRealmAuthentication(Realm realm, AuthenticationToken token) {
        if (!realm.supports(token)) {
            String msg = "Realm [" + realm + "] does not support authentication token [" +
                    token + "].  Please ensure that the appropriate Realm implementation is " +
                    "configured correctly or that the realm accepts AuthenticationTokens of this type.";
            throw new UnsupportedTokenException(msg);
        }
        AuthenticationInfo info = realm.getAuthenticationInfo(token);
        if (info == null) {
            String msg = "Realm [" + realm + "] was unable to find account data for the " +
                    "submitted AuthenticationToken [" + token + "].";
            throw new UnknownAccountException(msg);
        }
        return info;
    }

    protected AuthenticationInfo doMultiRealmAuthentication(Collection<Realm> realms, AuthenticationToken token) {

        AuthenticationStrategy strategy = getAuthenticationStrategy();

        AuthenticationInfo aggregate = strategy.beforeAllAttempts(realms, token);

        if (log.isTraceEnabled()) {
            log.trace("Iterating through {} realms for PAM authentication", realms.size());
        }

        for (Realm realm : realms) {

            aggregate = strategy.beforeAttempt(realm, token, aggregate);

            if (realm.supports(token)) {

                log.trace("Attempting to authenticate token [{}] using realm [{}]", token, realm);

                AuthenticationInfo info = null;
                Throwable t = null;
                try {
                    info = realm.getAuthenticationInfo(token);
                } catch (Throwable throwable) {
                    t = throwable;
                    if (log.isDebugEnabled()) {
                        String msg = "Realm [" + realm + "] threw an exception during a multi-realm authentication attempt:";
                        log.debug(msg, t);
                    }
                }

                aggregate = strategy.afterAttempt(realm, token, info, aggregate, t);

            } else {
                log.debug("Realm [{}] does not support token {}.  Skipping realm.", realm, token);
            }
        }

        aggregate = strategy.afterAllAttempts(token, aggregate);

        return aggregate;
    }
```
`doSingleRealmAuthentication(..)`方法不需要解释。  
`doMultiRealmAuthentication(..)`方法大体就是便利所有的`Realm`，如果这个`Realm`支持当前这个Token，则调用`getAuthenticationInfo(AuthenticationToken)`方法获取认证信息，然后根据使用的认证策略去处理认证信息。

而`getAuthenticationInfo(AuthenticationToken)`方法很简单：
```java
    public final AuthenticationInfo getAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {

        AuthenticationInfo info = getCachedAuthenticationInfo(token);
        if (info == null) {
            //otherwise not cached, perform the lookup:
            info = doGetAuthenticationInfo(token);
            log.debug("Looked up AuthenticationInfo [{}] from doGetAuthenticationInfo", info);
            if (token != null && info != null) {
                cacheAuthenticationInfoIfPossible(token, info);
            }
        } else {
            log.debug("Using cached authentication info [{}] to perform credentials matching.", info);
        }

        if (info != null) {
            assertCredentialsMatch(token, info);
        } else {
            log.debug("No AuthenticationInfo found for submitted AuthenticationToken [{}].  Returning null.", token);
        }

        return info;
    }
```
先从缓存中取，如果缓存里没有则调用`doGetAuthenticationInfo(AuthenticationToken)`方法获取（我们实现自己的Realm时一般就要实现这个方法，这个是获取认证信息的方法，还有一个获取权限信息的方法） 
最后，如果info不为`null`，调用`assertCredentialsMatch(..)`方法判断token和info是否匹配。默认使用的`CredentialsMatcher`是`SimpleCredentialsMatcher`，即调用`AuthenticationToken`和`AuthenticationInfo`的`getCredentials()`后进行简单的equal匹配。

## 一个前后端分离案例

**JWTToken**
```java
public class JWTToken implements HostAuthenticationToken {
    
    public static final JWTToken NONE = new JWTToken(null, null);
    
    private String token;
    private String host;
    
    public JWTToken(String token) {
        this(token, null);
    }
    
    public JWTToken(String token, String host) {
        this.token = token;
        this.host = host;
    }
    
    public String getToken() {
        return this.token;
    }
    
    public String getHost() {
        return host;
    }
    
    @Override
    public Object getPrincipal() {
        return token;
    }
    
    @Override
    public Object getCredentials() {
        return token;
    }
    
    @Override
    public String toString() {
        return "JWTToken{" +
                "token='" + token + '\'' +
                ", host='" + host + '\'' +
                '}';
    }
}
```

**JwtHttpAuthenticationFilter**
```java
package me.lolico.blog.web.filter;

import me.lolico.blog.web.JWTToken;
import org.apache.shiro.authc.AuthenticationException;
import org.apache.shiro.authc.AuthenticationToken;
import org.apache.shiro.subject.Subject;
import org.apache.shiro.web.filter.authc.AuthenticatingFilter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.HashSet;
import java.util.Locale;
import java.util.Set;

/**
 * @author lolico
 */
public class JwtHttpAuthenticationFilter extends AuthenticatingFilter {
    
    /**
     * This class's private logger.
     */
    private static final Logger log = LoggerFactory.getLogger(JwtHttpAuthenticationFilter.class);
    
    /**
     * HTTP Authorization header, equal to <code>Authorization</code>
     */
    private static final String AUTHORIZATION_HEADER = "Authorization";
    /**
     * HTTP Authentication header, equal to <code>WWW-Authenticate</code>
     */
    private static final String AUTHENTICATE_HEADER = "WWW-Authenticate";
    
    /**
     * The authzScheme default value to look for in the <code>Authorization</code> header
     */
    private static final String DEFAULT_AUTHORIZATION_SCHEME = "Bearer";
    
    /**
     * The name that is displayed during the challenge process of authentication, defauls to <code>application</code>
     * and can be overridden by the {@link #setApplicationName(String) setApplicationName} method.
     */
    private String applicationName = "application";
    
    /**
     * The authzScheme value to look for in the <code>Authorization</code> header, defaults to <code>Bearer</code>
     * Can override by {@link #setAuthzScheme(String)}
     */
    private String authzScheme = DEFAULT_AUTHORIZATION_SCHEME;
    
    /**
     * <code>true</code> will enable "OPTION" request method, <code>false</code> otherwise
     */
    private boolean isCorsEnable = true;
    
    /**
     * the callback handler for successful authentication
     */
    private SuccessfulHandler successfulHandler;
    
    /**
     * the callback handler for unsuccessful authentication
     */
    private UnsuccessfulHandler unsuccessfulHandler;
    
    
    public JwtHttpAuthenticationFilter() {
        unsuccessfulHandler = (token, e, request, response) -> {
            //defaults to set 401-unauthorized http status
            HttpServletResponse httpResponse = ((HttpServletResponse) response);
            String authcHeader = getAuthzScheme() + " realm=\"" + getApplicationName() + "\"";
            httpResponse.setHeader(AUTHENTICATE_HEADER, authcHeader);
            httpResponse.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        };
    }
    
    public JwtHttpAuthenticationFilter(String authzScheme, boolean isCorsEnable, SuccessfulHandler successfulHandler, UnsuccessfulHandler unsuccessfulHandler) {
        this.authzScheme = authzScheme;
        this.isCorsEnable = isCorsEnable;
        this.successfulHandler = successfulHandler;
        this.unsuccessfulHandler = unsuccessfulHandler;
    }
    
    /**
     * Returns the name that is displayed during the challenge process of authentication
     * Default value is <code>application</code>
     *
     * @return the name that is displayed during the challenge process of authentication
     */
    public String getApplicationName() {
        return applicationName;
    }
    
    /**
     * Sets the name that is displayed during the challenge process of authentication
     * Default value is <code>application</code>
     *
     * @param applicationName the name that is displayed during the challenge process of authentication
     */
    public void setApplicationName(String applicationName) {
        this.applicationName = applicationName;
    }
    
    /**
     * Returns the HTTP <b><code>Authorization</code></b> header value that this filter will respond to as indicating
     * a login request.
     * <p/>
     * Unless overridden by the {@link #setAuthzScheme(String)} method, the
     * default value is <code>Bearer</code>.
     *
     * @return the Http 'Authorization' header value that this filter will respond to as indicating a login request
     */
    public String getAuthzScheme() {
        return authzScheme;
    }
    
    /**
     * Sets the HTTP <b><code>Authorization</code></b> header value that this filter will respond to as indicating a
     * login request.
     * <p/>
     * Unless overridden by this method, the default value is <code>Bearer</code>
     *
     * @param authzScheme the HTTP <code>Authorization</code> header value that this filter will respond to as
     *                    indicating a login request.
     */
    public void setAuthzScheme(String authzScheme) {
        this.authzScheme = authzScheme;
    }
    
    /**
     * Default value is <code>true</code>
     *
     * @param corsEnable <code>true</code> will enable "OPTION" request method, <code>false</code> otherwise
     */
    public void setCorsEnable(boolean corsEnable) {
        isCorsEnable = corsEnable;
    }
    
    /**
     * Default value is <code>true</code>
     *
     * @return is cors enable
     */
    public boolean isCorsEnable() {
        return isCorsEnable;
    }
    
    /**
     * Returns the callback handler for successful authentication
     *
     * @return the callback handler for successful authentication
     */
    public SuccessfulHandler getSuccessfulHandler() {
        return successfulHandler;
    }
    
    /**
     * @param successfulHandler the callback handler for successful authentication
     */
    public void setSuccessfulHandler(SuccessfulHandler successfulHandler) {
        this.successfulHandler = successfulHandler;
    }
    
    /**
     * Returns the callback handler for unsuccessful authentication
     *
     * @return the callback handler for unsuccessful authentication
     */
    public UnsuccessfulHandler getUnsuccessfulHandler() {
        return unsuccessfulHandler;
    }
    
    /**
     * @param unsuccessfulHandler the callback handler for successful authentication
     */
    public void setUnsuccessfulHandler(UnsuccessfulHandler unsuccessfulHandler) {
        this.unsuccessfulHandler = unsuccessfulHandler;
    }
    
    /**
     * The Basic authentication filter can be configured with a list of HTTP methods to which it should apply. This
     * method ensures that authentication is <em>only</em> required for those HTTP methods specified. For example,
     * if you had the configuration:
     * <pre>
     *      [urls]
     *      /basic/** = authcJwt[POST,PUT,DELETE]
     * </pre>
     * then a GET request would not required authentication but a POST would.
     *
     * @param request     The current HTTP servlet request.
     * @param response    The current HTTP servlet response.
     * @param mappedValue The array of configured HTTP methods as strings. This is empty if no methods are configured.
     */
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        HttpServletRequest httpRequest = ((HttpServletRequest) request);
        String httpMethod = httpRequest.getMethod();
        
        // Check whether the current request's method requires authentication.
        // If no methods have been configured, then all of them require auth,
        // otherwise only the declared ones need authentication.
        
        Set<String> methods = httpMethodsFromOptions((String[]) mappedValue);
        boolean authcRequired = methods.size() == 0;
        
        // If enable cors and this request_method is equal to "OPTION", do not authentication.
        // Can override by configuration:
        // /** = authcJwt[POST,DELETE,OPTION]
        // then a OPTION request would required authentication.
        // if (isCorsEnable && httpMethod.equalsIgnoreCase("OPTION")) {
        //     authcRequired = false;
        // }
        for (String m : methods) {
            if (httpMethod.toUpperCase(Locale.ENGLISH).equals(m)) { // list of methods is in upper case
                authcRequired = true;
                break;
            }
        }
        // if (isCorsEnable && httpMethod.equalsIgnoreCase("OPTION")) {
        //     responseForCors(request, response);
        // }
        
        if (authcRequired) {
            return super.isAccessAllowed(request, response, mappedValue);
        } else {
            return true;
        }
    }
    
    /**
     * cors support
     */
    @Override
    protected boolean preHandle(ServletRequest request, ServletResponse response) throws Exception {
        if (isCorsEnable() && ((HttpServletRequest) request).getMethod().equals("OPTIONS")) {
            responseForCors(request, response);
            return false;
        }
        return super.preHandle(request, response);
    }
    
    private void responseForCors(ServletRequest request, ServletResponse response) {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        httpResponse.setHeader("Access-Control-Allow-Origin", httpRequest.getHeader("Origin"));
        httpResponse.setHeader("Access-Control-Allow-Headers", httpRequest.getHeader("Access-Control-Allow-Headers"));
        httpResponse.setHeader("Access-Control-Allow-Methods", "GET,POST,PUT,DELETE,OPTIONS");
        httpResponse.setHeader("Access-Control-Allow-Credentials", "true");
        
        httpResponse.setStatus(HttpServletResponse.SC_OK);
    }
    
    private Set<String> httpMethodsFromOptions(String[] options) {
        Set<String> methods = new HashSet<>();
        
        if (options != null) {
            for (String option : options) {
                // to be backwards compatible with 1.3, we can ONLY check for known args
                // ideally we would just validate HTTP methods, but someone could already be using this for webdav
                if (!option.equalsIgnoreCase(PERMISSIVE)) {
                    methods.add(option.toUpperCase(Locale.ENGLISH));
                }
            }
        }
        return methods;
    }
    
    /**
     * Processes unauthenticated requests. It handles the two-stage request/challenge authentication protocol.
     *
     * @param request  incoming ServletRequest
     * @param response outgoing ServletResponse
     * @return true if the request should be processed; false if the request should not continue to be processed
     */
    @Override
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
        boolean loggedIn = false; //false by default or we wouldn't be in this method
        if (isLoginRequest(request, response)) {
            if (log.isDebugEnabled()) {
                log.debug("Attempting to execute login with auth header");
            }
            loggedIn = executeLogin(request, response);
        }
        return loggedIn;
    }
    
    /**
     * Returns <code>true</code> if the incoming request have {@link #getAuthzHeader(ServletRequest)}
     * and the header's value is start with {@link #getAuthzScheme()}, <code>false</code> otherwise.
     *
     * @param request  the incoming <code>ServletRequest</code>
     * @param response the outgoing <code>ServletResponse</code>
     * @return <code>true</code> if the incoming request is required auth, <code>false</code> otherwise.
     */
    @Override
    protected boolean isLoginRequest(ServletRequest request, ServletResponse response) {
        String authzHeader = getAuthzHeader(request);
        String scheme = getAuthzScheme().toLowerCase(Locale.ENGLISH);
        return authzHeader != null && authzHeader.toLowerCase(Locale.ENGLISH).startsWith(scheme);
    }
    
    /**
     * @param request the incoming <code>ServletRequest</code>
     * @return the <code>Authorization</code> header's value
     */
    private String getAuthzHeader(ServletRequest request) {
        return ((HttpServletRequest) request).getHeader(AUTHORIZATION_HEADER);
    }
    
    /**
     * Returns the authentication token encapsulated by the value of the Authorization header
     *
     * @param request  the incoming <code>ServletRequest</code>
     * @param response the outgoing <code>ServletResponse</code>
     * @return the authentication token encapsulated by the value of the Authorization header
     */
    @Override
    protected AuthenticationToken createToken(ServletRequest request, ServletResponse response) throws Exception {
        String authzHeader = getAuthzHeader(request);
        if (authzHeader == null || authzHeader.length() == 0) {
            return JWTToken.NONE;
        }
        String scheme = getAuthzScheme();
        if (scheme != null && scheme.length() != 0) {
            authzHeader = authzHeader.substring(scheme.length());
        }
        String token = authzHeader.trim();
        String host = request.getRemoteHost();
        return new JWTToken(token, host);
    }
    
    /**
     * Callback processing after authentication successful.
     */
    @Override
    protected boolean onLoginSuccess(AuthenticationToken token, Subject subject, ServletRequest request, ServletResponse response) throws Exception {
        if (successfulHandler != null) {
            if (log.isDebugEnabled()) {
                log.debug("{} can pass auth, the auth subject is {}", token, subject);
            }
            successfulHandler.onSuccessful(token, subject, request, response);
        }
        return true;
    }
    
    /**
     * Callback processing after authentication failure.
     */
    @Override
    protected final boolean onLoginFailure(AuthenticationToken token, AuthenticationException e, ServletRequest request, ServletResponse response) {
        if (unsuccessfulHandler != null) {
            if (log.isDebugEnabled()) {
                log.debug("{} can not pass auth, the auth exception message is {}", token, e.getMessage());
            }
            unsuccessfulHandler.onUnsuccessful(token, e, request, response);
        }
        return false;
    }
    
}

interface UnsuccessfulHandler {
    /**
     * Callback processing when auth successful
     *
     * @param token    the token can not pass authentication
     * @param e        the exception thrown during authentication
     * @param request  the incoming <code>ServletRequest</code>
     * @param response the outgoing <code>ServletResponse</code>
     */
    void onUnsuccessful(AuthenticationToken token, AuthenticationException e, ServletRequest request, ServletResponse response);
}

interface SuccessfulHandler {
    /**
     * Callback processing when auth unsuccessful
     *
     * @param token    the token can pass authentication
     * @param subject  the incoming auth <code>Subject</code>
     * @param request  the incoming <code>ServletRequest</code>
     * @param response the outgoing <code>ServletResponse</code>
     */
    void onSuccessful(AuthenticationToken token, Subject subject, ServletRequest request, ServletResponse response);
}
```
**JwtRealm**
```java
@Component
public class JwtRealm extends AuthorizingRealm {
    
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        return null;
    }
    
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        String jwt = ((JWTToken) token).getToken();
        boolean authResult = JwtUtils.verifyToken(jwt);
        if (!authResult) {
            throw new IncorrectCredentialsException();
        }
        return new SimpleAuthenticationInfo(jwt, jwt, getName());
    }
    
    @Override
    public boolean supports(AuthenticationToken token) {
        return token instanceof JWTToken;
    }
}
```
**JwtUtils**
```java
@Slf4j
@Component
public class JwtUtils {
    
    private static String issuer;
    private static String secret;
    private static Duration lifeTime;
    private static SignatureAlgorithm algorithm;
    
    @Autowired
    public void init(SecurityProperties securityProperties) {
        SecurityProperties.JWT jwt = securityProperties.getJwt();
        JwtUtils.issuer = jwt.getIss();
        JwtUtils.secret = jwt.getSecret();
        JwtUtils.lifeTime = jwt.getLifeTime();
        JwtUtils.algorithm = jwt.getAlg();
    }
    
    public static String createToken(String subject) {
        Map<String, Object> claims = Collections.emptyMap();
        return doCreateToken(subject, claims, lifeTime);
    }
    
    public static String createToken(String subject, Consumer<Map<String, Object>> consumer) {
        Map<String, Object> claims = new HashMap<>();
        consumer.accept(claims);
        return doCreateToken(subject, claims, lifeTime);
    }
    
    public static String createToken(String subject, Map<String, Object> claims) {
        return createToken(subject, map -> map.putAll(claims));
    }
    
    public static String createToken(String subject, Map<String, Object> claims, long expiration, ChronoUnit unit) {
        return doCreateToken(subject, claims, Duration.of(expiration, unit));
    }
    
    private static String doCreateToken(String subject, Map<String, Object> claims, Duration expiration) {
        Instant now = Instant.now();
        JwtBuilder builder = Jwts.builder()
                .setHeaderParam(Header.TYPE, Header.JWT_TYPE) //添加令牌类型
                .addClaims(claims) //添加自定义Claims
                .setSubject(subject) //接受人
                .setIssuedAt(Date.from(now)) //签发时间
                .signWith(algorithm, secret);
        if (StringUtils.hasText(issuer)) {
            builder.setIssuer(issuer); //签发人
        }
        if (expiration != null && expiration.isNegative()) {
            builder.setExpiration(Date.from(now.plus(expiration))); //过期时间
        }
        String token = builder.compact();
        if (log.isTraceEnabled()) {
            log.trace("Create token[{}] in {}", token, Date.from(now));
        }
        return token;
    }
    
    public static Claims getClaims(String token) {
        Claims body;
        try {
            body = Jwts.parser()
                    .setSigningKey(secret)
                    .parseClaimsJws(token)
                    .getBody();
        } catch (ExpiredJwtException e) {
            // ignoring expired token exception
            body = e.getClaims();
        } catch (Exception e) {
            if (log.isDebugEnabled()) {
                log.debug("An exception[{}] occurred during get claims", e.getMessage());
            }
            body = null;
        }
        return body;
    }
    
    public static boolean verifyToken(String token) {
        //noinspection rawtypes
        Jwt jwt;
        try {
            jwt = parseToken(token);
        } catch (Exception e) {
            if (log.isDebugEnabled()) {
                log.debug("Token[{}] verify failed, message: {}", token, e.getMessage());
            }
            return false;
        }
        Claims claims = (Claims) jwt.getBody();
        String issuer = claims.getIssuer();
        if (issuer != null && !issuer.equals(JwtUtils.issuer)) {
            if (log.isDebugEnabled()) {
                String msg = "Incorrect issuer: " + issuer;
                log.debug("Token[{}] verify failed, message: {}", token, msg);
            }
            return false;
        }
        String subject = claims.getSubject();
        if (!StringUtils.hasText(subject)) {
            if (log.isDebugEnabled()) {
                String msg = "subject cannot be null or empty";
                log.debug("Token[{}] verify failed, message: {}", token, msg);
            }
            return false;
        }
        
        return true;
    }
    
    /**
     * @see JwtParser#parse(String)
     */
    @SuppressWarnings("rawtypes")
    private static Jwt parseToken(String token) throws ExpiredJwtException, MalformedJwtException, SignatureException {
        return Jwts.parser()
                .setSigningKey(secret)
                .parse(token);
    }
    
    public static boolean isValidToken(String token, String subject) {
        Optional<String> username = getClaimFromToken(token, Claims::getSubject);
        return username.filter(s -> (s.equals(subject) && !isTokenExpired(token))).isPresent();
    }
    
    public static boolean isTokenExpired(String token) {
        Optional<Date> expiration = getClaimFromToken(token, Claims::getExpiration);
        return expiration.map(date -> date.before(new Date())).orElse(false);
    }
    
    public static <T> Optional<T> getClaimFromToken(String token, Function<Claims, T> resolver) {
        Claims claims = getClaims(token);
        if (claims == null) {
            if (log.isDebugEnabled()) {
                log.debug("Cannot operate on a null Claims");
            }
            return Optional.empty();
        }
        return Optional.ofNullable(resolver.apply(claims));
    }
}
```
**ShiroConfig**
```java
@Configuration
@EnableConfigurationProperties(SecurityProperties.class)
@ConditionalOnProperty(name = "me.lolico.blog.security.enable", matchIfMissing = true)
public class ShiroConfig {
    
    @Bean
    public SecurityManager securityManager(Collection<Realm> realms) {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager(realms);
        
        /*
         * 关闭shiro自带的session
         */
        DefaultSubjectDAO subjectDAO = new DefaultSubjectDAO();
        DefaultSessionStorageEvaluator defaultSessionStorageEvaluator = new DefaultSessionStorageEvaluator();
        defaultSessionStorageEvaluator.setSessionStorageEnabled(false);
        subjectDAO.setSessionStorageEvaluator(defaultSessionStorageEvaluator);
        securityManager.setSubjectDAO(subjectDAO);
        
        //认证器
        ModularRealmAuthenticator authenticator = new ModularRealmAuthenticator();
        authenticator.setRealms(realms);
        //认证策略
        authenticator.setAuthenticationStrategy(new FirstSuccessfulStrategy());
        securityManager.setAuthenticator(authenticator);
        
        return securityManager;
    }
    
    @Bean
    public ShiroFilterFactoryBean shiroFilterFactoryBean(SecurityManager securityManager) {
        ShiroFilterFactoryBean factoryBean = new ShiroFilterFactoryBean();
        factoryBean.setSecurityManager(securityManager);
        
        // 自定义过滤器
        Map<String, Filter> map = new HashMap<>();
        map.put("auth", new JwtHttpAuthenticationFilter());
        factoryBean.setFilters(map);
        
        // 自定义路由策略
        Map<String, String> definitionMap = new LinkedHashMap<>();
        definitionMap.put("/login", "anon");
        definitionMap.put("/register", "anon");
        definitionMap.put("/u/confirm/**", "anon");
        definitionMap.put("/**", "auth");
        factoryBean.setFilterChainDefinitionMap(definitionMap);
        
        return factoryBean;
    }
    
    /**
     * A {@code credentialsMatcher},
     * using SHA256 algorithm and iteration three times
     *
     * @return a credentialsMatcher
     */
    @Bean
    public CredentialsMatcher hashedCredentialsMatcher() {
        HashedCredentialsMatcher credentialsMatcher = new HashedCredentialsMatcher();
        credentialsMatcher.setHashAlgorithmName(Sha256Hash.ALGORITHM_NAME);
        credentialsMatcher.setHashIterations(Constant.security.ITERATIONS);
        credentialsMatcher.setStoredCredentialsHexEncoded(Constant.security.TO_HEX);
        return credentialsMatcher;
    }
    
    /**
     * Annotation support
     */
    @Bean
    @DependsOn("lifecycleBeanPostProcessor")
    public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator() {
        DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator = new DefaultAdvisorAutoProxyCreator();
        // 强制使用cglib，防止重复代理和可能引起代理出错的问题
        // https://zhuanlan.zhihu.com/p/29161098
        defaultAdvisorAutoProxyCreator.setProxyTargetClass(true);
        return defaultAdvisorAutoProxyCreator;
    }
    
    @Bean
    public LifecycleBeanPostProcessor lifecycleBeanPostProcessor() {
        return new LifecycleBeanPostProcessor();
    }
    
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager) {
        AuthorizationAttributeSourceAdvisor advisor = new AuthorizationAttributeSourceAdvisor();
        advisor.setSecurityManager(securityManager);
        return advisor;
    }
    
}

```
- `JwtRealm#doGetAuthorizationInfo`获取权限可以从token中解析然后返回，注意在生成token的时候要放到Claims中
- `JwtAuthenticationFilter#isAccessAllowed`方法中注释的部分和`JwtAuthenticationFilter#preHandle`方法是支持cors的两种方式。

至于其他地方，应该不难理解。

## 后语
Shiro的认证流程到现在，那么也就分析完了。从Shiro的`Filter`到`SecurityManager`再到前后端分离案例，其实一路看下来，Shiro框架的代码是十分优雅并且简单易用的。  

*文章中代码较多。如有任何错误，还请指出，谢谢。*
