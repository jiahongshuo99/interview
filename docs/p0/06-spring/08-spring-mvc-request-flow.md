# Spring MVC 请求处理链路

## 面试定位

Spring MVC 请求处理链路是 Web 后端面试核心题。面试官通常关注：

- `DispatcherServlet` 的作用。
- HandlerMapping、HandlerAdapter、HandlerInterceptor 分别做什么。
- Controller 参数如何绑定。
- 返回值如何处理。
- `@ResponseBody` 为什么能返回 JSON。
- 异常如何统一处理。
- Filter、Interceptor、AOP 在请求链路中的位置。

回答时要围绕前端控制器模式：所有请求先进入 `DispatcherServlet`，再由它协调映射、适配、调用、渲染和异常处理。

## 总体链路

典型 Spring MVC 请求链路：

```text
客户端请求
  -> Servlet Filter 链
  -> DispatcherServlet
     -> HandlerMapping 查找 Handler
     -> HandlerExecutionChain
        -> Interceptor.preHandle()
     -> HandlerAdapter 调用 Handler
        -> 参数解析 HandlerMethodArgumentResolver
        -> 执行 Controller 方法
        -> 返回值处理 HandlerMethodReturnValueHandler
     -> Interceptor.postHandle()
     -> 视图解析 / HTTP 消息转换
     -> Interceptor.afterCompletion()
  -> 响应客户端
```

如果 Controller 返回 JSON，通常不会走传统 JSP 视图渲染，而是通过 `HttpMessageConverter` 写入响应体。

## DispatcherServlet

`DispatcherServlet` 是 Spring MVC 的核心前端控制器。

职责：

- 接收请求。
- 查找处理器。
- 调用拦截器。
- 选择合适的适配器执行处理器。
- 处理参数绑定和返回值。
- 处理异常。
- 渲染视图或写响应体。

入口方法：

```text
FrameworkServlet.service()
  -> processRequest()
     -> DispatcherServlet.doService()
        -> doDispatch()
```

核心逻辑在：

```text
DispatcherServlet.doDispatch()
```

## HandlerMapping

`HandlerMapping` 负责根据请求找到处理器。

常见实现：

- `RequestMappingHandlerMapping`：处理 `@RequestMapping`、`@GetMapping`、`@PostMapping` 等注解方法。
- `BeanNameUrlHandlerMapping`：根据 BeanName 映射 URL。
- 静态资源相关 HandlerMapping。

示例：

```java
@RestController
@RequestMapping("/orders")
class OrderController {
    @GetMapping("/{id}")
    public OrderVO get(@PathVariable Long id) {
        return orderService.get(id);
    }
}
```

`RequestMappingHandlerMapping` 会在启动阶段扫描这些注解，建立 URL、HTTP 方法、参数条件到 HandlerMethod 的映射。

## HandlerExecutionChain

`HandlerMapping` 返回的不只是 Handler，还包括拦截器链。

```text
HandlerExecutionChain
  -> handler
  -> interceptors
```

拦截器执行顺序：

```text
preHandle  正序执行
postHandle 倒序执行
afterCompletion 倒序执行
```

如果某个 `preHandle()` 返回 false：

- 后续 Controller 不执行。
- 后续未执行的拦截器不进入。
- 已执行 `preHandle()` 的拦截器会触发 `afterCompletion()`。

## HandlerAdapter

`HandlerAdapter` 负责真正调用 Handler。

为什么需要 Adapter：

- Spring MVC 支持多种 Handler 形式。
- DispatcherServlet 不直接知道每种 Handler 如何调用。
- 通过适配器屏蔽差异。

最常见实现：

```text
RequestMappingHandlerAdapter
```

它负责调用 `@RequestMapping` 标注的 Controller 方法。

核心能力：

- 参数解析。
- 数据绑定。
- 类型转换。
- 校验。
- 执行 Controller。
- 返回值处理。

## 参数解析

Controller 方法参数由 `HandlerMethodArgumentResolver` 解析。

常见参数注解：

- `@RequestParam`。
- `@PathVariable`。
- `@RequestBody`。
- `@RequestHeader`。
- `@CookieValue`。
- `@ModelAttribute`。

示例：

```java
@PostMapping("/orders/{id}")
public OrderVO update(
        @PathVariable Long id,
        @RequestBody UpdateOrderRequest request,
        @RequestHeader("X-User-Id") Long userId) {
    return orderService.update(id, request, userId);
}
```

大致解析：

- `@PathVariable` 从 URI 模板变量取值。
- `@RequestParam` 从 query/form 参数取值。
- `@RequestBody` 通过 `HttpMessageConverter` 读取请求体。
- 普通对象可通过数据绑定从请求参数组装。

## 类型转换与数据绑定

Spring MVC 会通过 ConversionService、PropertyEditor、Formatter 等机制把字符串转换成目标类型。

例如：

```text
"123" -> Long
"2026-06-19" -> LocalDate
```

绑定对象：

```java
public List<OrderVO> query(OrderQuery query) {
}
```

请求参数：

```text
?status=PAID&pageNo=1&pageSize=20
```

会绑定到 `OrderQuery` 对象对应字段。

校验通常配合：

```java
public void create(@Valid @RequestBody CreateOrderRequest request) {
}
```

校验失败会抛出相关绑定或校验异常，再进入异常处理链路。

## HttpMessageConverter

`HttpMessageConverter` 负责 HTTP body 和 Java 对象之间的转换。

常见场景：

- `@RequestBody`：请求体 JSON 转 Java 对象。
- `@ResponseBody`：Java 对象转 JSON 写入响应体。

常见实现：

- `MappingJackson2HttpMessageConverter`：JSON。
- `StringHttpMessageConverter`：字符串。
- `ByteArrayHttpMessageConverter`：字节数组。
- `FormHttpMessageConverter`：表单。

示例：

```java
@RestController
class OrderController {
    @GetMapping("/orders/{id}")
    public OrderVO get(@PathVariable Long id) {
        return orderService.get(id);
    }
}
```

`@RestController` 等价于 `@Controller + @ResponseBody`，返回对象会通过消息转换器写成 JSON。

## 返回值处理

Controller 返回值由 `HandlerMethodReturnValueHandler` 处理。

常见返回类型：

- `ModelAndView`。
- `String` 视图名。
- 普通对象配合 `@ResponseBody`。
- `ResponseEntity<T>`。
- `void`。
- 异步返回：`Callable`、`DeferredResult`、`WebAsyncTask`。

JSON 返回链路：

```text
Controller 返回 OrderVO
  -> RequestResponseBodyMethodProcessor
  -> HttpMessageConverter.write()
  -> response body
```

视图返回链路：

```text
Controller 返回 viewName
  -> ViewResolver
  -> View.render()
```

## 视图解析

传统 MVC 会通过 `ViewResolver` 把逻辑视图名解析成具体视图。

示例：

```java
@Controller
class PageController {
    @GetMapping("/index")
    public String index() {
        return "index";
    }
}
```

可能解析为：

```text
/WEB-INF/views/index.jsp
```

前后端分离项目中，更多使用 `@RestController` 返回 JSON，传统视图解析使用较少。

## 异常处理链路

Controller 执行异常后，会进入 `HandlerExceptionResolver`。

常见实现：

- `ExceptionHandlerExceptionResolver`：处理 `@ExceptionHandler`。
- `ResponseStatusExceptionResolver`：处理 `@ResponseStatus`。
- `DefaultHandlerExceptionResolver`：处理 Spring MVC 内置异常。

全局异常处理：

```java
@RestControllerAdvice
class GlobalExceptionHandler {
    @ExceptionHandler(BizException.class)
    public ErrorResponse handleBiz(BizException e) {
        return ErrorResponse.of(e.getCode(), e.getMessage());
    }
}
```

链路：

```text
Controller 抛异常
  -> HandlerExceptionResolver
  -> @ExceptionHandler 方法
  -> 返回错误响应
```

## Interceptor 执行时机

接口：

```java
public interface HandlerInterceptor {
    boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler);

    void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView);

    void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex);
}
```

执行顺序：

```text
preHandle
  -> Controller
  -> postHandle
  -> 视图渲染或响应写出
  -> afterCompletion
```

注意：

- `postHandle` 在 Controller 正常执行后、视图渲染前。
- 如果返回 JSON，`postHandle` 时响应体可能还没完全写出。
- `afterCompletion` 用于资源清理、日志收尾。
- 如果 Controller 抛异常，`postHandle` 不一定执行，但 `afterCompletion` 会在完成后执行。

## Filter 在链路中的位置

Filter 属于 Servlet 规范，位于 `DispatcherServlet` 之前。

```text
客户端
  -> Filter1
  -> Filter2
  -> DispatcherServlet
  -> Spring MVC
```

常见用途：

- 编码处理。
- CORS。
- 鉴权入口。
- 请求日志。
- TraceId。
- 安全框架入口，例如 Spring Security FilterChain。

Filter 可以拦截不进入 Spring MVC 的请求，例如静态资源或其他 Servlet。

## AOP 在链路中的位置

AOP 作用于 Spring Bean 方法调用。

Controller 如果是 Spring Bean，也可以被 AOP 增强：

```text
DispatcherServlet
  -> HandlerAdapter
     -> Controller 代理对象
        -> AOP 拦截器
        -> Controller 方法
```

Service 层事务 AOP：

```text
Controller
  -> Service 代理对象
     -> TransactionInterceptor
     -> Service 方法
```

AOP 不直接理解 HTTP 请求，它只理解方法调用。

## Spring Boot 自动配置中的 MVC

Spring Boot 会自动配置很多 MVC 组件：

- `DispatcherServlet`。
- `RequestMappingHandlerMapping`。
- `RequestMappingHandlerAdapter`。
- JSON 消息转换器。
- 静态资源处理。
- 格式化器和转换器。
- 异常处理。

常见扩展：

```java
@Configuration
class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/api/**");
    }
}
```

注意：通常不要随便使用 `@EnableWebMvc`，它会接管 Spring Boot 的 MVC 自动配置，导致一些默认配置失效。

## 源码主线

核心方法：

```text
DispatcherServlet.doDispatch()
```

简化源码流程：

```text
mappedHandler = getHandler(processedRequest)
ha = getHandlerAdapter(mappedHandler.getHandler())
mappedHandler.applyPreHandle()
mv = ha.handle(processedRequest, response, mappedHandler.getHandler())
mappedHandler.applyPostHandle()
processDispatchResult()
mappedHandler.triggerAfterCompletion()
```

`RequestMappingHandlerAdapter` 内部：

```text
handleInternal()
  -> invokeHandlerMethod()
     -> ServletInvocableHandlerMethod.invokeAndHandle()
        -> getMethodArgumentValues()
        -> doInvoke()
        -> handleReturnValue()
```

## 工程实践

- Controller 保持薄层，只做参数接收、校验、响应转换，业务放 service。
- 全局异常处理统一响应格式，不要在每个 Controller 重复 try-catch。
- 认证鉴权入口优先考虑 Filter 或 Spring Security；业务权限也可结合 Interceptor 或 AOP。
- 请求日志和 TraceId 更适合 Filter，因为能覆盖更早链路。
- HandlerInterceptor 适合基于 HandlerMethod 的 MVC 级拦截。
- 不要在 Controller 中开启复杂长事务，事务边界通常放 service。
- `@RequestBody` 只能读取请求体，GET 请求复杂参数不要滥用 body。
- 明确区分参数绑定异常、业务异常和系统异常。

## 常见追问

### 1. DispatcherServlet 的作用是什么

它是 Spring MVC 前端控制器，统一接收请求，协调 HandlerMapping、HandlerAdapter、拦截器、异常处理、视图解析和消息转换。

### 2. HandlerMapping 和 HandlerAdapter 区别

HandlerMapping 负责根据请求找到 Handler；HandlerAdapter 负责以合适方式调用 Handler。

### 3. `@RequestBody` 如何转成 Java 对象

由参数解析器识别 `@RequestBody`，再通过 `HttpMessageConverter` 读取请求体，例如用 Jackson 把 JSON 转成 Java 对象。

### 4. `@ResponseBody` 如何返回 JSON

返回值处理器识别响应体写出语义，再通过 `HttpMessageConverter` 把 Java 对象序列化成 JSON 写入 HTTP response。

### 5. `@ControllerAdvice` 如何生效

Spring MVC 启动时发现并注册全局增强组件。Controller 抛异常后，`ExceptionHandlerExceptionResolver` 会查找匹配的 `@ExceptionHandler` 方法处理。

### 6. Interceptor 的 `postHandle` 和 `afterCompletion` 区别

`postHandle` 在 Controller 正常执行后、视图渲染前；`afterCompletion` 在请求完成后执行，适合资源清理和最终日志。

### 7. 为什么不要随便加 `@EnableWebMvc`

在 Spring Boot 中，`@EnableWebMvc` 会导入完整 MVC 配置并可能使 Boot 的 Web MVC 自动配置退让，导致消息转换器、静态资源、格式化等默认能力变化。

## 易错点

- 把 HandlerMapping 和 HandlerAdapter 职责混淆。
- 以为 `@RestController` 是新 Controller 类型，实际是 `@Controller + @ResponseBody`。
- 以为 JSON 转换由 Controller 自己完成，实际由消息转换器处理。
- 以为 Interceptor 能拦截所有请求，实际它在 DispatcherServlet 之后，只处理进入 Spring MVC 的请求。
- 以为 Filter 能直接拿到 HandlerMethod，实际 Filter 在 MVC 映射之前。
- 在 Spring Boot 中误用 `@EnableWebMvc` 导致默认配置丢失。

## 自检清单

- 能否画出完整请求链路。
- 能否说明 DispatcherServlet 的核心职责。
- 能否区分 HandlerMapping 和 HandlerAdapter。
- 能否解释参数解析和返回值处理。
- 能否说明 HttpMessageConverter 的作用。
- 能否说清 Interceptor 三个方法的执行时机。
- 能否解释全局异常处理链路。
- 能否把 Filter、Interceptor、AOP 放到正确位置。
