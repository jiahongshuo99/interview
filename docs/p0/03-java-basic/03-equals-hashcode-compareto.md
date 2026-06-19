# equals / hashCode / compareTo

## 面试定位

这一题是 Java 集合、对象建模和排序比较的基础。面试官通常会问 `equals` 和 `==` 的区别、为什么重写 `equals` 必须重写 `hashCode`、`HashMap` 如何使用它们、`compareTo` 和 `Comparator` 有什么区别，以及 BigDecimal、TreeSet、HashSet 中的典型坑。

这类问题的核心不是背规范，而是理解对象“相等性”的不同维度：

- 引用相等：两个引用是否指向同一个对象。
- 逻辑相等：两个对象在业务语义上是否相同。
- 哈希相等：散列表定位桶时使用的整数摘要。
- 排序相等：排序比较结果是否为 0。

## `==` 与 `equals`

`==` 对基本类型比较值，对引用类型比较引用地址是否相同。

```java
int a = 1;
int b = 1;
System.out.println(a == b); // true

String x = new String("hello");
String y = new String("hello");
System.out.println(x == y);      // false
System.out.println(x.equals(y)); // true
```

`Object.equals` 默认实现就是引用相等：

```java
public boolean equals(Object obj) {
    return this == obj;
}
```

如果业务对象需要逻辑相等，必须重写 `equals`。

## equals 契约

`equals` 必须满足：

- 自反性：`x.equals(x)` 为 `true`。
- 对称性：`x.equals(y)` 为 `true`，则 `y.equals(x)` 也为 `true`。
- 传递性：`x.equals(y)`、`y.equals(z)` 为 `true`，则 `x.equals(z)` 为 `true`。
- 一致性：对象参与比较的字段不变时，多次调用结果一致。
- 非空性：`x.equals(null)` 为 `false`。

违反契约会导致集合、缓存、去重、排序等行为异常。

## hashCode 契约

`hashCode` 返回整数，用于散列表快速定位桶。它也有契约：

- 如果两个对象 `equals` 为 `true`，它们的 `hashCode` 必须相等。
- 如果两个对象 `hashCode` 相等，它们不一定 `equals`。
- 对象参与哈希计算的字段不变时，多次调用 `hashCode` 应返回相同结果。

重写 `equals` 必须重写 `hashCode`，否则 `HashMap`、`HashSet` 等集合会出现逻辑相等对象无法命中的问题。

```java
public final class UserKey {
    private final long tenantId;
    private final long userId;

    public UserKey(long tenantId, long userId) {
        this.tenantId = tenantId;
        this.userId = userId;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (!(o instanceof UserKey)) {
            return false;
        }
        UserKey userKey = (UserKey) o;
        return tenantId == userKey.tenantId && userId == userKey.userId;
    }

    @Override
    public int hashCode() {
        return Objects.hash(tenantId, userId);
    }
}
```

## HashMap 如何使用 equals 和 hashCode

`HashMap` 查找 key 的大致过程：

1. 调用 key 的 `hashCode` 得到哈希值，并做扰动计算。
2. 根据数组长度定位桶下标。
3. 如果桶中有元素，先比较哈希值，再用 `==` 或 `equals` 判断 key 是否相等。
4. 如果找到相等 key，返回对应 value；否则继续查找链表或红黑树。

所以：

- `hashCode` 决定大致位置。
- `equals` 决定是否真的是同一个逻辑 key。
- 哈希冲突不可避免，冲突后必须靠 `equals` 精确判断。

## compareTo

`Comparable<T>` 定义对象的自然顺序。

```java
public final class Version implements Comparable<Version> {
    private final int major;
    private final int minor;

    public Version(int major, int minor) {
        this.major = major;
        this.minor = minor;
    }

    @Override
    public int compareTo(Version other) {
        int r = Integer.compare(this.major, other.major);
        if (r != 0) {
            return r;
        }
        return Integer.compare(this.minor, other.minor);
    }
}
```

`compareTo` 返回值约定：

- 小于 0：当前对象小于参数对象。
- 等于 0：排序意义上相等。
- 大于 0：当前对象大于参数对象。

不要用减法实现整数比较：

```java
// 错误：可能溢出
return this.score - other.score;

// 正确
return Integer.compare(this.score, other.score);
```

## Comparable 与 Comparator

| 维度 | Comparable | Comparator |
| --- | --- | --- |
| 所在位置 | 对象类自身实现 | 外部比较器 |
| 语义 | 自然顺序 | 临时或多种排序规则 |
| 方法 | `compareTo` | `compare` |
| 修改类 | 需要改类代码 | 不需要改被比较类 |

示例：

```java
List<User> users = new ArrayList<>();

users.sort(Comparator
        .comparing(User::getAge)
        .thenComparing(User::getName, Comparator.nullsLast(String::compareTo)));
```

## compareTo 与 equals 的一致性

强烈建议 `compareTo` 返回 0 时，`equals` 也为 `true`。否则在 `TreeSet`、`TreeMap` 中会出现看似“不丢失却被覆盖”或“HashSet 和 TreeSet 去重结果不同”的现象。

典型例子是 `BigDecimal`：

```java
BigDecimal a = new BigDecimal("1.0");
BigDecimal b = new BigDecimal("1.00");

System.out.println(a.equals(b));    // false，精度 scale 不同
System.out.println(a.compareTo(b)); // 0，数值相等
```

因此：

```java
Set<BigDecimal> hashSet = new HashSet<>();
hashSet.add(new BigDecimal("1.0"));
hashSet.add(new BigDecimal("1.00"));
System.out.println(hashSet.size()); // 2

Set<BigDecimal> treeSet = new TreeSet<>();
treeSet.add(new BigDecimal("1.0"));
treeSet.add(new BigDecimal("1.00"));
System.out.println(treeSet.size()); // 1
```

## 底层机制

### 对象头与默认 hashCode

`Object.hashCode` 通常与对象身份有关，但 JVM 规范没有要求它必须是内存地址。不要把默认 hashCode 理解成稳定的物理地址。

### HashMap key 可变字段问题

如果对象作为 `HashMap` key 后，参与 `equals` 和 `hashCode` 的字段被修改，会导致定位桶变化，从而无法找到原 key。

```java
class User {
    String name;

    User(String name) {
        this.name = name;
    }

    @Override
    public boolean equals(Object o) {
        return o instanceof User && Objects.equals(name, ((User) o).name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name);
    }
}

Map<User, String> map = new HashMap<>();
User user = new User("a");
map.put(user, "value");

user.name = "b";
System.out.println(map.get(user)); // 可能为 null
```

工程上，Map key 应使用不可变对象，或者至少不要修改参与相等和哈希计算的字段。

## 常见追问

### 为什么重写 equals 必须重写 hashCode？

因为散列表先用 `hashCode` 定位桶，再用 `equals` 判断逻辑相等。如果两个逻辑相等对象的 hashCode 不同，它们会落到不同桶里，`HashMap.get`、`HashSet.contains` 可能找不到。

### hashCode 相等，对象一定 equals 吗？

不一定。hashCode 是有限的 `int`，对象空间理论上无限，冲突必然可能发生。HashMap 允许冲突，冲突后依靠 `equals` 精确比较。

### equals 中用 `getClass` 还是 `instanceof`？

取决于相等语义。

- 用 `getClass`：只有完全同类对象才可能相等，适合值对象、不可变对象。
- 用 `instanceof`：允许子类对象参与相等，但继承层级中容易破坏对称性或传递性。

实体类如果可能被 ORM 代理，直接使用 `getClass` 可能受代理类影响；需要结合框架实践谨慎设计。

### compareTo 返回 0 但 equals false 会怎样？

在 `TreeSet`、`TreeMap` 中，排序比较为 0 通常被视为同一个 key，会导致后加入元素被认为重复或覆盖旧值。`BigDecimal` 是典型例子。

### 为什么不要用减法写 compareTo？

整数减法可能溢出，导致比较结果方向错误。应使用 `Integer.compare`、`Long.compare` 或 `Comparator.comparing`。

## 代码示例

### 正确的值对象

```java
public final class OrderKey {
    private final long shopId;
    private final String orderNo;

    public OrderKey(long shopId, String orderNo) {
        this.shopId = shopId;
        this.orderNo = Objects.requireNonNull(orderNo, "orderNo");
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || getClass() != o.getClass()) {
            return false;
        }
        OrderKey orderKey = (OrderKey) o;
        return shopId == orderKey.shopId && orderNo.equals(orderKey.orderNo);
    }

    @Override
    public int hashCode() {
        return Objects.hash(shopId, orderNo);
    }
}
```

### 多字段排序

```java
List<Order> orders = queryOrders();
orders.sort(Comparator
        .comparing(Order::getCreateTime, Comparator.nullsLast(Comparator.naturalOrder()))
        .thenComparing(Order::getOrderId));
```

### BigDecimal 比较

```java
BigDecimal amount = new BigDecimal("1.00");

if (amount.compareTo(BigDecimal.ZERO) > 0) {
    // 金额大于 0
}

// 判断数值相等时通常使用 compareTo，不直接使用 equals
if (amount.compareTo(new BigDecimal("1.0")) == 0) {
    // true
}
```

## 工程实践

- 值对象建议设计为不可变，并同时重写 `equals` 和 `hashCode`。
- Map key、Set 元素不要使用会变化的字段参与哈希计算。
- 使用 IDE 或 Lombok 生成 `equals`/`hashCode` 时，要检查字段选择是否符合业务身份。
- JPA/MyBatis 实体的相等性要谨慎，不能简单把所有字段都纳入，尤其是数据库自增 ID 未生成前。
- 排序时优先使用 `Comparator.comparing`、`thenComparing`，可读性更好。
- 整数比较不要用减法，浮点比较要考虑精度和 NaN。
- 金额相等判断根据业务语义选择 `compareTo` 或统一 scale 后比较。
- 分布式系统中对象相等不应依赖 JVM 内对象身份，要使用业务主键。

## 易错点

- 只重写 `equals` 不重写 `hashCode`。
- 用可变对象作为 `HashMap` key，并修改参与哈希的字段。
- `compareTo` 用减法导致溢出。
- `BigDecimal.equals` 与 `compareTo` 语义不同。
- `TreeSet` 依赖比较器去重，不依赖 `equals` 去重。
- `equals` 里没有处理 `null` 或类型不匹配。
- `equals` 中使用浮点字段直接 `==`，忽略精度问题。
- 继承体系中粗暴使用 `instanceof`，破坏对称性。

## 自检清单

- 能说清 `==`、`equals`、`hashCode`、`compareTo` 分别解决什么问题。
- 能写出符合契约的 `equals` 和 `hashCode`。
- 能解释 HashMap 为什么需要同时使用 hashCode 和 equals。
- 能说明可变 key 的问题。
- 能说明 Comparable 与 Comparator 的区别。
- 能解释 compareTo 与 equals 不一致时 TreeSet 的行为。
- 能举出 BigDecimal 的经典陷阱。
