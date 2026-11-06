---
title: JVM初探
cover: /img/xi.png
top_img: /img/xi.png
date: 2026-11-06 18:10:46
categories:
  - Java
tags:
 - JVM
 - 类加载
 - 垃圾回收
---

# JVM 初探

> 本笔记按 **「JVM 整体结构 → 类加载机制 → 对象创建 → 运行时内存区域 → 垃圾回收 → 调优排查」** 的主线梳理 JVM 核心知识，便于复习时快速定位。

**学习主线**：

| 阶段 | 要回答的问题 | 对应内容 |
|---|---|---|
| 整体结构 | JVM 由哪几部分组成 | 类加载器 + 运行时数据区 + 执行引擎 |
| 类怎么进来 | `.class` 文件如何加载到内存 | 类加载过程、类加载器、双亲委派 |
| 对象怎么造 | 一个 `new` 背后发生了什么 | 对象创建五步、对象内存布局 |
| 数据放哪里 | 各块内存存什么、会不会 OOM | 程序计数器、栈、堆、方法区 |
| 怎么清理 | 垃圾怎么判定、怎么回收 | GC 算法、分代回收 |
| 出问题怎么办 | OOM / 内存泄漏怎么排查 | MAT、JProfiler、转储参数 |

---

# 一、JVM 的整体位置与结构

```
.java 源码
    ↓ javac 编译
Class File（字节码）
    ↓
类加载器（ClassLoader）
    ↓
┌─────────────────────────────────────────────────┐
│        运行时数据区（Runtime Data Area）          │
│                                                 │
│  方法区（元空间）     Java 栈      本地方法栈     │
│                                                 │
│  堆（Heap）          程序计数器                  │
└─────────────────────────────────────────────────┘
    ↕                                    ↕
执行引擎  ←──── 本地方法接口 ←──── 本地方法库
```

**各部分职责**：

| 组成 | 职责 |
|---|---|
| **类加载器** | 把 `.class` 字节码加载到 JVM 内存，生成 `Class` 对象 |
| **运行时数据区** | 存放类信息、对象实例、方法调用栈等运行时数据 |
| **执行引擎** | 解释执行 / JIT 编译执行字节码 |
| **本地方法接口** | 调用 C/C++ 实现的本地方法（`native`） |
| **本地方法库** | 各平台底层的 native 库 |

> **重点**：运行时数据区中，**堆和方法区是线程共享**的；**Java 栈、本地方法栈、程序计数器是线程私有**的。

---

# 二、类加载机制

## （一）类加载的整体流程

```text
加载 (Loading)
   ↓
验证 (Verification)     ┐
准备 (Preparation)      ├── 链接 (Linking)
解析 (Resolution)       ┘
   ↓
初始化 (Initialization)
   ↓
使用 (Using)
   ↓
卸载 (Unloading)
```

| 阶段 | 主要动作 |
|---|---|
| **加载** | 通过类的全限定名获取二进制字节流（磁盘/JAR/网络），生成 `Class` 对象 |
| **验证** | 文件格式、元数据、字节码、符号引用验证 |
| **准备** | 为静态变量分配内存并设置**零值**（`static int a = 10` 此时 `a = 0`） |
| **解析** | 将常量池中的符号引用替换为直接引用 |
| **初始化** | 执行 `<clinit>()`，为静态变量赋真实值、执行静态代码块 |

> **易错点**：准备阶段只赋**零值**，真正的赋值 `10` 要到**初始化**阶段才执行。

## （二）六种触发初始化的时机

有且仅有以下 6 种（主动引用）：

1. `new` 创建实例、访问静态字段、调用静态方法
2. 反射调用（`Class.forName`）
3. **初始化子类时先初始化父类**
4. 虚拟机启动时的主类（含 `main` 方法）
5. `MethodHandle` 解析结果为 `REF_getStatic` 等
6. 接口中定义的 `default` 方法被实现类调用时

> 其余引用（如通过数组定义引用类、访问 `static final` 常量）属于**被动引用**，不会触发初始化。

## （三）类加载器层级

```
Bootstrap ClassLoader（启动类加载器，C++ 实现）
        ↑
Extension / Platform ClassLoader（扩展 / 平台类加载器）
        ↑
Application ClassLoader（应用程序类加载器，也叫系统类加载器）
        ↑
Custom ClassLoader（自定义类加载器）
```

| 类加载器 | 加载路径 | 说明 |
|---|---|---|
| **Bootstrap** | `JAVA_HOME/jre/lib`（如 `rt.jar`） | C++ 实现，`getClassLoader()` 返回 `null` |
| **Extension**（JDK8）/ **Platform**（JDK9+） | `jre/lib/ext` 或 `jdk/modules` | 加载扩展类 |
| **Application** | classpath | 加载用户类路径上的类 |
| **Custom** | 自定义 | 继承 `ClassLoader` 重写 `findClass` |

> **记忆点**：Bootstrap 由 C++ 实现，不是 Java 类，所以 `String.class.getClassLoader()` 返回 `null`。

## （四）双亲委派机制

**工作流程**：

1. 类加载器收到加载请求，**先委派给父加载器**
2. 父加载器再委派给它的父加载器……直到 Bootstrap
3. 父加载器**无法加载**（自己搜索范围内找不到）时，子加载器才尝试自己加载

```
App ClassLoader → Ext ClassLoader → Bootstrap ClassLoader（最终先尝试）
        ↑                                    │
        └──────── 都失败才自己加载 ←──────────┘
```

**JDK 源码实现**（`ClassLoader.loadClass`）：

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // 1. 检查是否已加载
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                if (parent != null) {
                    // 2. 委派给父加载器
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // 父加载器加载失败
            }
            if (c == null) {
                // 3. 自己加载
                c = findClass(name);
            }
        }
        return c;
    }
}
```

**双亲委派的优势**：

| 优势 | 说明 |
|---|---|
| **安全性** | 防止核心类库被篡改（自定义 `java.lang.String` 不会被加载） |
| **唯一性** | 同一个类在 JVM 中只被加载一次，避免重复加载 |

> **重点**：双亲委派**不是强制约束**，而是 `ClassLoader.loadClass()` 的默认实现。子类重写 `loadClass()` 即可打破。

## （五）自定义类加载器

**方式一：重写 `findClass`（遵循双亲委派）**

```java
public class MyClassLoader extends ClassLoader {
    private String classPath;

    public MyClassLoader(String classPath) {
        this.classPath = classPath;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            byte[] data = loadByte(name);
            return defineClass(name, data, 0, data.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }

    private byte[] loadByte(String name) throws IOException {
        name = name.replace('.', '/');
        try (InputStream in = new FileInputStream(classPath + "/" + name + ".class")) {
            return in.readAllBytes();
        }
    }
}
```

**方式二：重写 `loadClass`（打破双亲委派）**

```java
@Override
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            // 不委派，直接自己加载（某些场景下）
            try {
                c = findClass(name);
            } catch (ClassNotFoundException e) {
                // 自己加载失败再委派
                if (parent != null) {
                    c = parent.loadClass(name, false);
                }
            }
        }
        if (resolve) resolveClass(c);
        return c;
    }
}
```

## （六）打破双亲委派的经典场景

| 场景 | 原因 | 解决方案 |
|---|---|---|
| **SPI 机制**（JDBC、JNDI） | 核心接口由 Bootstrap 加载，实现类在 classpath 上 | **线程上下文类加载器（TCCL）** |
| **Tomcat** | 应用间类隔离，应用需优先加载自己的类 | `WebappClassLoader` 先自己加载，找不到再委派 |
| **OSGi** | 模块化热部署 | 网状委派模型 |
| **Spring Boot** | 加载嵌套 JAR | `LaunchedURLClassLoader` |
| **热部署 / 热替换** | 修改类后重新加载 | 自定义 ClassLoader，丢弃旧实例 |

**核心 API**：

```java
ClassLoader.getSystemClassLoader()              // 获取 App ClassLoader
Thread.currentThread().getContextClassLoader()  // 获取 TCCL（线程上下文类加载器）
Class.forName("X")                              // 加载 + 初始化
ClassLoader.loadClass("X")                      // 只加载，不初始化
Class.getClassLoader()                          // Bootstrap 返回 null
```

> **对比记忆**：`Class.forName` **会**触发初始化，`loadClass` **不会**。这是 JDBC 驱动注册的经典考点。

---

# 三、对象的创建过程

## （一）创建对象的几种方式

| 方式 | 示例 |
|---|---|
| `new` 关键字 | `new User()` |
| 反射 | `Class.newInstance()`、`Constructor.newInstance()` |
| `clone()` | 复制已有对象，**不走构造方法** |
| 反序列化 | `ObjectInputStream.readObject()`，**不走构造方法** |
| 工厂方法 / 静态方法 | 内部终究还是 `new` |

> **注意**：`clone` 和反序列化**不会调用构造方法**，但依然会分配内存并初始化对象头。

## （二）对象创建的五个步骤

```
1. 类加载检查
      ↓
2. 分配内存
      ↓
3. 内存空间零值初始化
      ↓
4. 设置对象头
      ↓
5. 执行 <init> 构造方法
```

| 步骤 | 说明 |
|---|---|
| **① 类加载检查** | 检查 `new` 的类是否已加载，未加载则先走类加载流程 |
| **② 分配内存** | 在堆中划出内存。用**指针碰撞**（规整内存）或**空闲列表**（碎片内存）；并发下用 **CAS + 失败重试** 或 **TLAB（本地线程分配缓冲）** |
| **③ 零值初始化** | 将分配的内存空间全部置为**零值**（保证实例字段有默认值） |
| **④ 设置对象头** | 写入 Mark Word、Klass Pointer、数组长度（若是数组） |
| **⑤ 执行 `<init>`** | 执行构造方法，此时才真正完成对象的初始化 |

> **顺序易错**：③ 零值初始化 → ④ 对象头 → ⑤ 构造方法。所以构造方法里能访问到的父类字段可能还是零值。

## （三）对象内存布局

```text
┌─────────────────────────────┐
│        对象头 (Header)       │
│  ┌───────────────────────┐  │
│  │ Mark Word (8字节, 64位)│  │
│  ├───────────────────────┤   │
│  │ Klass Pointer (4/8字节)│  │
│  ├───────────────────────┤   │
│  │ 数组长度 (仅数组, 4字节)│  │
│  └───────────────────────┘  │
├─────────────────────────────┤
│   实例数据 (Instance Data)   │  ← 父类字段在前，子类在后；同宽字段相邻
├─────────────────────────────┤
│   对齐填充 (Padding)         │  ← 补齐到 8 字节整数倍
└─────────────────────────────┘
```

**Mark Word 在 64 位下的状态复用**（**重点，面试高频**）：

| 锁状态 | 存储内容 |
|---|---|
| 无锁 | hash(31) + 分代年龄(4) + 偏向标志(1) + 锁标志(01) |
| 偏向锁 | 线程ID(54) + epoch(2) + 分代年龄(4) + 偏向标志(1) + 锁标志(01) |
| 轻量级锁 | 指向栈中锁记录的指针 + 锁标志(00) |
| 重量级锁 | 指向 Monitor 的指针 + 锁标志(10) |
| GC 标记 | 空 + 锁标志(11) |

> Mark Word 只有 64 位却要存这么多信息，靠的是**按状态复用同一块空间**。

## （四）`<init>` 与 `<clinit>` 的区别

| 对比项 | `<init>()` | `<clinit>()` |
|---|---|---|
| 对应 | **实例构造方法** | **类静态初始化** |
| 触发时机 | 对象实例化时（`new`、反射等） | 类初始化时（首次主动使用） |
| 内容 | 实例字段赋值、实例代码块、构造方法体 | 静态字段赋值、静态代码块 |
| 执行次数 | 每创建一个对象执行一次 | 每个类加载器只执行一次 |
| 父类关系 | 隐式调用 `super()`（第一句） | 隐式先执行父类 `<clinit>` |

## （五）父子类完整实例化顺序

```text
1. 加载 Father 和 Son 类（如果还没加载，走类加载流程）
2. 执行 Father 的 <clinit>（静态变量、静态块）
3. 执行 Son 的 <clinit>
4. 分配 Son 对象内存
5. 零值初始化（所有实例字段，含继承来的）
6. 设置对象头
7. 执行 Father 的 <init>
   ├── Father 实例字段显式赋值
   ├── Father 实例代码块
   └── Father 构造方法体
8. 执行 Son 的 <init>
   ├── Son 实例字段显式赋值
   ├── Son 实例代码块
   └── Son 构造方法体
```

> **一句话记忆**：**类初始化先父后子，对象实例化也先父后子；先静态后实例，先零值后赋值。**

## （六）延伸：对象一定在堆上吗？

**不一定**。HotSpot 的**逃逸分析（Escape Analysis）**会判断对象是否逃逸出方法 / 线程：

| 优化 | 说明 |
|---|---|
| **栈上分配** | 对象未逃逸 → 直接在栈帧分配，随栈帧销毁，**无需 GC** |
| **标量替换** | 对象可拆解为多个基本类型变量，不真正创建对象 |
| **锁消除** | 对象仅线程内可见 → 去掉同步 |

可通过 `-XX:+DoEscapeAnalysis` 开启（JDK 默认开启）。

> 这解释了为什么"Java 对象都在堆上"这种说法在 JIT 优化后并不严格成立。

---

# 四、运行时数据区

| 区域 | 线程 | 作用 | 是否 OOM |
|---|---|---|---|
| **程序计数器** | 私有 | 记录当前线程执行的字节码行号 | **不会** |
| **Java 虚拟机栈** | 私有 | 存放栈帧（局部变量表、操作数栈等） | StackOverflowError / OOM |
| **本地方法栈** | 私有 | 为 native 方法服务 | SOE / OOM |
| **堆** | 共享 | 存放对象实例、数组 | **OOM 主战场** |
| **方法区** | 共享 | 类信息、常量、静态变量 | OOM |

## （一）程序计数器（PC Register）

- **线程私有**，每个线程一份
- 记录当前线程正在执行的**字节码指令地址**
- 唯一一个**不会抛出 OOM** 的区域
- 执行 native 方法时值为 `undefined`

## （二）Java 虚拟机栈

- **线程私有**，生命周期与线程相同
- 每个方法执行时创建一个**栈帧（Stack Frame）**入栈，方法返回时出栈
- **程序正在执行的方法一定在栈顶**

**栈帧结构**：

| 组成 | 作用 |
|---|---|
| **局部变量表** | 存放方法参数和局部变量（基本类型、对象引用） |
| **操作数栈** | 字节码运算的工作区 |
| **动态链接** | 指向运行时常量池中该方法的符号引用 |
| **方法返回地址** | 方法退出后继续执行的位置 |

**两种异常**：

- **StackOverflowError**：栈深度超过限制（如无限递归）
- **OutOfMemoryError**：栈扩展时无法申请到足够内存

> **设置参数**：`-Xss` 设置单个线程栈大小。

## （三）堆（Heap）

**JVM 内存中最大的一块**，线程共享，几乎所有对象实例都在这里分配。

**分代结构**：

```
┌──────────────────────────────────────────────┐
│                  堆 (Heap)                    │
│ ┌───────────────────────┬──────────────────┐ │
│ │      新生代 (Young)     │  老年代 (Old)     │ │
│ │ ┌──────┬─────┬─────┐  │                  │ │
│ │ │ Eden │ S0  │ S1  │  │                  │ │
│ │ └──────┴─────┴─────┘  │                  │ │
│ └───────────────────────┴──────────────────┘ │
└──────────────────────────────────────────────┘
```

| 区域 | 说明 |
|---|---|
| **Eden** | 新对象优先在此分配 |
| **Survivor 0 / 1** | 经过 Minor GC 存活的对象在两区间来回复制 |
| **Old（老年代）** | 存活够久或对象过大时晋升到此 |

**默认比例**：Eden : S0 : S1 = **8 : 1 : 1**；新生代 : 老年代 = **1 : 2**

**关联参数**：

| 参数 | 作用 |
|---|---|
| `-Xms` | 堆初始大小 |
| `-Xmx` | 堆最大大小 |
| `-Xmn` | 新生代大小 |
| `-XX:NewRatio` | 新生代与老年代比例 |
| `-XX:SurvivorRatio` | Eden 与单个 Survivor 比例 |

## （四）方法区

- **线程共享**，存放**类元信息、常量、静态变量、JIT 编译后的代码**
- **JDK 7 及以前**：**永久代（PermGen）**，属于堆内存，受 `-XX:MaxPermSize` 限制
- **JDK 8 及以后**：**元空间（Metaspace）**，使用**本地内存**

> **元空间特性**：「**逻辑上存在，物理上不存在**」—— 逻辑上属于方法区，物理上不在 JVM 堆内，而是直接使用本地内存，因此默认不受堆大小限制（受本机内存限制）。

**演进对比**：

| 版本 | 实现 | 位置 | 参数 |
|---|---|---|---|
| JDK 7- | 永久代 | JVM 堆内 | `-XX:MaxPermSize` |
| JDK 8+ | 元空间 | **本地内存** | `-XX:MaxMetaspaceSize` |

**为什么改元空间**：永久代大小难以预估，容易 OOM；元空间使用本地内存后更灵活。

---

# 五、沙箱安全机制

**沙箱机制**：将 Java 代码限定在 JVM 中特定的运行范围，并**严格限制对系统本地资源（文件、网络、系统调用）的访问**。

**核心思想**：

- 不可信代码（如远程加载的 Applet）运行在受限的沙箱中
- 沙箱通过 `ClassLoader` + `SecurityManager` + 访问控制器协同实现
- 不同来源的类由不同类加载器加载，赋予不同权限

> 双亲委派正是沙箱安全的基础之一 —— 它保证了**核心类库不会被自定义类篡改**。

---

# 六、垃圾回收（GC）机制

> **垃圾回收只发生在堆（和方法区），不会发生在栈空间中** —— 栈帧随方法调用自动入栈出栈，无需 GC。

## （一）如何判定是垃圾

| 算法 | 说明 | 缺点 |
|---|---|---|
| **引用计数法** | 对象被引用则计数 +1，归零即回收 | **无法解决循环引用** |
| **可达性分析**（Java 采用） | 从 **GC Roots** 出发，不可达的对象即为垃圾 | —— |

**GC Roots 包括**：

- 虚拟机栈中局部变量表引用的对象
- 方法区中静态变量、常量引用的对象
- 本地方法栈中 native 方法引用的对象
- 虚拟机内部的引用（基本类型 Class、常驻异常对象等）
- 被 `synchronized` 持有的对象

**对象的"死亡"要经历两次标记**：

1. 可达性分析后**不可达** → 第一次标记
2. 检查是否重写了 `finalize()` 且未执行过 → 若需要执行，放入 F-Queue，**第二次标记**仍不可达才真正回收

## （二）垃圾回收算法

| 算法 | 原理 | 优点 | 缺点 |
|---|---|---|---|
| **标记-清除** | 标记存活对象，清除其余 | 实现简单 | **内存碎片**、效率低 |
| **标记-复制** | 存活对象复制到另一块区域 | 无碎片、效率高 | **浪费空间** |
| **标记-整理** | 标记后把存活对象移向一端 | 无碎片 | 移动成本高 |
| **分代收集** | 新生代用复制，老年代用标记-清除/整理 | 综合最优 | —— |

> **分代依据**：绝大多数对象**朝生夕死**（弱分代假说），因此新生代用复制算法更划算。

## （三）分代回收流程

```
新对象 → Eden
          ↓ Eden 满 触发 Minor GC
       存活对象 → S0（年龄 +1）
          ↓ 再次 Minor GC
       S0 存活对象 → S1（年龄 +1），Eden 清空
          ↓ 对象在两 Survivor 间来回复制
       年龄达到阈值（默认 15）→ 晋升老年代
          ↓ 老年代满
       触发 Full GC
```

**两种 GC**：

| 类型 | 触发时机 | 范围 |
|---|---|---|
| **Minor GC / Young GC** | Eden 区满 | 只回收新生代（**频繁**） |
| **Full GC** | 老年代满、元空间不足、显式 `System.gc()` | 整个堆 + 方法区（**慢**，应尽量避免） |

> **易错点**：大对象（如超长数组）会**直接进入老年代**，可通过 `-XX:PretenureSizeThreshold` 设置阈值。

---

# 七、OOM 排查

## （一）排查思路

```
1. 拿到内存快照（Heap Dump）
      ↓
2. 用分析工具打开
      ↓
3. 查找占用最大的对象 / 对象数量异常增长的类
      ↓
4. 定位到具体代码（GC Root 引用链）
      ↓
5. 修复后验证
```

## （二）导出堆转储

**方式一：启动参数（推荐，自动导出）**

```bash
-Xms1m -Xmx8m -XX:+HeapDumpOnOutOfMemoryError
```

- `-XX:+HeapDumpOnOutOfMemoryError`：OOM 发生时**自动导出**堆转储文件
- `-XX:HeapDumpPath=./dump.hprof`：指定导出路径

**方式二：手动导出**

```bash
jmap -dump:format=b,file=heap.hprof <pid>
```

## （三）分析工具

| 工具 | 作用 |
|---|---|
| **MAT（Eclipse Memory Analyzer）** | 分析 dump 文件，**快速定位内存泄漏**、查看大对象、对象引用链 |
| **JProfiler** | 实时监控 + dump 分析，可视化强 |
| **jvisualvm** | JDK 自带，轻量级监控与分析 |

**MAT 核心用法**：

- **Leak Suspects**：自动给出内存泄漏可疑点
- **Histogram**：按类统计对象数量与占用内存
- **Dominator Tree**：找出真正持有内存的"支配对象"
- **Path to GC Roots**：查看对象为何无法被回收（**定位泄漏根源的关键**）

## （四）能否看到代码第几行出错

- **能**：内存快照分析工具（MAT、JProfiler）可通过引用链定位到具体代码位置
- 也可配合 DEBUG 逐行分析
- 但更推荐先用工具定位到**类**和**大对象**，再回溯业务代码

---

# 八、总结

## （一）知识地图

| 模块 | 核心要点 |
|---|---|
| **整体结构** | 类加载器 + 运行时数据区 + 执行引擎 |
| **类加载** | 加载 → 验证 → 准备 → 解析 → 初始化；准备赋零值、初始化赋真值 |
| **双亲委派** | 先委派父加载器，保证**安全性与唯一性**；重写 `loadClass` 可打破 |
| **对象创建** | 类加载检查 → 分配内存 → 零值初始化 → 设置对象头 → `<init>` |
| **内存布局** | 对象头（Mark Word + Klass Pointer）+ 实例数据 + 对齐填充 |
| **内存区域** | 堆、方法区共享；栈、PC、本地方法栈私有 |
| **元空间** | JDK 8+ 用本地内存，逻辑上属于方法区 |
| **GC** | 可达性分析 + 分代收集（新生代复制、老年代标记整理） |
| **排查** | `-XX:+HeapDumpOnOutOfMemoryError` → MAT → Path to GC Roots |

## （二）易错清单

1. **准备阶段只赋零值**：`static int a = 10` 在准备阶段 `a = 0`
2. **`Class.forName` 会初始化，`loadClass` 不会**
3. **Bootstrap 类加载器的 `getClassLoader()` 返回 `null`**
4. **双亲委派不是强制约束**，重写 `loadClass` 即可打破
5. **`clone` 和反序列化不走构造方法**，但仍会分配内存
6. **零值初始化在构造方法之前**，构造方法中可能读到父类字段的零值
7. **元空间在本地内存**，不在堆内
8. **程序计数器是唯一不会 OOM 的区域**
9. **GC 不回收栈空间**，栈帧随方法调用自动释放
10. **Full GC 比 Minor GC 慢得多**，生产环境应尽量减少
