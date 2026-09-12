# 第 5 章 Spring / Spring Boot / MyBatis

> 摘自《Java 面试全栈通关指南》——161 道题 / 12 个模块，覆盖 Java 后端与 Java AI 应用开发。
> 在线阅读（可搜索、自测、记进度）：https://java-interview-guide.app.workbuddy.host/

---

#### 5.1 IOC 和 DI 是什么？解决了什么问题？
> **难度**：L2 进阶 ｜ **考察频次**：高 ｜ **标签**：Spring、IOC

**参考答案要点**
- **IOC（Inversion of Control，控制反转）**：把"对象的创建、装配、生命周期管理"从业务代码转移到**容器**。传统写法是 `new UserService(new UserDao())`，调用方主动控制依赖；IOC 下变成容器把 `UserDao` **注入**给 `UserService`，控制权反转。
- **DI（Dependency Injection，依赖注入）** 是 IOC 的**具体实现形式**：构造器注入、setter 注入、字段注入。Spring 通过 `BeanDefinition`（配置元数据：XML / 注解 / JavaConfig）描述 Bean，容器在启动或 `getBean` 时按描述创建并装配。
- **解决的问题**：① **解耦**——依赖抽象（接口）而非实现，替换实现类只改配置不改代码；② **可测试**——单测时注入 Mock 非常容易（构造器注入尤其友好）；③ **统一生命周期与横切能力**——初始化回调、销毁、AOP 代理、事务、缓存都由容器统一织入；④ **可配置化**——`@Profile`、`@Conditional` 按环境切换实现。
- 实现载体：`BeanFactory`（顶层容器接口）与 `ApplicationContext`（企业级增强）；核心数据结构是 `BeanDefinition` 与三级缓存的单例池。
- 一句话总结：**IOC 是思想（谁来控制），DI 是手段（怎么把依赖给进去），Spring 是容器实现**。

**考察意图**
考察是否理解"反转"到底反转了什么（控制权：从调用方 → 容器），而不是只背"控制反转就是 DI"。能说出"依赖抽象 + 可测试 + 横切能力统一"三个收益即为合格。

**延伸追问**
- IOC 与工厂模式有什么区别？
- 为什么 Spring 推荐面向接口编程？没有接口也能注入吗？
- "Spring 的 IOC 容器"和"Servlet 容器"是一个概念吗？

---

#### 5.2 Spring Bean 的完整生命周期？有哪些可扩展点？
> **难度**：L3 深入 ｜ **考察频次**：高 ｜ **标签**：Spring、Bean生命周期

**参考答案要点**
以单例 Bean 为例，完整流程：

1. **实例化**：`createBeanInstance()` 通过构造器（或工厂方法）创建对象，此时属性全为默认值。*扩展点*：`InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()` 可返回代理短路整个流程。
2. **属性填充**：`populateBean()` 完成依赖注入。*扩展点*：`postProcessAfterInstantiation()`、`postProcessProperties()`（`@Autowired` 由 `AutowiredAnnotationBeanPostProcessor` 在此处理）。
3. **Aware 回调**：`BeanNameAware` → `BeanClassLoaderAware` → `BeanFactoryAware`；ApplicationContext 环境下还有 `EnvironmentAware`、`ApplicationContextAware` 等（由 `ApplicationContextAwareProcessor` 处理）。
4. **初始化前置**：`BeanPostProcessor.postProcessBeforeInitialization()`；**`@PostConstruct` 由 `CommonAnnotationBeanPostProcessor` 在这一步调用**。
5. **初始化**：先 `InitializingBean.afterPropertiesSet()`，再执行自定义 `init-method`（`@Bean(initMethod="...")` / XML 的 `init-method`）。
6. **初始化后置**：`BeanPostProcessor.postProcessAfterInitialization()`——**AOP 代理正是在这里生成**（`AbstractAutoProxyCreator`）；若存在循环依赖，代理会提前到三级缓存的 `getEarlyBeanReference()` 生成。
7. **使用**：Bean 就绪，进入单例池 `singletonObjects`。
8. **销毁**：容器关闭时 `@PreDestroy` → `DisposableBean.destroy()` → `destroy-method`。

- 关键扩展接口：`BeanPostProcessor`（对所有 Bean 生效）、`BeanFactoryPostProcessor` / `BeanDefinitionRegistryPostProcessor`（修改 BeanDefinition，如 `ConfigurationClassPostProcessor` 解析 `@Configuration`）、`Aware` 系列、`InitializingBean`、`SmartInitializingSingleton`（所有单例就绪后回调）。
- **注意**：`BeanPostProcessor` 自身也是 Bean，会**先于普通 Bean 实例化**。

**考察意图**
能否按顺序说出 8 个阶段并指出"@PostConstruct 与 afterPropertiesSet 谁先"以及"AOP 代理在哪个阶段生成"，是区分 L2/L3 的关键。

**延伸追问**
- `@PostConstruct`、`afterPropertiesSet`、`init-method` 三者的执行顺序？
- `BeanFactoryPostProcessor` 和 `BeanPostProcessor` 的作用对象有什么不同？
- 构造器里调用 `@Autowired` 注入的字段，为什么是 null？（注入发生在实例化之后）

---

#### 5.3 循环依赖：三级缓存如何解决？为什么必须三级？哪些情况解决不了？
> **难度**：L3 深入 ｜ **考察频次**：高 ｜ **标签**：Spring、循环依赖

**参考答案要点**
- **三级缓存**（`DefaultSingletonBeanRegistry`）：① `singletonObjects`（一级，完整成品 Bean）；② `earlySingletonObjects`（二级，早期半成品引用）；③ `singletonFactories`（三级，`ObjectFactory` 工厂，用于生成早期引用）。
- **解决流程**（A ↔ B，setter/字段注入）：A 实例化 → 立刻 `addSingletonFactory(A, () -> getEarlyBeanReference(A, mbd, bean))` → 填充属性时发现需要 B → B 实例化并暴露工厂 → B 填充属性需要 A → 依次查一级、二级，**命中三级缓存**调用 `getEarlyBeanReference` 得到 A 的早期引用（若 A 需要 AOP，此时就生成代理对象），放入二级并移除三级 → B 初始化完成进一级 → A 拿到 B 完成初始化进一级。
- **为什么必须是三级**：如果只有两级，就必须在 Bean 实例化后**立即**为所有 Bean 创建早期引用，这就意味着要么统一提前 AOP（破坏"代理在初始化后生成"的语义），要么注入原始对象导致**注入的对象与最终 Bean 不是同一个**（代理对象不一致，事务/AOP 全部失效）。三级缓存的 `ObjectFactory` 是**延迟执行**的：只有真正发生循环依赖时才提前生成代理，非循环依赖场景下代理时机完全不变；同时通过 `earlyProxyReferences` 记录，保证后续 `postProcessAfterInitialization` 不会重复代理，**注入的早期引用与最终 Bean 是同一个对象**。
- **解决不了的场景**：① **构造器注入**——对象都还没实例化完，无法提前暴露引用，直接抛 `BeanCurrentlyInCreationException`；② **prototype 作用域**——不进缓存，每次新建，无限递归；③ Spring Boot 2.6 起 `spring.main.allow-circular-references` **默认 false**，字段/setter 循环依赖也会直接启动失败（官方认为循环依赖是设计缺陷，应重构）。
- **正确解法**：抽取第三个公共类、用事件（`ApplicationEvent`）解耦、`@Lazy` 注入代理、`ObjectProvider` 延迟获取——**不推荐靠配置开关放行**。

**考察意图**
本题是 Spring 最经典的深挖题。能否解释"为何三级而非两级"以及"代理对象的时机一致性"，直接决定评级；答出 Boot 2.6 默认关闭则是版本敏感度的加分。

**延伸追问**
- 为什么在 `@PostConstruct` 里调用循环依赖注入进来的另一个 Bean 的方法可能拿到 null 字段？
- `@Lazy` 为什么能破环？它生成的是 JDK 代理还是 CGLIB 代理？
- 构造器注入明明更容易出问题，为什么官方还推荐它？（能**尽早暴露**设计缺陷，而不是把问题藏到运行期）

---

#### 5.4 BeanFactory、FactoryBean、ApplicationContext 的区别？
> **难度**：L2 进阶 ｜ **考察频次**：高 ｜ **标签**：Spring、容器

**参考答案要点**
- **BeanFactory**：IOC 容器的**顶层接口**，定义 `getBean`、`containsBean`、`getType` 等基础能力；默认**懒加载**（`getBean` 时才创建 Bean）。典型实现 `DefaultListableBeanFactory`。
- **ApplicationContext**：继承 `BeanFactory`（并组合了 `ListableBeanFactory`、`HierarchicalBeanFactory` 等），在其之上叠加企业级能力：① **国际化** `MessageSource`；② **事件机制** `ApplicationEventPublisher`（`ApplicationEvent` / `@EventListener`）；③ **资源访问** `ResourceLoader`（classpath:/file:/http:）；④ **环境抽象** `Environment` + `PropertySource`；⑤ 启动时**自动注册** `BeanPostProcessor`、`BeanFactoryPostProcessor`、`ApplicationListener`；⑥ **默认预实例化所有非懒加载单例**（`finishBeanFactoryInitialization`），启动即可发现配置错误；⑦ 原生集成 AOP、事务、校验。
- **FactoryBean**：一个**能生产对象的特殊 Bean**（注意名字里没有"ory"）。实现 `getObject()` / `getObjectType()` / `isSingleton()` 三个方法。容器中名为 `xxx` 的 FactoryBean，`getBean("xxx")` 返回的是**它生产的对象**；要拿 FactoryBean 本身需加 `&` 前缀，即 `getBean("&xxx")`。
  - 典型应用：MyBatis 的 `SqlSessionFactoryBean`、`MapperFactoryBean`；Spring 的 `ProxyFactoryBean`；Spring Cloud Feign 的 `FeignClientFactoryBean`。
- 一句话：**BeanFactory 是容器，ApplicationContext 是增强版容器，FactoryBean 是容器里的一个"对象工厂" Bean**。

**考察意图**
`FactoryBean` 与 `BeanFactory` 名字相似，是经典混淆点。能否说出 `&` 前缀和 MyBatis 里的实际用例，是判断真懂的关键。

**延伸追问**
- 为什么说 `ApplicationContext` 是"饿汉"而 `BeanFactory` 是"懒汉"？
- `@Bean` 方法里返回 `FactoryBean` 类型，最终注册的是哪个对象？
- `ObjectFactory`、`FactoryBean`、`ObjectProvider` 三者有什么区别？

---

#### 5.5 Bean 的作用域有哪些？单例 Bean 线程安全吗？
> **难度**：L2 进阶 ｜ **考察频次**：高 ｜ **标签**：Spring、作用域、线程安全

**参考答案要点**
- **六种作用域**：`singleton`（默认，每个 IOC 容器一个实例）、`prototype`（每次 `getBean`/注入都新建，**容器不负责销毁**）、`request`、`session`、`application`（ServletContext 级）、`websocket`。后四种仅在 Web 环境（`WebApplicationContext`）可用，且注入到单例 Bean 时需要作用域代理（`proxyMode = ScopedProxyMode.TARGET_CLASS`）。
- **单例 Bean 本身不保证线程安全**：Spring 的 singleton 是"每容器一个实例"，与"线程安全"无关。是否被多线程共享取决于使用方式——`@Controller`/`@Service` 默认单例，天然被所有请求线程共享。
- **安全的前提是无状态**：Service/Dao 只依赖方法参数与局部变量（在栈上，线程私有），自然安全。
- **有状态时的四种解法**：① 消除成员变量（最推荐，把状态改为方法参数）；② 使用 `ThreadLocal` 保存请求级状态（用完必须 `remove()`）；③ 改用 `prototype`（有性能与依赖管理代价）；④ 使用并发容器/`AtomicXxx`/加锁。
- **典型事故**：在 `@Controller` 里写 `private int count;` 或 `private User currentUser;` 做"临时变量"，导致请求间数据串号——这是最常见的 Spring 线程安全 Bug。

**考察意图**
考察"Spring 不保证线程安全"这个常被误解的结论，以及能否给出"无状态优先"这一工程判断。能举出 Controller 成员变量串号的真实例子是加分项。

**延伸追问**
- Controller 为什么默认是单例？改成 prototype 有什么副作用？
- 单例 Bean 注入 prototype Bean，为什么 prototype 不生效？如何解决？（`lookup-method` / `ObjectProvider` / 作用域代理）
- `ThreadLocal` 在单例 Bean 里用，线程池环境下要注意什么？

---

#### 5.6 @Autowired 与 @Resource 的区别？推荐哪种注入方式？
> **难度**：L2 进阶 ｜ **考察频次**：高 ｜ **标签**：Spring、依赖注入

**参考答案要点**

| 维度 | `@Autowired` | `@Resource` |
| --- | --- | --- |
| 来源 | Spring 自有（`org.springframework.beans.factory.annotation`） | **JSR-250 规范**（`javax.annotation` → Spring Boot 3 起为 `jakarta.annotation`） |
| 默认匹配 | **byType** | **byName**（先按字段名/name 找，找不到再按类型） |
| 指定名称 | 需配合 `@Qualifier("xxx")` | 直接 `@Resource(name="xxx")` |
| 必需性 | `required = false` 允许为空 | 无此属性（找不到即报错） |
| 适用范围 | 构造器、字段、setter、方法参数 | 字段、setter（**不能用于构造器**） |
| 多实现选择 | `@Qualifier` 或 `@Primary` | `name` 属性 |

- **推荐写法：构造器注入**（Lombok `@RequiredArgsConstructor` + `private final XxxService`）：① 依赖不可变、可声明为 `final`；② 依赖**显式可见**，一眼看出类有多少依赖（字段注入容易藏依赖，导致类变成"上帝类"）；③ **便于单元测试**（直接 new 传 mock，无需反射）；④ 能**在启动期暴露循环依赖**；⑤ 单一构造器时 Spring 可**省略 `@Autowired`**。
- 不推荐字段注入：无法 `final`、测试需反射注入、容易掩盖循环依赖与设计问题（Spring 官方文档与 IDEA 都会给出警告）。
- 补充：`@Inject`（JSR-330）与 `@Autowired` 行为基本一致但缺少 `required` 属性；构造器注入 + `@Qualifier` 可精确指定多实现。

**考察意图**
基础但高频。能否说出"推荐构造器注入"的**四条理由**（尤其"启动期暴露循环依赖"和"便于单测"），是判断是否理解依赖注入本质的标准。

**延伸追问**
- 一个接口两个实现类，`@Autowired` 会报什么错？有几种解决方式？
- `@Primary` 和 `@Qualifier` 同时存在谁优先？（`@Qualifier` 更精确，优先）
- 为什么 `@Resource` 不能用在构造器上？（规范未定义 `CONSTRUCTOR` 目标）

---

#### 5.7 Spring AOP 原理？JDK 动态代理 vs CGLIB？同类方法调用为什么增强失效？
> **难度**：L3 深入 ｜ **考察频次**：高 ｜ **标签**：Spring、AOP

**参考答案要点**
- **两种代理**：① **JDK 动态代理**——`java.lang.reflect.Proxy` + `InvocationHandler`，运行时生成实现**目标接口**的代理类，只能代理接口；② **CGLIB**——继承目标类生成子类 + `MethodInterceptor`（配合 FastClass 索引调用），**可代理无接口的类**，但不能代理 `final` 类/`final`/`private`/`static` 方法。
- **Spring 的选择**：目标有接口用 JDK，无接口用 CGLIB；**Spring Boot 2.x 起 `spring.aop.proxy-target-class` 默认 `true`，统一使用 CGLIB**（避免"注入接口可以、注入实现类报错"的不一致）。
- **代理时机**：Bean 初始化完成后，由 `AnnotationAwareAspectJAutoProxyCreator`（`BeanPostProcessor`）在 `postProcessAfterInitialization` 中生成代理对象，放进单例池的是**代理对象**。
- **同类方法调用失效的根因**：`this.inner()` 中的 `this` 是**原始目标对象**（在代理内部执行时，目标对象实例就是原对象），不经过代理，因此拦截链不生效。只有**从外部通过代理对象调用**才会被拦截。
- **三种解法**：① **注入自身代理**（`private final XxxService self;` 并 `self.inner()`，自注入）；② **`AopContext.currentProxy()`**（需 `@EnableAspectJAutoProxy(exposeProxy = true)`，侵入性较强）；③ **把方法拆到另一个 Bean**（最干净，本质是重构职责）；此外还可用 AspectJ 编译期/加载期织入（LTW）彻底解决，但复杂度高。
- 执行顺序：多个切面用 `@Order` 或实现 `Ordered`，值越小越靠外层（先执行 before、后执行 after）。

**考察意图**
几乎必考。"同类调用失效"是 Spring 最经典的坑，能说出 `this` 指向原始对象、并给出**拆分为另一个 Bean** 这一根治方案，才算真正解决过问题。

**延伸追问**
- 为什么 `@Transactional` 在同一个类里 `this.method()` 调用会失效？（同一个坑，事务也是 AOP）
- CGLIB 为什么不能代理 `final` 方法？（无法覆写）
- JDK 动态代理为什么要求目标类实现接口？（生成的代理类已继承 `Proxy`，Java 单继承）
- `private` 方法上加 `@Transactional` 有用吗？（无效，子类无法覆写）

---

#### 5.8 @Transactional 失效的 7 类场景？
> **难度**：L3 深入 ｜ **考察频次**：高 ｜ **标签**：Spring、事务

**参考答案要点**

1. **自调用（同类方法内部调用）**：`this.method()` 走原始对象而非代理，拦截链不生效。解法：注入自身代理 / `AopContext.currentProxy()` / 拆到另一个 Bean。
2. **方法不是 `public`**（或为 `final`/`private`/`static`）：`AbstractFallbackTransactionAttributeSource` 只认 public 方法；CGLIB 也无法覆写 final/private 方法。
3. **异常被 `catch` 吞掉**：拦截器靠捕获异常来决定回滚，`try-catch` 后不再抛出，事务会**正常提交**。必须重新抛出或手动 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`。
4. **抛出的异常类型不对**：默认只回滚 **`RuntimeException` 和 `Error`**，受检异常与自定义非运行时异常**不回滚**。必须写 `@Transactional(rollbackFor = Exception.class)`。
5. **传播行为配置错误**：如 `NOT_SUPPORTED`（挂起事务非事务执行）、`NEVER`、`SUPPORTS`（无事务时非事务执行），都会导致"看起来加了注解却没有事务"。
6. **类未被 Spring 管理**：没有 `@Service`/`@Component`，或对象是自己 `new` 出来的，代理根本不存在。
7. **数据库/引擎不支持事务**：MySQL 使用 **MyISAM** 引擎（无事务；必须 InnoDB），或 DDL/存储引擎层面的限制。
8. **（补充高频）多线程/异步调用**：事务上下文（`Connection`）绑定在 `ThreadLocal` 上，子线程、`@Async`、线程池提交的任务不在同一事务中，回滚互不影响。

**考察意图**
本题是 Spring 面试出现率最高的一题。能否**分类枚举出 7 类**并额外提到"异步/多线程"和"异常类型"，是判断实战经验的硬指标。

**延伸追问**
- `@Transactional` 加在接口上和加在实现类上有什么区别？
- 为什么 `rollbackFor` 默认只认 `RuntimeException`？（受检异常通常表示"可恢复"，按 EJB 传统约定）
- 事务方法里做了 RPC 调用，回滚能撤回远程操作吗？（不能，需要 TCC/Saga/本地消息表等分布式事务方案）

---

#### 5.9 事务传播行为有哪 7 种？REQUIRES_NEW 与 NESTED 的区别？
> **难度**：L3 深入 ｜ **考察频次**：高 ｜ **标签**：Spring、事务传播

**参考答案要点**

| 传播行为 | 语义 |
| --- | --- |
| `REQUIRED`（默认） | 有事务则加入，无则新建 |
| `SUPPORTS` | 有则加入，无则**非事务**执行 |
| `MANDATORY` | 必须有事务，无则抛异常 |
| `REQUIRES_NEW` | **挂起当前事务**，新建独立事务 |
| `NOT_SUPPORTED` | 挂起当前事务，非事务执行 |
| `NEVER` | 有事务则抛异常 |
| `NESTED` | 在当前事务内建 **savepoint**（无事务时等同 REQUIRED） |

- **REQUIRES_NEW vs NESTED（核心区别）**：
  - `REQUIRES_NEW`：**完全独立的物理事务**——外层事务被挂起，内层使用**新的数据库连接**、独立提交/回滚；**内层提交后，外层再回滚也撤不回内层的提交结果**。适用于"必须独立落库的日志/审计/流水"。
  - `NESTED`：**同一个物理事务 + 保存点**——共用外层连接，内层回滚只回到 savepoint，**外层回滚会连带回滚嵌套部分**；内层提交对外层无实际提交效果。适用于"主流程失败则子流程一并回滚，子流程失败不影响主流程"。
  - `NESTED` 依赖 JDBC savepoint（或嵌套事务支持），JPA/某些数据源可能不支持。
- **隔离级别**（`Isolation` 枚举）：`DEFAULT`（用数据库默认）、`READ_UNCOMMITTED`、`READ_COMMITTED`、`REPEATABLE_READ`、`SERIALIZABLE`。MySQL InnoDB 默认 **REPEATABLE READ**，Oracle/PostgreSQL 默认 **READ COMMITTED**。
- 补充：`@Transactional(readOnly = true)` 可让 JDBC/ORM 走只读优化（MySQL 8 主从路由、`flushMode=MANUAL`），但**仅在没有写操作的业务方法上用**，否则可能不报错但不生效。
- `timeout`（超时，依赖底层）、`rollbackFor/noRollbackFor`（回滚异常规则）。

**考察意图**
能否把 REQUIRES_NEW 与 NESTED 的"独立提交 vs 保存点"和"外层回滚是否影响内层"讲清楚，是本题唯一的分水岭；其余六种能准确描述即可。

**延伸追问**
- 一个 `REQUIRED` 方法调用另一个 `REQUIRED` 方法，事务是几个？（一个，同连接）
- `REQUIRES_NEW` 会不会导致数据库连接耗尽？（会，内外层各占一条连接，需评估连接池大小）
- 为什么 `NESTED` 在某些数据源下等价于 `REQUIRED`？

---

#### 5.10 Spring 中用到了哪些设计模式？分别对应哪里？
> **难度**：L2 进阶 ｜ **考察频次**：高 ｜ **标签**：Spring、设计模式

**参考答案要点**

| 设计模式 | Spring 中的落点 |
| --- | --- |
| **工厂模式** | `BeanFactory`（简单工厂）、`FactoryBean`（工厂方法，如 `SqlSessionFactoryBean`） |
| **单例模式** | Bean 默认 `scope="singleton"`，由 `DefaultSingletonBeanRegistry` 的单例池保证 |
| **代理模式** | AOP 的 JDK 动态代理 `JdkDynamicAopProxy` 与 CGLIB `CglibAopProxy`；`@Transactional` 的事务代理 |
| **模板方法** | `JdbcTemplate` / `RestTemplate` / `RedisTemplate` / `TransactionTemplate`；`AbstractApplicationContext.refresh()` 定义容器刷新骨架 |
| **观察者模式** | `ApplicationEvent` + `ApplicationListener` / `@EventListener`；`ContextRefreshedEvent`、`ApplicationReadyEvent` |
| **适配器模式** | `HandlerAdapter`（适配各种 Controller 写法）、`AdvisorAdapter`（把 Advice 适配成 MethodInterceptor） |
| **策略模式** | `InstantiationStrategy`（CGLIB vs 简单反射）、`Resource` 的多种实现、`AopProxy` 的两种实现、`PlatformTransactionManager` 的多种实现 |
| **责任链模式** | `FilterChainProxy`（Spring Security）、`HandlerInterceptor` 链、`BeanPostProcessor` 链、`Interceptor` 责任链 |
| **装饰器模式** | `HttpRequestWrapper`、`TransactionAwareCacheDecorator`、`HttpServletRequestWrapper` |
| **建造者模式** | `BeanDefinitionBuilder`、`MockMvcRequestBuilders`、`ResponseEntity.BodyBuilder` |
| **委派/门面** | `DispatcherServlet`（统一入口，`Front Controller` 模式）、双亲委派式的 `ClassLoader` 选择 |

- 加分点：指出 `JdbcTemplate` 解决的是"样板代码"（打开连接—执行—关连接—异常处理），把不变流程固化、变化部分交给 `RowMapper`/`PreparedStatementSetter` 回调。

**考察意图**
要求"至少 8 个并指出具体位置"——只背名字不给落点会被追问。能说出 `DispatcherServlet` 是前端控制器、`JdbcTemplate` 是模板方法，说明确实读过源码。

**延伸追问**
- `BeanFactory` 是简单工厂还是抽象工厂？
- 为什么 AOP 用代理模式而不用装饰器模式？
- Spring 的事件机制是同步还是异步的？如何改成异步？（`@Async` + `SimpleApplicationEventMulticaster.setTaskExecutor`）

---

#### 5.11 Spring Boot 自动装配的原理？2.7 → 3.x 有什么变化？
> **难度**：L3 深入 ｜ **考察频次**：高 ｜ **标签**：Spring Boot、自动装配

**参考答案要点**
- **入口**：`@SpringBootApplication` = `@SpringBootConfiguration`（即 `@Configuration`）+ `@EnableAutoConfiguration` + `@ComponentScan`。
- **核心链路**：`@EnableAutoConfiguration` 上标注 `@Import(AutoConfigurationImportSelector.class)` → 容器刷新时 `ConfigurationClassPostProcessor` 解析 `@Import` → `AutoConfigurationImportSelector.getAutoConfigurationEntry()`：
  1. `getCandidateConfigurations()` 读取候选类；
  2. **去重**（`LinkedHashSet`）；
  3. **排除**（`@SpringBootApplication(exclude=...)` / `spring.autoconfigure.exclude`）；
  4. **过滤**：`OnClassCondition` / `OnBeanCondition` / `OnPropertyCondition` 等条件匹配；
  5. 触发 `AutoConfigurationImportEvent`，最后把这些类作为**配置类**注册进容器。
- **配置文件的版本差异（必答）**：
  - **Spring Boot ≤ 2.6**：候选类写在 `META-INF/spring.factories` 的 `org.springframework.boot.autoconfigure.EnableAutoConfiguration` 键下，由 `SpringFactoriesLoader` 读取。
  - **Spring Boot 2.7**：引入新机制 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（**每行一个全限定类名，无 key、无反斜杠续行**），并新增 `@AutoConfiguration` 注解（等价于 `@Configuration(proxyBeanMethods = false)` + 排序语义）。2.7 对两种方式**都支持并去重**。
  - **Spring Boot 3.0+**：**彻底移除** `spring.factories` 中 `EnableAutoConfiguration` 键的支持，`.imports` 成为唯一方式；其他 SPI（如 `ApplicationListener`、`ApplicationContextInitializer`）也各自迁移到 `META-INF/spring/<接口全名>.imports`。
  - 迁移原因：职责单一（按需加载，只解析需要的文件）、解析更快、为 **AOT / GraalVM Native Image** 提前确定 Bean 做准备。
- **常用条件注解**：`@ConditionalOnClass`（类路径存在某类，用 ASM 读元数据所以"类缺失也不报错"）、`@ConditionalOnMissingBean`（**用户自定义优先**）、`@ConditionalOnProperty(prefix, name, havingValue, matchIfMissing)`、`@ConditionalOnBean`、`@ConditionalOnWebApplication`、`@ConditionalOnExpression`。
- 调试：`--debug` 或 `actuator/conditions` 查看装配报告（哪些生效、哪些因条件不满足被跳过）。

**考察意图**
区分 2.7 与 3.0 的文件差异是版本敏感度的硬指标；能解释"为什么改"（AOT + 按需加载）以及"@ConditionalOnMissingBean 保证用户配置优先"才算理解自动装配的精髓。

**延伸追问**
- 为什么自动配置类的 `@Bean` 方法要配 `@ConditionalOnMissingBean`？
- 写一个自定义 starter 的完整步骤？（见 5.12）
- `@AutoConfiguration` 为什么默认 `proxyBeanMethods = false`？（避免 CGLIB 代理开销，自动配置类之间不互相调用 @Bean 方法）
- Spring Boot 3 的 AOT 对自动装配有什么影响？（编译期生成 `__BeanDefinitions` 类，运行时不再扫描 `.imports`）

---

#### 5.12 Spring Boot 的启动流程？内置 Tomcat 如何启动？如何自定义 starter？
> **难度**：L2 进阶 ｜ **考察频次**：中 ｜ **标签**：Spring Boot、启动流程

**参考答案要点**
- **阶段一 `new SpringApplication(...)`**：推断应用类型（Servlet / Reactive / None）、从 `spring.factories`（或 `.imports`）加载 `ApplicationContextInitializer` 与 `ApplicationListener`、推断主启动类。
- **阶段二 `run(args)`**：① 记录启动耗时；② 获取 `SpringApplicationRunListeners` 并发布 `ApplicationStartingEvent`；③ **准备 Environment**（加载 `application.yml/properties`、命令行参数、Profile，发布 `ApplicationEnvironmentPreparedEvent`——`ConfigDataEnvironmentPostProcessor` 在这里工作）；④ 打印 Banner；⑤ **创建 ApplicationContext**（Servlet 环境为 `AnnotationConfigServletWebServerApplicationContext`）；⑥ **prepareContext**（关联 Environment、执行 Initializer、注册主类 BeanDefinition，发布 `ApplicationPreparedEvent`）；⑦ **refreshContext**——核心是 `AbstractApplicationContext.refresh()`：`invokeBeanFactoryPostProcessors`（`ConfigurationClassPostProcessor` 解析 `@Configuration`/`@Import`/`@ComponentScan`）→ `registerBeanPostProcessors` → `initMessageSource` → `initApplicationEventMulticaster` → **`onRefresh()`** → `finishBeanFactoryInitialization`（实例化所有非懒加载单例）→ `finishRefresh`；⑧ 发布 `ApplicationStartedEvent`；⑨ `callRunners`（`ApplicationRunner` / `CommandLineRunner`）；⑩ 发布 **`ApplicationReadyEvent`**（K8s 就绪探针应基于此）。失败则发布 `ApplicationFailedEvent`。
- **内置 Tomcat 原理**：`spring-boot-starter-web` 引入 `tomcat-embed-core`；`ServletWebServerApplicationContext.onRefresh()` → `createWebServer()` → `getWebServerFactory()` 拿到 `TomcatServletWebServerFactory`（由 `ServletWebServerFactoryAutoConfiguration` 装配）→ `getWebServer()` 中 `new Tomcat()`、设置 `Connector`（端口来自 `server.port`）、`Context`，再 `tomcat.start()`。替换容器只需排除 tomcat starter 并引入 `spring-boot-starter-jetty` / `undertow`；WebFlux 默认用 Netty。
- **自定义 starter 三步**：① `xxx-spring-boot-starter`（**只做依赖聚合，无代码**）；② `xxx-spring-boot-autoconfigure` 中写 `@AutoConfiguration` + `@ConditionalOnXxx` + `@EnableConfigurationProperties(XxxProperties.class)`（用 `@ConfigurationProperties(prefix="xxx")` 绑定配置）；③ 在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 中逐行登记配置类。命名规范：官方 `spring-boot-starter-xxx`，第三方 `xxx-spring-boot-starter`。

**考察意图**
能否说出 `onRefresh()` 是内置容器的切入点、以及 `ApplicationReadyEvent` 的语义，是判断有没有读过启动源码的关键；自定义 starter 的"两个模块 + .imports 文件"是实操能力的检验。

**延伸追问**
- `ApplicationRunner` 与 `CommandLineRunner` 的区别？
- 想在 Bean 全部初始化完成后执行逻辑，有几种方式？
- Spring Boot 3 的 AOT / Native Image 对启动流程做了哪些改造？

---

#### 5.13 Spring MVC 一个请求的完整处理流程？
> **难度**：L2 进阶 ｜ **考察频次**：高 ｜ **标签**：Spring MVC

**参考答案要点**
1. 用户请求到达 **`DispatcherServlet`**（前端控制器，`.doDispatch()`）。
2. **`HandlerMapping`**（`RequestMappingHandlerMapping`）根据 URL 匹配到 `HandlerMethod` 及拦截器链，返回 `HandlerExecutionChain`。
3. 根据 Handler 选择 **`HandlerAdapter`**（`RequestMappingHandlerAdapter`）。
4. 依次执行拦截器的 **`preHandle()`**（任一返回 false 则中断）。
5. `HandlerAdapter.handle()`：
   - **参数解析**：`HandlerMethodArgumentResolver` 处理 `@RequestParam`、`@PathVariable`、`@RequestBody`、`@RequestHeader` 等；`@RequestBody` 由 `RequestResponseBodyMethodProcessor` + **`HttpMessageConverter`**（如 `MappingJackson2HttpMessageConverter`）反序列化。
   - **反射调用** Controller 方法。
   - **返回值处理**：`HandlerMethodReturnValueHandler`；有 `@ResponseBody` 时用 `HttpMessageConverter` 序列化写回（`@RestController` = `@Controller` + `@ResponseBody`）；否则返回 `ModelAndView`。
6. 执行拦截器的 **`postHandle()`**。
7. `processDispatchResult()`：若抛异常则交给 **`HandlerExceptionResolver`**（`@ControllerAdvice` + `@ExceptionHandler` 由 `ExceptionHandlerExceptionResolver` 处理）生成错误响应；否则**视图解析**（`ViewResolver` → 渲染 HTML/JSON）。
8. 最后执行拦截器的 **`afterCompletion()`**（无论成功失败都会执行，用于清理资源、日志、耗时统计）。
- 补充：参数校验 `@Valid` 由 `MethodArgumentNotValidException` + `@ControllerAdvice` 统一处理；跨域由 `CorsFilter`/`@CrossOrigin` 处理。

**考察意图**
能否完整走完 8 步并说出 `HandlerMapping`/`HandlerAdapter`/`HttpMessageConverter` 三个核心组件的职责，以及 `@ControllerAdvice` 所处的环节。

**延伸追问**
- `DispatcherServlet` 是不是单例？它是线程安全的吗？（是单例，靠无状态设计保证安全）
- `@RequestBody` 与 `@RequestParam` 分别由哪个解析器处理？
- 拦截器 `preHandle` 返回 false 后，`afterCompletion` 还会执行吗？（**不会**，只有已执行过 preHandle 且返回 true 的拦截器才会被回调 afterCompletion）
- Spring MVC 与 WebFlux 的核心差异？

---

#### 5.14 Filter、Interceptor、Aspect 的执行顺序？三者有什么区别？
> **难度**：L2 进阶 ｜ **考察频次**：中 ｜ **标签**：Spring MVC、AOP

**参考答案要点**
- **完整调用链**：
```
Filter.doFilter(前半段)
  └─ DispatcherServlet.doDispatch()
       └─ HandlerInterceptor.preHandle()
            └─ [Aspect @Around 前半 / @Before]
                 └─ Controller 方法
            └─ [Aspect @AfterReturning / @Around 后半]
       └─ HandlerInterceptor.postHandle()
       └─ 视图渲染 / HttpMessageConverter 写回
       └─ HandlerInterceptor.afterCompletion()
  └─ Filter.doFilter(后半段)
```

| 维度 | Filter | Interceptor | Aspect |
| --- | --- | --- | --- |
| 归属 | **Servlet 规范**（`javax/jakarta.servlet.Filter`，Tomcat 层） | **Spring MVC**（`HandlerInterceptor`） | **Spring AOP**（`@Aspect`） |
| 拦截范围 | 所有请求（含静态资源、非 DispatcherServlet 请求） | 只拦截进入 `DispatcherServlet` 的请求 | 只能切 **Spring Bean 的方法** |
| 可获取的信息 | `ServletRequest/Response`（拿不到 Handler 信息） | `HandlerMethod`（可知方法、注解、参数） | 方法签名、参数、返回值、异常 |
| 注入 Bean | 可用（`DelegatingFilterProxy` 或 `@Component` + `FilterRegistrationBean`） | 完全由 Spring 管理 | 完全由 Spring 管理 |
| 典型用途 | 编码、跨域、XSS、认证入口（Spring Security 的 `FilterChainProxy`） | 登录校验、权限、日志、幂等 | 事务、日志、审计、限流 |

- 顺序控制：Filter 用 `@Order` / `FilterRegistrationBean.setOrder`；Interceptor 在 `WebMvcConfigurer.addInterceptors` 中的注册顺序（或 `Ordered`）；Aspect 用 `@Order`（值越小越靠外层）。
- **优先级结论**：**Filter > Interceptor.preHandle > Aspect > Controller > Aspect > Interceptor.postHandle > afterCompletion > Filter 尾部**。

**考察意图**
考察是否能画出完整的嵌套调用链。常见错误是把 AOP 放在 Interceptor 之前——实际上 AOP 发生在 `HandlerAdapter` 内部调用 Controller 时，晚于 `preHandle`。

**延伸追问**
- 为什么 Spring Security 用 Filter 而不是 Interceptor？（需要在进入 DispatcherServlet 之前完成认证，且不依赖 Spring MVC）
- 想拿到 Controller 方法上的自定义注解做权限控制，应该用 Filter、Interceptor 还是 AOP？（Interceptor 或 AOP，因为需要 `HandlerMethod`）
- 全局异常处理放在 Filter 层为什么捕获不到 Controller 异常？（异常已被 DispatcherServlet 内的 `HandlerExceptionResolver` 消化）

---

#### 5.15 MyBatis：`#{}` 与 `${}` 的区别？Mapper 接口为什么没有实现类？一二级缓存如何失效？
> **难度**：L3 深入 ｜ **考察频次**：高 ｜ **标签**：MyBatis

**参考答案要点**
- **`#{}` vs `${}`**：
  - `#{}` 是**预编译占位符**，MyBatis 会生成 `PreparedStatement` 并用 `?` 占位，执行前 `setXxx()` 赋值，**自动加引号、做类型处理、能防 SQL 注入**。
  - `${}` 是**字符串直接拼接**（`Statement`），有 SQL 注入风险。仅用于**动态表名、列名、`ORDER BY` 字段**等无法参数化的位置，且**必须对传入值做白名单校验**。
  - 补充：`#{}` 在 `LIKE` 中应写 `LIKE CONCAT('%', #{kw}, '%')`；能参数化的地方一律用 `#{}`。
- **Mapper 接口原理（JDK 动态代理）**：① 启动时 `@MapperScan`（`MapperScannerConfigurer` → `ClassPathMapperScanner`）把接口注册为 **`MapperFactoryBean`**（`FactoryBean`，`getObject()` 返回 `sqlSession.getMapper(type)`）；② `getMapper` 通过 `MapperProxyFactory` 创建 `MapperProxy`（**JDK 动态代理**）与 `MapperMethod`；③ 调用方法 → `MapperMethod.execute()` 根据 `SqlCommandType`（SELECT/INSERT/UPDATE/DELETE）调用 `SqlSession.selectOne/insert/...` → `Executor` → 查 `MappedStatement`（XML/注解解析得到，含 `BoundSql`、参数映射、结果映射）→ `StatementHandler` → `ParameterHandler` → `ResultSetHandler` 映射结果。
  - 因为本质是 JDK 代理，所以 **Mapper 接口不能有重载方法**（XML 里按"全限定名 + 方法名"唯一定位 `MappedStatement`）。
- **一级缓存**：`SqlSession` 级别（`BaseExecutor.localCache`，`PerpetualCache` 内部是 `HashMap`），**默认开启**，作用域 `SESSION`（可设 `STATEMENT`）。失效场景：① 执行 `insert/update/delete`（无论是否提交）会 `clearLocalCache()`；② 手动 `sqlSession.clearCache()`；③ 不同 `SqlSession` 不共享。**与 Spring 集成后基本失效**——`SqlSessionTemplate` 是非单例的线程安全代理，每次操作结束就关闭 SqlSession；只有在同一事务内才复用同一个 SqlSession（此时一级缓存生效）。
- **二级缓存**：`namespace`（Mapper）级别，默认**关闭**，需 XML 加 `<cache/>` 或 `@CacheNamespace`，实体类需 `Serializable`。失效：同 namespace 的 CUD 操作（`flushCache="true"`）会清空该 namespace 缓存。缺点：**多表关联查询时极易产生脏数据**（跨 namespace 的更新不会清空对方缓存），分布式环境必须换成 Redis（实现 `Cache` 接口或引入 `mybatis-redis`）。

**考察意图**
三个高频点合在一起考。`#{}` 的预编译原理、"Mapper 是 JDK 动态代理 + MapperFactoryBean"、"Spring 集成后一级缓存形同虚设"是必须答出的三句关键结论。

**延伸追问**
- 一级缓存会不会导致脏数据？（会，一个 SqlSession 内两次查询之间，其他会话改了数据）
- 为什么生产环境通常关掉二级缓存？（脏读风险 + 分布式不一致，收益不如在 Service 层用 Redis）
- `SqlSessionTemplate` 为什么是线程安全的？（代理模式，每次操作通过 `SqlSessionInterceptor` 从 `ThreadLocal` 获取/新建再关闭）

---

#### 5.16 MyBatis 插件机制与分页插件原理？动态 SQL、ResultMap 延迟加载、批量插入？
> **难度**：L2 进阶 ｜ **考察频次**：中 ｜ **标签**：MyBatis

**参考答案要点**
- **插件机制**：实现 `Interceptor` 并用 `@Intercepts(@Signature(type=..., method=..., args=...))` 声明拦截点。MyBatis 只允许拦截四大对象的方法：**`Executor`**（`update`/`query`/`flushStatements`）、**`ParameterHandler`**（`getParameterObject`/`setParameters`）、**`ResultSetHandler`**（`handleResultSets`）、**`StatementHandler`**（`prepare`/`parameterize`/`batch`）。原理：`InterceptorChain.pluginAll()` 用 **`Plugin.wrap()` 生成 JDK 动态代理并层层包裹**（责任链），执行到 `invocation.proceed()` 才继续下一个。
- **PageHelper 分页原理**：`PageHelper.startPage(pageNum, pageSize)` 把分页参数放进 **`ThreadLocal`**（`Page` 对象）→ 插件拦截 `Executor.query()` → 判断存在 Page 参数，则先执行一次 `count` 查询得到总数，再**改写 SQL**（按方言拼 `LIMIT ?, ?` 或 `ROW_NUMBER()`/`ROWNUM`）执行 → 结果封装为 `PageInfo`。**注意事项**：`startPage` 后必须紧跟**第一条**查询语句，中间插入其他查询会导致分页错位；用完建议 `PageHelper.clearPage()`。
- **动态 SQL**：`<if>`、`<choose>/<when>/<otherwise>`、`<where>`、`<set>`、`<trim>`（自定义前后缀与忽略分隔符）、`<foreach>`（批量 `IN` / 批量插入）、`<bind>`；OGNL 表达式；注解中用 `<script>` 包裹。
- **ResultMap**：解决列名与属性名不一致（`mapUnderscoreToCamelCase` 可开驼峰自动映射）；复杂映射用 `<association>`（一对一，嵌套查询或嵌套结果）与 `<collection>`（一对多）。**延迟加载**：全局 `lazyLoadingEnabled=true` + `aggressiveLazyLoading=false`（或属性级 `fetchType="lazy"`），原理是给结果对象创建代理，调用 getter 时才发起第二次查询。
- **批量插入三种写法**：① `foreach` 拼接 `INSERT INTO t (...) VALUES (...),(...)`（最快，但受 `max_allowed_packet` 限制，**建议每批 500~1000 条**）；② `ExecutorType.BATCH` 的 `SqlSession`（复用 PreparedStatement + `addBatch` + 定期 `flushStatements`，注意此模式下 `insert` 返回值不可用）；③ MyBatis-Plus `saveBatch`（本质也是 BATCH 模式 + 分批）。
- 连接池：生产用 `HikariCP`（Spring Boot 默认），配合 `rewriteBatchedStatements=true`（MySQL）才能真正发挥批量插入性能。

**考察意图**
考察 MyBatis 的"工程用法"而非 CRUD：能否说出四大拦截对象、PageHelper 的 ThreadLocal 陷阱、`rewriteBatchedStatements` 这类真实调优参数。

**延伸追问**
- 分页插件为什么要先 count？大数据量表如何优化分页？（延迟关联 / 游标分页 `WHERE id > lastId`）
- `<foreach>` 拼 IN 条件时元素过多会怎样？（SQL 过长、解析慢，需拆分）
- 延迟加载在什么情况下会失效或产生 N+1 问题？

---

#### 5.17（加分）Spring Boot 3 / Spring Framework 6 有哪些关键变化？
> **难度**：L2 进阶 ｜ **考察频次**：中 ｜ **标签**：Spring Boot 3、新特性

**参考答案要点**
- **基线升级**：要求 **Java 17+**（Spring Framework 6 / Boot 3.0，2022-11）；Jakarta EE 9+，所有 `javax.*` 包名迁移为 **`jakarta.*`**（`jakarta.servlet`、`jakarta.persistence`、`jakarta.validation`）——这是升级最大的工作量。
- **自动装配文件变更**：`spring.factories` 的 `EnableAutoConfiguration` 键在 3.0 移除，统一为 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（详见 5.11）。
- **AOT 与 GraalVM Native Image**：Spring Framework 6 引入 AOT 引擎（编译期生成 Bean 定义 Java 源码、`RuntimeHints`、代理类），Spring Boot 3 提供 `native-maven-plugin`，可把应用编译为**原生可执行文件**（启动 < 100ms、内存大幅下降）。代价：构建慢、反射/动态代理需显式声明 `@RegisterReflectionForBinding` 或 `RuntimeHintsRegistrar`。
- **可观测性**：Micrometer Tracing 取代 Spring Cloud Sleuth；统一 `Observation` API 覆盖 trace + metrics；Actuator 端点增强。
- **Web 层**：`ProblemDetail`（RFC 7807 错误响应）；Spring 6.0 的 HTTP Interface（`@HttpExchange` 声明式客户端）、6.1 引入 `RestClient`（`RestTemplate` 的现代替代，fluent API）；Spring Security 6 的 Lambda DSL 与 `AuthorizationFilter`。
- **其他**：`@ConfigurationProperties` 构造器绑定、`spring-boot-starter-*` 模块化、`spring.main.allow-circular-references` 默认 false、Log4j2 扩展与结构化日志。
- **更前沿（了解即可）**：**Spring Boot 4.0**（2025-11）基于 **Spring Framework 7**，基线 Jakarta EE 11、**Jackson 3**、JSpecify 空安全注解、代码库彻底模块化、增强 `@HttpExchange` 与 API 版本管理；Spring Boot 4.1（2026-06）新增 gRPC 自动装配、HTTP 客户端 SSRF 防护等。**国内多数存量项目仍在 2.x/3.x**，面试以 3.x 为准，能提一句 4.x 的演进方向即可体现技术视野。

**考察意图**
版本演进敏感度题：能说出 `jakarta.*` 迁移、Java 17 基线、AOT/Native、`.imports` 四点即为优秀；顺带提到 Boot 4.x 方向是加分。

**延伸追问**
- 从 Spring Boot 2.x 升级到 3.x，最常见的三个障碍是什么？（javax → jakarta、JDK 版本、第三方库兼容）
- Native Image 的代价是什么？什么场景适合？
- `RestTemplate` 为什么被标记"维护模式"？`RestClient` 与 `WebClient` 怎么选？

---

---

> 完整 161 题见在线版：https://java-interview-guide.app.workbuddy.host/
> 下一章持续更新中，欢迎收藏。
