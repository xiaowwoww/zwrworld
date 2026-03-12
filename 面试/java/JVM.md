JVM（Java Virtual Machine）内存结构在 Java 8 及之后版本中主要分为以下几个区域，每个区域有其特定的作用和生命周期。   
以下是 基于 **Java 8+ 的标准内存模型** （注意：Java 8 移除了永久代，引入了元空间）：

✅ **JVM 内存主要分为两大类：**

**一、线程私有区域（Thread-Private）**

每个线程独享，随线程创建而创建，随线程销毁而释放。

**1. 程序计数器（Program Counter Register）**

**• 作用：** 记录当前线程正在执行的字节码指令地址（行号）。
> 如果执行的是 native 方法，则值为 undefined。
  
**• 特点：**
> 唯一一个不会发生 OutOfMemoryError 的区域。
> 
> 线程私有，生命周期与线程一致。

**2. 虚拟机栈（Java Virtual Machine Stack）**

**• 作用：存储方法调用的栈帧（Stack Frame），每个方法调用都会创建一个栈帧，包含：**
> 局部变量表（Local Variables）
> 
> 操作数栈（Operand Stack）
> 
> 动态链接（Dynamic Linking）
> 
> 方法出口（Return Address）

**• 异常：**
> 栈深度超限 → StackOverflowError
>
> 栈无法扩展（如 -Xss 设置过小且系统内存不足）→ OutOfMemoryError
>
**• 线程私有**
> ⚠️ 注意：常说的“堆内存”和“栈内存”中的“栈”，就是指这个虚拟机栈。

**3. 本地方法栈（Native Method Stack）**
**• 作用：为 JVM 调用 native 方法（如 C/C++ 编写的 JNI 函数）服务。**
>有些 JVM（如 HotSpot）将它和虚拟机栈合并实现。
>同样可能抛出 StackOverflowError 或 OutOfMemoryError。

**二、线程共享区域（Shared by All Threads）**

**4. 堆（Heap） —— 最大、最重要的内存区域**

**• 作用：** 存放**几乎所有对象实例和数组**（JVM 规范规定对象必须在堆上分配，但 JIT 优化可能逃逸分析后栈上分配）。

**• GC 主战场：** 垃圾回收器主要管理此区域。

**• 结构（分代）：**
**• 新生代（Young Generation）：**
>Eden 区
>Survivor From / To（S0, S1）
**• 老年代（Old Generation）**

