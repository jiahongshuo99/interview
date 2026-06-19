# 双亲委派模型及破坏场景

## 面试定位

双亲委派是类加载机制中最常考的设计题。面试官通常不只想听“先交给父加载器”，还会追问为什么要这样设计，哪些场景会破坏它，Tomcat、JDBC、SPI、OSGi、热部署为什么需要自定义类加载。

回答框架：

1. 类加载器收到加载请求后，先委派给父类加载器。
2. 父加载器无法完成时，子加载器才尝试自己加载。
3. 这样可以保证核心类库安全和类的唯一性。
4. 破坏双亲委派通常是为了隔离、热部署、插件化或让父加载器使用子加载器可见的类。

## 类加载器层次

HotSpot 常见类加载器：

- Bootstrap ClassLoader：启动类加载器，加载核心类库，例如 `java.lang.*`。
- Extension ClassLoader：扩展类加载器，JDK 9 模块化后传统扩展机制变化较大。
- Application ClassLoader：应用类加载器，加载 classpath 下的业务类。
- Custom ClassLoader：自定义类加载器，用于插件、容器、热部署、加密加载等。

JDK 9 以后平台类加载器 Platform ClassLoader 替代了一部分扩展类加载器角色。面试中可以说明常见口径以 JDK 8 为主，同时知道 JDK 9 后有模块化变化。

## 双亲委派流程

`ClassLoader.loadClass` 的典型流程：

1. 检查类是否已经加载过。
2. 如果没有，委派父加载器加载。
3. 父加载器加载失败后，调用当前加载器的 `findClass`。
4. 如果仍失败，抛出 `ClassNotFoundException`。

简化伪代码：

```java
protected Class<?> loadClass(String name, boolean resolve) {
    Class<?> c = findLoadedClass(name);
    if (c == null) {
        try {
            if (parent != null) {
                c = parent.loadClass(name, false);
            } else {
                c = findBootstrapClassOrNull(name);
            }
        } catch (ClassNotFoundException ignored) {
            c = findClass(name);
        }
    }
    if (resolve) {
        resolveClass(c);
    }
    return c;
}
```

实践建议：自定义类加载器时通常重写 `findClass`，不要随意重写 `loadClass`，否则容易破坏双亲委派。

## 为什么需要双亲委派

### 保证核心类库安全

如果没有双亲委派，应用可以自定义一个 `java.lang.String` 并优先加载，核心类就可能被篡改。双亲委派保证核心类优先由 Bootstrap ClassLoader 加载。

### 保证类唯一性

同一个全限定名的核心类由顶层加载器统一加载，避免各个子加载器重复加载出多个版本，造成类型混乱。

### 隔离职责

不同类加载器负责不同范围：

- 启动类加载器负责 JDK 核心类。
- 应用类加载器负责应用 classpath。
- 自定义类加载器负责插件或容器内部类。

## 类身份判断

JVM 判断两个类是否相同，不只看全限定名，还要看加载它们的类加载器。

```text
类身份 = 类全限定名 + 定义类加载器
```

因此，同一个 `com.demo.User` 如果由两个不同类加载器加载，会被 JVM 视为两个不同类型。常见后果：

- `ClassCastException`。
- `instanceof` 返回 false。
- 反射方法参数类型不匹配。
- 静态变量隔离成多份。

## 破坏双亲委派的典型场景

### 1. JNDI、JDBC、SPI

一些 Java 核心 API 位于启动类加载器可见范围，但具体实现类在应用 classpath 中。父加载器需要反过来加载子加载器可见的实现类，这与纯粹的父优先模型冲突。

典型方案是线程上下文类加载器：

```java
ClassLoader loader = Thread.currentThread().getContextClassLoader();
```

SPI 如 `ServiceLoader` 会使用上下文类加载器查找实现。

### 2. Tomcat Web 应用隔离

Tomcat 需要做到：

- 每个 Web 应用可以加载自己的依赖版本。
- Web 应用之间类隔离。
- 容器公共类由公共加载器加载。
- Web 应用优先加载 `WEB-INF/classes` 和 `WEB-INF/lib` 中的类。

因此 Tomcat 的 WebAppClassLoader 对部分路径采用子优先策略，破坏了标准双亲委派。

### 3. 热部署和插件化

热部署需要卸载旧版本类并加载新版本类。因为类卸载依赖类加载器可回收，常见做法是每次发布或插件加载使用新的类加载器。

插件化常见诉求：

- 插件之间依赖隔离。
- 插件可动态安装、卸载、升级。
- 插件使用不同版本第三方库。

这些都需要自定义类加载器策略。

### 4. OSGi

OSGi 使用更复杂的网状类加载模型，模块之间通过显式导入导出包建立依赖，不是简单树状父子关系。

### 5. 框架增强和字节码生成

动态代理、CGLIB、Byte Buddy、Javassist 等会在运行时生成类。生成类要放到合适的类加载器中，否则可能访问不到目标类或导致类泄漏。

## 线程上下文类加载器

线程上下文类加载器解决的问题是：父加载器加载的代码需要找到子加载器中的实现。

典型例子：

```java
ServiceLoader<MyService> loader = ServiceLoader.load(MyService.class);
```

`ServiceLoader` 是 JDK 类，通常由启动类加载器或平台类加载器加载；业务实现类在应用 classpath 中，需要通过上下文类加载器加载。

风险：

- 线程池复用线程时，上下文类加载器未恢复，可能导致类加载器泄漏。
- 容器或插件环境下错误设置上下文类加载器，可能加载到错误版本的类。

## 自定义类加载器要点

最小实现通常重写 `findClass`：

```java
public class MyClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] bytes = loadBytes(name);
        return defineClass(name, bytes, 0, bytes.length);
    }
}
```

注意事项：

- 优先复用 `loadClass` 的父委派逻辑。
- 明确哪些包父优先，哪些包子优先。
- 避免加载 JDK 核心包和公共 API 包。
- 关注类加载器生命周期，避免静态引用、线程、ThreadLocal 持有类加载器。
- 生成类过多时注意 Metaspace。

## 排查方法

类加载冲突常见排查步骤：

1. 打印类加载器：

```java
System.out.println(obj.getClass());
System.out.println(obj.getClass().getClassLoader());
System.out.println(Thread.currentThread().getContextClassLoader());
```

2. 查看类加载日志：

```bash
java -verbose:class ...
# JDK 9+
java -Xlog:class+load=info,class+unload=info ...
```

3. 线上进程检查：

```bash
jcmd <pid> VM.classloader_stats
jcmd <pid> VM.system_properties
jcmd <pid> Thread.print
```

4. 检查依赖树：

```bash
mvn dependency:tree
gradle dependencies
```

5. 对 `ClassCastException`，重点看两个类的类加载器和来源 Jar。

## 常见追问

### 双亲委派能防止所有核心类被伪造吗？

不能只靠双亲委派解决所有安全问题，但它能保证核心类优先由上层加载器加载。JVM 对 `java.*` 等包也有额外限制，不能简单自定义核心包类覆盖 JDK。

### 为什么 Tomcat 要破坏双亲委派？

因为 Web 应用需要隔离依赖版本，并优先加载自己 `WEB-INF` 下的类。如果完全父优先，多个应用会被公共 classpath 中的类绑定，无法做到应用级隔离。

### 为什么 SPI 要用线程上下文类加载器？

SPI 接口通常由 JDK 或公共加载器加载，但实现类在应用 classpath。父加载器看不到子加载器的类，所以需要通过线程上下文类加载器反向查找实现。

### 自定义类加载器为什么可能造成 Metaspace 泄漏？

类元数据能否卸载取决于类加载器是否可回收。如果线程、静态变量、ThreadLocal、缓存等持有类加载器或其加载的类，就会导致旧类无法卸载。

## 易错点

- 把“双亲”理解为两个父亲；这里指父类加载器链。
- 认为父加载器是 Java 继承关系中的父类，实际通常是组合关系。
- 认为所有类加载器都严格遵守双亲委派，忽略容器和插件场景。
- 认为类名相同就一定是同一个类，忽略定义类加载器。
- 自定义类加载器时直接重写 `loadClass` 且不保留父委派，容易引发核心类冲突。
- 忽略线程上下文类加载器在线程池中的清理问题。

## 项目表达

可以这样表达：

> 我们在插件化能力里遇到过依赖冲突。宿主和插件都依赖同一个包但版本不同，完全父优先会导致插件使用宿主版本，完全子优先又可能加载出两份公共 API。最后我们把 API 包固定由父加载器加载，插件私有实现和三方依赖由插件类加载器加载，并在线程执行前后设置和恢复上下文类加载器，避免线程池复用造成泄漏。

Tomcat 类问题：

> 排查 `ClassCastException` 时，类名完全一致但强转失败。我们打印类加载器后发现一个对象来自 WebAppClassLoader，另一个来自公共加载器。最终调整 Jar 放置位置，让公共接口只在公共加载器加载，业务实现放在 Web 应用内。

## 自检清单

- 能否画出类加载器父委派流程？
- 能否说明双亲委派解决了什么问题？
- 能否解释类身份为什么包含类加载器？
- 能否列出 Tomcat、SPI、插件化破坏双亲委派的原因？
- 能否说明线程上下文类加载器解决什么问题？
- 能否给出类加载冲突和 Metaspace 泄漏的排查方法？
