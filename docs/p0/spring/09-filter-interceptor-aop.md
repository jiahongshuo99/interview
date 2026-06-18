# Filter、Interceptor、AOP 的区别

## 面试定位

Filter、Interceptor、AOP 都能“拦截”请求或方法，因此面试里很容易混淆。高质量回答要从层级、规范来源、拦截范围、可获取上下文、执行时机和典型用途来区分。

核心结论：

- Filter 是 Servlet 规范，工作在 Spring MVC 之前，面向 Web 请求。
- Interceptor 是 Spring MVC 机制，工作在 HandlerMapping 之后、Controller 调用前后，面向 MVC Handler。
- AOP 是 Spring 代理机制，工作在 Spring Bean 方法调用处，面向方法增强，不限于 Web 请求。

## 总体位置

请求进入一个 Spring MVC 应用的大致链路：

```text
客户端
  -> Servlet Filter 链
  -> DispatcherServlet
     -> HandlerMapping
     -> HandlerInterceptor.preHandle()
     -> HandlerAdapter
        -> Controller 代理对象
           -> AOP 拦截器
           -> Controller 方法
        -> Service 代理对象
           -> AOP 拦截器
           -> Service 方法
     -> HandlerInterceptor.postHandle()
     -> 视图渲染 / JSON 写出
     -> HandlerInterceptor.afterCompletion()
  -> Filter 返回链路
  -> 客户端
```

## Filter

Filter 是 Servlet 规范提供的过滤器。

接口：

```java
public interface Filter {
    void doFilter(ServletRequest request, ServletResponse response, FilterChain chain);
}
```

特点：

- 由 Servlet 容器调用。
- 位于 `DispatcherServlet` 之前。
- 可拦截所有匹配 URL 的请求。
- 不依赖 Spring MVC HandlerMapping。
- 对静态资源、非 MVC Servlet 请求也可能生效。
- 只能拿到 Servlet 层面的 request / response。

典型用途：

- 编码设置。
- CORS。
- 请求日志。
- TraceId。
- 安全认证入口。
- XSS 包装。
- 请求体缓存包装。
- Spring Security 过滤器链。

示例：

```java
@Component
class TraceFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        try {
            MDC.put("traceId", UUID.randomUUID().toString());
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

注册方式：

- `@WebFilter`。
- `FilterRegistrationBean`。
- `@Component`。
- Spring Security 内部注册。

## Interceptor

Interceptor 是 Spring MVC 的拦截器。

接口：

```java
public interface HandlerInterceptor {
    boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler);

    void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView);

    void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex);
}
```

特点：

- 属于 Spring MVC。
- 位于 `DispatcherServlet` 之后。
- HandlerMapping 找到 Handler 后执行。
- 可以拿到 Handler，常见是 `HandlerMethod`。
- 可以基于 Controller 方法、注解、路径做拦截。
- 只对进入 Spring MVC 的请求生效。

典型用途：

- 登录态检查。
- 基于注解的权限判断。
- Controller 级请求日志。
- 接口幂等 token 检查。
- 多租户上下文准备。
- Locale / Theme 处理。

示例：

```java
class AuthInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        if (handler instanceof HandlerMethod handlerMethod) {
            RequireLogin requireLogin = handlerMethod.getMethodAnnotation(RequireLogin.class);
            if (requireLogin != null) {
                // check login
            }
        }
        return true;
    }
}
```

注册：

```java
@Configuration
class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/login");
    }
}
```

## AOP

AOP 是 Spring 基于代理的面向切面编程能力。

特点：

- 作用于 Spring Bean 方法调用。
- 不限于 Web 层。
- 依赖代理对象。
- 可以用于 Controller、Service、Repository 等 Bean。
- 可基于方法签名、注解、包路径等切点匹配。
- 不能直接拦截未经过代理的方法调用。

典型用途：

- 事务。
- 缓存。
- 方法级权限。
- 审计日志。
- 指标采集。
- 限流。
- 幂等。
- 分布式锁。

示例：

```java
@Aspect
@Component
class AuditAspect {
    @Around("@annotation(audit)")
    public Object around(ProceedingJoinPoint pjp, Audit audit) throws Throwable {
        try {
            return pjp.proceed();
        } finally {
            // write audit log
        }
    }
}
```

## 核心区别表

| 维度 | Filter | Interceptor | AOP |
| --- | --- | --- | --- |
| 来源 | Servlet 规范 | Spring MVC | Spring AOP |
| 作用位置 | DispatcherServlet 之前 | HandlerMapping 之后，Controller 前后 | Spring Bean 方法调用处 |
| 拦截对象 | Servlet 请求 | MVC Handler | Bean 方法 |
| Web 依赖 | 强依赖 Web | 强依赖 Spring MVC | 不依赖 Web |
| 能否拿到 HandlerMethod | 默认不能 | 可以 | 切 Controller 时可从方法签名拿 |
| 是否能拦截静态资源 | 可以，取决于 URL 匹配 | 通常不拦截未进入 MVC Handler 的资源 | 不适合 |
| 是否依赖 Spring 代理 | 不依赖 | 不依赖 | 依赖 |
| 常见用途 | 安全入口、编码、CORS、TraceId | 登录、权限、Handler 注解检查 | 事务、缓存、方法日志、审计 |

## 执行顺序

### 多个 Filter

Filter 按注册顺序形成链：

```text
Filter1 before
  Filter2 before
    DispatcherServlet
  Filter2 after
Filter1 after
```

可用 `FilterRegistrationBean#setOrder()` 或 `@Order` 控制顺序。

### 多个 Interceptor

注册顺序为：

```text
Interceptor1.preHandle()
  Interceptor2.preHandle()
    Controller
  Interceptor2.postHandle()
Interceptor1.postHandle()
Interceptor2.afterCompletion()
Interceptor1.afterCompletion()
```

`preHandle` 正序，`postHandle` 和 `afterCompletion` 逆序。

### 多个 AOP 切面

通过 `@Order` 控制。

环绕通知类似：

```text
Aspect1 before
  Aspect2 before
    target method
  Aspect2 after
Aspect1 after
```

## 异常时的表现

### Filter

Filter 包在整个 Servlet 调用外层。

如果后续链路抛异常：

```java
try {
    chain.doFilter(request, response);
} catch (Exception e) {
    throw e;
} finally {
    // always execute
}
```

Filter 可以观察到 MVC 未处理的异常，但如果异常已被 MVC 全局异常处理转换成正常响应，Filter 外层看到的可能只是正常返回。

### Interceptor

- `preHandle` 返回 false：Controller 不执行。
- Controller 正常执行：`postHandle` 执行。
- 请求完成后：`afterCompletion` 执行。
- Controller 抛异常：`postHandle` 可能不执行，`afterCompletion` 通常执行，`ex` 参数可能有异常。

### AOP

AOP 是否感知异常取决于切点和通知类型。

```java
@AfterThrowing(pointcut = "serviceMethods()", throwing = "ex")
public void afterThrowing(Throwable ex) {
}
```

环绕通知要注意不要吞异常，否则可能影响事务回滚。

## 典型选型

### TraceId 放哪里

通常放 Filter。

原因：

- 越早越好。
- 覆盖整个请求链路。
- 包括未进入 Controller 的请求。

### 登录校验放哪里

取决于系统：

- 使用 Spring Security：核心在 FilterChain。
- 简单 MVC 系统：可用 Interceptor。
- 方法级权限：可用 AOP 或 Spring Security Method Security。

### 事务放哪里

放 AOP，通常作用于 service 方法。

原因：

- 事务是方法级业务边界。
- 不应该和 HTTP 强绑定。
- 同一 service 方法可能被 HTTP、MQ、定时任务等入口调用。

### 接口耗时统计放哪里

看统计目标：

- 统计完整 HTTP 请求耗时：Filter。
- 统计 Controller 方法耗时：Interceptor 或 AOP。
- 统计 Service 方法耗时：AOP。

### 基于 Controller 注解的权限放哪里

Interceptor 或 AOP 都可。

Interceptor 优势：

- 天然拿到 `HandlerMethod`。
- 适合 Web 权限。

AOP 优势：

- 可复用于非 Web 方法。
- 可做更细粒度方法级权限。

## 与 Spring Security 的关系

Spring Security 的 Web 安全主要基于 Filter 链。

请求链路：

```text
DelegatingFilterProxy
  -> FilterChainProxy
     -> SecurityFilterChain
        -> UsernamePasswordAuthenticationFilter
        -> AuthorizationFilter
        -> ...
  -> DispatcherServlet
```

这也是为什么很多认证授权逻辑发生在 MVC Interceptor 之前。

方法级安全如 `@PreAuthorize` 则基于 AOP 思路，在方法调用处拦截。

## 常见追问

### 1. Filter 和 Interceptor 最大区别是什么

Filter 是 Servlet 规范，位于 DispatcherServlet 之前，面向所有匹配请求；Interceptor 是 Spring MVC 机制，位于 HandlerMapping 之后，面向 MVC Handler。

### 2. Interceptor 和 AOP 区别是什么

Interceptor 面向 Web 请求和 Handler，能拿到 request、response、HandlerMethod；AOP 面向 Spring Bean 方法调用，不依赖 Web，常用于事务、缓存、审计等方法级增强。

### 3. 为什么 Filter 拿不到 Controller 方法注解

Filter 执行时还没有进入 DispatcherServlet，也没有经过 HandlerMapping，自然还不知道当前请求会匹配哪个 HandlerMethod。

### 4. 为什么 Interceptor 不能拦截所有请求

Interceptor 是 Spring MVC 的机制，只对进入 DispatcherServlet 并匹配到 MVC Handler 的请求生效。未进入 MVC 的请求不经过它。

### 5. AOP 能不能替代 Interceptor

不能完全替代。AOP 可以增强 Controller 方法，但它不天然覆盖完整 HTTP 链路，也不适合处理请求进入 MVC 之前的逻辑。Interceptor 更适合 MVC Handler 级别的 Web 拦截。

### 6. 事务为什么不用 Filter 或 Interceptor 做

事务边界应绑定业务方法，而不是 HTTP 请求。Service 方法可能被 MQ、定时任务、RPC 等入口调用，AOP 方法级事务更通用、更准确。

## 易错点

- 把 Filter、Interceptor、AOP 都说成“拦截器”，不区分层级。
- 以为 Interceptor 在 DispatcherServlet 之前执行。
- 以为 Filter 能直接判断 Controller 方法注解。
- 以为 AOP 能拦截所有方法，包括 private、自调用、非 Spring Bean。
- 在 Filter 中读取请求体后不包装，导致 Controller 无法再次读取。
- 在 AOP 环绕通知中吞异常，影响事务回滚。
- 把登录、权限、事务、日志全部堆到一种机制里。

## 自检清单

- 能否画出 Filter、Interceptor、AOP 的相对位置。
- 能否说明三者分别来自哪个规范或框架。
- 能否说明三者适合的典型场景。
- 能否解释为什么 Filter 拿不到 HandlerMethod。
- 能否说明 Interceptor 三个方法的执行顺序。
- 能否说明 AOP 为什么依赖代理对象。
- 能否根据 TraceId、权限、事务、耗时统计做技术选型。
