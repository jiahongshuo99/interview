# 面向对象、接口、抽象类、继承、多态

## 面试定位

这一题考察 Java 面向对象基础，也考察候选人是否能把语言特性落到工程设计。面试官常从“接口和抽象类有什么区别”切入，再追问继承与组合、多态实现机制、重载与重写、默认方法、访问控制、构造顺序和设计原则。

优秀回答不应只背概念，而要能说明：

- 面向对象的封装、继承、多态分别解决什么问题。
- 接口和抽象类在语义、状态、演进和多继承能力上的差异。
- 多态依赖动态绑定，重写在运行期选择目标方法，重载在编译期确定签名。
- 后端工程中优先面向接口编程，谨慎使用继承，常用组合表达复用。

## 核心概念

### 封装

封装是把数据和操作数据的行为放在一起，并通过访问控制隐藏内部实现。

```java
public class Account {
    private long balance;

    public void deposit(long amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("amount must be positive");
        }
        balance += amount;
    }

    public long getBalance() {
        return balance;
    }
}
```

封装的关键不是机械地给字段加 getter/setter，而是保护对象不变量。比如账户余额不能被外部随意改成负数，订单状态不能绕过状态机跳转。

### 继承

继承表达“is-a”关系，子类继承父类的字段和方法，并可重写父类方法。

```java
class Animal {
    void speak() {
        System.out.println("unknown");
    }
}

class Dog extends Animal {
    @Override
    void speak() {
        System.out.println("wang");
    }
}
```

Java 类只支持单继承，避免复杂的多继承菱形问题。接口支持多实现。

继承的优点是复用和抽象，缺点是耦合强、父类改动容易影响子类。工程中如果只是复用代码，优先考虑组合；如果确实存在稳定的层级抽象，再使用继承。

### 多态

多态是同一接口或父类引用在运行期表现出不同子类行为。

```java
Animal animal = new Dog();
animal.speak(); // 调用 Dog.speak()
```

多态的必要条件：

- 存在继承或接口实现关系。
- 子类重写父类或接口方法。
- 父类或接口引用指向子类对象。

多态使调用方依赖抽象而不是具体实现，是策略模式、模板方法、Spring Bean 注入、JDBC Driver 等大量框架设计的基础。

### 抽象类

抽象类用 `abstract` 修饰，可以有抽象方法，也可以有普通方法、字段和构造方法。抽象类不能直接实例化。

```java
abstract class AbstractExporter {
    public final void export() {
        validate();
        doExport();
    }

    protected void validate() {
        // 默认校验
    }

    protected abstract void doExport();
}
```

抽象类适合承载一组强相关类的公共状态和公共流程，例如模板方法模式中的固定算法骨架。

### 接口

接口定义能力或契约。Java 8 之后接口可以有 `default` 方法和 `static` 方法；Java 9 之后可以有私有方法辅助默认方法复用。

```java
public interface PaymentService {
    PayResult pay(PayRequest request);

    default boolean supportRefund() {
        return false;
    }
}
```

接口适合表达能力边界，例如 `Runnable`、`Comparable`、`Serializable`、业务中的 `OrderRepository`、`PaymentClient`。一个类可以实现多个接口。

## 接口与抽象类对比

| 维度 | 接口 | 抽象类 |
| --- | --- | --- |
| 语义 | 能力、契约、对外行为 | 同类事物的公共基类 |
| 继承关系 | 一个类可实现多个接口 | 一个类只能继承一个抽象类 |
| 状态 | 字段默认是 `public static final` 常量 | 可以有实例字段 |
| 构造方法 | 没有实例构造方法 | 可以有构造方法，供子类初始化 |
| 方法 | 抽象方法、默认方法、静态方法、私有方法 | 抽象方法、普通方法、静态方法、final 方法等 |
| 版本演进 | default 方法可兼容新增行为，但要谨慎 | 可在基类中增加公共实现 |
| 典型应用 | SPI、能力抽象、依赖倒置 | 模板方法、公共状态复用 |

面试回答建议：

- 如果要定义一组外部可替换的能力，用接口。
- 如果多个子类有共同状态、共同流程，且层级关系稳定，可以用抽象类。
- Java 不支持多继承类，但支持多实现接口；接口更适合解耦和测试替换。

## 底层机制

### 方法调用类型

JVM 字节码中常见方法调用指令：

- `invokestatic`：调用静态方法。
- `invokespecial`：调用构造方法、私有方法、父类方法。
- `invokevirtual`：调用普通实例方法，支持动态分派。
- `invokeinterface`：调用接口方法，支持动态分派。
- `invokedynamic`：动态语言支持、Lambda、字符串拼接等场景。

多态的核心是动态分派。编译期只知道引用类型，运行期根据实际对象类型选择重写后的目标方法。

### 重载与重写

重载是同一个类中方法名相同、参数列表不同，编译期根据静态类型和参数类型决定调用哪个方法。

```java
void print(Object value) {
    System.out.println("object");
}

void print(String value) {
    System.out.println("string");
}

Object v = "hello";
print(v); // object，重载选择看编译期类型
```

重写是子类重新实现父类可继承的方法，运行期根据实际对象类型决定调用哪个实现。

```java
class Parent {
    void run() {
        System.out.println("parent");
    }
}

class Child extends Parent {
    @Override
    void run() {
        System.out.println("child");
    }
}

Parent p = new Child();
p.run(); // child
```

### 字段没有多态

字段访问在编译期由引用类型决定，不存在方法重写意义上的动态分派。

```java
class Parent {
    String name = "parent";
}

class Child extends Parent {
    String name = "child";
}

Parent p = new Child();
System.out.println(p.name); // parent
```

### 构造与初始化顺序

对象创建时大致顺序：

1. 父类静态字段和静态代码块。
2. 子类静态字段和静态代码块。
3. 父类实例字段和实例代码块。
4. 父类构造方法。
5. 子类实例字段和实例代码块。
6. 子类构造方法。

同一类中静态字段和静态代码块按源码顺序执行，实例字段和实例代码块也按源码顺序执行。

构造方法中调用可重写方法是危险的，因为子类字段可能还没初始化完成。

```java
class Parent {
    Parent() {
        init(); // 可能调用子类重写方法
    }

    void init() {
    }
}

class Child extends Parent {
    private String value = "ready";

    @Override
    void init() {
        System.out.println(value); // 可能输出 null
    }
}
```

## 常见追问

### 接口能不能有实现？

可以。Java 8 开始接口支持 `default` 方法和 `static` 方法，Java 9 开始支持私有方法。默认方法主要用于接口演进，避免给接口新增方法后所有实现类都编译失败。但默认方法不应承载过多业务状态，否则接口会变得臃肿。

### 接口里的字段是什么？

接口字段默认是 `public static final`，也就是常量。即使不显式写修饰符，也是这个语义。接口不能定义普通实例字段。

### 抽象类一定要有抽象方法吗？

不一定。抽象类可以没有抽象方法。这样做通常是为了阻止直接实例化，或者表达这个类只作为基类存在。

### 静态方法能不能被重写？

不能。静态方法属于类，不参与运行期多态。子类可以定义同名静态方法，但这是隐藏，不是重写；调用哪个静态方法由引用类型或类名在编译期决定。

### `private` 方法能不能被重写？

不能。`private` 方法对子类不可见，不参与继承和重写。子类定义同名方法只是新方法。

### `final` 的作用是什么？

- `final` 类不能被继承，例如 `String`。
- `final` 方法不能被重写。
- `final` 变量只能赋值一次；引用变量不能重新指向别的对象，但对象内部状态仍可能可变。

### 多态会不会影响性能？

动态分派有一定成本，但现代 JVM 会通过内联缓存、类型分析、JIT 内联等方式优化。工程上不应为了微小调用成本放弃良好抽象，除非在经过性能分析确认的极端热点路径。

## 代码示例

### 面向接口编程

```java
public interface UserRepository {
    Optional<User> findById(long userId);
}

public class MysqlUserRepository implements UserRepository {
    @Override
    public Optional<User> findById(long userId) {
        // 查询 MySQL
        return Optional.empty();
    }
}

public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public Optional<User> getUser(long userId) {
        return userRepository.findById(userId);
    }
}
```

好处：

- `UserService` 不依赖 MySQL 细节。
- 单测可注入内存实现或 mock。
- 后续切换缓存、RPC、分库实现时调用方改动少。

### 抽象类模板方法

```java
public abstract class AbstractJob {
    public final void execute() {
        before();
        try {
            doExecute();
        } finally {
            after();
        }
    }

    protected void before() {
    }

    protected abstract void doExecute();

    protected void after() {
    }
}

public class SyncUserJob extends AbstractJob {
    @Override
    protected void doExecute() {
        // 同步用户
    }
}
```

模板方法中通常把主流程定义为 `final`，避免子类破坏算法骨架。

### 组合优于继承

```java
public class RateLimitedClient {
    private final RemoteClient remoteClient;
    private final RateLimiter rateLimiter;

    public RateLimitedClient(RemoteClient remoteClient, RateLimiter rateLimiter) {
        this.remoteClient = remoteClient;
        this.rateLimiter = rateLimiter;
    }

    public Response call(Request request) {
        rateLimiter.acquire();
        return remoteClient.call(request);
    }
}
```

这里限流能力通过组合增强客户端，而不是创建复杂继承层级。

## 工程实践

- 业务服务层依赖接口，不直接依赖难替换的具体实现，便于测试和迁移。
- 抽象类适合固定流程和公共状态，不适合做过大的“万能父类”。
- 如果只是为了复用一段代码，优先使用组合、工具类或委托，不急着继承。
- 继承层级不要太深。层级越深，理解初始化顺序、重写影响和状态来源越困难。
- 重写方法必须加 `@Override`，让编译器帮助发现签名错误。
- 父类构造方法中不要调用可重写方法。
- 接口默认方法要克制使用，避免接口变成带大量逻辑的半抽象类。
- 领域模型要封装不变量，不要把所有字段暴露成可随意 set 的贫血对象。

## 易错点

- 把重载当成多态。重载是编译期选择，重写才是运行期多态。
- 认为字段也有多态。字段访问看引用类型。
- 认为接口不能有方法实现。Java 8 后接口可以有默认方法。
- 认为抽象类必须有抽象方法。抽象类可以没有抽象方法。
- 在构造方法中调用被子类重写的方法，导致使用未初始化状态。
- 用继承表达“has-a”关系，例如 `Car extends Engine`，应改成组合。
- 子类静态方法“重写”父类静态方法，这其实是隐藏。
- 忽略访问权限：子类重写方法不能缩小父类方法可见性。

## 自检清单

- 能清楚说明封装、继承、多态的目的和代价。
- 能比较接口和抽象类，并给出适用场景。
- 能解释重载与重写的区别。
- 能解释为什么字段没有多态。
- 能说明构造初始化顺序和构造方法调用可重写方法的风险。
- 能在工程设计中说出什么时候用接口、抽象类、组合和继承。
- 能写出一个面向接口编程或模板方法的简单例子。
