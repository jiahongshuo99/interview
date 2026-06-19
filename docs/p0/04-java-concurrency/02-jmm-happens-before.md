# JMM、happens-before、可见性、原子性、有序性

## 面试定位

JMM 是 Java 并发的理论底座。面试中常从 `volatile`、`synchronized`、单例模式、指令重排、线程安全问题切入，最终考察候选人能否解释：

- 为什么多线程会读到旧值。
- 什么是可见性、原子性、有序性。
- happens-before 是什么，和真实执行顺序有什么关系。
- `volatile` 解决了什么，没有解决什么。
- `synchronized`、`final`、线程启动和线程终止如何建立可见性。

回答时不要把 JMM 讲成 CPU 缓存模型，也不要把 happens-before 误解为时间先后。JMM 是 Java 语言层面对多线程读写共享变量的规范。

## 核心概念

### JMM 是什么

JMM，全称 Java Memory Model，Java 内存模型，定义了：

- 线程之间如何通过共享变量通信。
- 一个线程对共享变量的写入何时对另一个线程可见。
- 编译器、处理器可以怎样重排序。
- 哪些同步动作可以建立 happens-before 关系。

JMM 屏蔽了不同 CPU 和操作系统内存模型差异，给 Java 程序提供统一的并发语义。

### 主内存和工作内存

JMM 抽象中：

- 共享变量存放在主内存。
- 每个线程有自己的工作内存，保存共享变量副本。
- 线程对变量的操作通常发生在工作内存中。
- 线程之间不能直接访问对方工作内存，只能通过主内存间接通信。

这只是规范抽象，不等于 JVM 里一定有一个叫工作内存的实体。

## 三大问题

### 可见性

一个线程修改共享变量后，另一个线程能否及时看到。

典型问题：

```java
class VisibilityDemo {
    private boolean running = true;

    void stop() {
        running = false;
    }

    void run() {
        while (running) {
            // 可能一直无法看到 running=false
        }
    }
}
```

没有同步约束时，读线程可能一直读本地缓存、寄存器或被 JIT 优化后的值。

解决方式：

- `volatile`
- `synchronized`
- `Lock`
- 原子类
- 线程安全容器
- 安全发布

### 原子性

一个操作是否不可分割。

`i++` 不是原子操作，大致包含：

```text
read i
add 1
write i
```

多线程下会丢失更新：

```java
class Counter {
    private int count;

    void incr() {
        count++;
    }
}
```

解决方式：

- `synchronized`
- `ReentrantLock`
- `AtomicInteger`
- `LongAdder`

注意：`volatile int count; count++` 仍然不是原子操作。

### 有序性

程序执行顺序可能被编译器或处理器重排，只要单线程语义不变。

示例：

```java
int a = 0;
boolean ready = false;

// 线程 1
a = 1;
ready = true;

// 线程 2
if (ready) {
    System.out.println(a); // 没有同步时理论上可能看到 0
}
```

如果没有 happens-before 约束，线程 2 看到 `ready=true` 时，不保证一定看到 `a=1`。

## happens-before

### 定义

happens-before 是 JMM 中判断内存可见性的规则。

如果操作 A happens-before 操作 B，则：

- A 的结果对 B 可见。
- A 的执行顺序在 JMM 语义上排在 B 前面。
- 编译器和处理器仍可在不改变 happens-before 结果的前提下优化和重排。

重要：happens-before 不是物理时间上的先发生，而是内存语义上的先行发生。

### 常见规则

#### 1. 程序次序规则

同一个线程内，前面的操作 happens-before 后面的操作。

```java
int a = 1;      // hb
int b = a + 1;  // 后续操作能看到 a=1
```

单线程内看起来像按代码顺序执行。

#### 2. monitor 锁规则

对同一个锁的 unlock happens-before 后续 lock。

```java
synchronized (lock) {
    shared = 1;
} // unlock

synchronized (lock) { // 后续 lock
    System.out.println(shared);
}
```

这就是 `synchronized` 同时保证互斥和可见性的原因。

#### 3. volatile 变量规则

对 volatile 变量的写 happens-before 后续对同一变量的读。

```java
data = 42;
ready = true; // volatile write

if (ready) {  // volatile read
    System.out.println(data); // 能看到 42
}
```

#### 4. 线程启动规则

调用线程中的 `Thread.start()` happens-before 新线程中的任意操作。

```java
int value = 1;
Thread t = new Thread(() -> System.out.println(value));
t.start();
```

新线程能看到 `start()` 前已经准备好的数据。

#### 5. 线程终止规则

线程中的所有操作 happens-before 其他线程检测到该线程终止。

检测方式包括：

- `Thread.join()` 成功返回。
- `Thread.isAlive()` 返回 `false`。

```java
Thread t = new Thread(() -> result = 42);
t.start();
t.join();
System.out.println(result); // 可见
```

#### 6. 线程中断规则

一个线程调用另一个线程的 `interrupt()` happens-before 被中断线程检测到中断事件。

#### 7. 对象终结规则

对象构造完成 happens-before 它的 `finalize()` 开始。现代工程中不应依赖 `finalize()`。

#### 8. 传递性

如果 A happens-before B，B happens-before C，则 A happens-before C。

这是组合并发语义的基础。

## 安全发布

对象构造完成后，要让其他线程安全看到它的完整状态。

不安全发布：

```java
class Holder {
    int value;

    Holder() {
        value = 42;
    }
}

Holder holder;

void init() {
    holder = new Holder(); // 没有同步，其他线程可能看到未完全初始化状态
}
```

安全发布方式：

- 通过 `static final` 初始化。
- 写入 `volatile` 字段。
- 在 `synchronized` 块内发布，并在相同锁下读取。
- 放入线程安全容器。
- 通过线程启动前准备好数据。

示例：

```java
class SafeHolder {
    private static final Holder HOLDER = new Holder();

    static Holder get() {
        return HOLDER;
    }
}
```

## final 字段语义

`final` 字段有特殊初始化安全性：

- 构造函数正确完成后，其他线程即使没有同步，也能看到 `final` 字段的初始化值。
- 前提是构造过程中没有让 `this` 逸出。

正确示例：

```java
final class UserConfig {
    private final String appName;
    private final int timeoutMs;

    UserConfig(String appName, int timeoutMs) {
        this.appName = appName;
        this.timeoutMs = timeoutMs;
    }
}
```

错误示例：

```java
class EscapeThis {
    static EscapeThis instance;
    final int value;

    EscapeThis() {
        instance = this; // this 逸出
        value = 42;
    }
}
```

## 双重检查锁定

错误写法：

```java
class Singleton {
    private static Singleton instance;

    static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

问题：`new Singleton()` 可能被拆成分配内存、初始化对象、引用赋值。没有 `volatile` 时，其他线程可能看到非空但未初始化完成的对象。

正确写法：

```java
class Singleton {
    private static volatile Singleton instance;

    static Singleton getInstance() {
        Singleton local = instance;
        if (local == null) {
            synchronized (Singleton.class) {
                local = instance;
                if (local == null) {
                    local = new Singleton();
                    instance = local;
                }
            }
        }
        return local;
    }
}
```

更推荐：

```java
class Singleton {
    private Singleton() {
    }

    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }

    static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

## 底层机制

### 内存屏障

JMM 通过编译器和 CPU 层面的内存屏障实现可见性和有序性约束。不同平台实现不同，面试中通常只需说明：

- volatile 写前后会加入相应屏障，防止前面的普通写被重排到 volatile 写后。
- volatile 读后会加入相应屏障，防止后面的普通读写被重排到 volatile 读前。
- monitor enter/exit 也会形成内存语义。

不要把 Java 层语义绑定到某一条具体 CPU 指令上。

### as-if-serial

编译器和处理器可以重排序，但必须保证单线程程序结果不变，这称为 as-if-serial。

多线程中，如果没有同步关系，另一个线程可能观察到重排影响。

## 代码示例：可见性修复

```java
class StopFlag {
    private volatile boolean stopped;

    void stop() {
        stopped = true;
    }

    void runLoop() {
        while (!stopped) {
            doWork();
        }
    }
}
```

如果 `doWork()` 内部可能阻塞，仅靠 volatile 不够，还需要中断、超时或关闭资源。

## 工程实践

- 共享可变状态必须有明确同步策略。
- 能用不可变对象就不要共享可变对象。
- 发布配置、开关、状态标记时优先考虑 `volatile` 或原子引用。
- 复合操作必须用锁或原子类，不要只靠 `volatile`。
- 对象构造期间不要启动线程、注册监听器或把 `this` 放入全局容器。
- 线程池任务之间不要通过普通字段隐式传递结果，使用 `Future`、队列、并发容器或同步工具。
- 面对线上偶现并发 bug，不要依赖本地复现概率，要先检查缺失 happens-before 的共享变量读写。

## 常见追问

### 1. happens-before 和代码先后顺序一样吗？

不一样。代码顺序是单线程内的程序次序；happens-before 是跨线程可见性规则。两个操作即使物理时间上先后发生，如果没有 happens-before，也不保证可见。

### 2. synchronized 为什么能保证可见性？

同一个 monitor 的 unlock happens-before 后续 lock。线程释放锁前对共享变量的写入，对随后获得同一把锁的线程可见。

### 3. volatile 为什么不能保证 `count++` 安全？

volatile 只保证单次读写的可见性和有序性，不保证读、改、写这个复合操作的原子性。

### 4. final 一定线程安全吗？

不是。`final` 只保证引用或字段初始化安全。若 `final` 引用指向可变对象，对象内部状态仍可能线程不安全。

### 5. 没有同步就一定出错吗？

不一定每次出错，但程序存在数据竞争，JMM 不保证跨线程可见性和顺序，线上可能偶现错误。

## 线上风险

- 开关变量不可见，导致线程无法停止。
- 单例对象未安全发布，偶现空字段或异常状态。
- 缓存对象发布后继续被无锁修改，读线程看到中间态。
- 错误使用 `volatile` 做计数，导致统计丢失。
- 异步任务写结果，主线程不 `join` 或不通过 `Future` 获取，读到旧值。
- 构造函数中 `this` 逸出，事件回调看到半初始化对象。

## 易错点

- 把 JMM 直接等同于堆和栈的内存划分。
- 认为 `volatile` 能替代所有锁。
- 认为时间上先执行就一定对另一个线程可见。
- 忽略 final 引用对象内部仍然可变。
- 双重检查锁定忘记加 `volatile`。
- 在构造函数里启动线程或注册回调。
- 用普通 `HashMap` 发布并发更新的数据。

## 自检清单

- 能否解释 JMM 解决的问题。
- 能否分别说明可见性、原子性、有序性。
- 能否列出至少 6 条 happens-before 规则。
- 能否解释 `volatile` 写和读建立的可见性。
- 能否说明 `synchronized` 的内存语义。
- 能否写出正确的双重检查锁定或 holder 单例。
- 能否解释安全发布和 `this` 逸出风险。
- 能否判断一个共享变量读写是否存在数据竞争。
