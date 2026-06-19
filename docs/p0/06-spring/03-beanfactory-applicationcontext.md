# BeanFactory 与 ApplicationContext

## 面试定位

`BeanFactory` 和 `ApplicationContext` 的区别是 Spring 基础面试题。浅层回答是“`BeanFactory` 是基础容器，`ApplicationContext` 是高级容器”。更好的回答要说明：

- 二者的继承和职责关系。
- Bean 创建时机差异。
- `ApplicationContext` 额外提供了哪些企业级能力。
- 实际工程中为什么几乎总是使用 `ApplicationContext`。
- Spring Boot 中上下文如何承载自动配置、事件、环境和 Web 能力。

## 一句话结论

`BeanFactory` 是 Spring IoC 的底层容器接口，负责 Bean 的注册、获取、创建和依赖管理；`ApplicationContext` 继承并扩展了 `BeanFactory`，在 IoC 基础上提供事件、国际化、资源加载、环境配置、自动注册后置处理器、应用生命周期等完整应用上下文能力。

## BeanFactory

`BeanFactory` 是最基础的 IoC 容器接口。

核心能力：

- 根据名称或类型获取 Bean。
- 判断 Bean 是否存在。
- 判断 Bean 是否单例或原型。
- 获取 Bean 类型。
- 管理 Bean 的创建和依赖注入。

常见方法：

```java
Object getBean(String name);

<T> T getBean(Class<T> requiredType);

boolean containsBean(String name);

boolean isSingleton(String name);

Class<?> getType(String name);
```

它的核心实现是：

```text
DefaultListableBeanFactory
```

`DefaultListableBeanFactory` 同时承担：

- BeanDefinition 注册表。
- BeanFactory。
- AutowireCapableBeanFactory。
- ConfigurableListableBeanFactory。

## ApplicationContext

`ApplicationContext` 是应用上下文接口，继承多个能力接口。

典型能力：

- BeanFactory 能力。
- 资源加载：`ResourceLoader`。
- 国际化：`MessageSource`。
- 应用事件：`ApplicationEventPublisher`。
- 环境抽象：`EnvironmentCapable`。
- 父子上下文：`HierarchicalBeanFactory`。
- 生命周期管理。

常见实现：

- `AnnotationConfigApplicationContext`：注解配置的非 Web 容器。
- `ClassPathXmlApplicationContext`：Classpath XML 容器。
- `FileSystemXmlApplicationContext`：文件系统 XML 容器。
- `AnnotationConfigServletWebServerApplicationContext`：Spring Boot Servlet Web 容器。
- `AnnotationConfigReactiveWebServerApplicationContext`：Spring Boot Reactive Web 容器。

## 继承关系理解

简化关系：

```text
BeanFactory
  <- ListableBeanFactory
  <- HierarchicalBeanFactory
  <- ApplicationContext
```

`ApplicationContext` 并不是替代 `BeanFactory`，而是基于 `BeanFactory` 构建应用上下文。它内部通常持有一个 `DefaultListableBeanFactory` 来完成底层 Bean 管理。

## 核心区别

| 维度 | BeanFactory | ApplicationContext |
| --- | --- | --- |
| 定位 | 基础 IoC 容器 | 完整应用上下文 |
| Bean 创建 | 默认懒加载，调用 `getBean()` 时创建 | 默认启动时预实例化非懒加载 singleton |
| 后置处理器 | 需要手动注册和触发 | 自动发现、注册和触发 |
| 国际化 | 不直接提供 | 支持 `MessageSource` |
| 事件机制 | 不直接提供完整应用事件 | 支持事件发布和监听 |
| 资源加载 | 能力有限 | 支持统一资源抽象 |
| 环境配置 | 能力有限 | 支持 Environment、Profile、PropertySource |
| Web 支持 | 不直接面向 Web | 有 WebApplicationContext 等实现 |
| 工程使用 | 偏底层框架 | 业务应用默认选择 |

## Bean 创建时机

### BeanFactory

`BeanFactory` 通常在第一次调用 `getBean()` 时才创建 Bean。

```text
创建 BeanFactory
  -> 加载 BeanDefinition
  -> getBean("userService")
  -> 创建 userService
```

### ApplicationContext

`ApplicationContext` 在 `refresh()` 末尾会预实例化非懒加载 singleton。

关键链路：

```text
refresh()
  -> finishBeanFactoryInitialization()
     -> beanFactory.preInstantiateSingletons()
```

因此默认情况下，单例 Bean 会在容器启动阶段创建。

例外：

- 标注 `@Lazy` 的 Bean。
- `prototype` Bean。
- 某些条件不满足的 Bean。
- 通过 `ObjectProvider` 等延迟获取的依赖。

## 后置处理器差异

`BeanFactory` 本身不会自动帮你完成所有上下文增强。比如直接使用 `DefaultListableBeanFactory` 时，如果没有手动注册 `AutowiredAnnotationBeanPostProcessor`，`@Autowired` 不会按预期生效。

`ApplicationContext` 在 `refresh()` 中会自动处理：

```text
invokeBeanFactoryPostProcessors()
registerBeanPostProcessors()
```

这意味着：

- `@Configuration` 能被解析。
- `@ComponentScan` 能生效。
- `@Autowired` 能生效。
- `@PostConstruct` 能生效。
- AOP 自动代理能生效。

这也是工程中基本使用 `ApplicationContext` 的原因。

## ApplicationContext 额外能力

### 资源加载

统一用 `Resource` 抽象不同位置的资源：

```java
Resource resource = applicationContext.getResource("classpath:app.yml");
```

支持：

- classpath。
- file。
- URL。
- ServletContext。

### 国际化

通过 `MessageSource` 获取国际化消息：

```java
String message = applicationContext.getMessage(
    "error.user.notFound",
    null,
    Locale.CHINA
);
```

### 事件机制

发布事件：

```java
applicationContext.publishEvent(new OrderCreatedEvent(orderId));
```

监听事件：

```java
@EventListener
public void onOrderCreated(OrderCreatedEvent event) {
}
```

底层核心：

- `ApplicationEventMulticaster`。
- `ApplicationListener`。
- `@EventListener` 适配。

### Environment 与 Profile

`ApplicationContext` 提供环境抽象：

- 当前激活 Profile。
- 配置属性来源。
- 系统环境变量。
- JVM 参数。
- 配置文件。

常见使用：

```java
String value = environment.getProperty("server.port");
```

### 父子上下文

`ApplicationContext` 支持父子层级。

查找 Bean 时：

- 先查当前上下文。
- 当前找不到再查父上下文。

Web 场景中曾经常见：

- Root WebApplicationContext：Service、Repository。
- DispatcherServlet 子上下文：Controller、MVC 组件。

Spring Boot 简化了很多传统 XML Web 应用中的上下文结构，但父子上下文仍是重要机制。

## WebApplicationContext

Web 环境下常见上下文：

```text
WebApplicationContext
  -> ServletWebServerApplicationContext
     -> AnnotationConfigServletWebServerApplicationContext
```

它在普通 ApplicationContext 基础上增加：

- ServletContext。
- ServletConfig。
- Web 作用域 Bean。
- 内嵌 WebServer 启动。
- DispatcherServlet 注册。

Spring Boot Web 应用启动大致链路：

```text
SpringApplication.run()
  -> createApplicationContext()
  -> refreshContext()
  -> ServletWebServerApplicationContext.onRefresh()
  -> createWebServer()
  -> finishRefresh()
```

## 源码链路理解

### BeanFactory 底层获取 Bean

```text
DefaultListableBeanFactory.getBean()
  -> AbstractBeanFactory.doGetBean()
  -> getSingleton()
  -> createBean()
```

### ApplicationContext 刷新流程

```text
AbstractApplicationContext.refresh()
  -> prepareRefresh()
  -> obtainFreshBeanFactory()
  -> prepareBeanFactory()
  -> postProcessBeanFactory()
  -> invokeBeanFactoryPostProcessors()
  -> registerBeanPostProcessors()
  -> initMessageSource()
  -> initApplicationEventMulticaster()
  -> onRefresh()
  -> registerListeners()
  -> finishBeanFactoryInitialization()
  -> finishRefresh()
```

面试时可以强调：`ApplicationContext` 的核心模板方法是 `refresh()`，它串起了环境准备、BeanFactory 准备、后置处理器、事件、多语言、单例初始化和生命周期发布。

## 工程实践

### 业务应用使用 ApplicationContext

普通业务应用不应该直接构造 `DefaultListableBeanFactory` 管理 Bean。直接使用 BeanFactory 很容易遗漏：

- 注解处理器。
- AOP。
- 事件。
- 配置解析。
- 生命周期回调。

Spring Boot 默认已经创建合适的 `ApplicationContext`。

### 不要滥用 ApplicationContext 获取 Bean

虽然 `ApplicationContext` 能获取 Bean，但业务代码中到处写 `getBean()` 会隐藏依赖。

更推荐构造器注入：

```java
@Service
class OrderService {
    private final UserService userService;

    OrderService(UserService userService) {
        this.userService = userService;
    }
}
```

少量合理场景：

- 框架扩展。
- 插件体系。
- 根据运行时类型动态选择 Bean。
- 解决懒加载或作用域代理问题。

这些场景也优先考虑 `ObjectProvider`、策略 Map 注入等方式。

### 用事件解耦，但不要滥用

ApplicationContext 提供事件机制，适合领域事件、应用启动事件、弱依赖通知。

但事件不是异步消息队列的替代品：

- 默认可能是同步执行。
- 事务边界需要额外关注。
- 失败处理和重试能力有限。
- 进程内事件无法跨服务。

## 常见追问

### 1. BeanFactory 和 ApplicationContext 的核心区别

`BeanFactory` 是底层 IoC 容器，负责 Bean 创建、依赖注入和获取；`ApplicationContext` 在它基础上提供完整应用能力，包括事件、国际化、资源、环境、Profile、自动注册后置处理器和预实例化单例。

### 2. 为什么 ApplicationContext 启动时就能发现配置错误

因为它默认在 `refresh()` 的 `finishBeanFactoryInitialization()` 阶段预实例化非懒加载 singleton。如果依赖缺失、配置错误、构造失败，会在启动阶段暴露。

### 3. BeanFactory 是否支持懒加载

BeanFactory 默认按需创建，调用 `getBean()` 时创建。ApplicationContext 默认会预实例化非懒加载 singleton，但也可以通过 `@Lazy` 延迟。

### 4. ApplicationContext 内部还用 BeanFactory 吗

用。ApplicationContext 是更高层的上下文，内部委托 BeanFactory 完成底层 Bean 管理，常见底层实现是 `DefaultListableBeanFactory`。

### 5. 为什么直接用 DefaultListableBeanFactory 时 `@Autowired` 可能不生效

因为 `@Autowired` 依赖 `AutowiredAnnotationBeanPostProcessor` 处理。直接使用底层 BeanFactory 如果没有注册相关后置处理器，注解不会自动生效。

### 6. 父子 ApplicationContext 查找 Bean 的规则是什么

先查子上下文，找不到再查父上下文。子上下文可以覆盖父上下文中的同名 Bean，但具体是否允许覆盖还受配置和版本约束。

## 易错点

- 只说 ApplicationContext 是 BeanFactory 子接口，不说额外能力。
- 以为 BeanFactory 完全不能处理 Bean 生命周期，实际它是生命周期底层基础，只是不提供完整上下文自动化能力。
- 以为 ApplicationContext 不使用 BeanFactory，实际内部依赖 BeanFactory。
- 以为所有 Bean 都会在 ApplicationContext 启动时创建，忽略 lazy、prototype 等例外。
- 以为事件监听默认一定异步，实际默认通常同步，除非配置异步执行器。
- 在业务代码中滥用 `applicationContext.getBean()`。

## 自检清单

- 能否一句话区分 BeanFactory 和 ApplicationContext。
- 能否说出 `DefaultListableBeanFactory` 的作用。
- 能否说明 ApplicationContext 的 `refresh()` 主流程。
- 能否解释非懒加载 singleton 的预实例化时机。
- 能否说明为什么 ApplicationContext 会自动支持 `@Autowired`、AOP、`@PostConstruct`。
- 能否说出 ApplicationContext 的事件、资源、国际化、环境能力。
- 能否说明 WebApplicationContext 的特殊能力。
- 能否解释父子上下文的 Bean 查找顺序。
- 能否说清业务代码为什么不该滥用 `getBean()`。
