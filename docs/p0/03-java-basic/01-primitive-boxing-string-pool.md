# 基本类型、包装类型、自动拆装箱、字符串常量池

## 面试定位

这一题通常考察 Java 语言基础是否扎实。面试官会从 `int` 和 `Integer` 的区别问起，逐步追到自动拆装箱、包装类型缓存、`==` 与 `equals`、`String` 不可变、字符串常量池和 `intern()`。后端面试中它还会延伸到对象内存、空指针、Map key、日志拼接、性能优化和线上隐患。

需要能回答三层内容：

- 语义层：基本类型存值，包装类型是对象；`String` 是不可变引用类型。
- 机制层：自动拆装箱由编译器改写，包装类型有缓存，字符串字面量进入字符串常量池。
- 工程层：避免拆箱空指针、避免用 `==` 比较包装对象和字符串、理解字符串拼接与内存分配。

## 核心概念

### Java 8 种基本类型

| 类型 | 位数 | 默认值 | 包装类型 | 典型场景 |
| --- | --- | --- | --- | --- |
| `byte` | 8 | `0` | `Byte` | 二进制数据、小范围整数 |
| `short` | 16 | `0` | `Short` | 少见，节省内存时可能使用 |
| `int` | 32 | `0` | `Integer` | 默认整数类型 |
| `long` | 64 | `0L` | `Long` | ID、时间戳、计数 |
| `float` | 32 | `0.0f` | `Float` | 单精度浮点，业务中较少直接用于金额 |
| `double` | 64 | `0.0d` | `Double` | 双精度浮点，统计计算 |
| `char` | 16 | `'\u0000'` | `Character` | UTF-16 code unit |
| `boolean` | JVM 规范未固定 | `false` | `Boolean` | 条件标记 |

局部变量没有默认值，必须显式初始化；成员变量和数组元素有默认值。

### 基本类型与包装类型

基本类型不是对象，不能为 `null`，不能作为泛型实参。包装类型是对象，可以为 `null`，可以放入集合、泛型、反射调用和需要对象的 API 中。

```java
int a = 1;
Integer b = 1;

List<Integer> list = new ArrayList<>();
list.add(a); // 自动装箱为 Integer.valueOf(a)
```

常见差异：

- 存储语义不同：基本类型直接表示值，包装类型变量保存对象引用。
- 空值能力不同：包装类型可表达“不存在”，但拆箱时可能触发 NPE。
- 比较方式不同：基本类型可用 `==` 比较值，包装类型应优先用 `equals` 或先转为基本类型。
- 泛型限制不同：`List<int>` 非法，只能使用 `List<Integer>`。
- 性能不同：大量装箱对象会带来分配、GC 和缓存局部性问题。

### 自动装箱与自动拆箱

自动装箱、拆箱是编译器语法糖，不是 JVM 在运行时凭空识别。

```java
Integer x = 10;      // 编译后近似为 Integer.valueOf(10)
int y = x;           // 编译后近似为 x.intValue()
Integer z = x + 1;   // x.intValue() + 1，再 Integer.valueOf(...)
```

典型风险：

```java
Integer count = null;
int value = count; // NullPointerException
```

在三目运算、算术运算、比较运算中尤其容易隐式拆箱：

```java
Integer a = null;
Integer b = 1;

boolean r1 = a == b;     // 比较引用，不拆箱
boolean r2 = a == 1;     // a 拆箱，NPE
int r3 = true ? a : b;   // 结果类型可能触发拆箱，NPE
```

### 包装类型缓存

部分包装类型的 `valueOf` 会使用缓存，避免频繁创建小范围对象。

常见缓存范围：

- `Byte`：全部缓存，范围 `-128` 到 `127`。
- `Short`：默认缓存 `-128` 到 `127`。
- `Integer`：默认缓存 `-128` 到 `127`，上界可通过 JVM 参数调整。
- `Long`：默认缓存 `-128` 到 `127`。
- `Character`：默认缓存 `0` 到 `127`。
- `Boolean`：缓存 `TRUE` 和 `FALSE`。
- `Float`、`Double`：通常没有这种整数小值缓存。

```java
Integer a = 127;
Integer b = 127;
System.out.println(a == b); // true，命中 IntegerCache

Integer c = 128;
Integer d = 128;
System.out.println(c == d); // false，通常是两个对象

System.out.println(c.equals(d)); // true
```

面试中要强调：缓存是 `valueOf` 的行为，`new Integer(127)` 会创建新对象；并且写业务代码不应依赖包装对象的引用相等。

### String 的基本特性

`String` 是不可变对象。所谓不可变，是指对象创建后其内容不可变，而不是引用变量不能重新指向别的对象。

```java
String s = "abc";
s = s + "d"; // s 指向新字符串 "abcd"，原 "abc" 没有被修改
```

不可变带来的价值：

- 可以安全作为 `HashMap` key，哈希值可缓存。
- 字符串常量池可以复用对象。
- 天然适合作为配置、路径、类名、SQL 片段等只读数据。
- 多线程共享更安全。

JDK 8 中 `String` 内部主要是 `char[]`；JDK 9 之后引入 Compact Strings，内部通常是 `byte[]` 加编码标记，以节省 Latin-1 字符串的内存。面试回答时可以说明版本差异，不要死背某个字段。

## 字符串常量池

字符串常量池用于复用字符串字面量和 `intern()` 后的字符串对象。JDK 7 之后，字符串常量池位于堆中；早期版本相关实现与永久代有关。

```java
String a = "hello";
String b = "hello";
System.out.println(a == b); // true，指向常量池同一对象

String c = new String("hello");
System.out.println(a == c); // false，c 是堆上的新对象
System.out.println(a == c.intern()); // true
```

### 字面量、new、intern 的区别

```java
String s1 = "ab";
String s2 = "a" + "b";
String s3 = new String("ab");
String s4 = s3.intern();

System.out.println(s1 == s2); // true，编译期常量折叠
System.out.println(s1 == s3); // false，new 创建新对象
System.out.println(s1 == s4); // true，intern 返回池中引用
```

变量参与拼接时，通常在运行期创建新对象：

```java
String x = "a";
String y = "b";
String z = x + y;

System.out.println(z == "ab"); // false，通常不是同一引用
System.out.println(z.equals("ab")); // true
```

`final` 常量如果编译期可确定，则可能被常量折叠：

```java
final String x = "a";
final String y = "b";
String z = x + y;

System.out.println(z == "ab"); // true
```

## 底层机制

### 自动装箱的字节码视角

源代码：

```java
Integer a = 1;
int b = a + 2;
```

编译器近似改写为：

```java
Integer a = Integer.valueOf(1);
int b = a.intValue() + 2;
```

所以：

- 装箱不一定创建对象，取决于 `valueOf` 是否命中缓存。
- 拆箱需要调用对象方法，目标为 `null` 时抛 NPE。
- 算术表达式中的包装类型多数会拆箱成基本类型计算。

### 字符串拼接

常量拼接可在编译期折叠：

```java
String s = "a" + "b" + "c"; // 编译后可视为 "abc"
```

变量拼接在不同 JDK 中实现不同：

- JDK 8 常见实现是编译成 `StringBuilder` 链式 `append`。
- JDK 9+ 常见实现使用 `invokedynamic` 和 `StringConcatFactory`。

工程实践中，不要把所有拼接都机械改成 `StringBuilder`。循环内大量拼接、热点路径拼接需要关注；普通日志和少量拼接可读性更重要。

## 常见追问

### `int` 和 `Integer` 有什么区别？

`int` 是基本类型，存放数值，不能为 `null`；`Integer` 是包装类型，变量保存对象引用，可以为 `null`，可以用于泛型和集合。二者之间的自动转换由编译器插入 `Integer.valueOf` 和 `intValue` 实现。业务中如果值必然存在并且计算频繁，优先用 `int`；如果需要表达缺失、参与泛型或与框架绑定，常用 `Integer`。

### `Integer a = 128; Integer b = 128; a == b` 是什么？

通常是 `false`。因为自动装箱调用 `Integer.valueOf(128)`，默认缓存范围只到 `127`，两个变量通常引用不同对象。`Integer a = 127; Integer b = 127; a == b` 通常是 `true`，但业务代码不要依赖包装对象引用相等，应使用 `Objects.equals(a, b)`。

### 为什么包装类型拆箱会 NPE？

拆箱本质是调用实例方法，例如 `Integer.intValue()`。如果引用是 `null`，调用方法就会抛出 `NullPointerException`。

### `String s = new String("abc")` 创建几个对象？

不能脱离上下文绝对回答。若类加载时常量池中还没有 `"abc"`，字面量会使常量池中存在一个 `"abc"`，`new String("abc")` 再在堆上创建一个新的 `String` 对象；若常量池中已存在 `"abc"`，则只创建堆上的新对象。面试里要强调上下文和类加载时机。

### `String` 为什么设计成不可变？

主要为了安全、复用和性能。不可变对象可安全共享，适合作为 Map key，可缓存 hash 值，也支持字符串常量池复用；同时减少被外部修改导致安全问题，例如类名、文件路径、网络地址、权限标识等。

### `StringBuilder` 和 `StringBuffer` 区别？

`StringBuilder` 非线程安全，性能更好，常用于局部变量拼接；`StringBuffer` 方法加了同步，线程安全但开销更高。实际后端代码中，局部拼接优先使用 `StringBuilder` 或直接使用字符串拼接，由编译器优化；跨线程共享可变字符串结构时需要额外同步，但这种场景很少。

## 代码示例

### 自动拆箱空指针

```java
public class BoxingNpeDemo {
    public static void main(String[] args) {
        Integer retryCount = getRetryCount();

        // 错误：retryCount 可能为 null，比较时会拆箱。
        // if (retryCount > 0) {}

        if (retryCount != null && retryCount > 0) {
            System.out.println("need retry");
        }
    }

    private static Integer getRetryCount() {
        return null;
    }
}
```

### 安全比较包装类型

```java
Integer a = 1000;
Integer b = 1000;

System.out.println(a == b);              // false
System.out.println(Objects.equals(a, b)); // true
```

### 安全比较字符串

```java
String status = getStatus();

if ("SUCCESS".equals(status)) {
    // 避免 status 为 null 时 status.equals("SUCCESS") 触发 NPE
}
```

### 循环拼接

```java
List<String> names = List.of("a", "b", "c");
StringBuilder builder = new StringBuilder();
for (String name : names) {
    if (builder.length() > 0) {
        builder.append(',');
    }
    builder.append(name);
}
String result = builder.toString();
```

## 工程实践

- DTO、数据库字段、RPC 入参中如果字段可缺失，使用包装类型表达 `null`，并在边界层完成校验和默认值处理。
- 内部计算和循环计数优先使用基本类型，减少装箱和空指针风险。
- 比较包装类型统一使用 `Objects.equals`，避免缓存范围导致的 `==` 偶现正确。
- 比较字符串时使用 `"CONST".equals(value)` 或 `Objects.equals(a, b)`。
- 金额不要使用 `float` 或 `double` 表示，优先使用 `BigDecimal` 或以分为单位的 `long`。
- 热点循环中避免无意识装箱，例如 `Map<Long, Long>` 的频繁加减可能产生大量对象。
- 不要滥用 `String.intern()`。它会影响字符串池，适合有限、重复度高、生命周期长且经过评估的字符串，不适合无限增长的用户输入。
- 日志参数优先使用占位符，例如 `log.info("uid={}, orderId={}", uid, orderId)`，避免日志级别关闭时提前拼接字符串。

## 易错点

- `Integer` 能为 `null`，参与算术和大小比较时会自动拆箱，可能 NPE。
- `==` 比较包装类型引用，不是稳定的值比较方式。
- `new String("x")` 与字面量 `"x"` 不是同一引用。
- `String` 不可变，但变量引用可重新赋值。
- 字符串常量池不是永久代概念的固定答案，JDK 版本不同。
- `final String` 只有在编译期可确定时才参与常量折叠。
- `BigDecimal` 用 `new BigDecimal("0.1")` 或 `BigDecimal.valueOf(0.1)`，不要直接 `new BigDecimal(0.1)`。
- `Long` 类型 ID 与字面量比较时也有缓存和拆箱问题。

## 自检清单

- 能说清 `int` 与 `Integer` 在空值、泛型、比较、性能上的区别。
- 能解释自动装箱对应 `valueOf`，自动拆箱对应 `xxxValue`。
- 能判断包装类型缓存导致的 `==` 结果。
- 能解释拆箱 NPE 的触发场景。
- 能区分字符串字面量、`new String`、`intern()`。
- 能说明字符串拼接在编译期和运行期的差异。
- 能给出后端代码中避免包装类型和字符串比较问题的实践。
