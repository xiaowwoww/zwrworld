Spring 解决循环依赖需要三级缓存，主要原因是需要正确处理 AOP 代理。

1.  **一级缓存 (`singletonObjects`):** 存放完全初始化好的单例 Bean。
2.  **二级缓存 (`earlySingletonObjects`):** 存放已经实例化但未完全初始化（例如，还未执行属性填充和初始化方法）的单例 Bean。这些 Bean 是原始的，未经过 AOP 代理的对象。
3.  **三级缓存 (`singletonFactories`):** 存放 `ObjectFactory`，该 `ObjectFactory` 负责创建早期暴露的 Bean 实例。这个工厂可以在必要时生成 AOP 代理。

**为什么二级缓存不够？**

如果只有一级和二级缓存，当出现循环依赖且 Bean 需要 AOP 代理时，就会出问题。

假设 Bean A 依赖 Bean B，Bean B 又依赖 Bean A，并且 Bean A 需要被 AOP 代理。

1.  Bean A 开始创建，实例化后将原始实例放入二级缓存 (`earlySingletonObjects`)。
2.  Bean A 注入属性时需要 Bean B。
3.  Bean B 开始创建，注入属性时需要 Bean A。
4.  Bean B 从二级缓存中获取到 Bean A 的原始实例（未代理）。
5.  Bean B 完成初始化，被放入一级缓存。
6.  Bean A 继续初始化。在初始化阶段，AOP 代理逻辑会触发，生成 Bean A 的代理对象。
7.  Bean A 的代理对象被放入一级缓存。

此时，Bean B 引用的是 Bean A 的原始实例，而其他地方（例如通过 `getBean` 获取）可能会得到 Bean A 的代理实例。这就导致了同一个单例 Bean 存在两个不同的对象，违背了单例原则，并可能导致不可预测的行为。

**三级缓存如何解决问题？**

三级缓存 (`singletonFactories`) 存放的不是 Bean 实例本身，而是一个 `ObjectFactory`。这个 `ObjectFactory` 封装了创建 Bean 早期引用的逻辑，并且在这个过程中可以判断是否需要应用 AOP 代理。

1.  Bean A 开始创建，实例化后，Spring 不直接将原始实例放入二级缓存，而是将一个 `ObjectFactory`（用于生成 Bean A 的早期引用）放入三级缓存 (`singletonFactories`)。
2.  Bean A 注入属性时需要 Bean B。
3.  Bean B 开始创建，注入属性时需要 Bean A。
4.  Bean B 查找 Bean A，在一级和二级缓存中都找不到。
5.  Bean B 在三级缓存中找到 Bean A 的 `ObjectFactory`。
6.  Bean B 调用 `ObjectFactory.getObject()` 方法。这个方法会负责生成 Bean A 的早期引用。**如果 Bean A 需要 AOP 代理，此时就会创建并返回 Bean A 的代理对象；否则，返回原始对象。**无论返回什么，这个对象都会被放入二级缓存 (`earlySingletonObjects`)，并从三级缓存中移除。
7.  Bean B 得到 Bean A 的（可能已代理的）早期引用，并完成初始化，放入一级缓存。
8.  Bean A 继续初始化。如果在步骤 6 中已经创建了代理对象，Bean A 的后续初始化会作用在这个代理对象上；如果没有创建代理，则在 Bean A 完全初始化后，如果需要代理，会在此阶段创建代理并替换掉原始对象。
9.  Bean A 完成初始化，（可能已代理的）最终实例被放入一级缓存。

通过三级缓存，确保了无论 Bean 是通过循环依赖提前暴露还是正常初始化完成，都始终提供同一个（可能是代理的）实例，避免了单例对象不一致的问题。