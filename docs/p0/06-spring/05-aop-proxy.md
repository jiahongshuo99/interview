# AOP 原理、JDK 动态代理、CGLIB

## 面试定位

AOP 是 Spring 事务、缓存、权限、日志、监控等能力的基础。面试常见问题包括：

- AOP 是什么，解决什么问题。
- Spring AOP 和 AspectJ 有什么区别。
- JDK 动态代理和 CGLIB 的区别。
- Spring 如何选择代理方式。
- 为什么自调用会导致事务失效。
- AOP 代理对象是在 Bean 生命周期哪个阶段创建的。
- 多个切面执行顺序如何控制。

回答时要抓住一句话：Spring AOP 本质上是基于代理对象的方法拦截机制，容器最终暴露的 Bean 可能不是原始对象，而是代理对象。

## AOP 核心概念

### Aspect

切面，表示一组横切逻辑。

```java
@Aspect
@Component
class LogAspect {
}
```

### Join Point

连接点，表示程序执行过程中的某个点。Spring AOP 只支持方法执行连接点。

### Pointcut

切点，用于匹配哪些连接点需要增强。

```java
@Pointcut("execution(* com.example.service..*(..))")
public void serviceMethods() {
}
```

### Advice

通知，即增强逻辑。

常见通知：

- `@Before`：方法执行前。
- `@After`：方法结束后，无论是否异常。
- `@AfterReturning`：方法正常返回后。
- `@AfterThrowing`：方法抛异常后。
- `@Around`：环绕通知，能力最强。

### Advisor

Spring 内部更常用 Advisor 表达增强器，它通常由 Pointcut 和 Advice 组成。

事务、缓存等底层也是通过 Advisor / Interceptor 机制接入。

## Spring AOP 的本质

Spring AOP 基于代理：

```text
调用方 -> 代理对象 -> 拦截器链 -> 目标对象方法
```

如果目标方法匹配切点，代理对象会执行增强逻辑；否则直接调用目标方法。

简化模型：

```java
class ProxyOrderService implements OrderService {
    private final OrderService target;
    private final List<MethodInterceptor> interceptors;

    public void createOrder() {
        // before
        target.createOrder();
        // after
    }
}
```

Spring 并不会修改目标类源码。它是在运行期创建代理对象，并把代理对象放入容器。

## Spring AOP 与 AspectJ

| 维度 | Spring AOP | AspectJ |
| --- | --- | --- |
| 实现方式 | 运行期代理 | 编译期、类加载期或运行期织入 |
| 支持连接点 | 主要是方法执行 | 字段、构造器、方法调用等更丰富 |
| 使用复杂度 | 低，和 Spring 容器集成 | 较高 |
| 适用场景 | 事务、缓存、权限、日志等常规业务增强 | 更底层、更广泛的织入需求 |
| 是否要求 Spring Bean | 通常要求目标对象是 Spring Bean | 不一定 |

多数 Java 后端业务系统使用 Spring AOP 已足够。

## JDK 动态代理

JDK 动态代理基于接口。

目标对象必须实现接口：

```java
interface PayService {
    void pay();
}

class PayServiceImpl implements PayService {
    public void pay() {
    }
}
```

核心 API：

```java
Proxy.newProxyInstance(
    classLoader,
    interfaces,
    invocationHandler
);
```

调用流程：

```text
客户端调用接口方法
  -> 代理对象
  -> InvocationHandler.invoke()
  -> 反射调用目标对象方法
```

特点：

- 只能代理接口方法。
- 不需要额外字节码库。
- 代理对象类型是接口的实现类，不是目标类子类。
- 不能代理没有接口的类。

## CGLIB 代理

CGLIB 基于继承生成目标类子类。

简化理解：

```java
class PayService$$EnhancerBySpringCGLIB extends PayService {
    @Override
    public void pay() {
        // advice
        super.pay();
    }
}
```

特点：

- 不要求目标类实现接口。
- 通过生成子类覆盖方法实现增强。
- 不能代理 `final` 类。
- 不能增强 `final` 方法。
- 对 `private` 方法无能为力，因为无法覆盖。

Spring Framework 4 之后 CGLIB 被重新打包进 Spring Core，不需要业务额外引入 CGLIB 依赖。

## Spring 如何选择代理方式

常见规则：

- 如果目标类实现了接口，默认优先使用 JDK 动态代理。
- 如果目标类没有接口，使用 CGLIB。
- 如果配置强制使用 class-based proxy，则使用 CGLIB。

常见配置：

```properties
spring.aop.proxy-target-class=true
```

在 Spring Boot 中，通常默认倾向使用 CGLIB 类代理。实际面试回答可以概括为：Spring 能根据目标对象是否有接口和配置决定使用 JDK 动态代理或 CGLIB，强制 class 代理时使用 CGLIB。

## AOP 代理创建时机

AOP 代理通常在 Bean 初始化后由 `BeanPostProcessor` 创建。

关键类型：

```text
AbstractAutoProxyCreator
  -> BeanPostProcessor
  -> SmartInstantiationAwareBeanPostProcessor
```

典型链路：

```text
Bean 创建
  -> 实例化
  -> 属性注入
  -> 初始化前 BeanPostProcessor
  -> 初始化方法
  -> 初始化后 BeanPostProcessor
     -> AbstractAutoProxyCreator.postProcessAfterInitialization()
        -> wrapIfNecessary()
        -> createProxy()
  -> 单例池保存代理对象
```

循环依赖场景下，代理可能通过三级缓存提前暴露：

```text
getEarlyBeanReference()
  -> wrapIfNecessary()
  -> createProxy()
```

## AOP 调用链

Spring AOP 方法调用链大致如下：

```text
代理对象方法
  -> JdkDynamicAopProxy.invoke()
     或 CglibAopProxy.DynamicAdvisedInterceptor.intercept()
  -> 获取目标方法匹配的拦截器链
  -> ReflectiveMethodInvocation.proceed()
     -> MethodInterceptor.invoke()
     -> 下一个 MethodInterceptor
     -> 目标方法
```

环绕通知本质上也是一个 `MethodInterceptor`。

```java
@Around("serviceMethods()")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.currentTimeMillis();
    try {
        return pjp.proceed();
    } finally {
        log.info("cost={}", System.currentTimeMillis() - start);
    }
}
```

`pjp.proceed()` 表示继续执行后续拦截器或目标方法。

如果不调用 `proceed()`，目标方法不会执行。

## 切面执行顺序

多个切面可通过 `@Order` 或实现 `Ordered` 控制顺序。

```java
@Aspect
@Order(1)
@Component
class TraceAspect {
}

@Aspect
@Order(2)
@Component
class LogAspect {
}
```

数字越小优先级越高。

环绕通知执行像洋葱模型：

```text
Trace before
  Log before
    target
  Log after
Trace after
```

## 自调用为什么失效

示例：

```java
@Service
class OrderService {
    public void create() {
        this.save();
    }

    @Transactional
    public void save() {
    }
}
```

外部调用时：

```text
调用方 -> 代理对象 -> save() -> 事务拦截器 -> 目标方法
```

自调用时：

```text
目标对象 create() -> this.save()
```

`this` 是目标对象本身，不是代理对象，因此不会经过代理拦截器链，事务、缓存、异步等基于 AOP 的能力都会失效。

解决方式：

- 把被增强方法拆到另一个 Bean。
- 从代理对象调用，而不是 `this` 调用。
- 使用 `AopContext.currentProxy()`，但需要开启暴露代理，不推荐作为常规方案。
- 调整业务边界，让事务方法成为外部入口。

## 哪些方法无法被 Spring AOP 增强

常见限制：

- 非 Spring 容器管理对象的方法。
- 同类内部自调用的方法。
- `private` 方法。
- `final` 方法。
- `static` 方法。
- 构造器。
- JDK 动态代理下接口中不存在的方法。
- CGLIB 下无法覆盖的方法。

事务失效、缓存失效、异步失效很多都和这些限制有关。

## 事务 AOP 的简化链路

`@Transactional` 也是 AOP。

简化流程：

```text
@EnableTransactionManagement
  -> 注册 AutoProxyCreator
  -> 注册 TransactionAttributeSourceAdvisor
  -> 匹配 @Transactional 方法
  -> 代理对象调用
  -> TransactionInterceptor.invoke()
     -> 开启事务
     -> 执行业务方法
     -> 提交或回滚事务
```

关键类型：

- `TransactionInterceptor`。
- `TransactionAttributeSource`。
- `PlatformTransactionManager`。
- `TransactionAttributeSourceAdvisor`。

## 工程实践

### AOP 适合做什么

适合横切能力：

- 事务。
- 缓存。
- 权限校验。
- 审计日志。
- 链路追踪。
- 指标监控。
- 幂等控制。
- 限流降级入口包装。

不适合：

- 复杂业务主流程。
- 依赖执行顺序很强的核心逻辑。
- 需要清晰显式调用关系的领域规则。

### 切点要收敛

不要写过宽切点：

```java
execution(* com.example..*(..))
```

更推荐：

- 限定包。
- 限定注解。
- 限定方法命名。
- 限定类名后缀。

### 环绕通知要正确处理异常

错误示例：

```java
try {
    return pjp.proceed();
} catch (Exception e) {
    log.error("error", e);
    return null;
}
```

问题：

- 吞异常。
- 影响事务回滚。
- 调用方无法感知失败。

更合理：

```java
try {
    return pjp.proceed();
} catch (Throwable e) {
    log.error("error", e);
    throw e;
}
```

## 常见追问

### 1. Spring AOP 是编译期织入还是运行期代理

Spring AOP 主要是运行期代理。它为 Spring Bean 创建代理对象，调用方调用代理对象时进入拦截器链。

### 2. JDK 动态代理和 CGLIB 有什么区别

JDK 动态代理基于接口，代理对象实现接口；CGLIB 基于继承，生成目标类子类。JDK 不能代理没有接口的类，CGLIB 不能代理 final 类和 final 方法。

### 3. Spring AOP 为什么要求对象是 Spring Bean

因为代理对象由 Spring 容器在 Bean 创建生命周期中生成并放入容器。自己 new 出来的对象不经过容器，也就不会被代理。

### 4. 为什么 `private` 方法不能被增强

Spring AOP 基于代理和方法拦截。`private` 方法不能被子类覆盖，也不会作为接口方法暴露，代理无法拦截它。

### 5. 为什么自调用会失效

自调用走的是目标对象内部的 `this` 调用，不经过代理对象，因此不会进入 AOP 拦截器链。

### 6. 多个切面如何排序

通过 `@Order` 或 `Ordered`。数字越小优先级越高。环绕通知按优先级进入，再按相反顺序退出。

### 7. AOP 代理和原始对象是什么关系

容器最终暴露的通常是代理对象，代理对象内部持有目标对象。调用代理方法时先执行拦截器链，再调用目标对象方法。

## 易错点

- 以为 Spring AOP 会修改源码或字节码，实际主要是运行期代理。
- 以为有注解就一定生效，忽略必须经过代理调用。
- 以为 CGLIB 可以代理 final 方法。
- 以为 private 方法加 `@Transactional` 能生效。
- 在环绕通知里吞异常，导致事务无法回滚。
- 切点范围过大，造成意外增强和性能问题。
- 忘记 JDK 动态代理下只能通过接口方法增强。

## 自检清单

- 能否解释 Spring AOP 的代理本质。
- 能否区分 Aspect、Pointcut、Advice、Advisor。
- 能否说清 JDK 动态代理和 CGLIB 区别。
- 能否说明 Spring 如何选择代理方式。
- 能否说出 AOP 代理创建的大致生命周期阶段。
- 能否解释自调用失效。
- 能否说明事务为什么也是 AOP。
- 能否说明哪些方法无法被 Spring AOP 增强。
- 能否写出一个正确的 `@Around` 通知。
