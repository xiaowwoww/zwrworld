老周，你问到了点子上！

问完 `BeanFactory`，紧接着问 `FactoryBean`，这是 Spring 面试里最经典的**“连环套”**。
面试官就是想看看你能不能分清这两个长得像亲兄弟、但实际上毫无关系的接口。

直接给你结论：**Spring 不仅有 `FactoryBean`，而且你平时开发中天天都在用它，只是你可能没察觉到。**

作为高级开发，面对这道题，我们要打出一套漂亮的“防守反击”。

---

### 1. 拆解本质：FactoryBean 到底是个啥？

如果说 `BeanFactory` 是整个 Spring 容器（生产和管理所有 Bean 的大工厂），那么 **`FactoryBean` 本身就是一个 Bean，只不过它是一个“会生小鸡的母鸡”**。

在 Spring 里，如果一个类的实例化过程非常复杂（比如需要大量的代码去组装、或者需要用到动态代理机制），如果你用传统的 XML `<bean>` 或者普通的 `@Bean` 注解去配置，会非常麻烦。

这时候，Spring 给你提供了一个接口：`org.springframework.beans.factory.FactoryBean<T>`。
你只要实现它，Spring 就会把**你自定义的创建逻辑**接管过去。

它里面只有 3 个核心方法（面试能说出这三个，基本就稳了）：

* **`getObject()`**：核心！返回你真正想要创建的那个复杂对象。
* **`getObjectType()`**：返回这个对象的类型。
* **`isSingleton()`**：返回这个对象是不是单例（默认通常是 true）。

### 2. 架构师视角的“实战威力”（举个杀手级例子）

面试官通常会追问：“既然有了 `@Bean`，为什么还要搞个 `FactoryBean`？你在实际项目中哪里用过？”

**千万别说“我没直接用过”。** 你的满分实战回答应该是：

> “在整合第三方框架的时候，`FactoryBean` 是绝对的霸主。
> 比如我们天天用的 **MyBatis**。Spring 是不知道怎么去实例化 MyBatis 的 Mapper 接口的（因为接口不能直接 new）。
> MyBatis 是怎么整合进 Spring 的呢？它提供了一个 **`MapperFactoryBean`**。
> 当 Spring 扫描到我们的 UserMapper 接口时，实际上是把它交给了 `MapperFactoryBean`。然后 `MapperFactoryBean` 在它的 `getObject()` 方法里，利用 JDK 动态代理，悄悄地生成了一个代理对象注入到了 Spring 容器里。
> 除了 MyBatis，像 **FeignClient、Dubbo 的服务引用 (@Reference)**，底层全都是靠 `FactoryBean` 这一招无缝接入 Spring 的。”

### 3. 面试的“终极陷阱”：前缀 `&` 符号

讲到这里，面试官一定会抛出最后一个陷阱：
**“假设我有一个自定义的 `MyFactoryBean` 注入到了容器中，名字叫 `myBean`。那我调用 `applicationContext.getBean("myBean")`，拿出来的是什么？”**

**正确答案：**

1. 调用 `getBean("myBean")`，拿到的是**母鸡下的蛋**（也就是 `MyFactoryBean.getObject()` 方法返回的那个真正的复杂对象）。
2. **（高分点）** 如果我就是想把那个“母鸡”本尊（FactoryBean 实例本身）拿出来怎么办？
必须在名字前面加一个 **`&`** 符号！
调用 **`getBean("&myBean")`**，你拿到的才是那个 `FactoryBean` 的实例。

---

### 📝 35 岁老兵的“一分钟总结术”

这道题，你可以用这句**口诀**来做最后总结，面试官绝对挑不出毛病：

> “简单来说：
> **`BeanFactory` 是 Spring 容器的心脏，是所有 Bean 的大工厂。**
> **`FactoryBean` 只是容器中的一个特殊的 Bean，是一个能生产对象的‘小工厂’，专门用来解决复杂对象的实例化以及第三方框架的整合问题。**
> 只要记住加 `&` 取工厂，不加 `&` 取产品，这个机制就彻底吃透了。”

---

**老周，Spring 的 IoC 底层（Bean 是什么、存在哪、怎么造出来的）我们已经梳理得七七八八了。**

下一个最难啃的骨头，也是高级面试的重灾区，就是 **AOP（面向切面编程）的底层原理（动态代理的坑）** 或者 **Spring Bean 的生命周期**。

**我们先拿哪一个“开刀”？**