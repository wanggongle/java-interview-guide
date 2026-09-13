# 第 1 章 Java 语言基础与 JDK 演进

> 摘自《Java 面试全栈通关指南》——192 道题 / 17 个模块，覆盖 Java 后端与 Java AI 应用开发，
> 其中第 13–16 章为**真实业务场景题**（每题先给业务约束与量级，再渐进追问）。
> 在线阅读（可搜索、自测、记进度）：https://java-interview-guide.app.workbuddy.host/

---

#### 1.1 面向对象三大特性与"多态"的底层实现是什么？

> **难度**：L2 进阶 ｜ **考察频次**：高 ｜ **标签**：语言基础 / JVM

**参考答案要点**

- 三大特性：封装（隐藏实现、暴露接口）、继承（复用与扩展）、多态（同一消息作用于不同对象产生不同行为）。
- 多态三要素：继承或实现、方法重写、父类引用指向子类对象（`Parent p = new Child();`）。
- 字节码层面方法调用有 5 条指令：`invokestatic`、`invokespecial`（私有/构造器/super 调用，编译期静态绑定）、`invokevirtual`、`invokeinterface`（运行期动态绑定）、`invokedynamic`。
- HotSpot 在类加载的连接阶段为每个类生成 **vtable 虚方法表**，数组中存放各可继承方法的直接入口地址；子类 vtable 沿用父类槽位布局，**重写方法覆盖同一下标**，因此在 `invokevirtual` 时只需从对象头的 Klass 指针取到 vtable，按下标 O(1) 取址，与继承深度无关。
- `private`、`static`、`final`（及构造器）不进 vtable，走静态绑定，可被 JIT 内联，这也是"final 方法略快"的根因，但现代 JIT 的 CHA（类层次分析）已让这个差距很小。
- 接口调用走 **itable**（因类可实现多个接口，需先定位接口表再索引），HotSpot 还对单态/双态调用点做内联缓存优化。
- `invokedynamic` 是 JDK 7 引入（源于 **JSR 292** 动态语言支持），首次执行时调用 Bootstrap Method 由 `LambdaMetafactory` 生成 `CallSite`，把"方法查找"从虚拟机下沉到 Java 代码层，用于 Lambda、字符串拼接（JDK 9 的 `StringConcatFactory`）、后续 Record/模式匹配的优化——**它与虚方法分派不是同一套机制**，不要混答。

**考察意图**

面试官要区分"背了三大特性定义"和"真的知道一次方法调用在 JVM 里发生了什么"。能说出 vtable/itable 槽位与内联缓存，说明你有源码/ VM 级认知；只会背"多态提高扩展性"通常会被判为 L1。

**延伸追问**

- `private` 方法在子类里再定义一次叫重写吗？（不是，是完全无关的新方法）
- 为什么 Lambda 表达式用 `invokedynamic` 而不是生成匿名内部类？（避免每处 Lambda 生成一个 class 文件，且允许后续替换实现策略而不重编译）
- 反射调用和 `invokedynamic` 的性能差异？
- 继承层次很深时多态会变慢吗？（不会，vtable 是按下标寻址）

---

#### 1.2 重载与重写有什么区别？返回值不同能否构成重载？

> **难度**：L1 入门 ｜ **考察频次**：高 ｜ **标签**：语言基础

**参考答案要点**

- **重载（Overload）**：同一类中方法名相同、**参数列表不同**（类型、个数、顺序任一不同）。编译器按实参的静态类型选择版本，属于**编译期静态多态**。可变参数为最后兜底选择。
- **重写（Override）**：子类重定义父类的方法，要求方法名与参数列表**完全相同**；访问权限不能更严格（`public` → `protected` 非法）；抛出的受检异常不能更宽；建议加 `@Override` 让编译器校验。
- **返回值不同不能构成重载**：Java 的方法签名 = 方法名 + 参数类型列表，**不含返回值**。若允许，`f(1)` 这种忽略返回值的调用编译器无法决定版本，因此编译器直接报 `method is already defined`。
- **唯一例外是协变返回类型**：重写时子类可返回父类返回类型的子类型（父 `Object clone()` → 子 `MyClass clone()`），这是重写而非重载。
- 泛型擦除陷阱：`void f(List<String>)` 与 `void f(List<Integer>)` 擦除后同为 `f(List)`，**不能共存**，报同样的 "have same erasure" 错误。
- 重载解析优先级：**精确匹配 → 基本类型自动提升 → 自动装箱 → 父类/接口参数 → 可变参数**；传 `null` 时会选继承树上最具体的那个类型。
- 父子类同名**静态**方法是"隐藏（hide）"而非重写，调用版本由引用的声明类型决定。

**考察意图**

验证你是否真的理解"编译期绑定 vs 运行期绑定"的分界线——这是后面答好 Lambda、动态代理、桥接方法的前置知识。

**延伸追问**

- `int` 和 `Integer` 参数同时重载，传 `1` 会调哪个？
- 为什么 JDK 里 `Arrays.asList` 、 `String.valueOf` 要提供那么多重载版本？（避免装箱/数组开销，提高 JIT 内联率）
- 手写一段代码让我判断输出（常见：传 `null` 给两个存在继承关系的重载参数）

---

#### 1.3 String、StringBuilder、StringBuffer 有什么区别？字符串常量池与 intern() 在 JDK 7 前后有何变化？

> **难度**：L2 进阶 ｜ **考察频次**：极高 ｜ **标签**：语言基础 / JVM

**参考答案要点**

- **String** 不可变：`final class`，JDK 9 起内部为 `byte[] value + byte coder`（Latin1 用 1 字节、UTF16 用 2 字节，JEP 254 紧凑字符串），此前为 `char[]`。不可变带来天然线程安全、可缓存 hash、可作 HashMap key、可进常量池复用。
- **StringBuilder** 非线程安全，继承 `AbstractStringBuilder`，默认容量 16，扩容规则 `newCapacity = old*2 + 2`（ArraysSupport.newLength）。**单线程首选**。
- **StringBuffer** 方法加 `synchronized`，JDK 本身还带 `toStringCache` 字段（写时置 null），多线程共享时使用，但锁开销明显。
- **常量池位置变化**：JDK 6 及之前 StringTable 位于**永久代 PermGen**，容量固定（默认 1009 桶），`intern()` 过多会抛 `OutOfMemoryError: PermGen space`；**JDK 7 起 StringTable 移到堆内存**，可被 GC 回收；JDK 8 用 Metaspace 取代永久代后，StringTable 依然在**堆**中。该项可用 `-XX:StringTableSize`（桶数）与 `-XX:+PrintStringTableStatistics` 观测。
- `new String("ab")` 若池中无 "ab"，先在池中创建字面量、再在堆上 new 一个对象，共 2 个对象。
- **`intern()` 语义差异**：JDK 6 是"把字符串复制一份到 PermGen 再返回新引用"，所以 `s.intern() == s` 恒为 false；**JDK 7+ 是把堆中该对象的引用直接登记到 StringTable 并返回该引用**，因此 `s.intern() == s` 可能为 true（要求 s 本身是堆上的 `new` 出来的对象）。
- 编译期常量折叠：`"a"+"b"` 编译期即为 "ab"；变量参与则编译为 `new StringBuilder().append().toString()`，因此**循环内用 `+` 拼接会反复创建 StringBuilder**，应显式声明一个 StringBuilder 复用。

**考察意图**

判断是否真正经历过"常量池太大导致 YGC 变慢"或"拼接热点导致对象分配过多"这类线上问题，而不只是背了一道八股。

**延伸追问**

- JDK 9 为什么把 `char[]` 改成 `byte[] + coder`？（堆占用平均下降约 30–40% 收益，代价是部分 char 操作要判别 coder）
- 海量去重场景（如风控名单）用 `HashSet<String>` 还是 `intern()`？（后者减轻堆占用但会撑大 StringTable、拖慢扫描，需权衡 `-XX:StringTableSize`）
- `String::hashCode` 缓存在哪个字段？为什么 String 适合作 HashMap key？

---

#### 1.4 `==` 与 `equals()` 有什么区别？为什么重写 equals 必须重写 hashCode？

> **难度**：L1 入门 ｜ **考察频次**：极高 ｜ **标签**：语言基础 / 集合

**参考答案要点**

- `==`：基本类型比值；引用类型比**对象地址**（是否指向同一块内存）。
- `Object.equals()` 默认实现就是 `return (this == obj)`，所以自定义类不重写 equals 等价于 `==`。
- equals 必须满足五条约定（自反、对称、传递、一致性、对 null 返回 false）；用 JDK 7 起的 `Objects.equals(a, b)` 可免判空。
- **hashCode 的契约**：`a.equals(b) == true` ⇒ `a.hashCode() == b.hashCode()`；反之不成立（哈希冲突允许）。hashCode 应保持稳定且分布均匀。
- **不重写 hashCode 的后果**：两个逻辑相等的对象 hashCode 不同 → 在 HashMap 中落到**不同的桶** → `map.get(k)` 返回 null、`set.contains(k)` 返回 false，集合语义直接失效。这是实际项目中最常见的隐性 Bug 之一。**必须成对重写**。
- 实现建议：Java 7+ 用 `Objects.hash(field1, field2)`；Java 16+ 直接用 `record`（编译器自动生成 equals/hashCode/toString）；或用 Lombok `@EqualsAndHashCode`。
- 陷阱一：作为 key 的对象**尽量不可变**，若把已放入 HashMap 的对象中参与 hash 的字段改掉，会导致再也查不到且无法清理（内存泄漏）。
- 陷阱二：equals 里类型判断用 `getClass()` 还是 `instanceof`，涉及"子类混入"时的对称性；`Objects.hash` 每次会新建 `Object[]`，热点路径可手写 31 累加。

**考察意图**

这是区分"背过八股"和"写过坑"的经典题：能说出"我把一个对象放进 HashSet 后改了它的 ID 字段，然后 contains 返回 false"这类真实经历的候选人，基本可直接判定为 L2。

**延伸追问**

- HashMap 的 `hash(key)` 为什么要 `(h = key.hashCode()) ^ (h >>> 16)`？（把高位信息混入低位，减少只按低位取模时的碰撞）
- 为什么重写 equals 时推荐同时实现 `Comparable` 以保持一致性？
- 两个对象 hashCode 相同一定相等吗？equals 相同 hashCode 一定相同吗？

---

#### 1.5 什么是自动装箱与拆箱？Integer 缓存的范围是多少？为什么推荐用 equals 比较包装类？

> **难度**：L1 入门 ｜ **考察频次**：高 ｜ **标签**：语言基础

**参考答案要点**

- 自动装箱 = 编译器插入 `Integer.valueOf(int)`；拆箱 = 插入 `intValue()`。触发于赋值、算术运算、放入泛型集合、switch、三目运算等。**是编译期语法糖，字节码里没有新指令**。
- `Integer.valueOf` 走 `IntegerCache`，**默认缓存 `[-128, 127]`**；该区间返回同一个对象，`Integer a = 100, b = 100; a == b` 为 **true**，而 `a = 200, b = 200` 为 **false**。上限可通过 `-XX:AutoBoxCacheMax=<n>` 或 `java.lang.Integer.IntegerCache.high` 调整。
- 各包装类缓存情况：`Byte`/`Short`/`Long`/`Integer` 均 `[-128,127]`，`Character` `[0,127]`，`Boolean` 缓存 `TRUE`/`FALSE` 两个常量；**`Float`/`Double` 没有缓存**（浮点无"常用小值"语义）。
- **结论：包装类一律用 `equals()` 比较**，或用 `Integer.compare(a, b)` / `Objects.equals(a, b)`。
- NPE 陷阱：`Integer i = null; int j = i;` 拆箱时抛 NPE；`Integer c = flag ? null : list.get(0)` 等三目/比较运算中的隐式拆箱同样会炸。
- 性能：装箱结果是对象，热点循环里反复装箱会产生大量临时对象，加大 YGC 压力且有 boxing/unboxing 指令开销；可用原始类型特化集合（fastutil、HPPC、Eclipse Collections 的 `IntArrayList`/`IntObjectHashMap`）替代 `List<Integer>`。
- 注意包装类的 `equals` 会先判类型：`Integer.valueOf(1).equals(1L)` 为 false（Long 实例）。

**考察意图**

确认候选人是否踩过"两个 Integer 用 == 比较线上偶发返回 false"的坑，并能把"为什么推荐 equals"讲成工程规范而不是教条。

**延伸追问**

- `Long a = 1L; Long b = 1L; a.equals(b) && a > b` 的输出是什么？（`equals` true，`>` 触发拆箱后为 false）
- 为什么 `-XX:AutoBoxCacheMax` 扩大缓存有时反而有害？（缓存数组常驻堆，且更多的复用对象延长了生命周期）
- `Integer i = Integer.valueOf(127); i++;` 之后 i 是同一个对象吗？（不是，拆箱加一再装箱）

---

#### 1.6 final、finally、finalize 有什么区别？

> **难度**：L1 入门 ｜ **考察频次**：中 ｜ **标签**：语言基础 / JVM

**参考答案要点**

- 三者毫无关系，只是长得像。
- **final**：修饰类（不可继承，如 `String`、`Integer`）、方法（不可重写，JIT 可做方法内联）、变量（基本类型值不可变；引用类型**地址不可变但对象内容可变**）。`final` 字段还有内存语义：JSR-133 保证正确构造的对象，其 final 字段无需同步即可被其他线程正确读到（禁止把 final 写重排序到构造器之外）。
- **finally**：`try-catch-finally` 中无论是否异常都会执行（`System.exit()`、JVM 崩溃、线程被 kill 除外）。**含 return 时的规则**：先计算返回值并暂存，再执行 finally，若 finally 中也有 `return` 会覆盖暂存值；finally 修改基本类型返回值无效，修改返回对象的内容有效。编译器实现是**把 finally 代码块复制到每个可能的出口分支**并生成异常表。
- **finalize**：`Object` 的 protected 方法，对象被判定为可回收时由 **Finalizer 守护线程**调用。问题在于：不保证及时、不保证一定执行、最多一次、可在其中让对象"复活"、拖慢 GC 且极易造成积压 OOM。**JDK 9 标记 `@Deprecated(since="9")`，JDK 18 起默认禁用 Finalization**（可用 `--finalization=disabled` 明确关闭）。
- 替代方案：**try-with-resources / AutoCloseable**（首选），或 JDK 9+ 的 `java.lang.ref.Cleaner`（基于 PhantomReference，独立线程，对象不会被复活）。实际项目中释放堆外内存、文件句柄都应走这两条路。

**考察意图**

验证是否知道 finalize 已被废弃及其原因。能说出"用 Cleaner 替代"和"JDK 18 起默认关闭"是明确的 L2 信号。

**延伸追问**

- `try { return 1; } finally { return 2; }` 返回什么？（2，且 javac 会警告）
- 为什么 finally 里修改返回值对象的内容会生效、修改基本类型不会？（返回值是值/引用的拷贝）
- `final` 字段的内存语义具体解决了什么问题？（逃逸的 this 引用导致的"部分构造对象"可见性问题）

---

#### 1.7 接口与抽象类有什么区别？JDK 8 之后接口为什么引入 default/static 方法？

> **难度**：L1 入门 ｜ **考察频次**：高 ｜ **标签**：语言基础

**参考答案要点**

- **抽象类**：表达 `is-a`，可含构造器、成员变量、静态代码块、具体方法，**单继承**，天然适合实现模板方法模式（如 `AbstractList`、`AbstractQueuedSynchronizer`）。
- **接口**：表达 `can-do` 契约，JDK 8 前只能有 `public abstract` 方法与 `public static final` 常量，**多实现**，不能有实例状态。
- **引入 default/static 方法的直接动机是"接口演化"**：已发布的接口（如 `Collection`、`List`）若要新增方法，所有实现类都会编译失败。有了 `default` 就能在接口里给出默认实现而不破坏二进制兼容——`Collection.forEach()`、`List.sort()`、`Map.getOrDefault()` 就是这样加进去的；同时也让"给接口的方法传 Lambda"变得可行。`static` 方法则提供工厂/工具入口（`Map.of()`、`Stream.of()`、`Comparator.comparing()`）。
- JDK 9 进一步引入 **`private` 方法**，让多个 default 方法能复用代码而不暴露 API。
- 冲突规则：**类优先**（父类具体方法覆盖接口 default）；若两个接口都有同名 default，实现类**必须重写**并用 `Xxx.super.method()` 显式指定。
- 硬限制：default 方法**不能**重写 `Object` 的 `equals`/`hashCode`/`toString`（编译错误）；接口仍不能有实例字段。
- 选型口诀：**需要共享状态、构造逻辑、骨架流程 → 抽象类；需要多继承、给外部扩点、定义 SPI → 接口**。JDK 中二者常组合使用（`List` + `AbstractList`）。

**考察意图**

判断是背了对比表，还是理解了"二进制兼容"这个真实工程约束——能在动机层面作答的候选人通常也对 API 演进（如 `/ deprecation` 策略）有认知。

**延伸追问**

- 抽象有抽象方法就必须类是抽象类吗？接口能有 default 方法之外的实现吗？
- 为什么 `@FunctionalInterface` 要求只有一个抽象方法，却能有多个 default 方法？（SAM 语义只关心待实现的那一个）
- Spring 的 `InitializingBean`、`BeanPostProcessor` 为什么用接口而不是抽象类？（避免占用单继承名额）

---

#### 1.8 受检异常与运行时异常有什么区别？try-with-resources 的原理是什么？异常对性能有什么影响？

> **难度**：L2 进阶 ｜ **考察频次**：高 ｜ **标签**：语言基础 / JVM

**参考答案要点**

- 体系：`Throwable` → `Error`（`OutOfMemoryError`、`StackOverflowError`、`NoClassDefFoundError`，程序无法处理）与 `Exception`；`Exception` → `RuntimeException`（Unchecked）与其余受检异常（`IOException`、`SQLException`、`ClassNotFoundException`、`InterruptedException`）。
- **Checked 必须捕获或声明 `throws`**，编译期强制；**Unchecked 不强制**。业务异常通常继承 `RuntimeException`，配合 `@RestControllerAdvice` 全局处理，避免层层 throws 污染签名。
- **try-with-resources 原理**：资源必须实现 `AutoCloseable`（`Closeable` 的父接口）。编译器将其展开为 `try + finally`，在 finally 中调用 `close()`，多个资源按**声明的逆序**关闭；若 body 与 close 都抛异常，close 的异常会作为 **suppressed exception** 通过 `addSuppressed()` 挂到主异常上（不会丢失主异常）。Java 9 起允许在 `try(...)` 中直接引用外部的 effectively-final 变量。
- **性能影响来源**：构造异常对象时调用 `fillInStackTrace()` 会遍历整个调用栈并生成 `StackTraceElement[]`，这是主要开销，通常比创建普通对象高**一个数量级以上**；栈越深越慢；含 try-catch 的方法也会限制 JIT 的部分优化（异常表影响内联/逃逸分析）；`synchronized`/`native` 边界同样会撑大栈。
- 实践准则：**不用异常做流程控制**；高频失败路径用返回值/`Optional`/状态码；catch 具体类型而非 `Exception`/`Throwable`；不吞异常（`log.error("xxx", e)` 带完整堆栈）；需要极致的场景可重写 `fillInStackTrace()` 返回 `this`（但要接受丢失堆栈的排查代价，Spring 的部分内部控制流异常就这么做）。
- 注意 JVM 默认启用 `-XX:+OmitStackTraceInFastThrow`，对频繁抛出的隐式异常（如热点路径的 `NullPointerException`）会**省略堆栈**，这是线上"日志里没有堆栈"的常见原因，排查时加 `-XX:-OmitStackTraceInFastThrow` 关闭。

**考察意图**

区分"知道 try-with-resources 怎么写"和"知道编译器做了什么、代价在哪"。能说出 `fillInStackTrace` 与 suppressed exception 两个关键词基本可判 L2+。

**延伸追问**

- 为什么 `InterruptedException` 捕获后要重新 `Thread.currentThread().interrupt()`？
- `Error` 能捕获吗？什么时候该捕获？（一般不该，除非你知道如何恢复，如某些 `LinkageError`）
- Spring 事务为什么默认只对 `RuntimeException`/`Error` 回滚？（受检异常需显式 `rollbackFor`）

---

#### 1.9 泛型的类型擦除是怎么回事？什么是桥接方法？PECS 原则怎么理解？

> **难度**：L3 深入 ｜ **考察频次**：高 ｜ **标签**：语言基础 / 集合

**参考答案要点**

- 泛型是 JDK 5 的**语法糖**。**类型擦除**：编译期做类型检查后，字节码中类型参数替换为其上界（无界则 `Object`），并在取出处插入 `checkcast`。因此 `new T()`、`new T[]`、`obj instanceof T`、`catch (T e)`、以及 `(List<String>` 与 `List<Integer>` 的同名重载都非法。
- **泛型信息并未完全丢失**：类/字段/方法/参数的泛型签名写入 class 文件的 **`Signature` 属性**，可通过 `getGenericSuperclass()`、`Method.getGenericReturnType()` 反射读到——这是 Gson 的 `TypeToken`、Jackson 的 `TypeReference`、Spring 的 `ResolvableType` 能还原泛型类型的依据。
- **桥接方法（bridge method）**：子类以具体类型实现泛型父类型时（如 `class MyList implements List<String> { public boolean add(String s) }`），擦除后父类要的是 `add(Object)`，编译器会在子类生成一个 `synthetic bridge` 修饰的 `add(Object)` 转发到 `add(String)`，以保证多态分派成立。可用 `Method.isBridge()`/`isSynthetic()` 判断。**这是反射或 AOP 拿到"参数为 Object 的奇怪方法"的根因**，遍历方法时应过滤。
- **数组协变、泛型不协变**：`String[]` 是 `Object[]` 的子类型，但 `List<String>` **不是** `List<Object>` 的子类型，否则会破坏类型安全，因此需要通配符。
- **PECS（Joshua Bloch, Effective Java）**：**P**roducer **e**xtends, **C**onsumer **s**uper。
  - `? extends T`（生产者）：可以 `get()` 成 T，**不能 `add()`**（除 null），因为无法确定具体子类型。
  - `? super T`（消费者）：可以 `add(T)` 及其子类，**`get()` 只能赋给 Object**。
  - 既读又写就别用通配符，直接用 `List<T>`。
  - 典范：`Collections.copy(List<? super T> dest, List<? extends T> src)`、`Stream.forEach(Consumer<? super T>)`。
- 其它限制：静态方法、静态变量、静态初始化块**不能**引用类的类型参数；裸类型（`List`）会静默关闭全部泛型检查。

**考察意图**

这是典型的分水岭题。能画出"擦除 → 桥接 → Signature 属性 → PECS"这条完整链路，说明你读过 JDK 源码或框架源码；只答"擦除就是运行时看不到类型"只能算 L1。

**延伸追问**

- `List<String>` 和 `List<Integer>` 的 `getClass()` 相等吗？（相等，都是 `class java.util.ArrayList`）
- Gson `TypeToken` 为什么要写成匿名内部类 `new TypeToken<List<User>>(){}`？（只有这样才能在 class 文件的 `Signature` 属性里保留具体泛型参数）
- 为什么 `Optional<T>` 里有很多 `<? super T>` / `<? extends Optional<? extends U>>` 这种签名？

---

#### 1.10 元注解有哪些？自定义注解如何在 Spring 中真正生效？

> **难度**：L2 进阶 ｜ **考察频次**：高 ｜ **标签**：语言基础 / 框架

**参考答案要点**

- **元注解**：`@Target`（作用位置：`TYPE`/`METHOD`/`FIELD`/`PARAMETER`/`CONSTRUCTOR`/`ANNOTATION_TYPE`/`PACKAGE`…）、`@Retention`（`SOURCE`/`CLASS`/`RUNTIME`）、`@Documented`、`@Inherited`（**只对类的继承有效，作用于接口不生效**）、`@Repeatable`（JDK 8）、`@Native`。
- **生命周期差异**：`SOURCE` 编译后即丢弃（`@Override`、Lombok `@Getter`），靠 APT/`javax.annotation.processing` 处理；`CLASS` 保留到 class 文件但 JVM 不加载（部分字节码增强场景）；**只有 `RUNTIME` 能被反射读到**——业务注解、Spring 全家注解必须是 `RUNTIME`。
- 定义方式：`public @interface LogRecord { String value() default ""; int ttl() default 60; }`，属性名为 `value` 时可省略属性名；属性类型只能是基本类型、`String`、`Class`、枚举、注解及其数组。
- **注解本身不产生任何行为，必须有人"读"它**——这是"加了注解没生效"的根本原因。Spring 中生效有三条路径：
  1. **AOP 切面**：`@Aspect` + 切点 `@annotation(com.xxx.LogRecord)` 或 `@within(...)`，运行时拦截（需 Bean 由 Spring 管理）。
  2. **BeanPostProcessor**：在 Bean 初始化前后扫描注解并处理，如 `ScheduledAnnotationBeanPostProcessor` 处理 `@Scheduled`、`AutowiredAnnotationBeanPostProcessor` 处理 `@Autowired`。
  3. **ImportSelector / ImportBeanDefinitionRegistrar**：配合 `@EnableXxx` 在 `BeanDefinition` 注册阶段生效，并可用 `ClassPathScanningCandidateComponentProvider` 扫描指定注解的类（`@MapperScan`、`@EnableFeignClients` 的做法）。
- Spring 还提供 `@AliasFor`（属性互为别名）、组合注解（`@RestController = @Controller + @ResponseBody`）与 `AnnotatedElementUtils.findMergedAnnotation()` 来读取元注解层级。
- 排查清单：Bean 是否被 Spring 托管？Retention 是不是 RUNTIME？方法是 private/self-invocation 导致代理失效了吗？少了 `@EnableXxx` 吗？

**考察意图**

区分"会用注解"和"知道注解如何被处理"。能说出 BeanPostProcessor 与 ImportBeanDefinitionRegistrar 两条路径，说明读过 Spring 源码，面试评级至少 L2+。

**延伸追问**

- `@Transactional` 在同一个类里自调用为什么失效？（未走代理对象）
- `@Inherited` 对方法上的注解有效吗？为什么 Spring 要自己做 `AnnotatedElementUtils` 搜索？
- Lombok 是 `SOURCE` 级别的，它怎么让下游模块也看到生成的方法？（编译期直接改写 AST，生成到 class 文件里）

---

#### 1.11 反射的原理是什么？有什么性能问题？为什么反射会破坏单例、怎么防护？

> **难度**：L2 进阶 ｜ **考察频次**：高 ｜ **标签**：语言基础 / JVM / 框架

**参考答案要点**

- **原理**：类加载后在堆上生成唯一的 `java.lang.Class` 实例，作为访问 `InstanceKlass` 元数据的入口。方法调用靠 `Method.invoke` → `MethodAccessor`。HotSpot 采用 **inflation 机制**：前若干次（默认阈值 `sun.reflect.inflationThreshold=15`）用 `NativeMethodAccessorImpl`，超过后由 `MethodAccessorGenerator` 动态生成字节码版 `MethodAccessorImpl`，以避免每次走 native。
- **性能问题**：① `Class.forName()` 触发类加载；② `getDeclaredMethods()` 每次返回**拷贝的数组并做权限过滤**，`getMethod` 还要遍历父类与接口；③ `Method.invoke` 需 `Object[]` 包装参数并对返回值拆装箱，产生临时对象；④ `setAccessible(true)` 会绕过语言可见性检查（并伴随安全检查开销）；⑤ 反射调用点对 JIT 几乎不可内联，逃逸分析失效。总体比直接调用慢数十倍到上百倍。
- **优化**：缓存 `Method`/`Field`/`Constructor` 对象（静态常量或 Map）；`setAccessible(true)` 只做一次；把 `accessible` 相关的安全检查一次性关闭；热点路径改用 **`MethodHandle`** 或 **`LambdaMetafactory`** 生成函数式接口（性能接近直接调用，这正是 JDK 8 Lambda 的实现方式）；JDK 9+ 模块化后需在 `module-info.java` 中 `opens` 包，否则抛 `InaccessibleObjectException`。
- **典型应用**：Spring IOC 实例化与 `@Autowired` 注入、MyBatis Mapper 参数绑定与结果映射、Jackson/Fastjson 序列化、JDK/CGLIB 动态代理、`Class.forName` 加载 JDBC 驱动、JUnit、BeanUtils。**框架级使用可接受，业务高频路径应避免**。
- **反射破坏单例**：私有构造器被 `constructor.setAccessible(true)` 后 `newInstance()`，就能造出第二个实例；`Unsafe.allocateInstance()`、反序列化、克隆同样可以。**防护措施**：
  1. 构造器中判断 `if (INSTANCE != null) throw new RuntimeException("单例被破坏")`（防反射）；
  2. **用枚举单例**——`Constructor.newInstance()` 对 Enum 会直接抛 `IllegalArgumentException: Cannot reflectively create enum objects`，JVM 层面保证不可反射创建，且天然防御反序列化；
  3. 序列化侧实现 `readResolve()` 返回 `INSTANCE`（Jackson/Java 原生反序列化同理）；
  4. 重写 `clone()` 抛异常或不实现 `Cloneable`。
- JDK 17 起强封装（JEP 403），反射 JDK 内部 API 默认被拒，需 `--add-opens`，这是很多老框架升级 JDK 17 的第一道坎。

**考察意图**

这道题横跨语言基础、JVM 与框架。能说出 inflation 机制、MethodHandle 替代方案、以及"枚举单例是 JVM 级防护"，基本可判定为有源码阅读与线上攻防经验的候选人。

**延伸追问**

- 为什么 `getDeclaredMethods()` 返回的**顺序**在不同 JDK 版本下不稳定？（HotSpot 不保证顺序，这也是很多依赖方法顺序的框架的坑）
- 反射通过设置 `accessible` 绕过检查时，什么情况下会被 SecurityManager/模块系统拒绝？
- Spring 为什么 6.x 起推荐构造函数注入而不是字段反射注入？（不可变性、可测试、避免循环依赖掩盖）

---

#### 1.12 Serializable 与 Externalizable 有什么区别？serialVersionUID 和 transient 的作用？为什么 JDK 序列化被诟病？

> **难度**：L2 进阶 ｜ **考察频次**：中 ｜ **标签**：语言基础 / 中间件

**参考答案要点**

- `Serializable` 是**标记接口**，`ObjectOutputStream` 递归写出类元数据（类名、 serialVersionUID、字段描述）与字段值；反序列化由 `ObjectInputStream` 读取并**完全绕过构造器**（因此不会执行构造器里的校验逻辑）。
- `Externalizable` 要求实现 `writeExternal(ObjectOutput)` / `readExternal(ObjectInput)`，且**必须有 public 无参构造器**（反序列化时先 new 再填充），序列化策略完全由你控制，性能与体积更优但需自行处理版本兼容。
- **`serialVersionUID`**：未显式声明时 JVM 按类名、父类、字段、方法等计算摘要，类结构一变（增删字段）摘要就变，反序列化直接抛 `InvalidClassException`。显式写 `private static final long serialVersionUID = 1L;` 后允许兼容演进：新增字段取类型默认值（对象为 null、int 为 0），删除字段被忽略。IDE 应开启该 warning。
- **`transient`**：标记该字段不参与默认序列化，用于敏感信息（密码、token）、不可序列化的第三方资源引用（Thread/Connection/Socket）或可推导字段。**静态字段天然不参与序列化**（属于类不属于对象）。替代方案：`writeObject`/`readObject` 私有钩子做加密后再写。
- **JDK 序列化被诟病的原因**：① **体积大**（携带完整类元数据与 field 描述）且仅 JVM 内可用，**无法跨语言**；② **性能差**（反射 + 递归 + 大量临时对象）；③ **安全**：反序列化会执行 `readObject` 链上的任意代码，历史上产生大量 RCE（Apache Commons Collections 利用链、Shiro RememberMe、WebLogic），因此安全基线通常要求开启反序列化白名单（JEP 290 `jdk.serialFilter`）或直接禁用；④ **演进能力差**，不支持 schema 管理、字段改名即失效；⑤ 与其它格式相比没有压缩/兼容能力。
- **生产替代**：JSON（Jackson，配合 `@JsonIgnoreProperties(ignoreUnknown = true)` 做向前兼容）、**Protobuf**（IDL + tag 编号，跨语言、天然向后/向前兼容，gRPC 首选）、Avro（Schema Registry）、Kryo/FST（JVM 内高性能）、Hessian、MessagePack（Msgpack）。
- 具体到中间件：`RedisTemplate` 的默认序列化器是 `JdkSerializationRedisSerializer`（JDK 二进制、不可跨语言且 key 常乱码），生产应换成 `Jackson2JsonRedisSerializer` / `GenericJackson2JsonRedisSerializer` / `StringRedisSerializer`。

**考察意图**

你是否真的处理过缓存对象升级后 `InvalidClassException` 导致线上读取失败的事故，以及是否具备"序列化格式选型"的工程判断。

**延伸追问**

- serialVersionUID 改成 `2L` 会发生什么？（老数据反序列化全部失败）
- 为什么建议在 Redis 里不要存 JDK 序列化结果？（跨服务不可读、无法与其它语言服务共存、类的改动即故障）
- Protobuf 为什么能做到向后兼容？（field number + wire type，未知 tag 直接跳过）

---

#### 1.13 BIO、NIO、AIO 有什么区别？NIO 三大组件是什么？零拷贝（mmap / sendfile）原理？

> **难度**：L2 进阶 ｜ **考察频次**：高 ｜ **标签**：语言基础 / IO / 中间件

**参考答案要点**

- 先厘清两对概念：**阻塞/非阻塞** = 线程等待数据的方式；**同步/异步** = 数据就绪后由谁完成拷贝与通知。BIO = 同步阻塞；NIO = 同步非阻塞；AIO = 异步非阻塞。
- **BIO**：`accept()`/`read()` 阻塞当前线程，一连接一线程；线程池版只是"伪异步"，连接数仍受线程数限制，C10K 不可行。
- **NIO（JDK 1.4）三大组件**：
  - **Buffer**：数据容器，核心属性 `capacity`/`position`/`limit`/`mark`，`flip()` 切读模式、`clear()`/`compact()` 切写模式；`DirectBuffer` 分配堆外内存（`ByteBuffer.allocateDirect`），避免一次堆到堆外拷贝但分配/回收成本高、不受 GC 直接管理。
  - **Channel**：全双工，必须与 Buffer 交互（`SocketChannel`/`ServerSocketChannel`/`FileChannel`/`DatagramChannel`），支持 `configureBlocking(false)`。
  - **Selector**：多路复用器，一个线程监听成千上万 Channel 的 `OP_ACCEPT`/`OP_CONNECT`/`OP_READ`/`OP_WRITE`，底层依赖操作系统的 `epoll`（Linux）/ `kqueue`（macOS）/ `IOCP`（Windows）。这就是 **Reactor 模型**（Netty、Redis、Kafka 网络层的基石）。
  - 附：`Zero-copy` 之外还有 Netty 的 `ByteBuf`（读写双指针、池化、`CompositeByteBuf` 实现用户态零拷贝）。
- **AIO（NIO.2，JDK 7）**：`AsynchronousSocketChannel` + `CompletionHandler`/`Future`，OS 完成 IO 后回调。但 **Linux 下 JDK 的实现仍是用线程池模拟 epoll**，并未真正使用异步 IO，收益有限；Netty 在 5.0 alpha 版本尝试 AIO 后回退到 NIO，Linux netty 实践几乎不用 AIO。真正的异步 IO 在 Windows IOCP 上才有意义。
- **零拷贝**：
  - **`sendfile`**：`FileChannel.transferTo()/transferFrom()` 在 Linux 走 `sendfile(2)`，数据从内核页缓存经 DMA 直接送到 socket 缓冲区，**省掉"内核→用户态"这一次拷贝和两次上下文切换**（传统 read+write 是 4 次拷贝、4 次上下文切换）。Kafka 消费端高效、Nginx `sendfile on` 依赖此机制。前提是发送端不需要对数据做加工。
  - **`mmap`**：`FileChannel.map()`（即 `MappedByteBuffer`）把文件映射到进程虚拟地址空间，用户态与内核共享同一页缓存，**减少一次内核到用户的拷贝**，适合小文件随机读、进程间共享内存。RocketMQ 的 CommitLog 用它做页缓存映射。
  - 代价：`MappedByteBuffer` 占虚拟地址空间，卸载时机不明确（靠 Cleaner），频繁映射易导致频繁缺页与 `map failed`；`sendfile` 无法在传输中加工数据。

**考察意图**

判断你是否有过高并发网络编程或中间件使用经历。能同时说出"epoll + Reactor"和"Kafka/RocketMQ 分别选 sendfile/mmap 及原因"，就是 L2+。

**延伸追问**

- Netty 为什么自研 `ByteBuf` 而不用 JDK 的 `ByteBuffer`？（position 需手动 flip、无池化、无引用计数）
- 为什么 Kafka 能这么快？（顺序写 + PageCache + sendfile 零拷贝 + 批量压缩）
- epoll 的 ET/ET 模式区别，JDK Selector 用的是哪种？（水平触发 + 就绪集合轮训）

---

#### 1.14 JDK 版本如何演进？Java 8/11/17/21/25 的关键特性有哪些？企业主流版本与升级痛点是什么？

> **难度**：L2 进阶 ｜ **考察频次**：高 ｜ **标签**：JDK 演进

**参考答案要点**

- **版本节奏**：自 JDK 9 起 Oracle 改为**每 6 个月一个版本，每 2 年一个 LTS**。已发布 LTS：**8（2014-03）、11（2018-09）、17（2021-09）、21（2023-09）、25（2025-09）**。最新非 LTS 是 **Java 26（2026-03-17 发布）**。
- **关键特性对比表**：

| 版本 | 里程碑特性 | 备注 |
|---|---|---|
| **JDK 8（LTS）** | Lambda、Stream API、`Optional`、方法引用、默认方法、`java.time`、Metaspace 取代 PermGen、HashMap 树化、ConcurrentHashMap 改 CAS | 面试仍可能被问，因其存量最大 |
| **JDK 11（LTS）** | `var` 局部变量、`HttpClient` 转正、ZGC（实验）、Epsilon GC、单文件源码运行 `java X.java`、`String.isBlank/lines/repeat`、TLS 1.3、移除 JavaFX/JAXB | 因 Java 17 更优而被跳过升级的典型"过渡版本" |
| **JDK 17（LTS）** | Record（16 转正，JEP 395）、Sealed Class 转正（JEP 409）、Text Block（15 转正，JEP 378）、instanceof 模式匹配（16 转正，JEP 394）、switch 模式匹配预览、**JEP 403 强封装 JDK 内部 API**、增强伪随机、移除 Applet | 当前企业"最稳锚点"，Oracle 商业支持至 2029 |
| **JDK 21（LTS）** | **虚拟线程转正（JEP 444）**、分代 ZGC（JEP 439）、Sequenced Collections（JEP 431）、Record Patterns（JEP 440）、switch 模式匹配转正（JEP 441）、结构化并发（预览，JEP 453）、KDF API | "要不要用虚拟线程"的分水岭，支持至 2031 |
| **JDK 25（LTS）** | ScopedValue 转正（替代 ThreadLocal 的更好方案，尤其配合虚拟线程）、Module Import Declarations、Flexible Constructor Bodies、Compact Source Files & Instance Main Methods、**紧凑对象头 JEP 450（实验特性，需 `-XX:+UseCompactObjectHeaders`，可显著降低堆占用）** | 最新 LTS（2025-09 发布），报告显示采用率已达约 10%；国内仍处于架构团队预研阶段 |

- **版本占比（不同统计口径差异极大，面试务必先说明口径）**：
  - Azul《State of Java Survey 2026》（n=2039）：Java 17 约 34%、Java 21 约 31%、Java 8 约 23%、Java 11 约 15%（Java 21 一年内 +23pp）。
  - Incus Data 2026-03 汇总（引 New Relic / Snyk 等遥测）：Java 21 约 45%、Java 17 约 25–30%、Java 8+11 合计不足 20%、Java 25 已约 10%（被紧凑对象头驱动）。
  - 国内社区/自媒体口径：JDK 17 约 42%、JDK 8 约 38%、JDK 21 仅约 4.5%。
  - **可取的表述**："不同统计口径差异很大（开发者问卷 vs 云厂商遥测 vs 国内自媒体），但共识是：**新项目与面试以 17/21 为主锚点，JDK 8 存量仍大但已不适合作为求职卖点，JDK 25 进入预研视野**"。
- **AI 相关的关键数据**（Azul 2026）：**约 62% 的企业已用 Java 承载 AI 功能**，约 50% 的 Java 应用已包含 AI 代码；约 35% 的 Java 开发者的主应用已通过 Java 原生库调用 LLM；Java 开发者最常用 AI 编码工具为 ChatGPT（58%）、Gemini（51%）、Amazon Q（32%）、Claude（31%）。生态侧：**Spring AI 1.0 GA（2025-05-20）→1.1 GA（2025-11-12，重点为 MCP）→2.0.0-M4（2026-03-26，基于 Spring Boot 4 / Spring Framework 7 / Jakarta EE 11）**；JetBrains 于 2026-03-17 发布 **Koog for Java**；Jakarta EE 12 正在制定 **Jakarta Agentic AI** 规范。
- **升级痛点清单**：① `javax.*` → `jakarta.*` 全量包名替换（Spring Boot 3 起）；② Spring Boot 3.x **强制 JDK 17+**，Lombok/Druid/老 mybatis-generator 等依赖需同步升版本；③ JEP 403 强封装导致 `setAccessible` 报 `InaccessibleObjectException`，需 `--add-opens`；④ JVM 参数与 GC 日志格式变更（`-Xlog:gc*` 取代 `-XX:+PrintGCDetails`）；⑤ 部分加密算法/协议套件被限制、时区与 Locale 数据变更（CLDR 默认 provider 变化导致日期格式不同）；⑥ 虚拟线程与 `synchronized` 的 pinning 问题（**JDK 24 JEP 491 才解决**，JDK 21 上应避免在 synchronized 块内做阻塞 IO）；⑦ Spring Boot 4.0 要求 Jakarta EE 11 且强制 Jackson 3，会影响日期序列化行为。

**考察意图**

面试官想知道你对技术债的态度：是"能用就行"，还是清楚现状、知道升级路径与阻碍。能说出"-add-opens 是因为 JEP 403"和"虚拟线程在 21 上有 pinning 问题"，说明你真的动过升级。

**延伸追问**

- 你们项目现在是哪个版本？如果要升，你的三步计划是什么？（建议答：先升到 17 打平 LTS、用 `jdeps` 扫依赖、引入 `-Xlog:gc` 重新采集基线、再评估 21 + 虚拟线程收益）
- 虚拟线程能完全替代线程池吗？（不能——虚拟线程适合 IO 密集型、不适合长时间 CPU 密集计算；且 synchronized / ThreadLocal / Thread pool 的 Pinning 都需要先清理）
- Record 能替代所有 DTO 吗？（不能继承、非 JavaBean 命名，很多框架反射时需要 getter，需权衡）

---

#### 1.15 综合编码题：实现一个线程安全且支持 LRU 淘汰的小缓存类

> **难度**：L3 深入 ｜ **考察频次**：高 ｜ **标签**：集合 / 并发 / 设计

**参考答案要点**

**选型分析（必须先说清楚，再写代码）**：

| 方案 | 做法 | 取舍 |
|---|---|---|
| **LinkedHashMap + 锁**（面试首选） | `new LinkedHashMap<>(cap, 0.75f, true)` + `removeEldestEntry` + `ReentrantLock` 包一层 | 代码最少（10 行），但**全局一把锁**，写并发高时吞吐受限 |
| **HashMap + 手写双向链表** | 自己维护链表 + `ReentrantReadWriteLock` | 可精细化控制；但"读命中即要 reorder"属于**结构性写**，读写锁收益有限 |
| **ConcurrentHashMap + 分段 LRU** | 按 key 的 hash 拆到 N 个带独立锁的 LRU 链表 | 降低锁竞争，接近 `ConcurrentLinkedHashMap` / Caffeine 的思路，复杂度高 |
| **直接用 Caffeine**（生产答案） | `Caffeine.newBuilder().maximumSize(10_000).expireAfterWrite(5, MINUTES).build()` | Window-TinyLFU + StripedBuffer + ring buffer，**命中率与吞吐均显著优于朴素 LRU**（LRU 在扫描型访问下会被冲刷） |

**关键代码骨架（方案一）**：

```java
public final class LruCache<K, V> {
    private final int maxSize;
    private final ReentrantLock lock = new ReentrantLock();
    private final Map<K, V> map;

    public LruCache(int maxSize) {
        if (maxSize <= 0) throw new IllegalArgumentException("maxSize must be positive");
        this.maxSize = maxSize;
        // accessOrder = true：每次 get 都会把节点移到链表尾部（即"最近使用"端）
        this.map = new LinkedHashMap<K, V>(Math.max(16, maxSize), 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > LruCache.this.maxSize;   // put / putAll 之后回调
            }
        };
    }

    public V get(K key) {
        Objects.requireNonNull(key);
        lock.lock();
        try { return map.get(key); }        // accessOrder=true 的 get 也是结构性修改，必须加锁
        finally { lock.unlock(); }
    }

    public void put(K key, V value) {
        Objects.requireNonNull(key);
        lock.lock();
        try { map.put(key, value); }
        finally { lock.unlock(); }
    }

    public V computeIfAbsent(K key, Function<K, V> loader) {   // 防"缓存击穿"的规范写法
        lock.lock();
        try {
            V v = map.get(key);
            if (v != null) return v;
            v = loader.apply(key);          // 加载必须在锁内或用 CompletableFuture 去重
            map.put(key, v);
            return v;
        } finally { lock.unlock(); }
    }

    public int size() { lock.lock(); try { return map.size(); } finally { lock.unlock(); } }
}
```

**必须讲清的四个得分点**：

1. **`accessOrder = true`** 是 LRU 的开关：默认 `false` 时 LinkedHashMap 只是"插入顺序"（即 FIFO），开了之后每次 `get`/`put` 都会把节点 `afterNodeAccess` 移到链表尾，链表头即最久未使用节点。
2. **`removeEldestEntry` 在 `put`/`putAll` 之后回调**（源码在 `LinkedHashMap.put` → `afterNodeInsertion`），返回 true 即删除头节点；它**不在 `get` 时触发**，所以过期清理只在写入时发生。
3. **为什么不能直接用 `ConcurrentHashMap`**：它没有"顺序"语义，无法表达 LRU 的访问顺序；也不能用 `Collections.synchronizedMap`，因为 `accessOrder` 模式下 **`get` 也是写操作**，而 `synchronizedMap` 的 get 与 put 之间仍可被并发穿插（`SynchronizedMap` 只是对每个方法加 synchronized，一次 `if (get == null) put` 的复合操作不是原子的）。
4. **工程化延伸（加分）**：① 加 TTL 用惰性清理或 `ScheduledExecutorService`；② 防**穿透**用布隆过滤器或缓存空值；③ 防**击穿**用 `CompletableFuture` 去重加载（Caffeine 的 `get(key, k -> ...)` 就是干这个的）；④ 防**雪崩**给 TTL 加随机抖动；⑤ 与 DB 一致性用延迟双删或 Canal 监听 binlog；⑥ 有界是硬约束，无界缓存就是内存泄漏。
5. **不要用 `computeIfAbsent` 里再做 map 更新**：`ConcurrentHashMap.computeIfAbsent` 内递归更新同一个 map 会造成死锁或 `IllegalStateException: Recursive update`（HashMap 则为死循环风险），这是高频线上事故点。

**考察意图**

这是一道把"集合 + 并发 + 工程意识"压在一起的题。面试官要的不是一个能跑的玩具，而是：你是否知道 `accessOrder` 这个参数、是否意识到 `get` 也是写、是否会主动谈起穿透/击穿/雪崩与一致性。答完还能补一句"生产我会直接用 Caffeine，因为 LRU 在扫描型流量下会被全量冲刷，TinyLFU 更抗污染"，通常就是终面级回答。

**延伸追问**

- 为什么生产更推荐 Caffeine 而不是自己实现？（Window-TinyLFU 命中率更高；采用 ring buffer + 无锁写缓冲，读几乎不阻塞；淘汰开销 O(1) 异步化）
- LFU 和 LRU 各自的失效场景？（LRU：周期性全表扫描会把热点全部挤掉；LFU：历史热点永不下线，需要 aging/window 机制）
- Redis 的 `maxmemory-policy` 里的 `allkeys-lru` 是真 LRU 吗？（不是，是**近似 LRU**，随机采样 `maxmemory-samples`（默认 5 个）key 取其中最久未用的）

---

> 完整 192 题见在线版：https://java-interview-guide.app.workbuddy.host/
> 场景面试题另有单册：`Java场景面试题.md`（31 题，含答题方法论导读）。
> 下一章持续更新中，欢迎收藏。
