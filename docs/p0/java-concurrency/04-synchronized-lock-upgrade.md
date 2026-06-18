# synchronized 锁升级、偏向锁、轻量级锁、重量级锁

## 面试定位

`synchronized` 是 Java 内置锁，也是并发面试核心题。常见问题包括：

- `synchronized` 修饰实例方法、静态方法、代码块分别锁什么。
- `synchronized` 如何保证原子性、可见性、有序性。
- monitor、对象头、Mark Word 是什么。
- 偏向锁、轻量级锁、重量级锁如何升级。
- 为什么锁可以优化，锁升级能否降级。
- JDK 新版本中偏向锁的状态。

回答时要先讲使用语义，再讲 JVM 优化。不要一上来只背锁升级流程。

## 基本用法

### 修饰实例方法

锁对象是当前实例 `this`。

```java
class Account {
    private int balance;

    public synchronized void deposit(int amount) {
        balance += amount;
    }
}
```

不同实例之间不互斥。

### 修饰静态方法

锁对象是类对象 `Account.class`。

```java
class Account {
    private static int total;

    public static synchronized void addTotal(int amount) {
        total += amount;
    }
}
```

同一个类的所有实例共享这把锁。

### 修饰代码块

锁对象是括号里的对象。

```java
class Account {
    private final Object lock = new Object();
    private int balance;

    void withdraw(int amount) {
        synchronized (lock) {
            balance -= amount;
        }
    }
}
```

推荐用私有 final 锁对象，避免外部代码拿到同一把锁造成不可控阻塞。

## synchronized 语义

### 原子性

同一时刻只有一个线程可以进入同一把 monitor 保护的临界区。

### 可见性

对同一把锁：

- 线程释放锁前的写入。
- 对后续获得这把锁的线程可见。

JMM 规则：unlock happens-before 后续 lock。

### 有序性

`synchronized` 的进入和退出会建立内存语义，限制临界区内外对共享变量访问的重排，保证其他线程按同步语义观察到正确结果。

### 可重入性

同一个线程已经持有某把锁时，可以再次进入同一把锁保护的代码。

```java
class Service {
    synchronized void outer() {
        inner();
    }

    synchronized void inner() {
        // 同一线程可重入
    }
}
```

## 字节码层面

同步代码块通常使用：

- `monitorenter`
- `monitorexit`

同步方法通常在方法访问标志上带 `ACC_SYNCHRONIZED`，由 JVM 在方法调用和返回时隐式获取和释放 monitor。

异常退出时也必须释放锁，所以编译器会生成异常路径上的 `monitorexit`。

## 对象头和 Mark Word

HotSpot 对象头通常包含：

- Mark Word：存储哈希码、GC 年龄、锁状态、偏向线程 ID 等信息。
- Klass Pointer：指向类元数据。
- 数组对象还有数组长度。

锁优化主要围绕 Mark Word 展开。不同锁状态下，Mark Word 中记录的信息不同。

## 锁状态

传统 HotSpot 锁状态可以理解为：

```text
无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁
```

锁升级方向通常是单向的，竞争加剧时升级。锁不会因为竞争降低而立刻回到更轻状态；具体实现和 JVM 版本有关。

重要版本差异：

- JDK 6 之后引入偏向锁、轻量级锁等优化。
- JDK 15 起偏向锁被废弃并默认关闭。
- 面试中讲偏向锁时要说明它主要是老版本 HotSpot 的优化机制，新版本默认行为可能不同。

## 偏向锁

### 解决的问题

很多锁从头到尾只被同一个线程使用，没有竞争。偏向锁希望这种场景下连 CAS 都尽量省掉。

### 基本思想

第一次获取锁时，将对象头 Mark Word 偏向到当前线程。后续同一线程再次进入同步块时，只需检查对象头是否偏向自己。

### 适合场景

- 单线程反复进入同步块。
- 几乎没有其他线程竞争。

### 撤销偏向

当其他线程尝试获取已经偏向的锁时，需要撤销偏向。撤销可能涉及：

- 到达安全点。
- 检查原持有线程是否仍存活。
- 检查是否仍在同步块中。
- 重新偏向、升级为轻量级锁或恢复无锁。

撤销偏向有成本，所以在竞争明显的场景偏向锁反而不划算。

## 轻量级锁

### 解决的问题

多个线程交替使用锁，但没有严重竞争。轻量级锁通过 CAS 避免直接阻塞和内核态互斥量。

### 基本流程

线程进入同步块时：

1. 在线程栈帧中创建 Lock Record。
2. 将对象头 Mark Word 复制到 Lock Record。
3. 用 CAS 尝试把对象头替换为指向 Lock Record 的指针。
4. CAS 成功，获得轻量级锁。
5. CAS 失败，说明存在竞争，可能自旋或升级。

### 自旋

竞争线程不会马上阻塞，可能短暂自旋等待持锁线程释放。适合临界区很短的场景。

自旋过多会浪费 CPU，因此 JVM 会根据运行情况优化。

## 重量级锁

### 何时升级

当竞争激烈、自旋失败或等待线程较多时，锁会膨胀为重量级锁。

重量级锁依赖 ObjectMonitor，竞争失败线程会进入阻塞，涉及操作系统 mutex/park/unpark 等机制，用户态和内核态切换成本更高。

### ObjectMonitor

可以粗略理解为包含：

- Owner：当前持锁线程。
- EntryList：等待进入锁的线程。
- WaitSet：调用 `wait()` 后等待条件的线程。

`BLOCKED` 通常对应等待进入 EntryList；`WAITING` 可能对应 WaitSet。

## 锁升级流程

简化流程：

```text
无锁
  |
  | 第一次被同一线程获取
  v
偏向锁
  |
  | 其他线程竞争，撤销偏向
  v
轻量级锁
  |
  | CAS 失败、自旋失败、竞争激烈
  v
重量级锁
```

注意：

- 这是便于面试理解的简化模型。
- 实际实现受 JVM 版本、启动参数、对象哈希码、竞争模式影响。
- 调用对象的 identity hash code 可能影响偏向锁，因为 Mark Word 需要存储哈希码。

## 锁优化

### 锁消除

JIT 通过逃逸分析发现对象不会逃出当前线程时，可以消除不必要的锁。

```java
void append() {
    StringBuffer sb = new StringBuffer();
    sb.append("a");
    sb.append("b");
    // sb 不逃逸，锁可能被消除
}
```

### 锁粗化

连续多次加锁解锁开销较大，JIT 可能把多个小同步块合并为一个大同步块。

```java
for (int i = 0; i < 100; i++) {
    synchronized (lock) {
        doSmallWork(i);
    }
}
```

如果频繁进出同一把锁，可能被粗化。

### 自适应自旋

JVM 会根据同一锁上之前自旋是否成功、持锁线程状态等信息决定自旋时间。

## wait/notify 与 synchronized

`wait()`、`notify()`、`notifyAll()` 必须在持有对象 monitor 时调用。

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }
    doWork();
}
```

`wait()` 会释放 monitor，进入 WaitSet。被 `notify` 唤醒后，需要重新竞争 monitor，成功后才能继续执行。

## 代码示例：保护共享状态

```java
class BoundedCounter {
    private final Object lock = new Object();
    private int value;

    int incrementAndGet() {
        synchronized (lock) {
            value++;
            return value;
        }
    }

    int get() {
        synchronized (lock) {
            return value;
        }
    }
}
```

读取也要加同一把锁，否则可见性和一致性不完整。

## 工程实践

- 锁对象使用私有 final 对象，避免锁泄漏。
- 临界区要小，只保护共享状态，不包住慢 I/O、RPC、SQL。
- 多把锁要固定获取顺序，避免死锁。
- 读多写少场景考虑 `ReadWriteLock`、不可变快照或无锁方案。
- 高并发计数优先考虑 `LongAdder`。
- 不要为了“性能”盲目替换 `synchronized`，现代 JVM 对低竞争锁优化已经很好。
- 使用线程 dump 分析 `BLOCKED` 时，重点找持有 monitor 的线程栈。

## 常见追问

### 1. synchronized 锁的是什么？

锁的是对象 monitor。实例方法锁 `this`，静态方法锁 `Class` 对象，代码块锁指定对象。

### 2. synchronized 是公平锁吗？

不是公平锁。JVM 不保证等待线程按进入顺序获得锁。

### 3. synchronized 可以中断吗？

线程等待进入 `synchronized` 时不能响应中断退出等待；获得锁后代码内部可以自行检查中断。

### 4. synchronized 和 ReentrantLock 区别？

`synchronized` 是 JVM 内置锁，自动释放，简单可靠；`ReentrantLock` 是显式锁，支持可中断等待、超时获取、公平锁、多个条件队列，但必须手动释放。

### 5. 锁升级可以降级吗？

面试中通常回答：锁升级主要是从轻到重，单个锁对象一般不会在运行中按竞争降低立即降回轻量状态。具体优化由 JVM 决定，不应依赖。

### 6. 偏向锁现在还重要吗？

重要在于理解 JVM 曾经的锁优化思路。实际工程要知道 JDK 15 起偏向锁默认关闭并被废弃，不能把偏向锁当成所有现代 JDK 的默认性能特征。

## 线上风险

- 锁内远程调用导致大量线程 `BLOCKED`。
- 公共字符串、Class 对象或暴露出去的对象被当锁，引发外部代码干扰。
- 多锁顺序不一致导致死锁。
- `wait()` 使用 `if` 判断条件，虚假唤醒后错误执行。
- 忘记 `notifyAll()`，多条件等待下唤醒错误线程。
- 同步方法粒度过大，串行化整个服务。
- 混用不同锁保护同一份状态，导致锁形同虚设。

## 易错点

- 认为 `synchronized` 只保证互斥，不保证可见性。
- 认为 `wait()` 不释放锁。
- 认为 `notify()` 后被唤醒线程立即执行。
- 认为所有 JDK 默认都有偏向锁。
- 锁对象使用可变引用，运行中被替换。
- 静态方法和实例方法锁对象混淆。
- 读方法不加锁，破坏同一保护策略。

## 自检清单

- 能否说明三种 synchronized 用法分别锁什么。
- 能否解释 monitor enter/exit 和异常释放锁。
- 能否说明 synchronized 的原子性、可见性、有序性来源。
- 能否讲清偏向锁、轻量级锁、重量级锁分别解决什么问题。
- 能否画出简化锁升级流程。
- 能否说明 JDK 15 后偏向锁默认关闭的版本差异。
- 能否解释 wait set 和 entry list 的区别。
- 能否根据线程 dump 定位 synchronized 锁竞争。
