# 编辑动作手册（Playbooks）

> 按「这次要做的动作」查，不按手册章节查。每个 playbook 给：默认半径、改前侦察、改法要点、常见坑、条文出处、收尾检查。
> 条文原文在 `../manual/`，需要确认措辞时再点进去读对应小节。

---

## §1 改方法签名 / 加参数

**默认半径**：R3（签名变了，所有调用方都要跟着变）

### 改前侦察

1. 读整个类，确认这个方法是否有重载版本。
2. `grep` 方法名，列出全部调用点（含测试）。
3. 若方法来自接口或父类：改接口还是只改实现？只改实现会导致编译失败或空实现。
4. 若方法是 `public` 且类在二方库里：**基本不允许改**（`01-(四)4【强制】`），只能新增重载。

### 改法要点

| 情况 | 做法 |
|---|---|
| 只是想加一个可选参数 | 新增重载，旧签名委托新签名，旧方法标 `@Deprecated` 并写清新方法名（`01-(四)4`） |
| 参数超过 4 个 | 考虑引入 `XxxQuery` / `XxxCommand` 对象，但**不要**在本次顺手改所有调用方 |
| 参数是布尔值 | 拆成两个语义明确的方法，或改用枚举，避免 `method(true, false)` |
| 返回值要加字段 | 不要改返回类型（破坏契约），在返回对象里加字段，或新增方法 |
| 会抛新的受检异常 | 优先转成非受检的业务异常，避免污染所有调用方 |

### 常见坑

- 只改了 `ServiceImpl`，忘了 `Service` 接口 / `@Override` 编译报错。
- 只改了 `public` 方法，漏掉 `src/test` 里的调用。
- 用了可变参数 `Object...` 图省事（`01-(四)3【强制】`：相同参数类型、相同业务含义才可用可变参数）。
- 改完签名顺手把参数名也改了，导致 MyBatis `#{param}`、Spring `@RequestParam` 绑定失败。

### 收尾检查

- [ ] 全部调用点已同步，`mvn -q -o compile` 通过
- [ ] 重载而非改签名（对外可见的 API）
- [ ] 参数名变更没有破坏 `#{...}`、`@RequestParam`、`@PathVariable` 绑定

---

## §2 加字段 / 改 DTO、VO、DO

**默认半径**：R4（DO 改动会牵动 SQL、序列化、前端）

### 改前侦察

1. 这个类是 DO（对应表）、DTO（入参）、VO（出参）还是 BO？改错层是最常见的返工原因（`06-(一)`）。
2. DO：对应表的列是否存在？没有列就要先加字段（R5）。
3. 字段是否参与序列化：`@JsonProperty`、`@TableField`、`resultMap`、`serialVersionUID`。
4. 前端是否已经依赖这个 VO 的字段名（grep 前端仓库或接口文档）。

### 改法要点

- **DO 属性类型必须与数据库字段类型匹配**（`01-(四)11【强制】`）：`datetime` → `LocalDateTime`，`decimal` → `BigDecimal`，`bigint` → `Long`。
- POJO 属性用**包装类型**，不用基本类型（`01-(四)13【强制】` 第 1 款）：`Integer` 而非 `int`，否则反序列化缺字段时得到 0 而非 null。
- **不要给 POJO 设默认值**（`01-(四)14【强制】`）：`private Integer status = 1;` 会让「没传」和「传了 1」无法区分。
- 布尔字段不加 `is` 前缀（`01-(一)9【强制】`）：字段叫 `deleted`，getter 是 `isDeleted()`。DB 列才是 `is_deleted`，用 `resultMap` 映射。
- 新增字段后补 `toString()`（`01-(四)17【强制】`）；继承父类时加 `super.toString()`。
- 序列化类新增属性不要动 `serialVersionUID`（`01-(四)15【强制】`）。

### 常见坑

- VO 里塞了 DO 的全部字段，把 `is_deleted`、`version`、内部备注一起返回给前端。
- 金额字段用 `Double`，前端拿到 `100.10000000000001`。
- 加字段忘了改 `resultMap`，查询结果该字段永远为 null。
- DTO 加了 `@NotNull` 但 Controller 没加 `@Valid`，校验不生效。

### 收尾检查

- [ ] 类型与 DB 列匹配；包装类型；无默认值
- [ ] `resultMap` / `@TableField` / `@JsonProperty` 同步
- [ ] 新增字段是否需要脱敏（`04-2`）
- [ ] 若字段对外可见，确认前端兼容

---

## §3 改 SQL / Mapper / resultMap

**默认半径**：R4

### 改前侦察

1. 这条 SQL 属于哪个业务链路？读 Service 层调用处，确认返回值怎么被消费。
2. 表有多大、有没有索引？`explain` 看一下执行计划，再改 where 条件（参考 `05-(二)` 索引规约）。
3. 是否是分页查询？深分页有没有 `limit 100000, 20` 的写法（`05-(二)7【推荐】`：用延迟关联优化）。
4. Mapper 接口方法是否被多处调用（`grep`）。

### 改法要点

- 参数一律 `#{}`，不用 `${}`（`05-(四)4【强制】`）。必须动态拼接列名/排序时，用白名单枚举校验。
- 不要 `select *`（`05-(四)1【强制】`），列变更会导致映射错位；明确列出需要的列。
- `count(*)` 不要写成 `count(列名)`（`05-(三)1【强制】`）。
- 多表查询给列加表别名（`05-(三)` 说明：操作列在多表存在时会抛异常）。
- 更新必须带 `update_time`（`05-(四)7【强制】`）；不要写「大而全」的更新接口，只更新变化字段（`05-(四)8【推荐】`）。
- 分页插件与手写 `limit` 不要混用。
- `resultMap` 的 `column` 与 `property` 一一对应，`is_deleted` → `deleted` 需要显式映射。

### 常见坑

- 加了 `where` 条件但没建索引，慢查询上线才暴露。
- `like '%#{keyword}%'` 写错成 `like '%${keyword}%'`。
- `in` 查询参数为空集合，生成 `in ()` 语法错误。
- 逻辑删除条件漏加 `is_deleted = 0`，把已删数据查出来。
- 更新语句漏 `and is_deleted = 0`，把已删除记录改活了。

### 收尾检查

- [ ] 无 `${}`、无 `select *`、无 `delete from`
- [ ] `update` 带 `update_time`，`where` 命中索引
- [ ] 空集合、null 参数有兜底
- [ ] 逻辑删除条件完整

---

## §4 加接口 / 改分层

**默认半径**：R4

### 改前侦察

1. 现有 Controller 的路径风格、返回包装（`Result<T>` / 直接返回对象）、错误码用法。
2. 同类接口的参数校验方式（`@Valid` + JSR303 还是手写 `Assert`）。
3. 权限注解（`@PreAuthorize` / 自定义注解）是否必须加。

### 改法要点

- 分层职责（`06-(一)`）：Controller 只做参数校验、调用 Service、组装返回；业务逻辑在 Service；可复用原子能力在 Manager；DAO 只做数据访问。
- **DO 不出 Service 层**：出参转 VO，入参用 DTO（`06-(一)`）。
- 参数必须校验（`04-4【强制】`）：`@Valid` + `@NotNull` / `@Size`，分页参数限制上限（`pageSize <= 100`）。
- 需要登录态的接口必须做权限校验（`04-1【强制】`），不能只校验登录。
- 错误码用既有体系，不新造（`02-(一)6【强制】`）；错误码不直接抛给用户看（`02-(一)7【强制】`）。
- 对外路径一旦发布就不能改（契约项 10）。

### 常见坑

- Controller 里写 `try-catch` 把异常转成 `Result.error("系统繁忙")`，吞掉真实错误。
- 新接口没有 `@Valid`，脏数据进库。
- 分页参数不设上限，`pageSize=1000000` 打爆内存。
- 返回 DO，把内部字段（如 `costPrice`）暴露给前端。

### 收尾检查

- [ ] DTO 校验注解 + Controller `@Valid`
- [ ] 权限校验（含数据归属）
- [ ] 返回 VO 而非 DO
- [ ] 错误码复用既有体系

---

## §5 改并发 / 线程池 / 锁

**默认半径**：R4（并发问题影响面大，且难测）

### 改前侦察

1. 这段代码运行在哪个线程模型里：Web 请求线程、线程池、定时任务、MQ 消费线程？
2. 共享了哪些可变状态（单例 Bean 字段、静态变量、缓存）？
3. 现有事务是否会被异步打断（`@Transactional` 只在当前线程生效）？
4. `ThreadLocal`（用户上下文、traceId）能否传递到子线程？

### 改法要点

- 线程池必须 `new ThreadPoolExecutor(...)` 显式参数，并给线程命名（`01-(七)4`、`01-(七)2`）。
- 队列必须有界；拒绝策略按业务选：`CallerRunsPolicy`（降级为同步）、`AbortPolicy`（快速失败）。
- `ThreadLocal` 在子线程中不继承，需 `InheritableThreadLocal` 或显式透传；用完 `remove()`（`01-(七)6`）。
- 锁粒度：能锁代码块不锁方法，能用对象锁不用类锁（`01-(七)7`）；多资源加锁顺序一致（`01-(七)8`）。
- 加锁代码必须在 `try` 之外获取锁，避免 finally 解锁失败（`01-(七)9`）。
- 并发更新同一条记录要防丢失更新：乐观锁 `version` 或分布式锁（`01-(七)11`）。
- 异步任务注意：事务未提交就异步读取会读不到数据；异常不会传播到主线程，必须自己记日志。

### 常见坑

- 把 `@Async` 加在同类内部调用的方法上，代理不生效，实际同步执行。
- `CompletableFuture` 用默认 `ForkJoinPool.commonPool()`，IO 任务把公共池打满。
- 线程池 Bean 被 `shutdown()` 后复用，抛 `RejectedExecutionException`。
- 双重检查锁忘了 `volatile`（`01-(七)16【推荐】`，单例场景几乎是必须）。

### 收尾检查

- [ ] 无 `Executors.new*`、无裸 `new Thread`
- [ ] 线程池有界队列 + 命名 + 明确拒绝策略
- [ ] `ThreadLocal` 有配对 `remove()`
- [ ] 异步路径的异常有日志

---

## §6 加事务 / 改事务边界

**默认半径**：R4

### 改前侦察

1. 当前方法是否已在事务里（看调用链上层有没有 `@Transactional`）。
2. 事务内是否有 RPC、HTTP、消息发送、文件 IO 等耗时操作。
3. 数据源是否多数据源 / 分库分表（本地事务是否够用）。
4. 异常类型：`@Transactional` 默认只回滚 `RuntimeException`。

### 改法要点

- `@Transactional(rollbackFor = Exception.class)`，显式声明回滚范围。
- 只读查询加 `@Transactional(readOnly = true)`，但别在只读事务里写库。
- 事务方法必须是 `public`，且通过代理调用（同类内部 `this.xxx()` 不走事务）。
- 事务里吞异常必须 `setRollbackOnly()`（`02-(二)5【强制】`）。
- 事务内不做远程调用；必须做时考虑「本地消息表 + 补偿」或事务外调用。
- 大事务拆分：先落库，再异步处理后续步骤。

### 常见坑

- 加在 `private` 方法或同类内部调用上，完全不生效。
- 捕获异常后 `return false`，事务照常提交。
- 事务里发了 MQ 消息，事务回滚但消息已发出（缺本地消息表）。
- 嵌套事务用默认 `REQUIRED`，内层回滚把外层一起标记 rollback-only，抛 `UnexpectedRollbackException`。

### 收尾检查

- [ ] `rollbackFor` 显式声明
- [ ] 事务内无远程调用 / 无长耗时操作
- [ ] catch 分支要么抛出、要么 `setRollbackOnly`
- [ ] 事务方法为 `public` 且经代理调用

---

## §7 加日志 / 改异常处理

**默认半径**：R2

### 改法要点

- 用门面 API（`02-(三)1【强制】`）：`@Slf4j` + `log.info`，不直接 `LoggerFactory.getLogger(Logback.class)`。
- 占位符不用字符串拼接：`log.info("orderId={}", id)`，不要 `log.info("orderId=" + id)`（后者无论如何都会拼接，且丢失结构化）。
- 异常日志把异常对象放最后一个参数：`log.error("支付失败, orderId={}", id, e)`，否则只打印 message 丢堆栈。
- 级别：`error` 只给需要人介入的；可恢复的用 `warn`；高频链路不打 `info` 循环日志。
- 不打印敏感信息（`04-2`）：手机号、身份证、密码、token、完整报文。
- 异常体系：抛有业务含义的自定义异常（`02-(二)12【推荐】`），不抛 `new RuntimeException()`，更不抛 `Exception` / `Throwable`。
- catch 要区分异常类型（`02-(二)3【强制】`），不要一个大 `catch (Exception e)` 包住整段。

### 常见坑

- `log.error(e.getMessage())` —— 丢堆栈，线上只会看到一句没有上下文的字符串。
- 在循环里打 `info`，日志量暴涨。
- 加日志时顺手把所有 `System.out.println` 全改了，diff 扩散到无关文件。
- 把异常转成 `null` 返回，调用方拿到 null 后 NPE。

### 收尾检查

- [ ] 无 `printStackTrace`、无 `System.out.println`
- [ ] 异常日志带上下文参数且异常对象在最后
- [ ] 无敏感信息
- [ ] catch 分支有处理动作（记日志、转译、补偿、抛出）

---

## §8 加缓存 / 改配置 / 改枚举常量

**默认半径**：R4（缓存）／ R5（枚举、公共常量）

### 缓存

- key 必须带业务前缀和版本：`order:detail:v1:{orderId}`，避免与别的业务撞 key。
- 必须设过期时间；无过期时间的缓存要有主动失效路径。
- 缓存穿透：空结果也缓存短 TTL，或用布隆过滤器。
- 缓存与 DB 一致性：先更新 DB 再删缓存，不要「先删缓存再更新 DB」。
- 缓存值序列化：`LocalDateTime` 需要配置 `JavaTimeModule`，否则反序列化失败。

### 配置

- 新增配置项要同时改：`application.yml`、配置中心、部署模板（K8s ConfigMap / 环境变量），漏一个就上线读到默认值。
- 配置 key 属于契约（§3.1-6），改名等于破坏契约。
- 敏感配置（密码、密钥）必须加密（`04-10【强制】`）。

### 枚举与常量

- 枚举值（`ordinal` / 自定义 `code`）一旦落库或对外输出就不能改（契约项 9）。
- 新增枚举值要检查所有 `switch` 是否有 `default`（`01-(八)1【强制】`），避免新值走不到分支。
- 常量用 `private static final`，全大写下划线分隔（`01-(一)6【强制】`），不用魔法值（`01-(二)`）。

### 收尾检查

- [ ] 缓存 key 有前缀和 TTL
- [ ] 新增配置项三处齐全
- [ ] 枚举新增值后所有 switch 有 default 且行为明确

---

## §9 补单元测试

**默认半径**：R2

### 原则（`03-unit-testing.md`）

- **AIR**：Automatic（可自动执行）、Independent（用例间不依赖）、Repeatable（可重复）。
- **BCDE**：Border（边界）、Correct（正确）、Design（设计）、Error（错误）四类用例都要有。
- 不写 `System.out` 断言，不用 `Thread.sleep` 等待。
- 单测不依赖外部环境：DB、MQ、第三方接口都要 mock 或内嵌。

### 写法要点

- 一个用例只验证一个行为，方法名说清「场景 + 预期」：`should_throw_when_stock_not_enough`。
- 用 `@Mock` / `@InjectMocks`，只 mock 边界（DAO、RPC），不 mock 被测类内部逻辑。
- 断言用 AssertJ 或 `assertEquals(expected, actual)`，参数顺序别反。
- 异常用例用 `assertThrows`，不要 try-catch 后 `fail()`。
- 涉及时间、随机数、UUID 的代码，注入 `Clock` / `IdGenerator` 而不是直接调静态方法。

### 常见坑

- 为了让测试过，把被测方法的逻辑改简单（改测试不改代码是原则，但别把 bug 当预期）。
- 只测 happy path，边界和异常路径没覆盖。
- 测试里用了真实数据库，CI 上跑不过。

### 收尾检查

- [ ] 覆盖新增分支的边界与异常路径
- [ ] 无外部依赖，可离线跑
- [ ] 测试命名表达场景与预期
