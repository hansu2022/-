## CPU 如何读写数据的？

先来认识 CPU 的架构，只有理解了 CPU 的 架构，才能更好地理解 CPU 是如何读写数据的，对于现代 CPU 的架构图如下：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/CPU%E6%9E%B6%E6%9E%84.png)

为了弥补 CPU 运算速度与内存访问速度之间的巨大差距，现代 CPU 采用了**多级分层缓存（Cache Hierarchy）**结构。

|存储设备|速度（访存延时）|特点|
|---|---|---|
|**寄存器**|极快（< 1 ns）|CPU 内部，极小容量，最快。|
|**L1 Cache**|极快（≈1 ns）|每个 CPU 核心独有，分为 **dCache**（数据）和 **iCache**（指令）。|
|**L2 Cache**|很快（≈4 ns）|每个 CPU 核心独有，比 L1 大。|
|**L3 Cache**|较快（≈10−40 ns）|**多个 CPU 核心共享**，容量更大。|
|**内存 (RAM)**|较慢（≈100 ns）|容量最大，是 CPU 访问速度的瓶颈。|
|**硬盘 (SSD/HDD)**|慢|永久存储。|
你可以看到， CPU 访问 L1 Cache 速度比访问内存快 100 倍，这就是为什么 CPU 里会有 L1~L3 Cache 的原因，目的就是把 Cache 作为 CPU 与内存之间的缓存层，以减少对内存的访问频率。

CPU 从内存中读取数据到 Cache 的时候，并不是一个字节一个字节读取，而是一块一块的方式来读取数据的，这一块一块的数据被称为 CPU Cache Line（缓存块），所以 **CPU Cache Line 是 CPU 从内存读取数据到 Cache 的单位**。

### 2. CPU Cache Line（缓存块）
**定义：** **CPU Cache Line**（缓存块）是 CPU 从内存中读取数据到 Cache 的**最小单位**
- **读取单位：** CPU 读取数据时，不是按字节读取，而是按块（Cache Line）读取。
- **大小：** 通常为 64 字节（可以通过 Linux 命令查看）。这意味着一次从内存载入 L1 Cache 的数据量是 64 字节。
- **对程序的影响：** 如果程序按照**物理内存地址连续**的顺序访问数据（如访问数组），由于这些连续数据很可能被一次性载入到同一 Cache Line，因此 **Cache 命中率会很高**，性能得到提升。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E7%BC%93%E5%AD%98/%E6%9F%A5%E7%9C%8BCPULine%E5%A4%A7%E5%B0%8F.png)


接下来，就来看看 Cache 伪共享是什么？又如何避免这个问题？

现在假设有一个双核心的 CPU，这两个 CPU 核心并行运行着两个不同的线程，它们同时从内存中读取两个不同的数据，分别是类型为 `long` 的变量 A 和 B，这个两个数据的地址在物理内存上是**连续**的，如果 Cahce Line 的大小是 64 字节，并且变量 A 在 Cahce Line 的开头位置，那么这两个数据是位于**同一个 Cache Line 中**，又因为 CPU Cache Line 是 CPU 从内存读取数据到 Cache 的单位，所以这两个数据会被同时读入到了两个 CPU 核心中各自 Cache 中。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/%E5%90%8C%E4%B8%80%E4%B8%AA%E7%BC%93%E5%AD%98%E8%A1%8C.png)

我们来思考一个问题，如果这两个不同核心的线程分别修改不同的数据，比如 1 号 CPU 核心的线程只修改了 变量 A，或 2 号 CPU 核心的线程的线程只修改了变量 B，会发生什么呢？
### 分析伪共享的问题

现在我们结合保证多核缓存一致的 MESI 协议，来说明这一整个的过程，如果你还不知道 MESI 协议，你可以看我这篇文章「[10 张图打开 CPU 缓存一致性的大门 (opens new window)](https://mp.weixin.qq.com/s/PDUqwAIaUxNkbjvRfovaCg)」。

①. 最开始变量 A 和 B 都还不在 Cache 里面，假设 1 号核心绑定了线程 A，2 号核心绑定了线程 B，线程 A 只会读写变量 A，线程 B 只会读写变量 B。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/%E5%88%86%E6%9E%90%E4%BC%AA%E5%85%B1%E4%BA%AB1.png)

②. 1 号核心读取变量 A，由于 CPU 从内存读取数据到 Cache 的单位是 Cache Line，也正好变量 A 和 变量 B 的数据归属于同一个 Cache Line，所以 A 和 B 的数据都会被加载到 Cache，并将此 Cache Line 标记为「独占」状态。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/%E5%88%86%E6%9E%90%E4%BC%AA%E5%85%B1%E4%BA%AB2.png)

③. 接着，2 号核心开始从内存里读取变量 B，同样的也是读取 Cache Line 大小的数据到 Cache 中，此 Cache Line 中的数据也包含了变量 A 和 变量 B，此时 1 号和 2 号核心的 Cache Line 状态变为「共享」状态。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/%E5%88%86%E6%9E%90%E4%BC%AA%E5%85%B1%E4%BA%AB3.png)

④. 1 号核心需要修改变量 A，发现此 Cache Line 的状态是「共享」状态，所以先需要通过总线发送消息给 2 号核心，通知 2 号核心把 Cache 中对应的 Cache Line 标记为「已失效」状态，然后 1 号核心对应的 Cache Line 状态变成「已修改」状态，并且修改变量 A。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/%E5%88%86%E6%9E%90%E4%BC%AA%E5%85%B1%E4%BA%AB4.png)

⑤. 之后，2 号核心需要修改变量 B，此时 2 号核心的 Cache 中对应的 Cache Line 是已失效状态，另外由于 1 号核心的 Cache 也有此相同的数据，且状态为「已修改」状态，所以要先把 1 号核心的 Cache 对应的 Cache Line 写回到内存，然后 2 号核心再从内存读取 Cache Line 大小的数据到 Cache 中，最后把变量 B 修改到 2 号核心的 Cache 中，并将状态标记为「已修改」状态。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/%E5%88%86%E6%9E%90%E4%BC%AA%E5%85%B1%E4%BA%AB5.png)

所以，可以发现如果 1 号和 2 号 CPU 核心这样持续交替的分别修改变量 A 和 B，就会重复 ④ 和 ⑤ 这两个步骤，Cache 并没有起到缓存的效果，虽然变量 A 和 B 之间其实并没有任何的关系，但是因为同时归属于一个 Cache Line ，这个 Cache Line 中的任意数据被修改后，都会相互影响，从而出现 ④ 和 ⑤ 这两个步骤。

因此，这种因为多个线程同时读写同一个 Cache Line 的不同变量时，而导致 CPU Cache 失效的现象称为**伪共享（_False Sharing_）**。

### 避免伪共享的方法

**伪共享**（False Sharing）是高性能并发编程中必须规避的问题。其本质是：多个线程独立修改处于**同一 Cache Line** 中的**不同变量**，导致缓存一致性协议频繁触发，引发缓存颠簸，浪费总线带宽和 CPU 周期。

**避免核心思想：** **用空间换时间**，确保经常被不同核心修改的**热点数据**位于**不同的 Cache Line** 中。

### 1. 编译层面的解决方案：Linux 内核宏

Linux 内核通过一个宏定义来强制变量进行 Cache Line 对齐，从而避免伪共享。
#### 机制：
- **宏定义：** 内核提供了 `__cacheline_aligned_in_smp` 宏。
- **对齐操作：**
    - 在**多核（MP）系统**中，该宏被定义为 `__cacheline_aligned`，其实际作用是告诉编译器将变量的起始地址设置为 **Cache Line 大小**的整数倍（通常为 64 字节）对齐。
    - 在**单核系统**中，该宏定义为空，因为单核环境下不存在伪共享问题。
        
- **应用：** 通过将结构体中**可能被不同核心竞争修改**的变量（如变量 `b`）进行 Cache Line 对齐，可以保证它和前一个变量 `a` 不会落入同一个 Cache Line 中

|修改前（可能发生伪共享）|修改后（避免伪共享）|效果|
|---|---|---|
|`long a; long b;`|`long a; long b __cacheline_aligned_in_smp;`|变量 `a` 和 `b` 被强制分开，位于不同的 Cache Line 中。|

举个例子，有下面这个结构体：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/struct_test.png)

结构体里的两个成员变量 a 和 b 在物理内存地址上是连续的，于是它们可能会位于同一个 Cache Line 中，如下图：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/struct_ab.png)

所以，为了防止前面提到的 Cache 伪共享问题，我们可以使用上面介绍的宏定义，将 b 的地址设置为 Cache Line 对齐地址，如下：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/struct_test1.png)

这样 a 和 b 变量就不会在同一个 Cache Line 中了，如下图：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/struct_ab1.png)

所以，避免 Cache 伪共享实际上是用空间换时间的思想，浪费一部分 Cache 空间，从而换来性能的提升。
### 2. 应用层面的解决方案：Java Disruptor 框架
高性能 Java 并发框架 **Disruptor** 采用了一种巧妙的**“字节填充 + 继承”**方式来解决伪共享，并进一步优化了缓存利用率
#### 机制
- **目标：** `RingBuffer` 类中的核心变量是多线程共享的**热点数据**。
- **Cache Line 大小：** 假设 Cache Line 大小为 **64 字节**，一个 `long` 类型数据占 **8 字节**。
- **填充策略（继承实现）：**
    1. **前置填充：** 在核心类 `RingBufferFields` 的父类 `RingBufferPad` 中定义了 **7 个 `long` 类型**的**无用变量**（如 `p1` 到 `p7`）。
    2. **核心变量：** 核心变量位于 `RingBufferFields` 中。
    3. **后置填充：** 在 `RingBufferFields` 的子类（继承了 `RingBufferFields` 的类）中定义额外的填充变量。
- **填充作用：** 通过在**核心变量**的前后都填充足够的字节（7×8=56 字节），可以保证核心变量被夹在两个 64 字节的 Cache Line 边界中间
    - 这样可以确保核心变量在自己的 Cache Line 中**占据独立位置**，从而避免被其他无关数据牵连。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/Disruptor.png)


![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/%E5%A1%AB%E5%85%85%E5%AD%97%E8%8A%82.png)

#### C++ 中避免伪共享的机制
在 C++ 中，解决伪共享问题的方法与 Linux 内核和 Java Disruptor 的思路是**一致的**，都是采用**字节填充**（Padding）和**内存对齐**（Alignment）的机制。
### 1. 使用 `alignas` 关键字进行内存对齐

这是 C++11 引入的标准机制，用于强制变量或结构体的起始地址按照指定的字节数对齐。

#### 场景一：对齐整个结构体/类

将整个结构体或类对齐到 Cache Line 的大小（通常是 **64 字节**）。
```c++
#include <iostream>

// 假设 Cache Line 大小为 64 字节
constexpr size_t CACHE_LINE_SIZE = 64;

// 结构体 A 被强制对齐到 64 字节边界
struct alignas(CACHE_LINE_SIZE) MyData {
    // 成员变量
    long value; // 8 bytes
    // 后面会自动填充到 64 字节
};

struct alignas(CACHE_LINE_SIZE) HotData {
    // 经常被线程 1 修改
    std::atomic<int> counter1;
    // 经常被线程 2 修改
    std::atomic<int> counter2;
    // ... 在 C++17 之前，需要手动填充或使用 alignas(64) 整个结构体
};
```

**效果：** 当创建 `MyData` 数组时，可以保证数组中的**每个元素**都会从一个新的 Cache Line 开始，从而避免不同数组元素之间可能发生的伪共享。

#### 场景二：对齐结构体内部成员（结合填充）

通过将热点成员对齐到 Cache Line 边界，然后手动添加填充（Padding）字节，确保下一个热点成员从下一个 Cache Line 开始。

```cpp
constexpr size_t CACHE_LINE_SIZE = 64;

struct AlignedCounters {
    // 线程 1 的计数器，起始地址对齐到 64 字节边界
    alignas(CACHE_LINE_SIZE) std::atomic<long> counter1; // 8 bytes (并对齐到 64 边界)

    // 手动填充字节，保证 counter2 不和 counter1 在同一 Cache Line
    // 填充 56 字节 (64 - 8)
    char padding[CACHE_LINE_SIZE - sizeof(std::atomic<long>)]; 

    // 线程 2 的计数器，起始地址自然会位于下一个 64 字节边界
    std::atomic<long> counter2; 
};
```

这种方式类似于 Java Disruptor 的**手动填充**，能更精确地控制内存布局。

### 2. 利用 C++17 的缓存干扰常量

从 C++17 开始，标准库提供了两个常量来辅助开发者进行缓存对齐：

|常量|含义|目的|
|---|---|---|
|**`std::hardware_constructive_interference_size`**|构造性干扰大小，通常等于 **Cache Line 大小**（如 64 字节）。|建议将**一起访问**的数据放在这个大小的内存块内，以提高缓存命中率。|
|**`std::hardware_destructive_interference_size`**|破坏性干扰大小，通常等于 **Cache Line 大小**或其**倍数**（如 64 或 128 字节）。|建议将**独立修改**的数据间隔开**至少**这个大小，以避免伪共享。|

在现代 C++ 代码中，开发者应该使用 `std::hardware_destructive_interference_size` 来定义填充或对齐大小：
```cpp
#include <new> // 包含 interference_size

struct FinalAlignedData {
    // 使用破坏性干扰大小来对齐，避免伪共享
    alignas(std::hardware_destructive_interference_size) std::atomic<long> counter1; 

    // 填充到下一个破坏性干扰大小边界
    char padding[std::hardware_destructive_interference_size - sizeof(std::atomic<long>)];
    
    std::atomic<long> counter2;
};
```

---
在 C++ 中解决伪共享问题，本质上是**手动或半自动地**管理内存对齐和填充。

1. **首选 `alignas`：** 这是标准 C++ 中最常用和最可移植的方法。
2. **使用标准常量：** 尽可能使用 C++17 的 `std::hardware_destructive_interference_size`，因为它能获取当前硬件平台建议的**最小安全隔离距离**，比手动写 `64` 更具可移植性。
3. **重新排序成员：** 除了填充，简单的**重新排序**结构体成员有时也能减轻伪共享，将那些可能被一起访问的或只读的成员放在一起，将被不同线程频繁修改的成员分开。
这些技术虽然牺牲了少量内存（填充字节），但在高并发、低延迟的场景（如高性能计算、交易系统、锁无关数据结构）中，能带来显著的性能提升。

---

## [#](https://xiaolincoding.com/os/1_hardware/how_cpu_deal_task.html#cpu-%E5%A6%82%E4%BD%95%E9%80%89%E6%8B%A9%E7%BA%BF%E7%A8%8B%E7%9A%84)CPU 如何选择线程的？
在 Linux 系统中，**进程**和**线程**在内核层面都被视为**任务（task）**，由统一的 `task_struct` 结构体来表示。Linux 内核的**调度器**（Scheduler）就是负责选择和管理这些任务的执行。
### 1. 任务的分类与优先级

Linux 任务根据其优先级和响应要求，分为两大类：

|任务类型|优先级范围|特点|
|---|---|---|
|**实时任务**|0 ~ 99|对响应时间要求极高，需尽快执行。|
|**普通任务**|100 ~ 139|响应时间要求不高，主要追求公平性。|
**注意：** 优先级的数值越小，优先级越高。
### [#](https://xiaolincoding.com/os/1_hardware/how_cpu_deal_task.html#%E8%B0%83%E5%BA%A6%E7%B1%BB)调度类

由于任务有优先级之分，Linux 系统为了保障高优先级的任务能够尽可能早的被执行，于是分为了这几种调度类，如下图：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/%E8%B0%83%E5%BA%A6%E7%B1%BB.png)

这三个调度类的**优先级顺序**为：`Deadline` > `Realtime` > `Fair`。这意味着 Linux 会先检查 `Deadline` 队列，再检查 `Realtime`，最后才考虑 `Fair` 队列。
Deadline 和 Realtime 这两个调度类，都是应用于实时任务的，这两个调度类的调度策略合起来共有这三种，它们的作用如下：

- _SCHED_DEADLINE_：是按照 deadline 进行调度的，距离当前时间点最近的 deadline 的任务会被优先调度；
- _SCHED_FIFO_：对于相同优先级的任务，按先来先服务的原则，但是优先级更高的任务，可以抢占低优先级的任务，也就是优先级高的可以「插队」；
- _SCHED_RR_：对于相同优先级的任务，轮流着运行，每个任务都有一定的时间片，当用完时间片的任务会被放到队列尾部，以保证相同优先级任务的公平性，但是高优先级的任务依然可以抢占低优先级的任务；

而 Fair 调度类是应用于普通任务，都是由 CFS 调度器管理的，分为两种调度策略：

- _SCHED_NORMAL_：普通任务使用的调度策略；
- _SCHED_BATCH_：后台任务的调度策略，不和终端进行交互，因此在不影响其他需要交互的任务，可以适当降低它的优先级。

### [#](https://xiaolincoding.com/os/1_hardware/how_cpu_deal_task.html#%E5%AE%8C%E5%85%A8%E5%85%AC%E5%B9%B3%E8%B0%83%E5%BA%A6)完全公平调度

我们平日里遇到的基本都是普通任务，对于普通任务来说，公平性最重要，在 Linux 里面，实现了一个基于 CFS 的调度算法，也就是**完全公平调度（_Completely Fair Scheduling_）**。

#### 核心机制：虚拟运行时间 (`vruntime`)

- **`vruntime`**：每个任务都有一个虚拟运行时间 `vruntime`。
- **计算方式：** 任务运行得越久，`vruntime` 值越大；未运行的任务 `vruntime` 不变。
- **调度选择：** CFS 调度器总是优先选择**vruntime** 值最小的任务来执行。

这就好比，让你把一桶的奶茶平均分到 10 杯奶茶杯里，你看着哪杯奶茶少，就多倒一些；哪个多了，就先不倒，这样经过多轮操作，虽然不能保证每杯奶茶完全一样多，但至少是公平的。

当然，上面提到的例子没有考虑到优先级的问题，虽然是普通任务，但是普通任务之间还是有优先级区分的，所以在计算虚拟运行时间 vruntime 还要考虑普通任务的**权重值**，注意权重值并不是优先级的值，内核中会有一个 nice 级别与权重值的转换表，nice 级别越低的权重值就越大，至于 nice 值是什么，我们后面会提到。 于是就有了以下这个公式：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/vruntime.png)

**权重与 `nice` 值：**
- `nice` 值范围为 **-20 到 19**，默认值为 0。
- `nice` 值越低，优先级越高，对应的**权重值越大**。
- 因此，优先级高的任务（`nice` 值低）在**相同的实际运行时间**内，其 `vruntime` 的增长会**更慢**，使其能更频繁地被调度。

### [#](https://xiaolincoding.com/os/1_hardware/how_cpu_deal_task.html#cpu-%E8%BF%90%E8%A1%8C%E9%98%9F%E5%88%97)CPU 运行队列

一个系统通常都会运行着很多任务，多任务的数量基本都是远超 CPU 核心数量，因此这时候就需要**排队**。

每个 CPU 核心都有自己的**运行队列（Run Queue）**，用于管理分配给该 CPU 的所有任务。

- **队列结构：** 一个运行队列通常包含三个子队列，分别对应不同的调度类：
    - `dl_rq`：存放 `Deadline` 任务。
    - `rt_rq`：存放 `Realtime` 任务。
    - `cfs_rq`：存放 `Fair` 任务，这是一个**红黑树**，任务按 `vruntime` 大小排序，最左侧的叶子节点就是下一个被调度的任务。
**调度顺序：** Linux 调度器会严格按照 `Deadline > Realtime > Fair` 的优先级，依次从这三个队列中选择下一个要执行的任务。
PS：下图中的 csf_rq 应该是 `cfs_rq`，由于找不到原图了，我偷个懒，我就不重新画了，嘻嘻。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/CPU%E9%98%9F%E5%88%97.png)

先从 `dl_rq` 里选择任务，然后从 `rt_rq` 里选择任务，最后从 `cfs_rq` 里选择任务。因此，**实时任务总是会比普通任务优先被执行**。

### [#](https://xiaolincoding.com/os/1_hardware/how_cpu_deal_task.html#%E8%B0%83%E6%95%B4%E4%BC%98%E5%85%88%E7%BA%A7)调整优先级

对于普通任务，可以通过调整其 **`nice` 值**来改变优先级。

- **`nice` 值范围：** -20 (最高优先级) 到 19 (最低优先级)，默认值为 0

是不是觉得 nice 值的范围很诡异？事实上，nice 值并不是表示优先级，而是表示优先级的修正数值，它与优先级（priority）的关系是这样的：priority(new) = priority(old) + nice。内核中，priority 的范围是 0~139，值越低，优先级越高，其中前面的 0~99 范围是提供给实时任务使用的，而 nice 值是映射到 100~139，这个范围是提供给普通任务用的，因此 nice 值调整的是普通任务的优先级。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/%E4%BC%98%E5%85%88%E7%BA%A7.png)

在前面我们提到了，权重值与 nice 值的关系的，nice 值越低，权重值就越大，计算出来的 vruntime 就会越少，由于 CFS 算法调度的时候，就会优先选择 vruntime 少的任务进行执行，所以 nice 值越低，任务的优先级就越高。

我们可以在启动任务的时候，可以指定 nice 的值，比如将 mysqld 以 -3 优先级：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/nice.png)

如果想修改已经运行中的任务的优先级，则可以使用 `renice` 来调整 nice 值：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/renice.png)

nice 调整的是普通任务的优先级，所以不管怎么缩小 nice 值，任务永远都是普通任务，如果某些任务要求实时性比较高，那么你可以考虑改变任务的优先级以及调度策略，使得它变成实时任务，比如：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/CPU%E4%BC%AA%E5%85%B1%E4%BA%AB/chrt.png)

---

## [#](https://xiaolincoding.com/os/1_hardware/how_cpu_deal_task.html#%E6%80%BB%E7%BB%93)总结

理解 CPU 是如何读写数据的前提，是要理解 CPU 的架构，CPU 内部的多个 Cache + 外部的内存和磁盘都就构成了金字塔的存储器结构，在这个金字塔中，越往下，存储器的容量就越大，但访问速度就会小。

CPU 读写数据的时候，并不是按一个一个字节为单位来进行读写，而是以 CPU Cache Line 大小为单位，CPU Cache Line 大小一般是 64 个字节，也就意味着 CPU 读写数据的时候，每一次都是以 64 字节大小为一块进行操作。

因此，如果我们操作的数据是数组，那么访问数组元素的时候，按内存分布的地址顺序进行访问，这样能充分利用到 Cache，程序的性能得到提升。但如果操作的数据不是数组，而是普通的变量，并在多核 CPU 的情况下，我们还需要避免 Cache Line 伪共享的问题。

所谓的 Cache Line 伪共享问题就是，多个线程同时读写同一个 Cache Line 的不同变量时，而导致 CPU Cache 失效的现象。那么对于多个线程共享的热点数据，即经常会修改的数据，应该避免这些数据刚好在同一个 Cache Line 中，避免的方式一般有 Cache Line 大小字节对齐，以及字节填充等方法。

系统中需要运行的多线程数一般都会大于 CPU 核心，这样就会导致线程排队等待 CPU，这可能会产生一定的延时，如果我们的任务对延时容忍度很低，则可以通过一些人为手段干预 Linux 的默认调度策略和优先级。