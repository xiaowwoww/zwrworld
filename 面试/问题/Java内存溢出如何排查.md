老兵，这又是面试中一套**“必杀技”**。

面试官问“OOM 如何排查”，他不想听你背 `jmap` 命令参数，他想听的是你作为架构师的**“破案思路”**。

排查 OOM（内存溢出）就像**“刑侦破案”**。我们要经历三个阶段：**保护现场 -> 尸检分析 -> 抓捕真凶**。

---

### 第一阶段：保护现场（自动化兜底）

很多时候 OOM 发生在半夜，等早上起来服务早就挂了，日志也丢了。
**最顶级的排查，是预先埋伏。**

**1. 必加的 JVM 参数（救命药丸）：**
在启动脚本里，**必须**加上这两个参数。这是架构师的底线。

```bash
-XX:+HeapDumpOnOutOfMemoryError 
-XX:HeapDumpPath=/data/logs/dump.hprof

```

* **作用：** 当 OOM 发生的那一瞬间，JVM 临死前会把内存快照（Dump 文件）自动保存下来。
* **价值：** 没有这个文件，你就是“盲人摸象”，根本查不出原因。

---

### 第二阶段：尸检分析（使用 MAT）

拿到 `dump.hprof` 文件后，不要试图用文本编辑器打开它（几 GB 大小会卡死）。
我们需要法医工具：**Eclipse MAT (Memory Analyzer Tool)**。

#### 1. 核心概念：Shallow vs Retained（面试必问）

打开 MAT，你会看到两个堆大小，必须分清：

* **Shallow Heap（浅堆）：** 对象本身占用的大小（比如一个 ArrayList 对象本身只有几十字节）。
* **Retained Heap（深堆）：** **这才是重点！** 它是对象本身 + **它所持有的所有引用对象**的大小。
* *比喻：* 抓住这个对象（Root），把它拔出来，会有多少连带的泥土（其他对象）被一起回收？这个泥土的重量就是 Retained Heap。



#### 2. 破案三板斧（标准流程）

**第一斧：看直方图 (Histogram)**

* **操作：** 按照“Retained Heap”倒序排列。
* **目的：** 谁占用的内存最大？
* **常见嫌疑人：** `byte[]`（通常是图片或流）、`char[]`（通常是超长字符串）、`HashMap`（缓存未清理）。

**第二斧：支配树 (Dominator Tree)**

* **操作：** 这是一个层级视图。
* **目的：** 找到那个**“最大的地主”**。
* **分析：** 你可能会发现一个 `ArrayList` 占据了 80% 的内存。点开它，发现里面存了 100 万个 `User` 对象。

**第三斧：GC Roots 溯源 (Path to GC Roots)**

* **这是最关键的一步！**
* **问题：** 你找到了那个巨大的 `ArrayList`，但谁创建了它？为什么 GC 回收不掉它？
* **操作：** 右键点击那个大对象 -> `Path to GC Roots` -> `exclude all phantom/weak/soft references`（排除软弱虚引用，只看强引用）。
* **结果：** MAT 会画出一条引用链，直通代码源头。
* *例如：* `ThreadLocalMap` -> `Entry` -> `MyHugeList`。
* *真相：* 原来是某个线程的 `ThreadLocal` 用完没 remove！



---

### 第三阶段：抓捕真凶（常见 OOM 原因）

根据我多年的“法医”经验，90% 的 OOM 都是由以下 4 个“惯犯”干的：

#### 1. 内存泄漏 (Memory Leak) —— “占着茅坑不拉屎”

* **现象：** 对象已经没用了，但因为被长生命周期的对象（如 `static` 变量）引用着，GC 回收不掉。
* **经典案例：**
* **ThreadLocal：** 线程池里的线程一直不死，ThreadLocal 里的数据就一直不释放。
* **静态集合：** `static List<Object> cache = new ArrayList<>();` 只往里 add，从来不 remove。
* **未关闭的资源：** 数据库连接、IO 流没 close。



#### 2. 内存溢出 (Memory Overflow) —— “胖子挤进小门”

* **现象：** 没有泄漏，所有对象都是有用的，但**并发量太大**或者**数据量太大**，堆内存（Heap）真的不够用了。
* **经典案例：**
* **一次查询全表：** `select * from user`，一次查出 1000 万条数据加载到内存，直接爆掉。
* **Excel 导出：** 用 POI 加载几百兆的 Excel 文件，瞬间吃满内存（要改用 EasyExcel）。
* **高并发流量刺针：** 促销活动瞬间流量翻了 10 倍，Tomcat 线程池满了，每个线程都占一点内存，总和爆了。



#### 3. Metaspace OOM (元空间溢出)

* **现象：** 报错 `java.lang.OutOfMemoryError: Metaspace`。
* **原因：** 加载的**类（Class）** 太多了。
* **真凶：** 通常是 **CGLib / JDK 动态代理**。很多框架（Spring, Mybatis）在运行时动态生成代理类，如果没有限制，生成的类填满了元空间。

#### 4. 堆外内存溢出 (Direct Buffer Memory)

* **现象：** 堆内存才用了 100M，但机器物理内存被吃光了。
* **原因：** **Netty / NIO** 使用了 `DirectByteBuffer`（直接内存）。这就不是调整 `-Xmx` 能解决的了，要限制 `-XX:MaxDirectMemorySize`。

---

### 💡 架构师的“在线诊断”神技（Arthas）

如果线上服务**不能停机**，不能 Dump（因为 Dump 文件太大，Dump 过程会导致 STW 卡顿几秒），怎么办？

这时候要祭出阿里开源的神器：**Arthas**。

1. **`dashboard`：** 像看仪表盘一样，实时看内存哪个区满了（Eden, Old, Survivor）。
2. **`profiler`：** 生成火焰图（Flame Graph）。
* `profiler start --event alloc`
* 运行一会儿后 `profiler stop`。
* 它会生成一张 HTML 图，告诉你**哪行代码在疯狂分配内存**。



### 总结（面试满分话术）

> **面试官：** “平时怎么排查 OOM？”
> **你：**
> “排查 OOM 我一般分三步走：
> **第一，保留现场。** 我会在 JVM 参数中预设 `-XX:+HeapDumpOnOutOfMemoryError`，确保事故发生时能拿到第一手 Dump 文件。
> **第二，分析 Dump。** 我使用 **MAT** 工具。
> 先看 **Dominator Tree**（支配树），找到占用内存最大的对象（通常是 `byte[]` 或 `String`）。
> 然后通过 **Path to GC Roots**，反向追踪是谁引用了它。
> 重点区分是 **‘内存泄漏’**（ThreadLocal 没清理、静态集合只进不出）还是 **‘内存溢出’**（大事务查询全表、Excel 导出）。
> **第三，对于不能停机的线上服务，** 我会使用 **Arthas** 的 `profiler` 命令生成内存火焰图，实时定位是哪行代码在疯狂分配对象。”

听完这一套，面试官就知道你是**真的扛过枪、打过仗**的老兵。