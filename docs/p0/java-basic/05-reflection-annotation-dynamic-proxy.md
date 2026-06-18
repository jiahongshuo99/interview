# 反射、注解、动态代理

## 面试定位

反射、注解、动态代理是 Java 后端框架的基础。Spring IOC、AOP、事务、MyBatis Mapper、JDK SPI、JSON 序列化、RPC 框架都大量依赖这些能力。面试时通常会从“反射是什么”问到“动态代理怎么实现 AOP”，再追问 JDK 动态代理和 CGLIB 的区别、注解保留策略、反射性能和安全问题。

这一题的关键是把三者串起来：

- 注解负责声明元数据。
- 反射负责在运行期读取类、方法、字段和注解信息，并调用成员。
- 动态代理负责在不改业务类代码的情况下增强方法调用。

## 反射

反射是 Java 在运行期检查和操作类结构的能力。通过反射可以获取类、构造方法、字段、方法、注解、泛型签名等信息，也可以创建对象和调用方法。

### 获取 Class 对象

```java
Class<String> c1 = String.class;
Class<?> c2 = Class.forName("java.lang.String");
Class<?> c3 = "hello".getClass();
```

三种方式的差异：

- `String.class` 最直接，编译期已知类型。
- `Class.forName` 常用于运行期按类名加载，默认会触发类初始化。
- `obj.getClass()` 从对象获取实际运行期类型。

### 创建对象

```java
Constructor<User> constructor = User.class.getDeclaredConstructor(String.class);
constructor.setAccessible(true);
User user = constructor.newInstance("jerry");
```

现代 Java 推荐通过 `Constructor.newInstance`，不要再使用已经不推荐的 `Class.newInstance`，因为后者只能调用无参构造且异常语义较弱。

### 访问字段和方法

```java
Field field = User.class.getDeclaredField("name");
field.setAccessible(true);
field.set(user, "tom");

Method method = User.class.getDeclaredMethod("getName");
String name = (String) method.invoke(user);
```

`setAccessible(true)` 可以绕过 Java 语言访问检查，但会受到模块系统、安全策略和运行参数影响。工程中应谨慎使用，避免破坏封装。

## 注解

注解是附加在代码元素上的元数据，本身不直接改变程序逻辑。注解需要被编译器、框架、反射或字节码工具读取后才产生作用。

### 定义注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface AuditLog {
    String value();

    boolean async() default true;
}
```

使用：

```java
public class OrderService {
    @AuditLog(value = "create_order", async = false)
    public void createOrder() {
        // create order
    }
}
```

读取：

```java
Method method = OrderService.class.getDeclaredMethod("createOrder");
AuditLog auditLog = method.getAnnotation(AuditLog.class);
System.out.println(auditLog.value());
```

### 元注解

常用元注解：

- `@Retention`：注解保留到哪个阶段。
- `@Target`：注解可标记的位置。
- `@Documented`：是否出现在 Javadoc。
- `@Inherited`：类注解是否可被子类继承。
- `@Repeatable`：是否可重复标注。

### RetentionPolicy

| 策略 | 含义 | 常见用途 |
| --- | --- | --- |
| `SOURCE` | 只在源码中存在，编译后丢弃 | Lombok、编译检查 |
| `CLASS` | 保留到 class 文件，运行期默认不可反射读取 | 字节码增强、编译工具 |
| `RUNTIME` | 运行期可通过反射读取 | Spring、JUnit、Jackson |

如果希望通过反射读取注解，必须使用 `RetentionPolicy.RUNTIME`。

### Target

常用目标：

- `TYPE`：类、接口、枚举。
- `FIELD`：字段。
- `METHOD`：方法。
- `PARAMETER`：参数。
- `CONSTRUCTOR`：构造方法。
- `ANNOTATION_TYPE`：注解类型。
- `TYPE_USE`：类型使用位置。

## 动态代理

动态代理是在运行期创建代理对象，代理对象实现与目标对象相同的接口或继承目标类，并在方法调用前后加入增强逻辑。

常见用途：

- AOP 切面。
- 事务、权限、审计、日志、指标、限流。
- RPC 客户端桩。
- MyBatis Mapper 接口实现。
- 声明式缓存。

## JDK 动态代理

JDK 动态代理基于接口生成代理类。目标类必须有接口，代理对象实现这些接口。

```java
public interface UserService {
    String getName(long userId);
}

public class DefaultUserService implements UserService {
    @Override
    public String getName(long userId) {
        return "user-" + userId;
    }
}

public class LogInvocationHandler implements InvocationHandler {
    private final Object target;

    public LogInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        long start = System.nanoTime();
        try {
            return method.invoke(target, args);
        } finally {
            long cost = System.nanoTime() - start;
            System.out.println(method.getName() + " cost=" + cost);
        }
    }
}
```

创建代理：

```java
UserService target = new DefaultUserService();
UserService proxy = (UserService) Proxy.newProxyInstance(
        target.getClass().getClassLoader(),
        new Class<?>[]{UserService.class},
        new LogInvocationHandler(target)
);

proxy.getName(1L);
```

调用链：

1. 调用方调用代理对象方法。
2. 代理对象把调用转发给 `InvocationHandler.invoke`。
3. `invoke` 中执行增强逻辑。
4. 通过反射调用目标对象方法。
5. 返回结果或抛出异常。

## CGLIB 动态代理

CGLIB 通过生成目标类的子类来代理方法，因此不要求目标类实现接口。Spring AOP 在没有接口或配置使用类代理时常使用类似机制。

特点：

- 可以代理普通类。
- 不能代理 `final` 类。
- 不能代理 `final` 方法，因为子类无法重写。
- 构造、方法可见性、模块限制等都可能影响代理。

现代 Spring 也可能使用 Byte Buddy 等字节码技术。面试中通常按“JDK 接口代理”和“CGLIB 子类代理”对比即可。

## JDK 动态代理与 CGLIB 对比

| 维度 | JDK 动态代理 | CGLIB |
| --- | --- | --- |
| 代理基础 | 接口 | 继承目标类 |
| 是否需要接口 | 需要 | 不需要 |
| final 类 | 不影响接口代理 | 不能代理 |
| final 方法 | 接口方法可代理 | 不能重写增强 |
| 典型场景 | 面向接口的服务、Mapper、RPC | 无接口类、Spring 类代理 |
| 底层 | 运行期生成实现接口的代理类 | 运行期生成目标类子类 |

## Spring AOP 与动态代理

Spring AOP 的基本思路：

1. 识别切点，例如有注解的方法或符合表达式的方法。
2. 为 Bean 创建代理对象。
3. 容器中注入的是代理对象而不是原始对象。
4. 方法调用经过代理对象，执行事务、日志、鉴权等增强。

典型问题是自调用失效：

```java
public class OrderService {
    public void outer() {
        inner(); // this.inner()，没有经过代理对象
    }

    @Transactional
    public void inner() {
        // 事务可能不生效
    }
}
```

因为 `outer` 内部调用 `inner` 是当前对象的直接调用，没有经过 Spring 代理。解决思路包括拆到另一个 Bean、通过代理对象调用、调整事务边界。

## 底层机制

### Class 元数据

JVM 加载 class 后，会在运行期维护类元数据。反射 API 基于这些元数据暴露字段、方法、构造器、注解等信息。

### 方法反射调用

`Method.invoke` 会做访问检查、参数包装、异常包装等工作。被调用方法抛出的异常会被包装成 `InvocationTargetException`，真正异常可通过 `getTargetException()` 或 `getCause()` 获取。

```java
try {
    method.invoke(target);
} catch (InvocationTargetException e) {
    Throwable actual = e.getTargetException();
    throw actual;
}
```

### 代理类生成

JDK 动态代理会在运行期生成一个实现指定接口的代理类。代理类的方法通常转发到同一个 `InvocationHandler`。这也是为什么 JDK 动态代理只能基于接口：代理类要实现接口方法。

## 常见追问

### 反射为什么慢？

反射调用通常涉及访问检查、参数装箱拆箱、方法分派、异常包装等额外开销，JIT 优化也更难直接内联。但现代 JDK 对反射有优化，普通业务不要夸大性能问题。热点路径应优先用直接调用、MethodHandle、缓存反射对象或代码生成。

### 注解本身会生效吗？

不会。注解只是元数据，必须有处理器读取它。比如 `@Transactional` 是 Spring 识别后创建代理并开启事务；如果对象不受 Spring 管理，注解不会自动生效。

### JDK 动态代理为什么必须有接口？

因为它生成的是实现接口的代理类。调用方通过接口引用调用方法，代理类把调用转发给 `InvocationHandler`。如果目标类没有接口，就无法用 JDK 动态代理按类方法代理。

### CGLIB 为什么不能代理 final 方法？

CGLIB 通过创建子类并重写方法来增强。`final` 方法不能被重写，所以无法拦截增强；`final` 类不能被继承，所以不能被 CGLIB 代理。

### 反射能不能访问 private 字段？

可以通过 `setAccessible(true)` 尝试绕过访问检查，但会破坏封装，也可能受模块系统、安全策略限制。工程中只有框架、序列化、测试工具等少数场景适合这样做。

## 代码示例

### 注解加反射实现简单路由

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Route {
    String value();
}

public class UserController {
    @Route("/user/detail")
    public String detail() {
        return "ok";
    }
}

Map<String, Method> routes = new HashMap<>();
for (Method method : UserController.class.getDeclaredMethods()) {
    Route route = method.getAnnotation(Route.class);
    if (route != null) {
        routes.put(route.value(), method);
    }
}
```

### 动态代理实现耗时统计

```java
public class TimingHandler implements InvocationHandler {
    private final Object target;

    public TimingHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return method.invoke(target, args);
        } catch (InvocationTargetException e) {
            throw e.getTargetException();
        } finally {
            System.out.println(method.getName() + " cost "
                    + (System.currentTimeMillis() - start) + " ms");
        }
    }
}
```

## 工程实践

- 反射对象如 `Class`、`Method`、`Field` 可以缓存，避免在热点路径反复查找。
- 注解要明确 `Retention` 和 `Target`，不要依赖默认值造成运行期读不到。
- 框架型代码使用反射时，要处理访问权限、异常包装、泛型擦除和类加载器问题。
- 业务代码中不要随意 `setAccessible(true)` 修改私有字段，优先使用公开 API。
- Spring AOP 中要知道代理边界，自调用、非 Spring 管理对象、final 方法都可能导致增强不生效。
- 动态代理适合横切逻辑，不适合把核心业务藏进 InvocationHandler 导致可读性下降。
- 反射和代理错误通常在运行期暴露，要补足单测和启动期校验。
- RPC、Mapper 这类代理接口要避免在接口方法上使用过度复杂的重载，降低框架解析难度。

## 易错点

- 认为注解写上就自动生效，忽略处理器。
- 忘记 `RetentionPolicy.RUNTIME`，运行期反射读不到注解。
- JDK 动态代理用于没有接口的类，导致创建失败或无法代理预期方法。
- CGLIB 代理 final 类或 final 方法。
- 反射调用异常被 `InvocationTargetException` 包装，日志只看外层异常。
- Spring AOP 自调用导致事务、缓存、异步等注解不生效。
- 反射访问私有成员破坏封装，升级 JDK 或模块化后失败。
- 动态代理中没有正确处理 `equals`、`hashCode`、`toString` 等 Object 方法。

## 自检清单

- 能说明反射能做什么以及代价是什么。
- 能定义一个运行期可读取的注解。
- 能解释 `Retention` 和 `Target`。
- 能写出 JDK 动态代理的基本代码。
- 能比较 JDK 动态代理和 CGLIB。
- 能解释 Spring AOP 自调用失效。
- 能处理 `Method.invoke` 的异常包装。
