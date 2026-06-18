# 循环依赖解决机制

## 面试定位

循环依赖是 Spring 面试中最容易被追问到底层缓存和代理机制的问题。常见问题包括：

- 什么是循环依赖。
- Spring 能解决哪些循环依赖，不能解决哪些。
- 三级缓存分别存什么。
- 为什么需要三级缓存，两级缓存行不行。
- AOP 代理对象在循环依赖中如何提前暴露。
- Spring Boot 为什么不建议依赖循环依赖。

回答时要明确：Spring 只解决部分 singleton、非构造器注入的循环依赖，不是所有循环依赖。

## 什么是循环依赖

两个或多个 Bean 相互依赖，形成闭环。

最简单的例子：

```java
@Service
class AService {
    @Autowired
    private BService bService;
}

@Service
class BService {
    @Autowired
    private AService aService;
}
```

依赖关系：

```text
AService -> BService -> AService
```

更复杂的闭环：

```text
A -> B -> C -> A
```

## Spring 能解决的场景

通常能解决：

- singleton Bean。
- 属性注入或 Setter 注入。
- 不涉及必须在构造时拿到完整依赖的场景。

示例：

```java
@Component
class A {
    @Autowired
    private B b;
}

@Component
class B {
    @Autowired
    private A a;
}
```

Spring 能先创建 A 的半成品，再创建 B，把 A 的早期引用注入 B，最后回到 A 完成 B 的注入。

## Spring 不能解决的典型场景

### 构造器循环依赖

```java
@Component
class A {
    A(B b) {
    }
}

@Component
class B {
    B(A a) {
    }
}
```

构造器注入要求创建 A 时必须先有 B，创建 B 时又必须先有 A。此时 A 连半成品实例都没有，无法提前暴露。

结果通常是：

```text
BeanCurrentlyInCreationException
```

### prototype 循环依赖

prototype Bean 每次获取都创建新实例，不进入 singleton 三级缓存体系，因此无法通过提前暴露单例引用解决。

### 初始化阶段强依赖完整对象

即使属性注入循环能被容器解开，如果初始化方法中立即调用对方复杂逻辑，也可能出现对象尚未完全初始化导致的问题。

### Spring Boot 默认禁止循环依赖

Spring Boot 2.6 起默认关闭循环依赖支持：

```properties
spring.main.allow-circular-references=false
```

可以改为 true，但工程上不建议依赖这个开关。循环依赖通常说明模块边界或职责设计有问题。

## 三级缓存

Spring 用三级缓存解决 singleton 循环依赖。

核心字段在 `DefaultSingletonBeanRegistry`：

```java
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

### 一级缓存：singletonObjects

保存完整初始化后的 singleton Bean。

特点：

- 成品对象。
- 对外正常提供。
- `getBean()` 优先从这里取。

### 二级缓存：earlySingletonObjects

保存提前暴露的早期 Bean 引用。

特点：

- 还不是完整生命周期结束的 Bean。
- 用于解决循环依赖中的重复获取。
- 可能是原始对象，也可能是早期代理对象。

### 三级缓存：singletonFactories

保存能够生成早期引用的对象工厂。

特点：

- 存的是 `ObjectFactory<?>`，不是 Bean 实例。
- 可以在真正需要早期引用时调用。
- 给 AOP 提供机会提前生成代理。

## 创建流程示例

以 A 属性注入 B，B 属性注入 A 为例。

### 1. 创建 A

```text
getBean("a")
  -> createBean("a")
  -> instantiate A
  -> addSingletonFactory("a", ObjectFactory)
  -> populateBean("a")
```

A 实例化完成后，还没注入 B。Spring 把 A 的对象工厂放入三级缓存。

### 2. A 注入 B，触发创建 B

```text
populateBean("a")
  -> resolveDependency("b")
  -> getBean("b")
```

### 3. 创建 B

```text
createBean("b")
  -> instantiate B
  -> addSingletonFactory("b", ObjectFactory)
  -> populateBean("b")
```

B 实例化后，也提前暴露对象工厂。

### 4. B 注入 A

```text
populateBean("b")
  -> resolveDependency("a")
  -> getBean("a")
  -> getSingleton("a", allowEarlyReference = true)
```

此时 A 正在创建中，一级缓存没有 A，二级缓存也可能没有 A，于是从三级缓存取 A 的 `ObjectFactory`。

调用工厂后：

```text
Object earlyA = singletonFactory.getObject()
earlySingletonObjects.put("a", earlyA)
singletonFactories.remove("a")
```

B 拿到 A 的早期引用，完成注入和初始化。

### 5. 回到 A 完成注入

B 创建完成后进入一级缓存。A 继续注入 B，然后初始化完成，最终 A 也进入一级缓存。

最终：

```text
singletonObjects:
  a -> 完整 A
  b -> 完整 B
```

## 源码链路

关键链路：

```text
AbstractBeanFactory.doGetBean()
  -> DefaultSingletonBeanRegistry.getSingleton(beanName)
  -> AbstractAutowireCapableBeanFactory.createBean()
     -> doCreateBean()
        -> createBeanInstance()
        -> addSingletonFactory()
        -> populateBean()
        -> initializeBean()
        -> addSingleton()
```

提前暴露判断：

```text
earlySingletonExposure =
    mbd.isSingleton()
    && allowCircularReferences
    && isSingletonCurrentlyInCreation(beanName)
```

提前暴露：

```text
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean))
```

获取早期引用：

```text
getSingleton(beanName, allowEarlyReference)
  -> singletonObjects.get(beanName)
  -> earlySingletonObjects.get(beanName)
  -> singletonFactories.get(beanName).getObject()
```

完成后放入一级缓存：

```text
addSingleton(beanName, singletonObject)
  -> singletonObjects.put(beanName, singletonObject)
  -> singletonFactories.remove(beanName)
  -> earlySingletonObjects.remove(beanName)
```

## 为什么需要三级缓存

核心原因：AOP 代理对象需要在“真正发生循环依赖并需要早期引用时”才提前生成。

如果只有一级缓存：

- 只能存完整 Bean，无法解决正在创建中的 Bean。

如果只有一级和二级缓存：

- 可以提前放原始对象。
- 但如果 Bean 最终需要 AOP 代理，其他 Bean 注入的可能是原始对象，而容器最终暴露的是代理对象，导致引用不一致。

三级缓存存的是工厂：

- 不急着生成早期引用。
- 如果没有循环依赖，就不需要提前生成代理。
- 如果发生循环依赖，调用工厂，通过 `getEarlyBeanReference()` 给 AOP 一个返回代理对象的机会。

## AOP 场景下的循环依赖

假设 A 需要事务代理，同时 A 和 B 循环依赖。

如果 B 注入的是 A 的原始对象，而容器最终暴露的是 A 的代理对象，会出现问题：

- B 调用 A 时绕过事务代理。
- 容器中 A 的引用和 B 中 A 的引用不一致。

Spring 通过三级缓存解决：

```text
singletonFactory.getObject()
  -> getEarlyBeanReference()
     -> SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference()
        -> AbstractAutoProxyCreator.getEarlyBeanReference()
           -> wrapIfNecessary()
           -> createProxy()
```

这样 B 注入的是 A 的早期代理对象，而不是原始对象。

## 为什么构造器循环依赖不能解决

构造器注入的问题在于“对象还没实例化出来”。

属性注入可以这样：

```text
实例化 A
  -> 暴露 A 的早期引用
  -> 注入 B
```

构造器注入只能这样：

```text
构造 A 需要 B
构造 B 需要 A
```

A 和 B 都没有半成品对象可以提前暴露，因此无法进入三级缓存解决链路。

## 如何解决业务中的循环依赖

### 拆分职责

如果 A 和 B 互相调用，往往说明职责边界不清。

可抽出第三个服务：

```text
A -> C
B -> C
```

或把某一段逻辑下沉到领域服务 / 协调服务。

### 引入事件

如果一方只是通知另一方，可用事件解耦：

```java
applicationContext.publishEvent(new OrderCreatedEvent(orderId));
```

监听方：

```java
@EventListener
public void onOrderCreated(OrderCreatedEvent event) {
}
```

### 使用延迟依赖

少量不得已场景可用：

```java
private final ObjectProvider<BService> bServiceProvider;
```

或：

```java
@Lazy
@Autowired
private BService bService;
```

但这通常只是缓解，不是设计上的根治。

### 把双向调用改成单向依赖

例如把：

```text
OrderService <-> PayService
```

改成：

```text
OrderApplicationService -> OrderService
OrderApplicationService -> PayService
```

由应用服务编排流程，领域服务保持单向依赖。

## 常见追问

### 1. Spring 通过什么解决循环依赖

通过 singleton 三级缓存提前暴露 Bean 的早期引用，解决部分 singleton 属性注入或 Setter 注入循环依赖。

### 2. 三级缓存分别是什么

- `singletonObjects`：完整单例 Bean。
- `earlySingletonObjects`：提前暴露的早期引用。
- `singletonFactories`：能创建早期引用的 ObjectFactory。

### 3. 为什么不能只用二级缓存

普通属性循环依赖理论上二级缓存可以解决，但涉及 AOP 时需要在真正被提前引用时生成代理对象。三级缓存的 ObjectFactory 给 AOP 提供了延迟生成早期代理的机会，避免提前创建不必要的代理，也避免原始对象和代理对象引用不一致。

### 4. 构造器循环依赖为什么失败

因为构造器注入要求依赖在实例化前准备好，双方都无法先实例化出半成品对象，也就无法提前暴露引用。

### 5. `@Lazy` 为什么能缓解循环依赖

`@Lazy` 注入的通常是代理对象，真实 Bean 延迟到方法调用时再获取。这样可以打破创建时必须立即拿到真实依赖的要求。

### 6. 循环依赖一定要开启支持吗

不建议。循环依赖通常代表设计问题。Spring Boot 默认关闭循环依赖后，更推荐通过拆分职责、事件、编排层等方式消除闭环。

### 7. AOP 代理在循环依赖中如何保证一致

通过三级缓存中的 `ObjectFactory` 调用 `getEarlyBeanReference()`，让 `AbstractAutoProxyCreator` 有机会提前返回代理对象，使其他 Bean 注入的引用和容器最终暴露的引用尽量一致。

## 易错点

- 说 Spring 能解决所有循环依赖。
- 忽略 Spring 主要解决 singleton 属性注入循环依赖。
- 把三级缓存说成三个不同生命周期的 Bean 池，但讲不出每级存什么。
- 只解释普通对象循环依赖，不解释 AOP 为什么需要三级缓存。
- 认为构造器注入也能靠三级缓存解决。
- 认为 `@Lazy` 是根治方案，实际通常只是延迟依赖获取。
- 在工程中靠开启 `allow-circular-references` 掩盖设计问题。

## 自检清单

- 能否给出 A -> B -> A 的创建过程。
- 能否说清三级缓存字段名和含义。
- 能否解释 `addSingletonFactory()` 的时机。
- 能否解释 `getEarlyBeanReference()` 的作用。
- 能否说明为什么构造器循环依赖失败。
- 能否说明 prototype 循环依赖为什么失败。
- 能否解释 AOP 代理和循环依赖的关系。
- 能否给出工程上消除循环依赖的方案。
