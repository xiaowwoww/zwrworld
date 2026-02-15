这是一个非常棒的“反直觉”问题。

直接给结论：**Spring 的三级缓存是默认开启的，不需要你做任何配置。**

但是，**Spring Boot 2.6.0 版本之后，默认禁止了循环依赖。** 如果你的代码里有循环依赖（Circular Dependency），启动时会直接报错。

所以，你现在的痛点可能不是“怎么开启三级缓存”，而是**“怎么让 Spring 允许我写循环依赖的代码”**。

---

### 1. 为什么 Spring Boot 2.6+ 要禁止循环依赖？

在此之前，Spring 一直是默许循环依赖的（只要是 Singleton + Setter注入）。

但官方认为：**循环依赖通常意味着代码设计有问题（Bad Design）。**

* **高耦合：** A 和 B 互相依赖，说明职责划分不清。
* **难以维护：** 改 A 动 B，改 B 动 A。
* **启动变慢：** 三级缓存机制虽然能解决问题，但增加了 Bean 创建的复杂度。

因此，从 Spring Boot 2.6.0 开始，官方决定“倒逼”开发者去优化代码，默认**抛出异常**，阻止应用启动。

---

### 2. 如何“强行开启”循环依赖支持？（如果你非要用）

如果你接手的是一个老项目，代码改不动，或者你就是不想重构，可以通过配置来**恢复**旧版本的行为。

**方法一：修改配置文件 (application.yml / properties)**

这是最简单的方法。在你的配置文件里加上这一行：

```yaml
spring:
  main:
    allow-circular-references: true  # 允许循环引用

```

或者 properties 格式：

```properties
spring.main.allow-circular-references=true

```

**方法二：代码重构（这才是正道）**

作为架构师，我不建议你直接开开关，而是用以下手段去**解耦**：

1. **使用 `@Lazy` 注解（推荐）：**
* **原理：** 告诉 Spring：“你先别急着注入 B，给我一个**代理对象（Proxy）**。等到我真的调用 B 的方法时，你再去容器里找真正的 B。”
* **写法：**
```java
@Component
public class A {
    private final B b;

    public A(@Lazy B b) { // 这里加 @Lazy
        this.b = b;
    }
}

```


* **优点：** 既解决了循环依赖，又不需要开启全局配置，只针对这一个注入点生效。


2. **提取第三者（Extract Class）：**
* **原理：** A 和 B 互相依赖，说明它们共用了一部分逻辑。把这部分逻辑抽出来，变成 C。
* **结果：** A -> C, B -> C。循环依赖被打破了。


3. **使用 `ApplicationContextAware` 或 `InitializingBean`：**
* **原理：** 不要在构造器或字段注入时拿 B，而是等到 A 初始化完成后，手动去容器里 `getBean(B.class)`。



---

### 3. 三级缓存的“生效条件”

虽然三级缓存是默认机制，但它**不是万能的**。它只能解决**特定场景**下的循环依赖。

**三级缓存能解决：**

* **Scope = Singleton (单例)**
* **注入方式 = Setter 注入 / @Autowired 字段注入**

**三级缓存不能解决（依然会报错）：**

* **Scope = Prototype (多例)：** 因为 Spring 不缓存多例 Bean，每次都 new，直接死循环 OOM。
* **构造器注入 (Constructor Injection)：** 因为构造器执行时对象还没实例化（没放进三级缓存），连“半成品”都不是，根本没法引用。
* *解法：* 用 `@Lazy` 或者改成 Setter 注入。


* **@Async / @Transactional 等 AOP 代理问题：** 有时候加上 AOP 会导致生成的 Bean 和原始 Bean 不一致，Spring 会报错 `BeanCurrentlyInCreationException`。

### 总结

1. **不用开：** 三级缓存机制本身是 Spring 内核自带的，一直都在。
2. **要允许：** Spring Boot 2.6+ 默认报错，需配置 `spring.main.allow-circular-references=true` 来允许循环依赖。
3. **最佳实践：** 尽量别用配置开启，遇到循环依赖，优先用 **`@Lazy`** 解决，或者**重构代码**。