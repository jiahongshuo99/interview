# Spring 事务传播机制

## 面试定位

Spring 事务传播机制考察的是你对“多个事务方法互相调用时事务边界如何合并、挂起、新建、回滚”的理解。面试常见追问：

- `REQUIRED` 和 `REQUIRES_NEW` 区别。
- `NESTED` 和 `REQUIRES_NEW` 区别。
- 外层事务回滚是否影响内层事务。
- 内层事务异常被捕获后为什么外层提交仍可能失败。
- 事务传播机制和数据库连接、保存点有什么关系。

回答时要先强调：传播行为只在通过 Spring 代理调用事务方法时生效，自调用不会触发事务拦截。

## 事务传播是什么

事务传播行为定义了一个事务方法被另一个事务方法调用时，应该如何处理当前事务。

入口注解：

```java
@Transactional(propagation = Propagation.REQUIRED)
public void createOrder() {
}
```

底层由 `TransactionInterceptor` 在方法调用前后处理：

```text
代理对象
  -> TransactionInterceptor.invoke()
     -> 获取事务属性
     -> 获取或创建事务
     -> 执行业务方法
     -> 提交或回滚
```

核心类型：

- `TransactionInterceptor`。
- `TransactionAttribute`。
- `PlatformTransactionManager`。
- `TransactionStatus`。
- `TransactionSynchronizationManager`。

## 七种传播行为

Spring 定义了七种传播行为：

- `REQUIRED`。
- `SUPPORTS`。
- `MANDATORY`。
- `REQUIRES_NEW`。
- `NOT_SUPPORTED`。
- `NEVER`。
- `NESTED`。

实际业务中最高频的是：

- `REQUIRED`。
- `REQUIRES_NEW`。
- `NESTED`。

## REQUIRED

默认传播行为。

规则：

- 当前存在事务：加入当前事务。
- 当前不存在事务：新建一个事务。

示例：

```java
@Transactional
public void outer() {
    inner();
}

@Transactional
public void inner() {
}
```

如果 `outer()` 通过代理开启事务，`inner()` 也通过代理被调用，`inner()` 会加入 `outer()` 的事务。

事务边界：

```text
outer transaction begin
  outer logic
  inner logic
outer transaction commit
```

特点：

- 内外层共用一个事务。
- 任意一处回滚会影响整个事务。
- 适合主业务流程中的原子操作。

## REQUIRES_NEW

规则：

- 当前存在事务：挂起当前事务，新建一个事务。
- 当前不存在事务：新建一个事务。

示例：

```java
@Transactional
public void createOrder() {
    orderRepository.save(order);
    auditService.recordLog();
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void recordLog() {
    auditRepository.save(log);
}
```

事务边界：

```text
outer transaction begin
  save order
  suspend outer transaction
    inner transaction begin
      save audit log
    inner transaction commit
  resume outer transaction
outer transaction rollback or commit
```

特点：

- 内层事务独立提交或回滚。
- 外层事务回滚通常不会回滚已经提交的内层 `REQUIRES_NEW`。
- 需要额外数据库连接，连接池不足时可能有风险。

常见用途：

- 记录操作日志。
- 记录失败日志。
- 发送状态落库。
- 某些必须独立提交的审计数据。

## NESTED

规则：

- 当前存在事务：在当前事务内创建嵌套事务，通常基于保存点。
- 当前不存在事务：行为类似 `REQUIRED`，新建事务。

保存点模型：

```text
outer transaction begin
  savepoint S1
    inner logic
    rollback to S1 or release S1
outer transaction commit
```

特点：

- 内层回滚可以回滚到保存点，不一定导致外层全部回滚。
- 外层事务最终回滚时，内层已经执行的内容也会一起回滚。
- 依赖事务管理器和数据库保存点支持。

与 `REQUIRES_NEW` 的关键区别：

- `REQUIRES_NEW` 是独立物理事务，外层回滚不影响内层已提交结果。
- `NESTED` 是外层物理事务内的保存点，外层回滚会回滚全部。

## SUPPORTS

规则：

- 当前存在事务：加入当前事务。
- 当前不存在事务：以非事务方式执行。

适合：

- 查询方法。
- 可参与事务但不强制要求事务的逻辑。

示例：

```java
@Transactional(propagation = Propagation.SUPPORTS, readOnly = true)
public User queryUser(Long id) {
    return userRepository.findById(id);
}
```

注意：如果没有事务，方法中的多次数据库访问不具备事务一致性边界。

## MANDATORY

规则：

- 当前存在事务：加入当前事务。
- 当前不存在事务：抛异常。

适合必须在上层事务中执行的内部方法。

```java
@Transactional(propagation = Propagation.MANDATORY)
public void deductInventory() {
}
```

如果调用方没有事务，会抛出：

```text
IllegalTransactionStateException
```

## NOT_SUPPORTED

规则：

- 当前存在事务：挂起当前事务，以非事务方式执行。
- 当前不存在事务：以非事务方式执行。

适合：

- 明确不希望在事务内执行的慢操作。
- 某些不需要事务的查询。
- 避免长事务包住外部调用。

注意：被挂起期间的数据库操作不会跟随外层事务回滚。

## NEVER

规则：

- 当前存在事务：抛异常。
- 当前不存在事务：以非事务方式执行。

适合极少数必须禁止事务的场景。

```java
@Transactional(propagation = Propagation.NEVER)
public void nonTransactionalOperation() {
}
```

## 传播行为对比表

| 传播行为 | 有当前事务 | 无当前事务 | 是否常用 |
| --- | --- | --- | --- |
| REQUIRED | 加入当前事务 | 新建事务 | 高 |
| REQUIRES_NEW | 挂起当前事务，新建事务 | 新建事务 | 高 |
| NESTED | 当前事务内建保存点 | 新建事务 | 中 |
| SUPPORTS | 加入当前事务 | 非事务执行 | 中 |
| MANDATORY | 加入当前事务 | 抛异常 | 低 |
| NOT_SUPPORTED | 挂起当前事务，非事务执行 | 非事务执行 | 低 |
| NEVER | 抛异常 | 非事务执行 | 低 |

## 回滚规则

默认情况下：

- 运行时异常 `RuntimeException` 回滚。
- `Error` 回滚。
- 受检异常 `Exception` 默认不回滚。

可配置：

```java
@Transactional(rollbackFor = Exception.class)
public void submit() throws Exception {
}
```

或：

```java
@Transactional(noRollbackFor = BizException.class)
public void submit() {
}
```

传播行为和回滚规则是两个维度：

- 传播行为决定事务边界如何参与。
- 回滚规则决定异常发生后是否标记回滚。

## REQUIRED 中异常被捕获的坑

示例：

```java
@Transactional
public void outer() {
    try {
        innerService.inner();
    } catch (Exception e) {
        log.warn("inner failed", e);
    }
}

@Transactional
public void inner() {
    throw new RuntimeException();
}
```

如果 `inner()` 与 `outer()` 同属 `REQUIRED`，它们共用一个事务。内层抛出运行时异常时，事务可能被标记为 rollback-only。即使外层捕获异常并继续执行，最终提交时也会失败，可能抛出：

```text
UnexpectedRollbackException
```

解决思路：

- 内层异常需要外层整体失败：不要吞异常。
- 内层失败不影响外层：考虑 `REQUIRES_NEW`，并明确异常处理。
- 使用业务返回值表达可恢复失败，而不是让事务进入 rollback-only 后再吞异常。

## REQUIRES_NEW 的连接问题

`REQUIRES_NEW` 会挂起外层事务并开启新事务，通常需要额外数据库连接。

如果大量并发中每个请求都持有外层连接，又进入 `REQUIRES_NEW` 等待新连接，连接池可能被耗尽。

风险场景：

```text
线程数 = 100
连接池大小 = 100
每个线程持有外层事务连接
每个线程都进入 REQUIRES_NEW 等待第二个连接
```

可能出现连接池等待甚至死锁式阻塞。

实践建议：

- 不要滥用 `REQUIRES_NEW`。
- 控制外层事务范围。
- 连接池大小要覆盖传播行为带来的额外连接需求。
- 审计日志可考虑异步消息或独立可靠通道。

## NESTED 的保存点限制

`NESTED` 依赖保存点：

- 数据库需要支持 Savepoint。
- 事务管理器需要支持嵌套事务。
- 常见 JDBC `DataSourceTransactionManager` 支持保存点。
- JPA 场景中要结合具体实现判断。

不能简单认为所有事务管理器都支持 `NESTED`。

## 源码链路理解

核心入口：

```text
TransactionInterceptor.invoke()
  -> TransactionAspectSupport.invokeWithinTransaction()
     -> createTransactionIfNecessary()
        -> PlatformTransactionManager.getTransaction()
           -> AbstractPlatformTransactionManager.getTransaction()
              -> handleExistingTransaction()
     -> invocation.proceedWithInvocation()
     -> commitTransactionAfterReturning()
     -> completeTransactionAfterThrowing()
```

已有事务时的传播处理在：

```text
AbstractPlatformTransactionManager.handleExistingTransaction()
```

这里会根据传播行为决定：

- 抛异常。
- 挂起事务。
- 创建新事务。
- 创建保存点。
- 加入当前事务。

## 工程实践

- 默认使用 `REQUIRED`，不要过度设计传播行为。
- 需要独立提交的日志、审计、流水可考虑 `REQUIRES_NEW`，但要评估连接池压力。
- 需要局部回滚且最终仍受外层事务控制，可考虑 `NESTED`，但要确认保存点支持。
- 外部 RPC、MQ 发送、耗时 IO 不要放在长事务中。
- 不要吞掉事务方法抛出的运行时异常，除非明确知道事务状态。
- 事务方法尽量放在 service 层，不要散落在 DAO 或 Controller。
- 一个事务方法只覆盖必要的数据库操作，事务边界越小越好。

## 常见追问

### 1. REQUIRED 和 REQUIRES_NEW 的区别

`REQUIRED` 有事务就加入当前事务，没有就新建；`REQUIRES_NEW` 总是开启新事务，如果当前有事务就挂起当前事务。`REQUIRES_NEW` 内层提交后，外层回滚通常不影响内层结果。

### 2. REQUIRES_NEW 和 NESTED 的区别

`REQUIRES_NEW` 是独立物理事务，外层事务会被挂起；`NESTED` 是当前物理事务内的保存点。外层回滚会回滚 `NESTED` 的内容，但不会回滚已提交的 `REQUIRES_NEW`。

### 3. 内层 REQUIRED 抛异常被外层 catch，外层还能提交吗

不一定。因为共用一个事务，内层异常可能已经把事务标记为 rollback-only，外层即使 catch 了异常，最终提交也可能抛 `UnexpectedRollbackException`。

### 4. 哪些传播行为会挂起当前事务

常见有：

- `REQUIRES_NEW`：挂起当前事务并新建事务。
- `NOT_SUPPORTED`：挂起当前事务并非事务执行。

### 5. NESTED 是否一定可用

不一定。它依赖数据库保存点和事务管理器支持。使用前要确认具体技术栈。

### 6. 传播行为在同类自调用时生效吗

不生效。事务传播由代理拦截器处理，同类 `this.method()` 不经过代理。

## 易错点

- 把 `NESTED` 当成 `REQUIRES_NEW`。
- 以为外层捕获异常后事务一定可以提交。
- 以为所有异常都会自动回滚。
- 滥用 `REQUIRES_NEW`，忽略连接池压力。
- 忽略传播机制必须经过 Spring 代理才生效。
- 在事务中做耗时远程调用，扩大锁持有时间。
- 以为 `readOnly = true` 等同于绝对禁止写入，实际更多是优化和语义提示，具体效果依赖数据库和驱动。

## 自检清单

- 能否说出七种传播行为。
- 能否重点解释 `REQUIRED`、`REQUIRES_NEW`、`NESTED`。
- 能否画出 `REQUIRES_NEW` 挂起外层事务的流程。
- 能否解释 `NESTED` 保存点模型。
- 能否说明默认回滚规则。
- 能否解释 `UnexpectedRollbackException` 的原因。
- 能否说明传播行为为什么自调用不生效。
- 能否给出事务边界设计实践。
