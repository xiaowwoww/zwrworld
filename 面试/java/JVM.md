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
  
  > 线程私有，生命周期与线程一致。
