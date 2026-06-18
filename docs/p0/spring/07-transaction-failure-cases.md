# Spring 事务失效场景：自调用、异常类型、非 public 方法、异步线程

## 面试定位

Spring 事务失效是 Java 后端面试高频题。面试官通常会给一段 `@Transactional` 代码，问为什么没有回滚、为什么事务没有开启、为什么异步方法不受事务控制。

核心判断标准：

- 是否由 Spring 容器管理。
- 是否通过代理对象调用。
- 方法是否能被代理增强。
- 异常是否满足回滚规则。
- 是否跨线程。
- 数据库和事务管理器是否真的支持事务。

## 总体原理

`@Transactional` 基于 Spring AOP。

典型调用链：

```text
调用方
  -> 事务代理对象
     -> TransactionInterceptor
        -> 开启或加入事务
        -> 调用目标方法
        -> 根据异常提交或回滚
```

只要调用没有经过代理对象，或者代理无法拦截目标方法，事务就可能失效。

## 场景一：自调用失效

### 问题代码

```java
@Service
class OrderService {
    public void createOrder() {
        saveOrder();
    }

    @Transactional
    public void saveOrder() {
        orderRepository.save(order);
    }
}
```

`createOrder()` 调用 `saveOrder()` 是同类内部调用，本质是：

```java
this.saveOrder();
```

这里的 `this` 是目标对象，不是 Spring 代理对象，因此不会进入 `TransactionInterceptor`。

### 正确理解

外部调用代理：

```text
Controller -> OrderServiceProxy.saveOrder() -> TransactionInterceptor -> target.saveOrder()
```

自调用：

```text
target.createOrder() -> this.saveOrder()
```

后者绕过代理，事务失效。

### 解决方案

推荐拆分到另一个 Bean：

```java
@Service
class OrderApplicationService {
    private final OrderTxService orderTxService;

    OrderApplicationService(OrderTxService orderTxService) {
        this.orderTxService = orderTxService;
    }

    public void createOrder() {
        orderTxService.saveOrder();
    }
}

@Service
class OrderTxService {
    @Transactional
    public void saveOrder() {
    }
}
```

其他方案：

- 注入自身代理对象后调用代理方法。
- 使用 `AopContext.currentProxy()`，需要开启 `exposeProxy`，不推荐常规使用。
- 调整事务边界，把 `@Transactional` 放到外部入口方法。

## 场景二：异常类型不匹配

### 默认回滚规则

默认回滚：

- `RuntimeException`。
- `Error`。

默认不回滚：

- 受检异常 `Exception`。

### 问题代码

```java
@Transactional
public void importData() throws IOException {
    repository.save(data);
    throw new IOException("import failed");
}
```

`IOException` 是受检异常，默认不会触发回滚。

### 解决方案

显式指定：

```java
@Transactional(rollbackFor = Exception.class)
public void importData() throws IOException {
    repository.save(data);
    throw new IOException("import failed");
}
```

也可以针对具体异常：

```java
@Transactional(rollbackFor = IOException.class)
```

### 易错变体：吞异常

```java
@Transactional
public void submit() {
    try {
        repository.save(data);
        remoteCall();
    } catch (Exception e) {
        log.error("submit failed", e);
    }
}
```

异常被 catch 后没有继续抛出，事务拦截器认为方法正常返回，于是提交事务。

解决：

```java
catch (Exception e) {
    log.error("submit failed", e);
    throw e;
}
```

或手动标记回滚：

```java
TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
```

但手动标记回滚会让业务代码侵入 Spring 事务 API，需谨慎使用。

## 场景三：非 public 方法

### 问题代码

```java
@Service
class OrderService {
    @Transactional
    private void saveOrder() {
    }
}
```

或：

```java
@Transactional
protected void saveOrder() {
}
```

传统 Spring AOP 代理模型下，事务注解通常应放在 `public` 方法上。非 public 方法可能无法被代理正确拦截。

原因：

- JDK 动态代理基于接口方法，接口方法通常是 public。
- CGLIB 基于子类覆盖，`private` 方法无法覆盖。
- Spring 的事务属性解析和代理调用入口通常面向可被外部调用的业务方法。

实践建议：

- 事务方法使用 `public`。
- 事务边界放在 service 层外部可调用方法上。
- 不要给 `private` 方法加 `@Transactional` 期望生效。

## 场景四：final 方法或 final 类

CGLIB 通过生成子类覆盖方法实现增强，因此：

```java
@Service
final class OrderService {
    @Transactional
    public void save() {
    }
}
```

final 类不能被继承，CGLIB 无法代理。

```java
@Transactional
public final void save() {
}
```

final 方法不能被覆盖，CGLIB 无法增强。

如果使用 JDK 动态代理，调用接口方法时仍可能生效，但工程上不要依赖这种模糊行为。

实践建议：

- 需要 AOP 增强的 Spring Bean 不要设计成 final 类。
- 需要事务增强的方法不要声明 final。

## 场景五：方法不属于 Spring Bean

### 问题代码

```java
class OrderService {
    @Transactional
    public void save() {
    }
}

OrderService service = new OrderService();
service.save();
```

对象是自己 new 的，没有经过 Spring 容器生命周期，也不会被创建为代理对象。

解决：

- 使用 `@Service` 等注解交给容器管理。
- 从容器注入该 Bean。
- 不要手动 new 需要事务增强的服务对象。

## 场景六：异步线程导致事务上下文丢失

Spring 事务上下文通常绑定在线程上，底层使用 `TransactionSynchronizationManager` 和 `ThreadLocal`。

### 问题代码

```java
@Transactional
public void createOrder() {
    orderRepository.save(order);

    new Thread(() -> {
        itemRepository.save(item);
        throw new RuntimeException();
    }).start();
}
```

新线程不会继承当前线程的事务上下文。异步线程中的数据库操作不属于外层事务。

### `@Async` 场景

```java
@Transactional
public void createOrder() {
    asyncService.saveItem();
}

@Async
public void saveItem() {
    itemRepository.save(item);
}
```

`@Async` 在线程池中执行，默认和外层事务不是同一个事务。

如果异步方法也标注事务：

```java
@Async
@Transactional
public void saveItem() {
}
```

它会在异步线程中开启自己的事务，但不是外层事务的一部分。

实践建议：

- 不要期望事务自动跨线程传播。
- 异步任务需要独立事务时，在异步方法上显式声明事务。
- 强一致操作不要拆到异步线程里。
- 跨线程一致性更适合用消息、状态机、补偿机制。

## 场景七：`@Async` 自调用也会失效

```java
@Service
class OrderService {
    public void create() {
        asyncSave();
    }

    @Async
    @Transactional
    public void asyncSave() {
    }
}
```

自调用绕过代理，`@Async` 和 `@Transactional` 都可能不生效。

解决方式和事务自调用类似：拆到另一个 Bean，通过代理调用。

## 场景八：数据库引擎不支持事务

例如 MySQL MyISAM 不支持事务。即使 Spring 开启事务，数据库层也无法回滚。

实践中要确认：

- 表引擎是否支持事务，例如 InnoDB。
- 数据源是否正确。
- SQL 是否经过同一个事务管理器管理。
- 是否有隐式提交语句。

## 场景九：多数据源事务管理器用错

多数据源项目中，如果事务管理器绑定错数据源，会导致事务没有覆盖实际 SQL。

示例：

```java
@Transactional(transactionManager = "orderTransactionManager")
public void updateUser() {
    userMapper.update(user);
}
```

如果 `userMapper` 实际走的是 user 数据源，而注解指定 order 事务管理器，就可能不生效。

实践建议：

- 多数据源明确指定事务管理器。
- Repository / Mapper 和 transactionManager 绑定关系清晰。
- 避免一个方法中随意跨多个本地事务数据源。

## 场景十：事务边界内使用错误连接

如果手动使用 JDBC：

```java
Connection conn = dataSource.getConnection();
```

而不是通过 Spring 管理的模板或工具，可能拿到不受当前事务管理的连接。

推荐：

- 使用 `JdbcTemplate`。
- 使用 MyBatis / JPA 等和 Spring 集成的访问方式。
- 必要时使用 `DataSourceUtils.getConnection(dataSource)`。

## 场景十一：传播行为理解错误

内层 `REQUIRES_NEW` 已提交后，外层回滚不会回滚内层：

```java
@Transactional
public void outer() {
    auditService.saveAuditRequiresNew();
    throw new RuntimeException();
}
```

如果 `saveAuditRequiresNew()` 使用 `REQUIRES_NEW` 并成功提交，外层回滚不会撤销审计记录。

这不是事务失效，而是传播行为本来如此。

## 场景十二：事务过早提交

如果事务方法返回后才执行逻辑，就不在事务内。

常见变体：

- 注册异步回调。
- 返回 Stream 后在外部消费。
- 延迟执行 Supplier。
- 提交线程池任务。

示例：

```java
@Transactional
public Supplier<Void> createTask() {
    return () -> {
        repository.save(data);
        return null;
    };
}
```

`Supplier` 真正执行时事务方法早已返回，不受原事务控制。

## 源码链路理解

事务生效入口：

```text
代理对象方法
  -> TransactionInterceptor.invoke()
     -> TransactionAspectSupport.invokeWithinTransaction()
        -> createTransactionIfNecessary()
        -> invocation.proceedWithInvocation()
        -> commitTransactionAfterReturning()
        -> completeTransactionAfterThrowing()
```

异常回滚判断：

```text
TransactionAttribute.rollbackOn(Throwable ex)
```

事务上下文绑定：

```text
TransactionSynchronizationManager
  -> ThreadLocal resources
  -> ThreadLocal synchronizations
  -> ThreadLocal transactionName
  -> ThreadLocal readOnly
```

所以事务通常不能自动跨线程传播。

## 工程实践

- 事务注解放在 service 层 public 方法上。
- 事务方法必须通过 Spring Bean 代理对象调用。
- 默认写 `rollbackFor = Exception.class` 要结合团队规范，不要机械套用。
- 不在事务内做耗时 RPC、MQ 同步等待、大文件 IO。
- 异步逻辑如果要落库，要显式设计独立事务和补偿。
- 多数据源要显式确认 transactionManager。
- 捕获异常后如果需要回滚，要重新抛出或标记 rollback-only。
- 事务方法中不要调用同类内部的事务方法来切分事务边界。

## 常见追问

### 1. `@Transactional` 为什么自调用失效

因为同类内部 `this.method()` 调用的是目标对象本身，不经过 Spring 代理对象，也就不会进入 `TransactionInterceptor`。

### 2. 为什么抛了 Exception 没回滚

Spring 默认只对 `RuntimeException` 和 `Error` 回滚，受检异常默认不回滚。需要配置 `rollbackFor`。

### 3. catch 异常后为什么没回滚

事务拦截器只看到方法正常返回，就会提交事务。需要重新抛出异常或手动标记 rollback-only。

### 4. private 方法上的事务为什么不生效

Spring AOP 基于代理，private 方法不能被接口代理暴露，也不能被 CGLIB 子类覆盖，因此无法作为代理增强入口。

### 5. 异步线程能不能复用外层事务

默认不能。Spring 事务上下文基于 ThreadLocal，异步线程和原线程不是同一个事务上下文。

### 6. `@Async` 和 `@Transactional` 同时使用要注意什么

异步方法通过代理进入时，可以在异步线程开启自己的事务；但它不是调用方线程的外层事务。若同类自调用，两个注解都可能失效。

### 7. 事务没有回滚一定是 Spring 失效吗

不一定。可能是传播行为符合预期、异常被吞、数据库不支持事务、多数据源事务管理器选错、手动连接绕过 Spring 管理。

## 易错点

- 以为加了 `@Transactional` 就一定生效。
- 忽略 Spring AOP 必须经过代理调用。
- 以为所有异常都会回滚。
- catch 异常后不抛出，却期望自动回滚。
- 在 private、final、static 方法上加事务。
- 期望事务自动跨线程传播。
- 多数据源项目中不指定或指定错 transactionManager。
- 把 `REQUIRES_NEW` 的独立提交误判为事务失效。

## 自检清单

- 能否解释事务自调用失效。
- 能否说出默认回滚异常类型。
- 能否说明 catch 异常后如何保证回滚。
- 能否说明 public 方法和代理的关系。
- 能否解释异步线程为什么没有外层事务。
- 能否排查多数据源事务失效。
- 能否区分真正失效和传播行为符合预期。
- 能否写出正确的事务边界拆分方案。
