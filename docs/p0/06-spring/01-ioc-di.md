# IoC / DI 原理

## 面试定位

IoC 和 DI 是 Spring 的底层入口。面试通常不会只问“控制反转是什么”，而是继续追问：容器到底管理了什么、对象什么时候创建、依赖如何解析、为什么能按类型注入、`@Autowired` 和 `@Resource` 有什么差异、循环依赖为什么只在部分场景能解决。

回答时要把抽象概念落到容器流程上：

- IoC 是一种设计思想：对象创建权和依赖组装权从业务代码转移给容器。
- DI 是 IoC 的主要实现方式：容器在创建 Bean 时把依赖注入进去。
- Spring 的核心工作是把配置、注解或扫描结果解析成 `BeanDefinition`，再由 `BeanFactory` 按生命周期创建、装配、初始化、缓存和销毁 Bean。

## 核心概念

### IoC：控制反转

传统代码里对象主动创建依赖：

```java
class OrderService {
    private final UserService userService = new UserService();
}
```

这会带来几个问题：

- 对象之间强耦合，替换实现困难。
- 测试困难，依赖无法方便 mock。
- 生命周期散落在业务代码里，无法统一增强。

IoC 下对象不再主动创建依赖，而是声明自己需要什么：

```java
@Service
class OrderService {
    private final UserService userService;

    OrderService(UserService userService) {
        this.userService = userService;
    }
}
```

容器负责：

- 发现 Bean。
- 创建 Bean。
- 解析依赖。
- 注入依赖。
- 管理生命周期。
- 提供扩展点和增强能力。

### DI：依赖注入

DI 是容器把依赖传给对象的过程，常见方式有：

- 构造器注入：依赖不可变，适合必需依赖，推荐优先使用。
- Setter 注入：适合可选依赖或运行期可替换依赖。
- 字段注入：写法简短，但不利于测试、不可变性和显式依赖表达，工程上不推荐。
- 方法注入：用于少量特殊场景，例如 `@Bean` 方法参数注入。

构造器注入示例：

```java
@Service
class PayService {
    private final RiskService riskService;

    PayService(RiskService riskService) {
        this.riskService = riskService;
    }
}
```

Setter 注入示例：

```java
@Service
class PayService {
    private RiskService riskService;

    @Autowired
    public void setRiskService(RiskService riskService) {
        this.riskService = riskService;
    }
}
```

字段注入示例：

```java
@Service
class PayService {
    @Autowired
    private RiskService riskService;
}
```

## 容器到底管理什么

Spring 容器不是简单的 Map。它至少管理以下信息：

- Bean 的元数据：类名、作用域、是否懒加载、初始化方法、销毁方法、依赖关系等。
- Bean 实例：单例池中的成品对象。
- 早期对象：用于解决部分循环依赖。
- 后置处理器：`BeanFactoryPostProcessor`、`BeanPostProcessor` 等扩展点。
- 环境与配置：`Environment`、`PropertySource`、Profile。
- 资源、事件、国际化等上下文能力。

其中最关键的是 `BeanDefinition` 和 Bean 实例。

`BeanDefinition` 是 Bean 的“配方”，不是 Bean 本身。容器先收集配方，再按配方创建对象。

## BeanDefinition

`BeanDefinition` 中常见信息包括：

- `beanClassName`：Bean 对应的类。
- `scope`：`singleton`、`prototype`、`request`、`session` 等。
- `lazyInit`：是否懒加载。
- `dependsOn`：显式依赖。
- `primary`：是否作为优先候选。
- `factoryBeanName` / `factoryMethodName`：工厂 Bean 或工厂方法。
- `initMethodName` / `destroyMethodName`：初始化和销毁方法。
- 构造参数和属性值。

注解、XML、Java Config 最终都会被解析成 `BeanDefinition`。

例如：

```java
@Service
class UserService {
}
```

扫描后会形成一个 `BeanDefinition`，注册到 `BeanDefinitionRegistry` 中。

## Spring 容器启动主流程

以 `AnnotationConfigApplicationContext` 为例，核心流程在 `refresh()`：

```text
AnnotationConfigApplicationContext()
  -> register(componentClasses)
  -> refresh()
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

其中和 IoC / DI 最相关的是：

- `invokeBeanFactoryPostProcessors()`：允许修改 BeanDefinition。
- `registerBeanPostProcessors()`：注册 Bean 创建过程中的增强器。
- `finishBeanFactoryInitialization()`：实例化非懒加载单例 Bean。

## Bean 创建与依赖注入调用链

典型单例 Bean 创建链路：

```text
getBean()
  -> doGetBean()
     -> getSingleton(beanName)
     -> getMergedLocalBeanDefinition(beanName)
     -> createBean(beanName, mbd, args)
        -> resolveBeforeInstantiation()
        -> doCreateBean()
           -> createBeanInstance()
           -> addSingletonFactory()
           -> populateBean()
           -> initializeBean()
           -> registerDisposableBeanIfNecessary()
```

关键点：

- `createBeanInstance()`：实例化对象，调用构造器或工厂方法。
- `populateBean()`：填充属性，完成依赖注入。
- `initializeBean()`：执行 Aware、BeanPostProcessor、初始化方法等。
- `addSingletonFactory()`：提前暴露单例工厂，服务于循环依赖。

## 依赖解析机制

当容器看到 `@Autowired` 或构造器参数时，会把依赖描述成 `DependencyDescriptor`，再调用依赖解析逻辑。

典型流程：

```text
populateBean()
  -> AutowiredAnnotationBeanPostProcessor.postProcessProperties()
     -> InjectionMetadata.inject()
        -> AutowiredFieldElement.inject()
           -> beanFactory.resolveDependency()
              -> doResolveDependency()
                 -> findAutowireCandidates()
                 -> determineAutowireCandidate()
                 -> getBean(candidateName)
```

### 按类型查找

`@Autowired` 默认按类型注入。若只有一个候选 Bean，直接注入。

```java
interface SmsClient {
}

@Component
class AliyunSmsClient implements SmsClient {
}

@Service
class NoticeService {
    NoticeService(SmsClient smsClient) {
    }
}
```

### 多候选处理

如果同一类型有多个 Bean，Spring 会按规则缩小范围：

1. 使用 `@Primary`。
2. 使用 `@Qualifier`。
3. 使用字段名、参数名或 Bean 名匹配。
4. 如果仍不能唯一确定，抛出 `NoUniqueBeanDefinitionException`。

示例：

```java
@Component
@Primary
class AliyunSmsClient implements SmsClient {
}

@Component
class TencentSmsClient implements SmsClient {
}
```

或者：

```java
@Service
class NoticeService {
    NoticeService(@Qualifier("tencentSmsClient") SmsClient smsClient) {
    }
}
```

### 可选依赖

常见写法：

```java
@Autowired(required = false)
private MetricsReporter reporter;
```

更推荐：

```java
private final ObjectProvider<MetricsReporter> reporterProvider;

NoticeService(ObjectProvider<MetricsReporter> reporterProvider) {
    this.reporterProvider = reporterProvider;
}
```

`ObjectProvider` 的好处：

- 延迟获取。
- 不存在时不会直接失败。
- 可以按需遍历多个候选。
- 能减少非必要 Bean 的提前初始化。

## `@Autowired` 与 `@Resource`

### `@Autowired`

- Spring 提供。
- 默认按类型。
- 可配合 `@Qualifier`。
- 支持构造器、字段、方法、参数。
- 可用 `required = false` 表示非必需。

### `@Resource`

- JSR-250 规范注解。
- 默认按名称。
- 找不到名称时再按类型。
- 常用于按 Bean 名精确注入。

对比：

```java
@Autowired
@Qualifier("orderRepository")
private Repository repository;
```

```java
@Resource(name = "orderRepository")
private Repository repository;
```

面试回答要点：`@Autowired` 更偏类型驱动，`@Resource` 更偏名称驱动。

## `@Component` 与 `@Bean`

### `@Component`

用于类级别扫描：

```java
@Component
class RedisClient {
}
```

适合项目内部可直接交给容器管理的类。

### `@Bean`

用于方法级别声明：

```java
@Configuration
class RedisConfig {
    @Bean
    RedisClient redisClient() {
        return new RedisClient("host", 6379);
    }
}
```

适合：

- 第三方类。
- 构造过程复杂的对象。
- 需要按配置显式创建的对象。
- 同一类型创建多个实例。

## `FactoryBean` 与普通 Bean

`FactoryBean<T>` 是一种特殊 Bean，它本身受容器管理，但 `getBean()` 默认拿到的是它生产出来的对象。

```java
class ClientFactoryBean implements FactoryBean<Client> {
    @Override
    public Client getObject() {
        return new Client();
    }

    @Override
    public Class<?> getObjectType() {
        return Client.class;
    }
}
```

获取区别：

- `getBean("clientFactoryBean")`：获取工厂生产的 `Client`。
- `getBean("&clientFactoryBean")`：获取工厂本身。

常见应用：

- MyBatis 的 Mapper 代理对象。
- Dubbo / RPC 客户端代理。
- 一些复杂代理对象或连接客户端。

## Bean 作用域

常见作用域：

- `singleton`：容器中一个 BeanName 对应一个实例，默认作用域。
- `prototype`：每次获取都创建新实例。
- `request`：一次 HTTP 请求一个实例。
- `session`：一次 HTTP Session 一个实例。
- `application`：ServletContext 级别一个实例。

注意：

- singleton 不是 JVM 全局单例，而是一个 Spring 容器内单例。
- prototype Bean 创建后，Spring 不负责完整销毁回调。
- singleton 依赖 prototype 时，prototype 默认只在 singleton 创建时注入一次。

如果 singleton 每次需要新的 prototype，可用：

```java
@Lookup
protected Task newTask() {
    return null;
}
```

或使用 `ObjectProvider<Task>`。

## 底层扩展点

### BeanFactoryPostProcessor

作用于 Bean 实例化之前，修改 BeanDefinition。

典型实现：

- `PropertySourcesPlaceholderConfigurer`：解析占位符。
- `ConfigurationClassPostProcessor`：处理 `@Configuration`、`@Bean`、`@ComponentScan`、`@Import`。

### BeanPostProcessor

作用于 Bean 初始化前后，处理 Bean 实例。

典型实现：

- `AutowiredAnnotationBeanPostProcessor`：处理 `@Autowired`。
- `CommonAnnotationBeanPostProcessor`：处理 `@Resource`、`@PostConstruct`、`@PreDestroy`。
- AOP 相关后置处理器：创建代理对象。

理解区别：

- `BeanFactoryPostProcessor` 面向 BeanDefinition。
- `BeanPostProcessor` 面向 Bean 实例。

## 工程实践

### 优先构造器注入

优点：

- 依赖关系显式。
- 支持 `final` 字段。
- 对象创建后状态完整。
- 单元测试更方便。
- 能更早暴露循环依赖问题。

不推荐字段注入的原因：

- 依赖隐藏。
- 无法声明 `final`。
- 脱离容器时难以构造。
- 容易产生过大的服务类而不自知。

### 避免滥用容器

不建议到处写：

```java
ApplicationContextHolder.getBean(Xxx.class);
```

问题：

- 破坏依赖显式性。
- 增加隐藏耦合。
- 测试困难。
- 容易绕过生命周期和代理边界的清晰设计。

只有在框架集成、动态插件、策略扩展等场景才考虑。

### 面向接口注入

推荐：

```java
class OrderService {
    private final PayGateway payGateway;
}
```

而不是：

```java
class OrderService {
    private final WechatPayGateway wechatPayGateway;
}
```

这样更容易替换实现、mock 测试和做策略扩展。

### 控制 Bean 的数量和边界

不是所有对象都要成为 Bean。适合成为 Bean 的对象通常具备：

- 有共享生命周期。
- 需要依赖注入。
- 需要 AOP、事务、事件、配置等容器能力。
- 是服务、组件、客户端、配置、策略等基础对象。

普通值对象、临时对象、DTO、Command、局部计算对象通常不需要交给 Spring 管理。

## 常见追问

### 1. IoC 和 DI 的区别是什么

IoC 是思想，强调控制权反转；DI 是实现方式，强调容器把依赖注入对象。Spring 通过 BeanFactory / ApplicationContext 管理 Bean 的创建和依赖装配，从而实现 IoC。

### 2. Spring 如何知道要创建哪些 Bean

来源包括：

- XML 配置。
- `@ComponentScan` 扫描。
- `@Bean` 方法。
- `@Import` 导入。
- 自动配置类。
- `ImportBeanDefinitionRegistrar` 编程注册。
- `BeanDefinitionRegistryPostProcessor` 动态注册。

这些来源最终都会注册为 `BeanDefinition`。

### 3. Spring 是先实例化 Bean 还是先注入属性

先实例化，再属性注入。构造器注入发生在实例化阶段；字段和 Setter 注入发生在属性填充阶段。

### 4. 为什么构造器循环依赖不能解决

构造器注入要求依赖在对象实例化前就准备好，A 需要构造 B，B 又需要构造 A，双方都没有可以提前暴露的半成品对象，因此无法通过三级缓存解决。

### 5. `@Autowired` 多个实现如何选择

先按类型找到候选，再按 `@Primary`、`@Qualifier`、Bean 名称或参数名缩小。如果仍不唯一，启动失败。

### 6. `BeanFactoryPostProcessor` 和 `BeanPostProcessor` 区别

前者处理 BeanDefinition，发生在 Bean 实例化之前；后者处理 Bean 实例，发生在 Bean 初始化前后。

### 7. `ApplicationContext.getBean()` 会不会每次创建新对象

取决于作用域。默认 singleton 只创建一次并缓存；prototype 每次获取都会创建新实例。

## 易错点

- 把 IoC 和 DI 说成同一个概念，没有区分思想和实现方式。
- 以为 singleton 是 JVM 级别全局单例，实际是容器级别。
- 以为 `@Autowired` 总是按名称注入，实际默认按类型。
- 以为 BeanDefinition 就是 Bean 实例，实际只是元数据。
- 忽略 `BeanPostProcessor` 在 AOP、注入、初始化注解中的作用。
- 以为所有对象都应该放进 Spring 容器。
- 在业务代码里大量使用 `ApplicationContext.getBean()`，导致依赖隐藏。
- 使用字段注入后，单测和构造约束变差。

## 自检清单

- 能否解释 IoC 是控制权反转，而 DI 是依赖注入实现。
- 能否说清 Spring 先收集 BeanDefinition，再创建 Bean。
- 能否写出 `refresh()` 中和 Bean 创建相关的关键步骤。
- 能否说清 `getBean()` 到 `createBean()` 的大致链路。
- 能否解释构造器注入、Setter 注入、字段注入的取舍。
- 能否说明 `@Autowired` 多候选时的选择规则。
- 能否区分 `BeanFactoryPostProcessor` 和 `BeanPostProcessor`。
- 能否说明 `FactoryBean` 和普通 Bean 的区别。
- 能否说清 singleton、prototype 的生命周期差异。
- 能否结合工程实践解释为什么推荐构造器注入。
