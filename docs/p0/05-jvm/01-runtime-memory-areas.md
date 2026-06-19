# JVM 内存区域：堆、栈、方法区、程序计数器、本地方法栈

## 面试定位

JVM 内存区域是 JVM 面试的入口题，考察你是否能把 Java 代码执行过程、对象分配、线程隔离、GC 管理范围和 OOM 现象串起来。回答时不要只背五块区域，应该说明哪些区域线程私有、哪些区域线程共享，哪些会触发 GC，哪些更容易出现 `OutOfMemoryError` 或 `StackOverflowError`，以及 JDK 版本变化。

一个稳定的回答框架：

1. JVM 运行时数据区分为线程共享和线程私有。
2. 线程共享区域主要是堆和方法区，是 GC 重点管理区域。
3. 线程私有区域包括虚拟机栈、本地方法栈、程序计数器，生命周期随线程创建和销毁。
4. JDK 8 以后 HotSpot 用 Metaspace 替代永久代，方法区是规范概念，永久代和元空间是 HotSpot 的实现。

## 总览

| 区域 | 是否线程共享 | 主要内容 | 常见异常 | GC 相关性 |
| --- | --- | --- | --- | --- |
| 堆 | 共享 | 对象实例、数组、字符串对象等 | `OutOfMemoryError: Java heap space` | GC 主战场 |
| 虚拟机栈 | 私有 | 栈帧、局部变量表、操作数栈、动态链接、返回地址 | `StackOverflowError`、`OutOfMemoryError` | 一般不被 GC 管理 |
| 方法区 | 共享 | 类元数据、运行时常量池、字段和方法信息、静态变量引用等 | `OutOfMemoryError: Metaspace` | 有类卸载 |
| 程序计数器 | 私有 | 当前线程执行的字节码行号指示器 | 基本不会 OOM | 不需要 GC |
| 本地方法栈 | 私有 | Native 方法调用栈 | `StackOverflowError`、`OutOfMemoryError` | 一般不被 GC 管理 |

## 堆

堆是 JVM 中最大的一块内存区域，主要存放对象实例和数组。绝大多数对象都会在堆上分配，堆也是垃圾收集器最主要的管理区域。

常见分区口径：

- 新生代：Eden、Survivor From、Survivor To。
- 老年代：长期存活对象、大对象、晋升对象。
- JDK 7 之前字符串常量池常见于永久代，JDK 7 起逐步移入堆；面试中要避免把字符串常量池永远说成方法区。
- G1/ZGC 等现代收集器对堆的物理组织和传统分代不完全一样，但逻辑上仍会关注新生代、老年代、对象年龄、晋升和回收。

堆相关参数：

```bash
-Xms<size>      # 初始堆大小
-Xmx<size>      # 最大堆大小
-Xmn<size>      # 新生代大小，部分收集器或版本下不推荐直接依赖
-XX:NewRatio=N  # 老年代与新生代比例
-XX:SurvivorRatio=N
```

堆常见问题：

- 对象分配速度过快导致 Young GC 频繁。
- 老年代增长导致 Full GC 或 Mixed GC。
- 缓存、集合、队列、ThreadLocal、监听器等持有对象引用导致内存泄漏。
- 大对象直接进入老年代，造成空间碎片或晋升失败。

## 虚拟机栈

虚拟机栈描述 Java 方法执行的线程内存模型。每个线程创建时都会有自己的栈，每次方法调用会创建一个栈帧，方法执行结束后栈帧出栈。

栈帧通常包含：

- 局部变量表：存放基本类型、对象引用、`returnAddress`。
- 操作数栈：字节码指令执行时的临时计算区。
- 动态链接：保存当前方法所属类的运行时常量池/类元数据引用，用于把字节码方法调用中的符号引用按需解析为直接引用。
- 方法返回地址：方法正常返回或异常返回后继续执行的位置。

常见参数：

```bash
-Xss<size>      # 每个线程的栈大小
```

栈相关问题：

- 递归过深通常触发 `StackOverflowError`。
- 线程数过多、每个线程栈过大，可能触发无法创建新线程或 native memory OOM。
- 局部变量表里的对象引用只要栈帧仍可达，就会影响对象是否可回收。

面试易错点：虚拟机栈不是存对象本体，而是存对象引用和方法执行上下文；对象实例主要在堆上。

## 方法区

方法区是 JVM 规范定义的运行时数据区，用于存储已被虚拟机加载的类型信息、常量、静态变量、即时编译后的代码缓存等。HotSpot 在 JDK 8 之前用永久代实现方法区的一部分，JDK 8 起用本地内存中的 Metaspace 替代永久代。

需要区分：

- 方法区：JVM 规范概念。
- 永久代：HotSpot JDK 7 及以前的实现方式。
- 元空间：HotSpot JDK 8 以后的实现方式，使用本地内存。
- 运行时常量池：方法区的一部分，用于存放编译期生成的字面量和符号引用等。
- 字符串常量池：HotSpot 中 JDK 7 起主要在堆中，不能简单等同于运行时常量池。

Metaspace 常见参数：

```bash
-XX:MetaspaceSize=<size>       # 触发 Full GC 的初始阈值，不是硬上限
-XX:MaxMetaspaceSize=<size>    # 元空间最大值
-XX:CompressedClassSpaceSize=<size>
```

常见问题：

- 动态生成类过多，例如 CGLIB、Byte Buddy、JDK Proxy、Groovy、脚本引擎、热部署。
- 类加载器泄漏，导致类元数据无法卸载。
- `MaxMetaspaceSize` 没有限制时，本地内存可能被持续吃掉。

## 程序计数器

程序计数器是线程私有的小内存区域，可以理解为当前线程正在执行的字节码行号指示器。多线程切换后，线程需要恢复到正确的执行位置，程序计数器就保存了这个上下文。

特点：

- 执行 Java 方法时，记录正在执行的字节码指令地址。
- 执行 Native 方法时，值通常为空或未定义，具体取决于虚拟机实现。
- 是 JVM 规范中唯一没有规定任何 `OutOfMemoryError` 情况的区域。

面试回答时可以强调：程序计数器存在的核心原因是线程切换和字节码解释执行需要记录下一条指令位置。

## 本地方法栈

本地方法栈服务于 Native 方法调用，例如 JVM 调用 C/C++ 实现的底层能力、操作系统接口或第三方 native 库。HotSpot 中虚拟机栈和本地方法栈在实现上联系较紧密，但规范上二者概念不同。

常见风险：

- JNI 调用栈过深。
- native 库分配内存未释放。
- 线程过多导致 native stack 或系统线程资源耗尽。

## 底层机制串联

下面用一次普通方法调用串联各区域：

```java
public class UserService {
    private static final String TYPE = "member";

    public User create(String name) {
        User user = new User(name);
        return user;
    }
}
```

执行过程可以这样理解：

1. `UserService` 类加载后，类元数据进入方法区，常量、方法描述、字段描述等也与方法区相关。
2. 线程调用 `create` 方法时，虚拟机栈压入一个栈帧。
3. 栈帧中的局部变量表保存 `name` 引用、`user` 引用等。
4. `new User(name)` 创建的 `User` 对象实例主要分配在堆上。
5. 程序计数器记录该线程当前执行到哪条字节码。
6. 如果调用 native 方法，则会涉及本地方法栈。

## 排查和调优方法

常见定位路径：

1. 堆 OOM：先保存 heap dump，再用 MAT、JProfiler、VisualVM 或 Arthas 分析大对象、支配树、GC Roots 引用链。
2. Metaspace OOM：检查动态类生成、类加载器数量、热部署、代理框架，配合 `jcmd VM.classloader_stats`。
3. 栈溢出：看异常栈是否递归或循环调用；必要时调大 `-Xss`，但更常见是修复代码。
4. 线程创建失败：检查线程数、`-Xss`、系统 `ulimit`、容器内存、native memory。
5. GC 问题：开启 GC 日志，结合堆大小、晋升速率、对象存活率分析。

常用命令：

```bash
jcmd <pid> VM.flags
jcmd <pid> VM.native_memory summary
jcmd <pid> GC.heap_info
jcmd <pid> Thread.print
jmap -dump:format=b,file=heap.hprof <pid>
jstat -gcutil <pid> 1000 10
```

## 常见追问

### 为什么堆是 GC 主战场？

因为对象实例主要分配在堆上，生命周期差异很大，且堆是线程共享区域。GC 的主要目标就是找出堆中不再被引用的对象并回收空间。

### 方法区会不会 GC？

会，但回收条件更苛刻，主要包括废弃常量和无用类。类卸载通常要求该类所有实例都被回收、加载该类的类加载器可回收、对应 `Class` 对象不可达。

### 栈会不会发生 OOM？

会。单线程栈深度过深通常是 `StackOverflowError`；线程过多或栈空间申请失败可能是 `OutOfMemoryError`，例如无法创建新的 native thread。

### 直接内存属于 JVM 运行时数据区吗？

直接内存不是 JVM 运行时数据区的一部分，但被 Java 程序广泛使用，例如 NIO `DirectByteBuffer`。它受本地内存限制，常用 `-XX:MaxDirectMemorySize` 控制，问题表现可能是 direct buffer memory OOM。

## 易错点

- 把方法区、永久代、元空间混为一谈。
- 认为所有常量都在方法区，忽略字符串常量池在 HotSpot JDK 7 以后主要位于堆。
- 认为栈里存对象本体，实际栈里主要存局部变量和对象引用。
- 忽略程序计数器线程私有，且基本不会 OOM。
- 把堆 OOM、Metaspace OOM、Direct Memory OOM、无法创建线程都笼统说成内存不够。
- 只背区域，不说明线程共享/私有和生命周期。

## 项目表达

可以这样表达一次真实排查：

> 我们线上遇到过一次内存持续上涨的问题。首先通过 GC 日志确认老年代占用只升不降，再用 `jmap` 导出 heap dump，用 MAT 看 dominator tree，发现某个本地缓存 Map 持有大量业务对象。结合代码确认缓存没有淘汰策略，最终增加最大容量和过期时间，并补充指标监控。这个问题本质上发生在堆内存区域，不是栈或元空间问题。

如果是 Metaspace：

> 某次服务因为动态脚本频繁发布出现 `Metaspace` 增长。我们通过 `jcmd VM.classloader_stats` 看到类加载器数量异常，定位到脚本引擎每次加载后没有释放类加载器引用。修复后限制脚本类加载器生命周期，并设置 `MaxMetaspaceSize` 防止无限吃本地内存。

## 自检清单

- 能否说清五大区域哪些线程共享、哪些线程私有？
- 能否区分方法区、永久代、元空间？
- 能否解释为什么程序计数器几乎不会 OOM？
- 能否说清栈帧里有哪些结构？
- 能否把堆 OOM、栈溢出、Metaspace OOM、直接内存 OOM 分开定位？
- 能否结合一次方法调用说明对象、引用、类元数据分别在哪里？
