---
date: 2018-8-6 19:08:05
title: 集群环境下做session共享
tags: [技术,后端,java,集群]
no_word_count: false
---

>在单机服务器上，我们将当前用户信息直接保存在session中，是完全没有问题的，但稍大点的网站，一般都是集群方式部署，通过负载均衡分发请求，如果用户在A服务器上登录后，在有效的session时间范围内，在此访问网站，此时请求可能被分发到B服务器，但是B服务器上并没有该用户的session，就会出现已经登录的用户被判断为未登录,需要用户重新登录的情况


## 什么是session
> session被用于表示一个持续的连接状态，在网站访问中一般指代客户端浏览器的进程从开启到结束的过程。session其实就是网站分析的访问（visits）度量，表示一个访问的过程。
> 
> 当用户在浏览器端发起请求到服务器端的时候，服务器首先会检查请求中有没有sessionid，如果没有，则创建一个名字叫jsessionid的cookie返回给浏览器，并且将这个sessionid以HashTable的结构形式保存在服务器中，当请求中有sessionid时，服务器会用该sessionid查找这个session的信息并直接使用，如果没有找到则重新创建一个新的sessionid返回给浏览器。
> 
> - session就是一个特殊的cookie
> - session由服务端生成
> - session表示一个会话状态
<!--more-->
## 解决session共享的方案

>搜一下，网上有很多解决方案大概如下：
>
> - 通过数据库共享session
>> 用一个数据库保存所有session信息，当用户请求来时，会到这个数据库服务器上检查session信息
> 
> - 通过cookie共享session
> >将用户的登录后的信息存在cookie中，当访问服务器时首先服务器判断有没有这个用户的session信息，如果没有在去查cookie中有没用户的信息，如果有，则将cookie中的用户信息同步到服务器中。
> 缺点：安全性太低，用户可以伪造cookie，浏览器还可以禁止cookie
> 
> - 通过多台服务器内部同步session
> > 使用一台服务器作为登录服务器,当用户登录后，登录服务器将用户的session同步到其他服务器上
> 
> - 通过NFS共享session
> >选择一台公共的NFS服务器做共享服务器，不管用户在哪台服务器登录后，都将session写到这台NFS服务器上，不管用户访问哪台服务器都来这个NFS服务器上拿session
> 
> - 使用memcache/redis共享session
> > 不管哪台服务器登录，都将用户session存放在内存中。后面也都在这里取。这种方式是最符合我们条件的方式。


## spring-session + redis 实现session共享

##### 依赖jar包
```xml
<dependency>
	<groupId>org.springframework.session</groupId>
	<artifactId>spring-session</artifactId>
	<version>1.0.2.RELEASE</version>
</dependency>
```
##### web.xml配置过滤，过滤器位置放在第一个
```xml
<filter>
    <filter-name>springSessionRepositoryFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>

<filter-mapping>
    <filter-name>springSessionRepositoryFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```
##### spring配置文件中配置bean
```xml
<bean class="org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration">
		<property name="maxInactiveIntervalInSeconds" value="1800"></property>
</bean>
```

##### 分析DelegatingFilterProxy过滤器代码
>DelegatingFilterProxy 继承自 GenericFilterBean
> 
>GenericFilterBean 实现Filter, BeanNameAware, EnvironmentAware, ServletContextAware, InitializingBean, DisposableBean接口
```java

public class DelegatingFilterProxy extends GenericFilterBean 

public abstract class GenericFilterBean implements Filter, BeanNameAware, EnvironmentAware, ServletContextAware, InitializingBean, DisposableBean

```
>GenericFilterBean 中实现了Filter中的init接口 并定义一个模板方法initFilterBean供子类实现在init中调用
```java 

public final void init(FilterConfig filterConfig) throws ServletException {
    Assert.notNull(filterConfig, "FilterConfig must not be null");
    if (this.logger.isDebugEnabled()) {
        this.logger.debug("Initializing filter '" + filterConfig.getFilterName() + "'");
    }

    this.filterConfig = filterConfig;

    try {
        PropertyValues pvs = new GenericFilterBean.FilterConfigPropertyValues(filterConfig, this.requiredProperties);
        BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
        ResourceLoader resourceLoader = new ServletContextResourceLoader(filterConfig.getServletContext());
        bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, this.environment));
        this.initBeanWrapper(bw);
        bw.setPropertyValues(pvs, true);
    } catch (BeansException var5) {
        String msg = "Failed to set bean properties on filter '" + filterConfig.getFilterName() + "': " + var5.getMessage();
        this.logger.error(msg, var5);
        throw new NestedServletException(msg, var5);
    }

    this.initFilterBean();//调用子类的initFilterBean方法
    if (this.logger.isDebugEnabled()) {
        this.logger.debug("Filter '" + filterConfig.getFilterName() + "' configured successfully");
    }

}

protected void initFilterBean() throws ServletException {
}
```

>DelegatingFilterProxy 中首先实现了initFilterBean()方法
```java
protected void initFilterBean() throws ServletException {
    Object var1 = this.delegateMonitor;
    synchronized(this.delegateMonitor) {
        if (this.delegate == null) {
            if (this.targetBeanName == null) {
                this.targetBeanName = this.getFilterName();//继承自父类，获取配置的过滤器名字即springSessionRepositoryFilter
            }

            WebApplicationContext wac = this.findWebApplicationContext();
            if (wac != null) {
                this.delegate = this.initDelegate(wac);
            }
        }

    }
}

protected Filter initDelegate(WebApplicationContext wac) throws ServletException {
    Filter delegate = (Filter)wac.getBean(this.getTargetBeanName(), Filter.class);//在容器中查找名为springSessionRepositoryFilter的过滤器实例 我们的主角上场了
    if (this.isTargetFilterLifecycle()) {
        delegate.init(this.getFilterConfig());//this.getFilterConfig()继承自GenericFilterBean返回FilterConfig对象，调用springSessionRepositoryFilter的init方法并将FilterConfig传过去
    }

    return delegate;
}

```

>SessionRepositoryFilter 这个类内部包装了原来的request，response 以及session对象,包装后的request的getsession方法返回重写的session对象。
```java

private final class SessionRepositoryRequestWrapper extends HttpServletRequestWrapper{

	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {

        request.setAttribute(SESSION_REPOSITORY_ATTR, this.sessionRepository);
        SessionRepositoryFilter<S>.SessionRepositoryRequestWrapper wrappedRequest = new SessionRepositoryFilter.SessionRepositoryRequestWrapper(request, response, this.servletContext);
        SessionRepositoryFilter<S>.SessionRepositoryResponseWrapper wrappedResponse = new SessionRepositoryFilter.SessionRepositoryResponseWrapper(wrappedRequest, response);
        HttpServletRequest strategyRequest = this.httpSessionStrategy.wrapRequest(wrappedRequest, wrappedResponse);
        HttpServletResponse strategyResponse = this.httpSessionStrategy.wrapResponse(wrappedRequest, wrappedResponse);

        try {
            filterChain.doFilter(strategyRequest, strategyResponse);//包装原来的HttpServletRequest和HttpServletResponse
        } finally {
            wrappedRequest.commitSession();//在看一下这个方法
        }

    }

	private void commitSession() {
            SessionRepositoryFilter<S>.SessionRepositoryRequestWrapper.HttpSessionWrapper wrappedSession = this.getCurrentSession();
            if (wrappedSession == null) {
                if (this.isInvalidateClientSession()) {
                    SessionRepositoryFilter.this.httpSessionStrategy.onInvalidateSession(this, this.response);
                }
            } else {
                S session = wrappedSession.session;
                SessionRepositoryFilter.this.sessionRepository.save(session);//sessionRepository这个属性通过查看 RedisHttpSessionConfiguration配置Bean发现其实就是RedisOperationsSessionRepository的实例，它的save方法中最终调用的是RedisTemple类中的方法。
                if (!this.isRequestedSessionIdValid() || !session.getId().equals(this.getRequestedSessionId())) {
                    SessionRepositoryFilter.this.httpSessionStrategy.onNewSession(session, this, this.response);
                }
            }

    }

	public HttpSession getSession(boolean create) {
            SessionRepositoryFilter<S>.SessionRepositoryRequestWrapper.HttpSessionWrapper currentSession = this.getCurrentSession();
            if (currentSession != null) {
                return currentSession;
            } else {
                String requestedSessionId = this.getRequestedSessionId();
                ExpiringSession session;
                if (requestedSessionId != null) {
                    session = (ExpiringSession)SessionRepositoryFilter.this.sessionRepository.getSession(requestedSessionId);
                    if (session != null) {
                        this.requestedSessionIdValid = true;
                        currentSession = new SessionRepositoryFilter.SessionRepositoryRequestWrapper.HttpSessionWrapper(session, this.getServletContext());
                        currentSession.setNew(false);
                        this.setCurrentSession(currentSession);
                        return currentSession;
                    }
                }

                if (!create) {
                    return null;
                } else {
                    session = (ExpiringSession)SessionRepositoryFilter.this.sessionRepository.createSession();
                    currentSession = new SessionRepositoryFilter.SessionRepositoryRequestWrapper.HttpSessionWrapper(session, this.getServletContext());
                    this.setCurrentSession(currentSession);
                    return currentSession;
                }
            }
    }
	public HttpSession getSession() {
            return this.getSession(true);
    }
	...
	private final class HttpSessionWrapper implements HttpSession{...}//在方法中定义了一个HttpSession的包装类，完全重写的原来的HttpSession对象。
	... 
}

```
>在看一下这个包装的HttpSessionWrapper  这个session包装类重写了所有HttpSession对象的方法。
>将session的数据保存在redis里以达到通过redis共享session的目的
```java
private final class HttpSessionWrapper implements HttpSession {
    private S session;  //这个S是SessionRepositoryFilter<S extends ExpiringSession>类定义的泛型，实际上它是谁呢？ 这时我们看一下最后一个配置的Bean对象发现这里的S其实就是RedisOperationsSessionRepository.RedisSession

	RedisHttpSessionConfiguration
    private final ServletContext servletContext;
    private boolean invalidated;
    private boolean old;

    public HttpSessionWrapper(S session, ServletContext servletContext) {
        this.session = session;
        this.servletContext = servletContext;
    }

    public long getCreationTime() {
        this.checkState();
        return this.session.getCreationTime();
    }

    public String getId() {
        return this.session.getId();
    }

    public long getLastAccessedTime() {
        this.checkState();
        return this.session.getLastAccessedTime();
    }

    public ServletContext getServletContext() {
        return this.servletContext;
    }

    public void setMaxInactiveInterval(int interval) {
        this.session.setMaxInactiveIntervalInSeconds(interval);
    }

    public int getMaxInactiveInterval() {
        return this.session.getMaxInactiveIntervalInSeconds();
    }

    public HttpSessionContext getSessionContext() {
        return SessionRepositoryFilter.NOOP_SESSION_CONTEXT;
    }

    public Object getAttribute(String name) {
        this.checkState();
        return this.session.getAttribute(name);
    }

    public Object getValue(String name) {
        return this.getAttribute(name);
    }

    public Enumeration<String> getAttributeNames() {
        this.checkState();
        return Collections.enumeration(this.session.getAttributeNames());
    }

    public String[] getValueNames() {
        this.checkState();
        Set<String> attrs = this.session.getAttributeNames();
        return (String[])attrs.toArray(new String[0]);
    }

    public void setAttribute(String name, Object value) {
        this.checkState();
        this.session.setAttribute(name, value);
    }

    public void putValue(String name, Object value) {
        this.setAttribute(name, value);
    }

    public void removeAttribute(String name) {
        this.checkState();
        this.session.removeAttribute(name);
    }

    public void removeValue(String name) {
        this.removeAttribute(name);
    }

    public void invalidate() {
        this.checkState();
        this.invalidated = true;
        SessionRepositoryRequestWrapper.this.requestedSessionInvalidated = true;
        SessionRepositoryRequestWrapper.this.setCurrentSession((SessionRepositoryFilter.SessionRepositoryRequestWrapper.HttpSessionWrapper)null);
        SessionRepositoryFilter.this.sessionRepository.delete(this.getId());
    }

    public void setNew(boolean isNew) {
        this.old = !isNew;
    }

    public boolean isNew() {
        this.checkState();
        return !this.old;
    }

    private void checkState() {
        if (this.invalidated) {
            throw new IllegalStateException("The HttpSession has already be invalidated.");
        }
    }
}


```

>RedisHttpSessionConfiguration 是一个配置Bean 
>- 方法：springSessionRepositoryFilter返回SessionRepositoryFilter对象。
>- 在new SessionRepositoryFilter对象时传递sessionRepository，而sessionRepository实际上是RedisOperationsSessionRepository的实例。并且上面包装session中使用的泛型S的实例就是在RedisOperationsSessionRepository类中指定为具体的RedisOperationsSessionRepository.RedisSession类型的。
```java
//springSessionRepositoryFilter
@Bean
public <S extends ExpiringSession> SessionRepositoryFilter<? extends ExpiringSession> springSessionRepositoryFilter(SessionRepository<S> sessionRepository, ServletContext servletContext) {
    SessionRepositoryFilter<S> sessionRepositoryFilter = new SessionRepositoryFilter(sessionRepository);
    sessionRepositoryFilter.setServletContext(servletContext);
    if (this.httpSessionStrategy != null) {
        sessionRepositoryFilter.setHttpSessionStrategy(this.httpSessionStrategy);
    }

    return sessionRepositoryFilter;
}

//sessionRepository实例，RedisOperationsSessionRepository中的sessionRedisOperations就是这个RedisTemplate实例
@Bean
public RedisOperationsSessionRepository sessionRepository(RedisTemplate<String, ExpiringSession> sessionRedisTemplate) {
    RedisOperationsSessionRepository sessionRepository = new RedisOperationsSessionRepository(sessionRedisTemplate);
    sessionRepository.setDefaultMaxInactiveInterval(this.maxInactiveIntervalInSeconds);
    return sessionRepository;
}

@Bean //使用sessionRepository创建SessionRepositoryFilter实例
public <S extends ExpiringSession> SessionRepositoryFilter<? extends ExpiringSession> springSessionRepositoryFilter(SessionRepository<S> sessionRepository, ServletContext servletContext) {
    SessionRepositoryFilter<S> sessionRepositoryFilter = new SessionRepositoryFilter(sessionRepository);
    sessionRepositoryFilter.setServletContext(servletContext);
    if (this.httpSessionStrategy != null) {
        sessionRepositoryFilter.setHttpSessionStrategy(this.httpSessionStrategy);
    }

    return sessionRepositoryFilter;
}
```

>RedisOperationsSessionRepository  指定了泛型S的具体类型RedisOperationsSessionRepository.RedisSession
```java
public class RedisOperationsSessionRepository implements SessionRepository<RedisOperationsSessionRepository.RedisSession> {

...  //RedisSession的定义
final class RedisSession implements ExpiringSession {
        private final MapSession cached;
        private Long originalLastAccessTime;
        private Map<String, Object> delta;

        RedisSession() {
            this(new MapSession());
            this.delta.put("creationTime", this.getCreationTime());
            this.delta.put("maxInactiveInterval", this.getMaxInactiveIntervalInSeconds());
            this.delta.put("lastAccessedTime", this.getLastAccessedTime());
        }

        RedisSession(MapSession cached) {
            this.delta = new HashMap();
            Assert.notNull("MapSession cannot be null");
            this.cached = cached;
        }

        public void setLastAccessedTime(long lastAccessedTime) {
            this.cached.setLastAccessedTime(lastAccessedTime);
            this.delta.put("lastAccessedTime", this.getLastAccessedTime());
        }

        public boolean isExpired() {
            return this.cached.isExpired();
        }

        public long getCreationTime() {
            return this.cached.getCreationTime();
        }

        public String getId() {
            return this.cached.getId();
        }

        public long getLastAccessedTime() {
            return this.cached.getLastAccessedTime();
        }

        public void setMaxInactiveIntervalInSeconds(int interval) {
            this.cached.setMaxInactiveIntervalInSeconds(interval);
            this.delta.put("maxInactiveInterval", this.getMaxInactiveIntervalInSeconds());
        }

        public int getMaxInactiveIntervalInSeconds() {
            return this.cached.getMaxInactiveIntervalInSeconds();
        }

        public Object getAttribute(String attributeName) {
            return this.cached.getAttribute(attributeName);
        }

        public Set<String> getAttributeNames() {
            return this.cached.getAttributeNames();
        }

        public void setAttribute(String attributeName, Object attributeValue) {
            this.cached.setAttribute(attributeName, attributeValue);
            this.delta.put(RedisOperationsSessionRepository.getSessionAttrNameKey(attributeName), attributeValue);
        }

        public void removeAttribute(String attributeName) {
            this.cached.removeAttribute(attributeName);
            this.delta.put(RedisOperationsSessionRepository.getSessionAttrNameKey(attributeName), (Object)null);
        }

        private void saveDelta() {
            String sessionId = this.getId();
            RedisOperationsSessionRepository.this.getSessionBoundHashOperations(sessionId).putAll(this.delta);
            this.delta = new HashMap(this.delta.size());
            RedisOperationsSessionRepository.this.expirationPolicy.onExpirationUpdated(this.originalLastAccessTime, this);
        }
    }
...

}
```
##### 最后还是贴个图吧
![](SessionRepositoryFilter.png)


## 总结

> 代码一点一点缕,整理的有点乱 
> - 整个过程就是通过SessionFilter过滤器包装Request，Response，Session替换Http原来的那帮对象，通过过滤器传到到了我们自己的业务代码中的Req、Reps已经是被掉包的对象了，通过这个对象获取的Session也是掉包后的Session，这个Session中保存的数据，会被保存到Redis中，获取也是从Redis中获取，最后对Session的操作变成了对Redis的操作，而整个过程对我们开发来说是完全透明的。
> 




