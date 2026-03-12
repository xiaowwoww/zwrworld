JVM（Java Virtual Machine）内存结构在 Java 8 及之后版本中主要分为以下几个区域，每个区域有其特定的作用和生命周期。   
以下是 基于 **Java 8+ 的标准内存模型** （注意：Java 8 移除了永久代，引入了元空间）：

✅ **JVM 内存主要分为两大类：**

**一、线程私有区域（Thread-Private）**

每个线程独享，随线程创建而创建，随线程销毁而释放。

**1. 程序计数器（Program Counter Register）**

**• 作用：** 记录当前线程正在执行的字节码指令地址（行号）。
>如果执行的是 native 方法，则值为 undefined。
  
**• 特点：**
>唯一一个不会发生 OutOfMemoryError 的区域。
>
>线程私有，生命周期与线程一致。

**2. 虚拟机栈（Java Virtual Machine Stack）**

**• 作用：存储方法调用的栈帧（Stack Frame），每个方法调用都会创建一个栈帧，包含：**
>局部变量表（Local Variables）
>
>操作数栈（Operand Stack）
>
>动态链接（Dynamic Linking）
>
>方法出口（Return Address）

**• 异常：**
>栈深度超限 → StackOverflowError
>
>栈无法扩展（如 -Xss 设置过小且系统内存不足）→ OutOfMemoryError
>
**• 线程私有**
>⚠️ 注意：常说的“堆内存”和“栈内存”中的“栈”，就是指这个虚拟机栈。

**3. 本地方法栈（Native Method Stack）**

**• 作用：为 JVM 调用 native 方法（如 C/C++ 编写的 JNI 函数）服务。**
>有些 JVM（如 HotSpot）将它和虚拟机栈合并实现。
>
>同样可能抛出 StackOverflowError 或 OutOfMemoryError。

**二、线程共享区域（Shared by All Threads）**

**4. 堆（Heap） —— 最大、最重要的内存区域**

**• 作用：** 存放**几乎所有对象实例和数组**（JVM 规范规定对象必须在堆上分配，但 JIT 优化可能逃逸分析后栈上分配）。

**• GC 主战场：** 垃圾回收器主要管理此区域。

**• 结构（分代）：**

**• 新生代（Young Generation）：**
>Eden 区
>
>Survivor From / To（S0, S1）

**• 老年代（Old Generation）**

**参数控制：**
>-Xms：初始堆大小
>
>-Xmx：最大堆大小

**• 异常：** java.lang.OutOfMemoryError: Java heap space

**5. 方法区（Method Area） —— 逻辑概念，Java 8+ 由元空间实现**

**• 作用：** 存储**类的元数据信息**，包括：
>类结构（字段、方法、接口）
>
>运行时常量池（Runtime Constant Pool）
>
>静态变量（static variables）
>
>即时编译器（JIT）优化后的代码缓存等

**Java 8 之前**：实现为 **永久代（PermGen）**，位于堆内。

**Java 8 及以后**：改为 **元空间（Metaspace）**，使用**本地内存（Native Memory）**，不再受 JVM 堆大小限制。
>参数：-XX:MetaspaceSize, -XX:MaxMetaspaceSize

**异常：**
>Java 7 及以前：java.lang.OutOfMemoryError: PermGen space
>
>Java 8+：java.lang.OutOfMemoryError: Metaspace

>💡 注意：字符串常量池在 Java 7 之后也从永久代移到了堆中。

**🧩 补充说明：运行时常量池（Runtime Constant Pool）**
>是方法区的一部分。
>
>存放编译期生成的各种字面量和符号引用（如 "hello"、类名、方法名等）。
>
>具有动态性：运行期间也可将新常量放入池中（如 String.intern()）。

**📊 总结表格**

|内存区域|线程私有/共享|作用|是否会发生 OOM|
|--|--|--|--|
|程序计数器|私有|记录字节码行号|❌不会|
|虚拟机栈|私有|存储方法栈帧（局部变量、操作数栈等）|✅ 会（StackOverflow / OOM）|
|本地方法栈|私有|支持 native 方法调用|✅ 会|
|堆（Heap）|共享|存放对象实例、数组|✅ 会（最常见 OOM）|
|方法区（元空间）|共享|存储类元数据、常量池、静态变量|✅ 会（Metaspace OOM）|

**🔚 关键记忆点**

**对象在堆上，方法在栈上。**

**堆 = 对象；栈 = 方法调用；方法区 = 类信息。**

**Java 8 没有永久代，只有元空间（在本地内存）。**

**字符串常量池在堆里（Java 7+）。**

理解 JVM 内存结构，是排查内存泄漏、调优 GC、分析 OOM 问题的基础。
