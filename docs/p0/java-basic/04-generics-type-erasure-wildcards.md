# 泛型、类型擦除、通配符

## 面试定位

泛型是 Java 集合、框架 API、序列化、反射和类型安全的基础。面试官常问：为什么 `List<String>` 不能赋值给 `List<Object>`、什么是类型擦除、`? extends` 和 `? super` 怎么选、泛型方法怎么写、为什么不能 `new T()`、运行期能不能拿到泛型类型。

优秀回答需要覆盖：

- 泛型提供编译期类型检查，减少强转和运行期类型错误。
- Java 泛型主要通过类型擦除实现，运行期大多没有具体泛型实参。
- 通配符处理的是泛型不变性和 API 的读写边界。
- 工程中要用泛型表达约束，但不要滥用复杂泛型牺牲可读性。

## 核心概念

### 为什么需要泛型

没有泛型时，集合只能保存 `Object`，取出时需要强转，错误会推迟到运行期。

```java
List list = new ArrayList();
list.add("hello");
list.add(1);

String value = (String) list.get(1); // ClassCastException
```

有泛型后，编译器提前检查类型：

```java
List<String> list = new ArrayList<>();
list.add("hello");
// list.add(1); // 编译失败

String value = list.get(0); // 不需要显式强转
```

泛型的主要收益：

- 编译期类型安全。
- 减少强制类型转换。
- API 语义更清楚。
- 支持通用容器、通用算法和框架扩展点。

### 泛型类

```java
public class Response<T> {
    private final int code;
    private final T data;

    public Response(int code, T data) {
        this.code = code;
        this.data = data;
    }

    public T getData() {
        return data;
    }
}
```

使用：

```java
Response<User> response = new Response<>(0, new User());
User user = response.getData();
```

### 泛型方法

泛型方法在返回类型前声明类型参数。

```java
public static <T> T first(List<T> list) {
    if (list == null || list.isEmpty()) {
        return null;
    }
    return list.get(0);
}
```

类型参数可以有上界：

```java
public static <T extends Comparable<T>> T max(List<T> values) {
    if (values == null || values.isEmpty()) {
        throw new IllegalArgumentException("empty values");
    }
    T max = values.get(0);
    for (T value : values) {
        if (value.compareTo(max) > 0) {
            max = value;
        }
    }
    return max;
}
```

### 泛型接口

```java
public interface Converter<S, T> {
    T convert(S source);
}

public class UserConverter implements Converter<UserDO, UserDTO> {
    @Override
    public UserDTO convert(UserDO source) {
        return new UserDTO(source.getId(), source.getName());
    }
}
```

## 类型擦除

Java 泛型是通过类型擦除实现的。编译后，大部分泛型类型信息会被擦除为原始类型或上界类型，并在必要位置插入强制类型转换。

```java
List<String> strings = new ArrayList<>();
strings.add("a");
String value = strings.get(0);
```

编译后近似为：

```java
List strings = new ArrayList();
strings.add("a");
String value = (String) strings.get(0);
```

如果类型参数有上界：

```java
class Box<T extends Number> {
    private T value;
}
```

擦除后 `T` 会变成上界 `Number`。

### 类型擦除带来的限制

#### 不能直接创建泛型数组

```java
// List<String>[] array = new List<String>[10]; // 编译失败
```

数组运行期知道元素类型并做协变检查，泛型运行期类型被擦除，两者结合容易破坏类型安全。

#### 不能直接 `new T()`

```java
class Factory<T> {
    // T create() {
    //     return new T(); // 编译失败
    // }
}
```

运行期不知道 `T` 具体是什么，也不知道构造方法。常见替代方案是传入 `Class<T>`、`Supplier<T>` 或工厂对象。

```java
public class Factory<T> {
    private final Supplier<T> supplier;

    public Factory(Supplier<T> supplier) {
        this.supplier = supplier;
    }

    public T create() {
        return supplier.get();
    }
}
```

#### 不能用基本类型作为泛型实参

```java
// List<int> values = new ArrayList<>(); // 编译失败
List<Integer> values = new ArrayList<>();
```

泛型要求引用类型，基本类型需要使用包装类型。

#### 不能用泛型实参做 instanceof

```java
Object value = new ArrayList<String>();
// if (value instanceof List<String>) {} // 编译失败
if (value instanceof List<?>) {
    // 可以判断是否是 List，但不能判断具体元素类型
}
```

#### 方法签名擦除冲突

```java
// 编译失败：擦除后两个方法都是 handle(List)
// void handle(List<String> values) {}
// void handle(List<Integer> values) {}
```

## 泛型不变性

即使 `String` 是 `Object` 的子类，`List<String>` 也不是 `List<Object>` 的子类。

```java
List<String> strings = new ArrayList<>();
// List<Object> objects = strings; // 编译失败
```

如果允许这样赋值，会破坏类型安全：

```java
// 假设允许
List<Object> objects = strings;
objects.add(1);
String s = strings.get(0); // 可能拿到 Integer
```

泛型默认不变，通配符用于表达受限的协变或逆变。

## 通配符

### `? extends T`

`? extends T` 表示某个未知类型，它是 `T` 或 `T` 的子类。适合生产数据，也就是从集合中读取为 `T`。

```java
public static double sum(List<? extends Number> values) {
    double result = 0;
    for (Number value : values) {
        result += value.doubleValue();
    }
    return result;
}
```

不能安全写入具体元素：

```java
List<? extends Number> values = new ArrayList<Integer>();
// values.add(1);      // 编译失败
// values.add(1L);     // 编译失败
values.add(null);      // 只有 null 可以
```

原因是编译器只知道这是某个 `Number` 子类型的列表，不知道到底是 `Integer`、`Long` 还是 `BigDecimal`。

### `? super T`

`? super T` 表示某个未知类型，它是 `T` 或 `T` 的父类。适合消费数据，也就是向集合写入 `T`。

```java
public static void addIntegers(List<? super Integer> values) {
    values.add(1);
    values.add(2);
}
```

读取时只能安全读成 `Object`：

```java
List<? super Integer> values = new ArrayList<Number>();
Object value = values.get(0);
```

### PECS 原则

PECS：Producer Extends, Consumer Super。

- 如果参数作为生产者，只从里面读数据，用 `? extends T`。
- 如果参数作为消费者，只向里面写数据，用 `? super T`。
- 如果既要读又要写同一种具体类型，通常不要用通配符，直接用 `List<T>`。

经典例子：

```java
public static <T> void copy(List<? extends T> src, List<? super T> dest) {
    for (T item : src) {
        dest.add(item);
    }
}
```

## 泛型边界

### 上界

```java
public static <T extends Number> double toDouble(T value) {
    return value.doubleValue();
}
```

多个上界：

```java
public static <T extends Number & Comparable<T>> T pick(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}
```

如果同时有类和接口，类必须放在第一个。

### 递归泛型

递归泛型常用于可比较对象、自类型链式调用等场景。

```java
public interface Entity<T extends Entity<T>> extends Comparable<T> {
}
```

这类设计表达力强，但复杂度高。面试能说出用途即可，工程中应避免滥用。

## 运行期泛型信息

虽然泛型大多被擦除，但某些声明位置的泛型签名会保存在 class 元数据中，反射可以读取。例如字段、父类、接口、方法返回值上的泛型声明。

```java
class UserRepository extends BaseRepository<User> {
}

Type type = UserRepository.class.getGenericSuperclass();
ParameterizedType parameterizedType = (ParameterizedType) type;
Type actualType = parameterizedType.getActualTypeArguments()[0];
System.out.println(actualType); // class User
```

框架常利用这个能力解析泛型，例如 JSON 反序列化、ORM Repository 泛型、Spring 的类型推断。但局部变量运行期一般拿不到泛型实参。

### TypeReference 模式

反序列化 `List<User>` 时只传 `List.class` 不够，因为元素类型丢失。常见方案是匿名子类捕获泛型签名。

```java
TypeReference<List<User>> typeRef = new TypeReference<List<User>>() {};
```

Jackson、Fastjson、Guava 等库都有类似设计。

## 常见追问

### 什么是类型擦除？

类型擦除是 Java 泛型的实现方式。编译器在编译期完成类型检查，生成字节码时把类型参数擦除为原始类型或上界类型，并在需要的位置插入强制类型转换。运行期通常不知道 `List<String>` 和 `List<Integer>` 的具体元素泛型。

### 为什么 `List<String>` 不能赋值给 `List<Object>`？

因为泛型是不变的。如果允许赋值，就能通过 `List<Object>` 往原本的 `List<String>` 中加入 `Integer`，破坏类型安全。

### `List<?>` 和原始类型 `List` 有什么区别？

`List<?>` 表示元素类型未知，但仍然是类型安全的，不能随意添加非 null 元素；原始类型 `List` 关闭了泛型检查，容易产生堆污染和运行期 `ClassCastException`。

### 什么是堆污染？

堆污染是指一个参数化类型变量引用了不符合其声明类型的对象。例如通过原始类型把 `Integer` 放入 `List<String>`，编译器无法完全阻止，后续读取时可能 `ClassCastException`。

```java
List<String> strings = new ArrayList<>();
List raw = strings;
raw.add(1);

String value = strings.get(0); // ClassCastException
```

### 泛型和数组有什么冲突？

数组是协变且运行期保留元素类型，泛型是不变且类型擦除。二者组合会让运行期类型检查和编译期泛型检查产生矛盾，所以 Java 禁止直接创建泛型数组。

## 代码示例

### 泛型 Repository

```java
public interface Repository<ID, E> {
    Optional<E> findById(ID id);

    void save(E entity);
}

public class UserRepository implements Repository<Long, User> {
    @Override
    public Optional<User> findById(Long id) {
        return Optional.empty();
    }

    @Override
    public void save(User entity) {
        // save user
    }
}
```

### PECS 使用

```java
public static <T> void merge(Collection<? extends T> source,
                             Collection<? super T> target) {
    for (T item : source) {
        target.add(item);
    }
}

List<Integer> integers = List.of(1, 2, 3);
List<Number> numbers = new ArrayList<>();
merge(integers, numbers);
```

### Class<T> 作为类型令牌

```java
public final class JsonCodec {
    public <T> T decode(String json, Class<T> type) {
        // 实际项目中委托 Jackson/Fastjson/Gson
        throw new UnsupportedOperationException();
    }
}
```

`Class<T>` 能解决简单对象的运行期类型问题，但不能完整表达 `List<User>` 这类嵌套泛型，需要 `Type` 或 `TypeReference`。

## 工程实践

- 集合、缓存、RPC 响应、转换器等 API 应使用泛型表达输入输出类型。
- 尽量避免原始类型 `List`、`Map`，除非与老接口兼容，并做好边界转换。
- 对只读入参使用 `? extends T`，对只写入参使用 `? super T`。
- 不要把泛型设计得过度复杂。超过两三个类型参数时，要重新审视抽象是否过度。
- JSON 反序列化嵌套泛型时使用 `TypeReference` 或框架提供的泛型类型对象。
- 公共 API 中通配符要谨慎，目标是让调用方更舒服，而不是展示类型技巧。
- 对 varargs 泛型方法要关注堆污染，必要时使用 `@SafeVarargs`，但前提是方法确实安全。
- 代码评审中看到 unchecked cast，要确认是否有运行期类型保证。

## 易错点

- 认为 `List<String>` 是 `List<Object>` 的子类。
- 把 `List<?>` 当成可以随意添加元素的列表。
- 忘记类型擦除导致不能 `new T()`、不能 `T.class`、不能 `instanceof List<String>`。
- 使用原始类型绕过泛型检查，埋下运行期类型错误。
- `? extends` 只能安全读，不能安全写具体值。
- `? super` 可以安全写 `T`，但读出来通常只能当 `Object`。
- 泛型方法的 `<T>` 写在返回类型前，不是写在方法名后。
- 用复杂递归泛型让简单业务代码难以理解。

## 自检清单

- 能解释泛型的作用和类型擦除。
- 能说明 `List<String>` 不能赋值给 `List<Object>` 的原因。
- 能区分原始类型、无界通配符和具体泛型。
- 能正确使用 `? extends`、`? super` 和 PECS。
- 能说明为什么不能 `new T()` 和 `new List<String>[10]`。
- 能写出泛型类、泛型方法、泛型接口。
- 能说明运行期哪些泛型信息可能通过反射获取。
