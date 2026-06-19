# Bean 生命周期

## 面试定位

Bean 生命周期是 Spring 面试高频问题。面试官通常希望你不只是背“实例化、属性注入、初始化、销毁”，而是能讲出：

- BeanDefinition 到 Bean 实例的转换过程。
- 各类扩展点的执行顺序。
- Aware、`BeanPostProcessor`、初始化方法、销毁方法分别在什么阶段执行。
- AOP 代理对象在哪个阶段产生。
- singleton 和 prototype 生命周期有什么差异。

一个完整回答要覆盖“容器启动阶段”和“单个 Bean 创建阶段”。

## 总览

典型 singleton Bean 生命周期：

```text
加载配置 / 扫描类
  -> 注册 BeanDefinition
  -> BeanFactoryPostProcessor 修改 BeanDefinition
  -> 实例化 Bean
  -> 属性填充 / 依赖注入
  -> Aware 回调
  -> BeanPostProcessor 前置处理
  -> 初始化回调
  -> BeanPostProcessor 后置处理
  -> 放入单例池，对外提供服务
  -> 容器关闭
  -> 销毁回调
```

更贴近源码的链路：

```text
ApplicationContext.refresh()
  -> invokeBeanFactoryPostProcessors()
  -> registerBeanPostProcessors()
  -> finishBeanFactoryInitialization()
     -> DefaultListableBeanFactory.preInstantiateSingletons()
        -> getBean()
           -> doGetBean()
              -> createBean()
                 -> doCreateBean()
                    -> createBeanInstance()
                    -> populateBean()
                    -> initializeBean()
                    -> registerDisposableBeanIfNecessary()
```

## BeanDefinition 阶段

Bean 不是一开始就被创建。Spring 会先解析配置并注册 `BeanDefinition`。

来源包括：

- XML。
- `@Component` / `@Service` / `@Repository` / `@Controller`。
- `@Bean` 方法。
- `@Import`。
- Spring Boot 自动配置。
- 手动注册。

`BeanDefinition` 保存 Bean 的元信息：

- Bean class。
- scope。
- lazy-init。
- primary。
- init method。
- destroy method。
- depends-on。
- 构造参数。
- 属性值。

面试重点：`BeanDefinition` 阶段还没有业务对象实例，主要是元数据注册和调整。

## BeanFactoryPostProcessor 阶段

`BeanFactoryPostProcessor` 在 Bean 实例化前执行，用于修改 BeanDefinition 或 BeanFactory 配置。

典型接口：

```java
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory);
}
```

常见实现：

- `ConfigurationClassPostProcessor`：解析 `@Configuration`、`@ComponentScan`、`@Import`、`@Bean`。
- `PropertySourcesPlaceholderConfigurer`：处理 `${}` 占位符。
- 自定义处理器：动态修改 BeanDefinition。

如果实现 `BeanDefinitionRegistryPostProcessor`，还可以在更早阶段注册新的 BeanDefinition。

关键区别：

- `BeanFactoryPostProcessor` 操作的是 BeanDefinition。
- `BeanPostProcessor` 操作的是 Bean 实例。

## Bean 实例化

实例化是创建对象本身，但还没有完成属性注入。

常见实例化方式：

- 构造器。
- 静态工厂方法。
- 实例工厂方法。
- Supplier。
- FactoryBean 间接创建。

源码入口：

```text
AbstractAutowireCapableBeanFactory.createBeanInstance()
```

构造器选择逻辑大致包括：

- 是否存在显式构造参数。
- 是否有 `@Autowired` 构造器。
- 是否只有一个构造器。
- 是否需要自动装配构造参数。
- 是否可以使用默认无参构造器。

构造器注入发生在实例化阶段：

```java
@Service
class OrderService {
    private final UserService userService;

    OrderService(UserService userService) {
        this.userService = userService;
    }
}
```

## 早期暴露阶段

对 singleton Bean，Spring 可能在实例化后、属性填充前，把一个对象工厂提前放入三级缓存。

关键代码位置在 `doCreateBean()` 附近：

```text
createBeanInstance()
  -> addSingletonFactory(beanName, () -> getEarlyBeanReference(...))
  -> populateBean()
```

早期暴露服务于循环依赖：

- 已经实例化但尚未完成初始化的对象可以被其他 Bean 引用。
- 如果存在 AOP，早期暴露可能返回代理对象。

注意：早期暴露只适用于部分 singleton 场景，不是所有循环依赖都能解决。

## 属性填充 / 依赖注入

属性填充阶段对应：

```text
AbstractAutowireCapableBeanFactory.populateBean()
```

这一阶段完成：

- `@Autowired` 字段注入。
- `@Autowired` 方法注入。
- `@Resource` 注入。
- XML property 注入。
- Setter 注入。

相关后置处理器：

- `AutowiredAnnotationBeanPostProcessor`：处理 `@Autowired`、`@Value`。
- `CommonAnnotationBeanPostProcessor`：处理 `@Resource`。

典型链路：

```text
populateBean()
  -> InstantiationAwareBeanPostProcessor.postProcessProperties()
     -> AutowiredAnnotationBeanPostProcessor.postProcessProperties()
        -> InjectionMetadata.inject()
```

## Aware 回调

属性注入后，初始化前，Spring 会处理部分 Aware 接口。

常见 Aware：

- `BeanNameAware`：注入 BeanName。
- `BeanClassLoaderAware`：注入 ClassLoader。
- `BeanFactoryAware`：注入 BeanFactory。
- `ApplicationContextAware`：注入 ApplicationContext。
- `EnvironmentAware`：注入 Environment。
- `ResourceLoaderAware`：注入 ResourceLoader。
- `ApplicationEventPublisherAware`：注入事件发布器。

示例：

```java
@Component
class DemoBean implements BeanNameAware, ApplicationContextAware {
    @Override
    public void setBeanName(String name) {
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
    }
}
```

面试中可以说明：Aware 是 Spring 把容器基础设施对象回调给 Bean 的机制，但业务代码不应滥用，否则会和 Spring 强耦合。

## BeanPostProcessor 前置处理

初始化方法执行前，会调用：

```java
postProcessBeforeInitialization(Object bean, String beanName)
```

入口：

```text
initializeBean()
  -> applyBeanPostProcessorsBeforeInitialization()
```

常见作用：

- 处理 `@PostConstruct`。
- 做初始化前的统一校验。
- 注入框架内部能力。

`@PostConstruct` 实际由 `CommonAnnotationBeanPostProcessor` 处理。

## 初始化回调

初始化回调常见三种：

1. `@PostConstruct`。
2. `InitializingBean.afterPropertiesSet()`。
3. 自定义 `initMethod`。

执行顺序通常是：

```text
@PostConstruct
  -> InitializingBean.afterPropertiesSet()
  -> custom init-method
```

示例：

```java
@Component
class CacheWarmUpBean implements InitializingBean {
    @PostConstruct
    public void postConstruct() {
    }

    @Override
    public void afterPropertiesSet() {
    }
}
```

`@Bean` 指定初始化方法：

```java
@Bean(initMethod = "init")
Client client() {
    return new Client();
}
```

实践建议：

- 简单初始化可用 `@PostConstruct`。
- 需要避免和 JSR 注解耦合时可用 `InitializingBean` 或 init method。
- 不要在初始化方法中做耗时过长且不可控的远程调用，可能拖慢应用启动。

## BeanPostProcessor 后置处理

初始化方法执行后，会调用：

```java
postProcessAfterInitialization(Object bean, String beanName)
```

入口：

```text
initializeBean()
  -> applyBeanPostProcessorsAfterInitialization()
```

AOP 代理常在这一阶段返回代理对象：

```text
AbstractAutoProxyCreator.postProcessAfterInitialization()
  -> wrapIfNecessary()
  -> createProxy()
```

这也是为什么有时容器中最终暴露的是代理对象，而不是原始对象。

## 单例池缓存

singleton Bean 初始化完成后，会放入单例池。

核心缓存：

- `singletonObjects`：一级缓存，保存完整单例 Bean。
- `earlySingletonObjects`：二级缓存，保存早期引用。
- `singletonFactories`：三级缓存，保存早期对象工厂。

完整对象最终会进入一级缓存，对外 `getBean()` 优先从一级缓存返回。

## SmartInitializingSingleton

所有非懒加载 singleton Bean 创建完成后，会回调：

```java
public interface SmartInitializingSingleton {
    void afterSingletonsInstantiated();
}
```

适合做“依赖所有单例 Bean 已准备好”的动作，例如：

- 启动后注册。
- 汇总策略 Bean。
- 做轻量级一致性校验。

它和普通 init 方法的区别：

- init 方法是单个 Bean 初始化完成后执行。
- `afterSingletonsInstantiated()` 是所有单例 Bean 初始化完成后执行。

## 容器刷新完成后的回调

Spring Boot 中还常见：

- `ApplicationRunner`。
- `CommandLineRunner`。
- `ApplicationReadyEvent`。

这些不是普通 Bean 初始化生命周期的一部分，而是应用启动流程中的后续回调。

执行时机大致更靠后：

```text
Bean 初始化完成
  -> ApplicationContext refresh 完成
  -> SpringApplication runners
  -> ApplicationReadyEvent
```

面试时要避免把它们和 Bean 初始化方法混在一起。

## 销毁阶段

容器关闭时，singleton Bean 会执行销毁逻辑。

常见销毁方式：

1. `@PreDestroy`。
2. `DisposableBean.destroy()`。
3. 自定义 destroy method。

示例：

```java
@Bean(destroyMethod = "close")
Client client() {
    return new Client();
}
```

执行顺序通常是：

```text
@PreDestroy
  -> DisposableBean.destroy()
  -> custom destroy-method
```

注意：

- prototype Bean 创建后，容器通常不负责完整销毁。
- Web 作用域 Bean 的销毁跟对应 request / session 生命周期相关。
- JVM 异常退出、`kill -9` 等情况下销毁回调不一定执行。

## singleton 与 prototype 生命周期区别

### singleton

容器负责：

- 创建。
- 依赖注入。
- 初始化。
- 缓存。
- 销毁。

### prototype

容器负责：

- 创建。
- 依赖注入。
- 初始化。

通常不负责：

- 后续状态管理。
- 完整销毁回调。

因此 prototype Bean 持有外部资源时，需要调用方自己释放，或避免使用 prototype 承载资源。

## AOP 与生命周期

AOP 代理对象通常由 `BeanPostProcessor` 创建。关键点：

- 原始对象先创建、注入、初始化。
- 后置处理器判断是否需要代理。
- 需要代理时创建代理对象并返回。
- 单例池最终保存的可能是代理对象。

循环依赖中如果涉及 AOP，Spring 可能通过 `getEarlyBeanReference()` 提前暴露代理对象，避免其他 Bean 注入原始对象而最终容器暴露代理对象导致不一致。

## 常见追问

### 1. `@PostConstruct` 在什么时候执行

在属性注入和 Aware 回调之后，在 `InitializingBean.afterPropertiesSet()` 和自定义 init method 之前。它由 `CommonAnnotationBeanPostProcessor` 在初始化前置处理阶段触发。

### 2. AOP 代理在哪一步生成

多数情况下在 `BeanPostProcessor.postProcessAfterInitialization()` 阶段生成，典型实现是 `AbstractAutoProxyCreator`。循环依赖场景下可能通过三级缓存的 `getEarlyBeanReference()` 提前暴露代理。

### 3. Bean 生命周期中 BeanPostProcessor 执行几次

至少有初始化前和初始化后两类回调：

- `postProcessBeforeInitialization()`。
- `postProcessAfterInitialization()`。

此外还有更早的 `InstantiationAwareBeanPostProcessor`、`SmartInstantiationAwareBeanPostProcessor` 等扩展点参与实例化和属性填充过程。

### 4. BeanFactoryPostProcessor 和 BeanPostProcessor 谁先执行

`BeanFactoryPostProcessor` 先执行，因为它要在 Bean 实例化前修改 BeanDefinition。`BeanPostProcessor` 后注册，并在 Bean 实例创建过程中处理实例。

### 5. prototype Bean 会执行销毁方法吗

通常不会由容器自动完整销毁。Spring 创建并初始化 prototype Bean 后就交给调用方，后续销毁责任一般由调用方承担。

### 6. `afterPropertiesSet()` 和 `@PostConstruct` 的区别

`@PostConstruct` 是 JSR 注解，由后置处理器识别；`afterPropertiesSet()` 是 Spring 的 `InitializingBean` 接口方法。执行顺序上 `@PostConstruct` 通常早于 `afterPropertiesSet()`。

## 工程实践

- 初始化逻辑保持轻量，不要把长时间远程调用放在 Bean 初始化中。
- 资源关闭逻辑要放在销毁方法中，例如线程池、连接、客户端。
- 业务代码尽量不要实现太多 Aware 接口，避免和容器强耦合。
- 单例 Bean 中不要保存请求级别、用户级别的可变状态。
- prototype Bean 不适合作为需要容器统一销毁的资源承载者。
- 生命周期扩展点要有明确边界，避免在 `BeanPostProcessor` 中做复杂业务逻辑。

## 易错点

- 把 BeanDefinition 当成 Bean 实例。
- 以为属性注入发生在构造器之前。
- 忽略构造器注入其实属于实例化阶段。
- 以为 `@PostConstruct` 是 Java 自动调用，实际由 Spring 后置处理器处理。
- 以为所有 Bean 都会在容器启动时创建，实际懒加载和 prototype 不一定。
- 以为 prototype Bean 会被容器完整管理销毁。
- 以为 AOP 代理对象在实例化阶段就生成，实际多数在初始化后生成。
- 把 ApplicationRunner、CommandLineRunner 当作普通 Bean 初始化回调。

## 自检清单

- 能否按顺序说出 Bean 生命周期主流程。
- 能否区分 BeanDefinition 阶段和 Bean 实例阶段。
- 能否说明 `BeanFactoryPostProcessor` 的执行时机。
- 能否说明构造器注入、属性注入分别发生在哪个阶段。
- 能否说明 Aware 回调的作用。
- 能否说出 `@PostConstruct`、`afterPropertiesSet()`、init method 的顺序。
- 能否解释 AOP 代理在哪个阶段产生。
- 能否说清 singleton 和 prototype 生命周期差异。
- 能否说明销毁回调的触发条件和局限。
- 能否结合源码链路说出 `doCreateBean()` 的关键步骤。
