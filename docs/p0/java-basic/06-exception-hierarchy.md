# 异常体系、checked / unchecked exception

## 面试定位

异常体系是 Java 后端开发的基本功。面试官会问 Throwable、Error、Exception、RuntimeException 的关系，checked exception 和 unchecked exception 的区别，什么时候捕获、什么时候抛出，finally 是否一定执行，以及业务异常如何设计。

这一题既考语言基础，也考工程习惯。好的回答应能说明：

- Java 异常体系的层级。
- checked exception 是编译器强制处理的异常。
- unchecked exception 通常表示编程错误、参数错误、状态错误或业务运行期失败。
- 后端服务应有统一异常边界、日志策略和错误码体系。

## 异常体系

Java 异常根类型是 `Throwable`，主要分为 `Error` 和 `Exception`。

```text
Throwable
├── Error
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── NoClassDefFoundError
└── Exception
    ├── IOException
    ├── SQLException
    └── RuntimeException
        ├── NullPointerException
        ├── IllegalArgumentException
        ├── IllegalStateException
        ├── IndexOutOfBoundsException
        └── ClassCastException
```

### Error

`Error` 表示 JVM 或系统层面的严重问题，应用程序通常不应该捕获并尝试恢复。

典型例子：

- `OutOfMemoryError`：内存不足。
- `StackOverflowError`：栈溢出。
- `NoClassDefFoundError`：运行期找不到类定义。
- `ExceptionInInitializerError`：类初始化失败。

工程中可以在最外层做兜底日志，但不要把 `Error` 当成普通业务异常吞掉。

### Exception

`Exception` 表示程序可以处理或声明抛出的异常。它又分为 checked exception 和 unchecked exception。

## checked exception 与 unchecked exception

### checked exception

checked exception 是编译器强制处理的异常，除了 `RuntimeException` 及其子类之外的 `Exception` 通常都属于 checked exception。

```java
public String read(String path) throws IOException {
    return Files.readString(Path.of(path));
}
```

调用方必须捕获或继续声明抛出：

```java
try {
    String content = read("/tmp/a.txt");
} catch (IOException e) {
    // handle
}
```

典型 checked exception：

- `IOException`
- `SQLException`
- `ClassNotFoundException`
- `InterruptedException`

checked exception 适合表达调用方有合理机会恢复或必须显式感知的外部失败，例如文件不存在、网络 IO 失败、线程被中断。

### unchecked exception

unchecked exception 包括 `RuntimeException` 及其子类，以及 `Error`。编译器不强制捕获。

典型 runtime exception：

- `NullPointerException`
- `IllegalArgumentException`
- `IllegalStateException`
- `IndexOutOfBoundsException`
- `ClassCastException`
- `NumberFormatException`
- `ConcurrentModificationException`

unchecked exception 常表示代码 bug、参数非法、对象状态不满足、业务前置条件失败等。

## 常见 RuntimeException 语义

### NullPointerException

访问 `null` 引用的字段、方法或拆箱时发生。

```java
Integer count = null;
int value = count; // NPE，自动拆箱
```

### IllegalArgumentException

方法参数不符合要求。

```java
public void setAge(int age) {
    if (age < 0) {
        throw new IllegalArgumentException("age must not be negative");
    }
}
```

### IllegalStateException

对象当前状态不允许执行该操作。

```java
if (order.isClosed()) {
    throw new IllegalStateException("order already closed");
}
```

### UnsupportedOperationException

对象不支持某操作，例如只读集合调用修改方法。

```java
List<String> values = List.of("a");
values.add("b"); // UnsupportedOperationException
```

## try-catch-finally

### 基本结构

```java
try {
    doWork();
} catch (IOException e) {
    log.warn("io failed", e);
} finally {
    cleanup();
}
```

`finally` 通常用于释放资源。无论 try 中是否抛异常，catch 是否处理，通常都会执行 finally。

### finally 不执行的情况

不能绝对说 finally 一定执行。以下情况可能不执行或无法完成：

- JVM 进程退出，例如 `System.exit`。
- JVM 崩溃。
- 机器断电、进程被强杀。
- 当前线程被无限阻塞或死循环，无法走到 finally。

### finally 中 return 的问题

不要在 finally 中写 return。它会覆盖 try 或 catch 中的返回值和异常，导致问题被吞掉。

```java
public int bad() {
    try {
        return 1;
    } finally {
        return 2; // 覆盖 try 中的 return
    }
}
```

如果 try 抛异常，finally 中 return 也会吞掉异常，这是严重坏味道。

## try-with-resources

实现 `AutoCloseable` 的资源应优先使用 try-with-resources 自动关闭。

```java
try (InputStream input = Files.newInputStream(Path.of("/tmp/a.txt"))) {
    return input.readAllBytes();
}
```

多个资源按声明顺序创建，关闭时按相反顺序关闭。

try-with-resources 能正确处理关闭资源时的异常。如果 try 块和 close 都抛异常，close 抛出的异常会作为 suppressed exception 挂到主异常上。

```java
catch (IOException e) {
    for (Throwable suppressed : e.getSuppressed()) {
        log.warn("suppressed exception", suppressed);
    }
}
```

## throw 与 throws

- `throw`：在方法体内实际抛出一个异常对象。
- `throws`：在方法签名上声明该方法可能抛出的异常类型。

```java
public void check(String name) throws IOException {
    if (name == null) {
        throw new IllegalArgumentException("name is null");
    }
}
```

重写方法时，子类不能抛出比父类方法更宽的 checked exception，但可以抛出更窄的 checked exception 或 unchecked exception。

## 异常链

捕获异常后重新抛出时，应保留 cause，避免丢失根因。

```java
try {
    remoteCall();
} catch (IOException e) {
    throw new RemoteAccessException("remote call failed", e);
}
```

不要这样写：

```java
catch (IOException e) {
    throw new RemoteAccessException("remote call failed");
}
```

后者丢失原始堆栈，排查线上问题会困难。

## 自定义业务异常

后端服务通常会定义业务异常，携带错误码、错误信息和上下文。

```java
public class BizException extends RuntimeException {
    private final String code;

    public BizException(String code, String message) {
        super(message);
        this.code = code;
    }

    public BizException(String code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }

    public String getCode() {
        return code;
    }
}
```

使用：

```java
if (stock <= 0) {
    throw new BizException("STOCK_NOT_ENOUGH", "库存不足");
}
```

业务异常常设计为 unchecked exception，因为多数业务失败由统一异常处理器转换为响应，不要求每层方法都声明。

## 统一异常处理

Web 服务中通常在 Controller 层或网关层做统一异常转换。

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(BizException.class)
    public ApiResponse<Void> handleBizException(BizException e) {
        return ApiResponse.fail(e.getCode(), e.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public ApiResponse<Void> handleException(Exception e) {
        log.error("unexpected error", e);
        return ApiResponse.fail("SYSTEM_ERROR", "系统异常");
    }
}
```

注意不要把所有异常都以 warn 级别吞掉。未知异常需要 error 日志和监控告警。

## InterruptedException

`InterruptedException` 是并发编程中非常重要的 checked exception，表示线程被请求中断。捕获后不能随便吞掉。

推荐处理方式：

```java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    throw new IllegalStateException("thread interrupted", e);
}
```

如果当前方法能继续声明抛出，直接 `throws InterruptedException` 也是合理的。

吞掉中断会导致线程池关闭、任务取消等机制失效。

## 常见追问

### checked exception 和 unchecked exception 的区别？

checked exception 在编译期被强制处理，调用方必须捕获或声明抛出；unchecked exception 不被编译器强制处理，通常表示程序错误、非法参数、非法状态或业务运行期失败。

### RuntimeException 要不要捕获？

不应在底层大量捕获 RuntimeException 并吞掉。应该在能处理的地方捕获；处理不了就向上抛，由统一异常边界记录日志、转换响应或触发告警。

### finally 一定执行吗？

通常会执行，但不是绝对。`System.exit`、JVM 崩溃、进程被强杀、断电等情况下可能不执行。并且 finally 中不要 return，否则会覆盖异常或返回值。

### catch Exception 和 catch Throwable 有什么问题？

`catch Exception` 可能过宽，掩盖具体处理策略；`catch Throwable` 会把 `Error` 也捕获，可能误吞严重系统错误。最外层框架可以兜底 `Throwable` 做日志和资源清理，但业务代码不应随意捕获。

### throw e 和 throw new Exception(e) 的区别？

`throw e` 保留原异常对象和堆栈；`throw new Exception(e)` 创建新异常，原异常成为 cause。重新包装时应保留 cause，并确保错误语义更清楚。

## 代码示例

### 参数校验

```java
public Order createOrder(CreateOrderRequest request) {
    if (request == null) {
        throw new IllegalArgumentException("request is null");
    }
    if (request.getUserId() <= 0) {
        throw new IllegalArgumentException("invalid userId");
    }
    return doCreate(request);
}
```

### 资源关闭

```java
public List<String> readLines(Path path) {
    try {
        return Files.readAllLines(path, StandardCharsets.UTF_8);
    } catch (IOException e) {
        throw new FileReadException("read file failed: " + path, e);
    }
}
```

### 不要吞异常

```java
try {
    sendMessage(message);
} catch (Exception e) {
    log.error("send message failed, messageId={}", message.getId(), e);
    throw e;
}
```

如果这里吞掉异常，上游会以为发送成功，可能造成数据不一致。

## 工程实践

- 异常只在能处理的层捕获；不能处理就保留上下文继续抛出。
- 捕获后重新包装异常时必须保留 cause。
- 业务异常建议带错误码，统一转换为接口响应。
- 未知异常要记录 error 日志，包含关键业务上下文，但避免打印敏感信息。
- 不要用异常做正常流程控制，例如循环中通过捕获异常判断结束。
- 不要在 finally 中 return 或抛出会覆盖原异常的新异常。
- IO、锁、连接等资源使用 try-with-resources 或 finally 保证释放。
- 捕获 `InterruptedException` 后要恢复中断状态或继续抛出。
- 对外接口不要直接暴露内部异常消息，避免泄漏实现和敏感数据。

## 易错点

- `finally` 中 return 吞掉异常。
- 捕获 `Exception` 后只打印日志不抛出，导致调用方误判成功。
- 重新抛出业务异常时丢失 cause。
- 把所有业务失败都当系统异常，无法区分用户错误和系统错误。
- 吞掉 `InterruptedException`。
- 在热路径中用异常做分支逻辑，性能和可读性都差。
- `catch Throwable` 误吞 `OutOfMemoryError` 等严重错误。
- checked exception 层层 `throws Exception`，丢失精确信息。

## 自检清单

- 能画出 Throwable、Error、Exception、RuntimeException 的层级。
- 能区分 checked exception 和 unchecked exception。
- 能说明 finally 的执行规则和 return 风险。
- 能正确使用 try-with-resources。
- 能解释异常链和 cause 的意义。
- 能设计一个简单业务异常和统一异常处理。
- 能说明 InterruptedException 为什么不能吞。
