### Spring4与Spring 5的AOP的不同

---

Spring4对应Spring Boot 1版本，Spring 5对应Spring Boot 2版本。它们对AOP的通知执行顺序是不同的。

五种通知有 ：

- `@Before`
- `@After`
- `@AfterReturning`
- `@AfterThrowing`
- `@Around`

假设一个方法五种通知都有，那么它的通知的执行顺序，在Spring4是 ：

1. `@Around`中方法执行前的代码
2. `@Before`方法
3. 方法执行
4. `@Around`中方法执行后的代码
5. `@After`方法
6. `@AfterReturning`方法



如果发送了异常，那么执行顺序是 ：

1. `@Around`中方法执行前的代码
2. `@Before`方法
3. 方法执行
4. `@After`
5. `@AfterThrowing`



从上述来看，其实是不太合理的 ：

**@Around先于@Before执行，但是@Around却先于@After**，按照方法栈的规则，先进应当后出。

因此在Spring 5针对这点做了修改，但也是埋了一个坑，如果项目中有代码依赖这些顺序，那么升级到Spring Boot2可能会出问题。



五种通知在Spring 5的顺序为 ：

1. `@Around`中方法执行前的代码
2. `@Before`
3. 方法
4. `@AfterReturning`
5. `@After`
6. `@Around`中方法执行后的代码



异常情况的顺序为 ：

1. `@Around`中方法执行前的代码
2. `@Before`
3. 方法
4. `@AfterThrowing`
5. `@After`方法



总结 ：**Spring 4的AOP的五种通知，并不符合后进先出的规则；Spring 5针对这个点做了修改**。