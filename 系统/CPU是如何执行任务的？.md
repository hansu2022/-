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