---
name: java-edit-guard
description: 在既有 Java 工程里改代码、加业务的作业协议（不是从零起项目的教程）。覆盖两类任务：修改存量代码（改方法、修 bug、重构、改 SQL），以及在既有工程里新增业务代码（新 Controller / Service / Mapper / DTO / 接口）。核心是控制编辑半径、保住既有契约、让 diff 可评审：改前读全文并 grep 调用方，改中最小 diff 且不动无关风格，改后编译/测试并给四段交付说明。Use when editing, patching, refactoring, fixing, or adding new business code to an existing Java project — Controller / Service / Manager / DAO / Mapper / DTO-VO / 线程池 / 事务 / SQL / 单元测试，或用户说「改一下 / 加个参数 / 加个接口 / 写个 Service / 修个 bug / 优化这段 / 帮我 review 这个类」。红线（被本次改动触碰到的范围内发现即修，不接受口头豁免）：禁 Executors 任何工厂方法建池（newFixedThreadPool/newCachedThreadPool/newScheduledThreadPool 等，须 new ThreadPoolExecutor 显式参数）、禁 new Thread(...).start() 自建线程、SimpleDateFormat 不得共享（static 字段或单例实例字段，一律改 java.time）、ThreadLocal 用后必须 remove、SQL 禁 ${} 拼接、catch 后不得吞异常、事务异常须回滚、资源须关闭、禁 finally 中 return、数据订正先 select、禁物理删除。另：禁整文件格式化、禁重排 import、禁重命名本次未改到的标识符——这些是 diff 噪声，视为违规。
agent_created: true
---

# Java 编辑守卫

> protocol: v1.0

> 这份 skill 不教「怎么从零写一个类」。它管的是**在已经有人写过的 Java 工程里写代码时，别把东西动坏**：改存量不破坏调用方，新增业务代码与既有约定对齐，不制造 diff 噪声，不把红线带进仓库，交付时说清楚改了什么。

## 何时启用

| 场景 | 启用 |
|---|---|
| 在现有类里加方法、加参数、加分支、改判断 | ✅ 完整协议（R2~R3） |
| **在既有工程里新增业务代码**：新 Controller / Service / Manager / Mapper / DTO / 定时任务 / 新接口 | ✅ 完整协议，R4（§3.1「契约不变」→「契约对齐」；流程见 §11 B1） |
| 修 bug、改逻辑、改 SQL / Mapper、改 DTO | ✅ 完整协议 |
| 重构、抽方法、换实现、升级依赖调用 | ✅ 完整协议 |
| 改表结构、改公共常量或枚举 | ✅ 完整协议（R5，先给方案） |
| Review 别人或自己的改动 | ✅ 走 §6 审查视角 |
| 空目录从零起新项目 | 🔸 按结构条款清单建（§11 B2），不做半径侦察 |
| 一次性脚本、Demo | 🔸 只保 §5 红线，侦察与报告可简化 |

**「新增」不等于「从零」。** 在既有工程里写一个新 Service，同样是受约束的动作：包放哪一层、抛什么异常、返回怎么包装、事务加在方法还是调用方、日志怎么打——这些不是自由发挥，要和周边代码对齐，否则就是给下一个人留坑。所以这类任务照走完整协议，只是把 §3.1 的「契约不变」换成「契约对齐」：不破坏既有约定、不引入第二套风格。真正的从零项目改用结构条款清单（§11 B2）。

## 三条铁律

1. **读全再改**——没读完目标文件、没找到调用方，不许落笔；新增文件时改为先读同层 2~3 个同类文件。凭片段猜测是编辑事故的第一来源。
2. **最小 diff**——只改任务要求的行，加被触碰范围内的 P0 红线。顺手优化、顺带重命名、顺带格式化，一律算违规。
3. **结论可溯源**——说「规约要求」必须给得出条文编号（如 `01-(七)4【强制】`）。手册没写的，不许包装成手册要求（见 §7）。

## 1. 编辑半径：先判断这次改动有多大

改之前先定级，半径决定侦察深度与是否要先征求方案。（新增业务代码不必定级，直接看 §11。）

| 半径 | 触碰范围 | 典型动作 | 改前必做 |
|---|---|---|---|
| **R1** | 单条语句 / 表达式 | 换 API、加判空、改常量引用 | 读该语句上下各 20 行 |
| **R2** | 单个方法体 | 加分支、改循环、改异常处理、加日志 | 读**整个方法** + 方法注释 + 入参从哪来 |
| **R3** | 方法签名 / 字段 / 构造器 | 加参数、改返回类型、加字段、改可见性 | R2 + 读**整个类** + grep 全仓调用方 |
| **R4** | 跨类 / 跨层 | 改接口、改 DTO、改 Mapper 与 SQL、加事务 | R3 + 读接口全部实现 + 走通 Controller→Service→Manager→DAO |
| **R5** | 公共契约 | 改二方库 API、公共常量、枚举值、表结构 | R4 + 影响面清单 + 兼容方案（重载 / 默认值 / 版本化），**先给方案再动手** |

门禁：

- R3 及以上，**没 grep 到调用方就不许改签名**。grep 不到不等于没人用（反射、SPI、外部系统调用），要在报告里说明搜索范围。
- R4 及以上，动数据库或接口契约前先输出方案，等确认。
- R5 默认只报告不实施，除非用户明确要求。

> 半径会膨胀：改一个 R2 的方法，发现它被 8 处调用，实际已是 R4。**发现膨胀就停下来重新定级**，别硬着头皮改。

## 2. 改前：侦察清单

按半径取用，不要机械全跑。

**必做（R1 及以上）**

- [ ] 用 `Read` 读目标文件的**完整内容**，不是区间。只看片段会漏掉同类方法、已有工具类、import 冲突。
- [ ] 新增文件时，先读同层 2~3 个同类类的完整实现，作为包结构、命名、异常、返回包装的样本。
- [ ] 判断这个类在项目里的角色（Controller / Service / Manager / DAO / DTO），别把业务逻辑写进 Controller（`06-(一)`）。

**R3 及以上**

- [ ] `Grep` 方法名、字段名，列出全部调用点，逐个判断改了还能不能编译。
- [ ] 方法若 `@Override` 了接口或父类，先读接口定义：签名不能只改实现。
- [ ] 字段若参与序列化（DTO / VO / DO），查 `@JsonProperty`、`@TableField`、MyBatis `resultMap`。

**R4 及以上**

- [ ] 读整条调用链，确认异常、事务、返回值语义在链路上怎么被消费。
- [ ] 找现有测试（`Grep` 类名 in `src/test`）；有测试先跑一遍留基线。
- [ ] 确认构建方式（`pom.xml` / `build.gradle` / `mvnw`），拿到可执行的编译与测试命令。

侦察产出（内部记录即可）：改动文件清单、受影响调用点、要遵守的既有约定、验证命令。

## 3. 改中：四条约束

### 3.1 契约不变

手册 `01-(四)4【强制】`：外部正在调用的接口或二方库接口，**不允许修改方法签名**。落到编辑动作上，下列十项默认不动；确需改动，先在报告里列出影响面。

| # | 契约项 | 破坏后的典型后果 |
|---|---|---|
| 1 | 方法名、参数顺序/类型、返回类型、可见性 | 调用方编译失败 |
| 2 | 已声明的受检异常、异常类型 | 全局异常处理器映射错、调用方 catch 不到 |
| 3 | 返回值语义（null ↔ 空集合 ↔ 抛异常） | 调用方判空逻辑失效 |
| 4 | 序列化字段名、getter/setter 名、枚举 name | 前后端、缓存、消息体解析失败 |
| 5 | 表名、列名、`@TableField`、SQL 里出现的列 | 上线即 `Unknown column` |
| 6 | Spring Bean 名、`@Qualifier`、`@Value` key、配置项 key | 启动失败或读到默认值 |
| 7 | 事务边界、传播行为、锁粒度 | 部分提交、死锁、并发超卖 |
| 8 | 同步/异步模型、线程池归属 | 上下文丢失、事务失效、线程泄漏 |
| 9 | 公共常量值、枚举值（库里存的是值，不是名字） | 历史数据错乱 |
| 10 | 对外 HTTP 路径、参数名、状态码、错误码 | 前端或第三方调用失败 |

**新增文件时，本节叫「契约对齐」。** 没有旧签名要保，但上面这十项要从周边代码里读出来并沿用：同层的类怎么命名、异常怎么抛、返回怎么包装、事务加在方法还是调用方、日志用什么门面、DTO 带不带 `serialVersionUID`。判断标准很简单——**新代码的调用方不应该需要知道「这是新加的」**，用法与既有代码一致。

### 3.2 红线预检

动手前对着 §5 的表扫一遍：这次要写的新行里有没有出现「禁止」列的 token。有就换写法，不要写完再返工。

存量代码里发现红线，按 §5 处理规则走——先完成任务，再单独修红线，修复依据写进报告。

### 3.3 风格继承：周边代码优先

编辑不是重写。文件里已有的风格优先于手册的风格类规约（P1），因为**一致的旧风格 > 不一致的新风格**。

禁止动作（除非任务明确要求）：

- 整文件格式化、改缩进宽度、tab↔空格、改行尾换行符
- 重排 `import`、删「看起来没用」的 import（可能带静态初始化副作用）
- 重命名本次没改到的标识符
- 把 `for` 改 `stream`、匿名类改 lambda、if-else 改三目
- 删改与改动无关的注释、`@author`、日志
- 调整方法在类中的顺序

新增代码跟随既有约定：项目用 SLF4J 就用 `@Slf4j`，有 `BizException` 就别抛 `RuntimeException`，项目用 MapStruct 就别手写转换循环，项目 DTO 带 `serialVersionUID` 就一起带上。

### 3.4 不扩大半径

改的过程中看到相邻问题（命名不规范、注释过期、SQL 可优化）：**记下来，写进报告的建议区，不要直接改**。唯一例外是 P0 红线——被触碰范围内的红线必须修。

## 4. 改后：验证与交付

### 4.1 验证（能跑就必须跑）

| 情况 | 动作 |
|---|---|
| 有构建工具 | 至少编译：`mvn -q -o compile` 或 `gradle compileJava`（离线优先，避免联网下载依赖） |
| 有相关测试 | 跑目标类对应测试；改了公共方法就跑同模块测试 |
| 无法执行（缺依赖、无网络、非本地环境） | **明说「未编译 / 未运行测试」并说明原因**。不许写「已验证通过」 |

### 4.2 diff 自检（交付前逐条过）

- [ ] diff 里除了任务需要的行，没有其他改动
- [ ] 没有整文件重写留下的格式差异
- [ ] 新增/修改的方法有必要注释（`01-(九)`），公共方法有 Javadoc
- [ ] 新写的代码没有违反 §5 红线
- [ ] 所有受影响的调用点已同步（含测试代码）
- [ ] 没有留下 `TODO`、`System.out.println`、调试用的 `e.printStackTrace()`
- [ ] 异常路径有日志，日志带业务上下文（订单号、用户 ID），不打印敏感信息（`04-2`）
- [ ] 有新增分支的地方，考虑过 null、空集合、边界值、并发

### 4.3 交付说明（固定四段）

```
改了什么     文件:行号 + 一句话说明（多处则列点）
依据         涉及的条文编号；没有条文支撑的决策单独说明理由
验证         执行了什么命令、结果如何；未执行则写「未验证：<原因>」
没做/风险    刻意没动的相邻问题、已发现未修的存量问题、需要用户确认的点
```

模板与示例见 `references/report-template.md`。

## 5. P0 红线：编辑时逐条预检

判定标准：违反后可能导致**线上故障、安全漏洞、资源泄露、数据损坏**。这类问题不看「是不是存量」，**被本次改动触碰到的范围内发现即修**。

分级对应：本章即 **P0**；手册其余【强制】属 **P1**（命名、格式、OOP、注释、控制语句等风格与结构类，只约束本次新写的代码）；【推荐】/【参考】属 **P2**（建议，用户说忽略即可）。

| # | 域 | 禁止 | 必须 | 出处 |
|---|---|---|---|---|
| 1 | 线程 | `Executors.newXxx(...)` **任何**工厂方法：`newFixedThreadPool` / `newCachedThreadPool` / `newSingleThreadExecutor` / `newScheduledThreadPool` / `newSingleThreadScheduledExecutor` / `newWorkStealingPool` | `new ThreadPoolExecutor(...)` 显式给出核心数、最大数、存活时间、**有界队列**、命名 ThreadFactory、拒绝策略 | `01-(七)4【强制】` |
| 2 | 线程 | `new Thread(...).start()` 自己建线程跑任务（`ThreadFactory` 内构造 Thread 是正当写法，不算）；用 `Timer` 跑多任务 | 线程资源统一走线程池；定时任务用 `ScheduledExecutorService` | `01-(七)3【强制】`、`01-(七)12【强制】` |
| 3 | 并发 | `SimpleDateFormat` 被共享：static 字段，或单例 Bean 的实例字段 | `java.time` + `DateTimeFormatter`（不可变、线程安全）；确需复用 `SimpleDateFormat` 时用 `ThreadLocal` 包装 | `01-(七)5【强制】` |
| 4 | 并发 | `ThreadLocal` 用完不清理 | `try { ... } finally { tl.remove(); }` | `01-(七)6【强制】` |
| 5 | 并发 | 共享可变状态（单例字段、缓存 Map）用非线程安全容器 | `ConcurrentHashMap` / `CopyOnWriteArrayList`；注意 `ConcurrentHashMap` 不接受 null 值 | `01-(七)1【强制】`、`01-(六)` 集合对照表 |
| 6 | 事务 | `catch` 掉异常后继续提交 | 需要回滚时重新抛出，或 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` | `02-(二)5【强制】` |
| 7 | 异常 | `catch (Exception e) { }` 空实现；只 `printStackTrace()` | 记日志 + 转译为有业务含义的异常向上抛 | `02-(二)4【强制】` |
| 8 | 资源 | 流、连接、锁未在 `finally` 关闭 | `try-with-resources`；在 `finally` 里关闭时自身也要 try-catch | `02-(二)6【强制】` |
| 9 | 异常 | `finally` 中 `return` 或 `throw` | 把返回逻辑移出 finally | `02-(二)7【强制】` |
| 10 | 安全 | SQL 用 `${}` 拼接用户输入、字符串拼 SQL | `#{}` / `PreparedStatement` 参数绑定 | `05-(四)4【强制】`、`04-3【强制】` |
| 11 | 安全 | 按 id 直接查改他人数据、无归属校验 | 校验当前用户对该资源的操作权限（水平越权） | `04-1【强制】` |
| 12 | 安全 | 手机号、身份证、银行卡原样输出或落日志 | 脱敏，如 `139****1219` | `04-2【强制】` |
| 13 | 数据 | 物理 `delete` | 逻辑删除（`is_deleted`） | `05-(一)10【强制】` |
| 14 | 数据 | 直接跑 `update` / `delete` 改数据 | 先 `select` 确认影响范围，再执行 | `05-(三)8【强制】` |
| 15 | 数据 | 更新记录不带 `update_time` | 同步更新 `update_time = now()` | `05-(四)7【强制】` |
| 16 | 类型 | 金额用 `BigDecimal.equals()` 比较、`new BigDecimal(double)`、`double` 算钱 | `compareTo()`；`BigDecimal.valueOf()` 或字符串构造；金额存储用最小单位整型 | `01-(四)10【强制】`、`01-(四)12【强制】`、`01-(四)8【强制】` |
| 17 | 类型 | 包装类用 `==` 比较 | `equals` / `Objects.equals` | `01-(四)7【强制】` |
| 18 | 日期 | `java.sql.Date` / `Time` / `Timestamp`；用 `new Date().getTime()` 取毫秒 | `java.time` 对应类型；`System.currentTimeMillis()` | `01-(五)4【强制】`、`01-(五)3【强制】` |

处理规则：

- **新写的代码**：直接写合规写法，不许先写反例再「优化」。
- **存量代码被本次改到**（含同类字段、同方法内其他语句）：顺手修掉，并把条文编号写进报告「依据」；若修复会改变公共契约或影响多个调用方（例如线程池被外部注入），先给方案再改。
- **与任务无关的存量红线**：报告里列一条，不擅自扩大改动范围。

完整反例/正例代码对照见 `references/redlines.md`。

## 6. 两种视角

| 视角 | 什么时候 | 输出形态 |
|---|---|---|
| **编写视角** | 用户让你改代码 | 直接给改好的代码 + 四段说明，不逐条讲规约 |
| **审查视角** | 用户让你 review，或问「这段有没有问题」 | 按红线域**汇总**，按严重度排序；每条给位置、条文、修法。存量风格问题一句话带过，不逐条刷屏 |

审查时不要改文件，除非用户要求「直接改」。

## 7. 手册没这么说的三件事（别误传）

社区流传的一些「阿里规约」，黄山版原文并不是那个意思。引用前先确认，否则会拿假规则去改别人的代码。

| 常见说法 | 黄山版原文实际是 | 怎么处理 |
|---|---|---|
| 「集合方法禁止 `return null`」 | `02-(二)10【推荐】`：**方法的返回值可以为 null**，不强制返回空集合或空对象，但必须注释说明什么情况返回 null。`01-(十)2【强制】` 只约束**前后端列表接口**返回空数组。 | 新代码返回空集合更好，但这是**选择**不是红线。改存量的返回语义属于契约变更（见 §3.1 第 3 项），要单独说明。 |
| 「双重检查锁必须 `volatile`」 | `01-(七)16` 是**【推荐】**，不是【强制】。 | 可以建议，不能当红线强制改。 |
| 「`update` / `delete` 必须带 `where`」 | 黄山版**没有这条原文**。相近的是 `05-(三)8【强制】`（数据订正先 select）和 `05-(一)10【强制】`（禁物理删除）。 | 这是合理的工程常识，按「团队规则」提，不要说成手册要求。 |

同理，下列内容手册未覆盖，属团队或个人约定：`Optional` 用法、Lombok、MapStruct、日志脱敏实现方式、接口幂等、缓存 key 规范。写进 `project/<项目名>.md` 或 `memory.md`，并标注来源。

## 8. 路由：按动作查文件

条文正文在 `references/manual/`，**只在需要确认原文时读对应小节，不要整章读**。

| 你要做的事 | 读这里 |
|---|---|
| 只知道要查什么（JWT、索引、分页、线程池…），不知道在哪一章 | `references/manual/index.md`（触发词 → 章节；321 条总览） |
| 大特性开工前（新增量大、要建表、跨模块） | `references/new-code.md` §1（B0 前置契约）+ `references/manual/07-design-standards.md` |
| 新增业务代码（新 Service / Controller / Mapper / DTO） | `references/new-code.md` §2 + `references/manual/06-project-structure.md` |
| 空目录从零起项目 | `references/new-code.md` §3 + `references/manual/06-project-structure.md`、`references/manual/05-mysql-database.md` |
| 改方法签名 / 加参数 / 加字段 | `references/playbooks.md` §1、§2 |
| 改 SQL / Mapper / resultMap | `references/playbooks.md` §3 + `references/manual/05-mysql-database.md` |
| 加接口 / 改 DTO-VO / 改分层 | `references/playbooks.md` §4 + `references/manual/06-project-structure.md` |
| 改并发 / 线程池 / 锁 | `references/playbooks.md` §5 + `references/manual/01-coding-standards.md`（七） |
| 加事务 / 改事务边界 | `references/playbooks.md` §6 + `references/manual/02-exception-logging.md`（二） |
| 加日志 / 改异常处理 | `references/playbooks.md` §7 + `references/manual/02-exception-logging.md` |
| 加缓存 / 改配置 / 改枚举常量 | `references/playbooks.md` §8 |
| 补单元测试 | `references/playbooks.md` §9 + `references/manual/03-unit-testing.md` |
| 红线具体反例与改法 | `references/redlines.md` |
| 交付前自检、diff 噪声排查 | `references/diff-review.md` |
| 写交付说明 | `references/report-template.md` |
| 命名、常量、格式、OOP、集合、控制语句、注释 | `references/manual/01-coding-standards.md` 对应小节 |
| 安全（越权 / 脱敏 / 上传 / 重放） | `references/manual/04-security-standards.md` |
| 设计（分层 / 依赖 / 降级） | `references/manual/07-design-standards.md` |
| 名词、错误码全表 | `references/manual/08-appendix.md` |

## 9. 项目画像与个人偏好

编辑前读两层配置，**项目层覆盖个人层，个人层覆盖手册基线**：

1. `memory.md`——个人跨项目习惯（金额类型、Lombok、异常风格、日志风格）
2. `project/<项目名>.md`——当前项目约定：构建与测试命令、分层与包结构、命名习惯、存量卫生度、红线豁免记录

项目名取工作目录顶层目录名（如 `D:\code\物业系统\contract-service` → `物业系统`）。文件不存在就跳过，不要因此停下任务；同类偏好重复出现两次以上，问一次是否落盘。

模板见 `project/_template.md`。

## 10. 豁免

| 层级 | 豁免方式 |
|---|---|
| 风格类（手册 P1 / P2） | 用户一句话即可，本次会话生效 |
| P0 红线 | 模糊指令（「别管了」「简单改改」）**不生效**。需同时满足：① 用户点名条文编号并说明理由；② 代码处加 `// JAVA-EDIT-P0-EXEMPT(01-(七)4): 压测需要，已评估内存风险`；③ 报告末尾列出「已豁免红线」 |

用户的「其他保持不变」不构成红线豁免，只构成「不要扩大风格改动」的授权。

## 11. 路径 B：新增与从零

§1~§4 是「改」的协议。新增业务代码走这条，共用 §5 红线、§3.3 风格继承、§4.1 验证、§4.3 交付说明。

先分清是哪种新增：**有同层代码可读、有调用链可接 → B1**；**空目录什么都没有 → B2**。

**新增量大时先做 B0**：预计 10 个文件以上、要建表、加对外接口、或跨 3 个以上模块——先产出一页纸（范围与边界 / 契约先行 / 分批 / 验收口径），契约定稿再动手。大特性返工的主因不是写错某一行，而是边写边改契约。

| | B1 既有工程里新增 | B2 空目录从零起项目 |
|---|---|---|
| 默认半径 | R4（新类要接进现有链路） | 不适用 |
| 核心动作 | **对齐**：读样本、接线路 | **定型**：先定分层与契约 |
| 失败方式 | 风格不一致、漏接线、分层错位 | 结构没定就写业务，整体返工 |

**B1 三步**（细节见 `references/new-code.md`）：

1. **对齐侦察**：读同层 2~3 个类的完整实现，走通一条同类链路，记下六项约定——异常、返回包装、事务位置、日志门面、持久化方式、校验方式；再找出接线点（路由前缀、`@MapperScan` 包范围、配置 key、权限注解）。
2. **按序建设，每步可编译**：DTO → DO/Mapper → Service 接口 → Impl → Controller → 单测 → 接线自检。先定接口再写实现，避免实现里顺手改契约。
3. **验收**：把新代码丢进同类文件堆里看不出哪个是新加的；新 Mapper、Bean、配置项、权限点都已接上。

**B2 五件事按顺序定**：分层与包结构（`06-(一)`）→ 依赖与版本（`06-(二)`）→ 数据模型与建表（`05-(一)`）→ 错误码与异常体系（`02-(一)`）→ 测试骨架（`03`）。

B2 的验收不是「写完了」，而是：编译通过、能启动且无 Bean 冲突、一个接口返回 200、一条端到端用例跑通、日志格式与 traceId 符合预期。

新增任务的报告，「没做/风险」重点写**未接的线**：未注册的 Bean、未加的配置、未接的权限、未覆盖的分支。新增代码最常见的故障不是逻辑错，而是没接上。

## 12. 反模式：这些行为直接判不合格

1. 只 `Read` 了 50 行就 `Edit`，没看全文
2. 改签名前没 `grep` 调用方
3. 用 `Write` 整文件覆盖，制造满屏 diff
4. `old_string` 取得过短，匹配到多处或匹配错位置
5. 为了「防止报错」加 `try-catch` 把异常吞掉
6. 用 `System.out.println` 代替日志
7. 加了 `!= null` 判断却改变了原有语义（原本 null 走另一分支）
8. 顺手把相邻代码重命名 / 格式化 / 换写法
9. 报告只写「已完成」，不写验证结果和影响面
10. 把「手册没写」的规则说成「阿里规约要求」

## 13. Google Java Style Guide 补充约束（冲突时阿里优先）

基线仍是《Java 开发手册（黄山版）》v1.7.1。Google Java Style Guide 只补充手册未覆盖、且不与现有约束冲突的项。以下冲突项按用户决策**一律以阿里为准，不采纳 Google**：4 空格缩进（非 2）、单行 120 字符（非 100）、续行相对首行 +4 且第三行起不再追加缩进、switch case +4；JUnit 测试方法不用下划线分段；`static final` 一律全大写；`@Override` 必须加（父类 `@Deprecated` 可省略的例外不采纳）；空 `catch` 仍属 P0；Javadoc 只用 `/** */`、类必须有 `@author`/`@date`，不引入 `///`、不强制 visible 成员覆盖；TODO 格式仍按 `01-(九)12`。

以下新增项**只约束本次新写代码**；存量文件按 §3.3 风格继承，不顺手改格式。

### 13.1 源文件与 import

- 一个源文件只有一个顶层类；顺序为 license/copyright（若有）→ package → imports → 类声明，每段之间一个空行。
- `package` 声明与 `import` 不换行，不受行宽限制。
- 不用通配符 import（`import xxx.*`），static 与非 static 都不用；import 不换行。
- import 分两组：static 一组、非 static 一组，组间一个空行；组内按 imported name 的 ASCII 序排列（注意 `.` 排在 `;` 前）；不静态导入静态嵌套类。
- 存量文件已有的通配符 import 或排序不因本次改动顺手调整（§3.3）。

### 13.2 格式补充

- 每条语句独占一行；变量声明一次只声明一个变量（`int a, b;` 禁止，`for` 头部例外）。
- 局部变量就近声明，声明即初始化或随后立即初始化，不习惯性堆在块首。
- 数组声明写 `String[] args`，不写 `String args[]`。
- 特殊转义序列优先于八进制/Unicode 等价写法（`\b \t \n \f \r \" \' \\`）；非 ASCII 字符可直接写实际字符或等价 Unicode 转义，但字符串和注释之外避免 Unicode 转义。
- 运算符优先级不显然时加括号，不假设读者记得优先级表。
- 文本块的 `"""` 开始符与结束符各占一行、同缩进；块内每行至少与分隔符同缩进；文本内容不受行宽限制。
- switch 表达式必须用箭头（`->`）新语法；switch 必须穷尽，未覆盖分支补 `default`。旧式 switch 的 fall-through 仍须注释（与 `01-(八)1` 一致）。
- 修饰符按 JLS 顺序：`public protected private abstract default static final sealed non-sealed transient volatile synchronized native strictfp`。
- 类/包/模块/方法/构造器上的多个注解每个单独一行；单个无参注解可与签名首行同行（如 `@Override`）；字段注解可同行；类型注解紧贴被注解类型。
- `if/else`、`try/catch/finally` 多块语句中不允许空 catch（与 §5 红线一致）。

### 13.3 命名补充

- 缩写按普通单词处理：先全小写，再按驼峰规则首字母大写。写 `XmlHttpRequest`、`newCustomerId`、`supportsIpv6OnIos`，不写 `XMLHTTPRequest`、`newCustomerID`、`supportsIPv6OnIOS`。这与 `01-(一)12`「杜绝不规范缩写」一致。
- 方法名用动词或动词短语，仍为 lowerCamelCase（与 `01-(一)5` 一致）。
- 泛型类型变量用单个大写字母可跟一位数字（`T`、`E`、`X`、`T2`），或 UpperCamelCase + `T`（`RequestT`、`FooBarT`）。
- 允许未命名变量 `_`（如 `_ -> true`），不改变参数仍需 lowerCamelCase 的要求。

### 13.4 编程实践补充

- 静态成员必须 `Class.staticMember` 形式访问（与 `01-(四)1` 一致）。
- 不覆写 `Object.finalize()`（Finalization 已计划移除）。
- 空 catch 按 §5 红线处理：记日志并转译；Google「注释说明后可忽略」不采纳。
- `@Override` 必须加（与 `01-(四)2` 一致）；Google 对 `@Deprecated` 父方法的例外不采纳。

### 13.5 明确不采纳的 Google 项（保留阿里口径）

- 2 空格缩进、100 字符行宽、续行缩进可变化、switch +2。
- JUnit 测试方法下划线分段；POJO 布尔字段加 `is` 前缀；包名每节多词或用数字；mutable `static final` 小写。
- `///` Markdown Javadoc、visible 成员强制 Javadoc、Javadoc 摘要短句与块标签顺序、取消 `@author/@date`。
- Google 风格 TODO（`TODO: url - reason`）替代阿里 `TODO(标记人, 标记时间, [预计处理时间])`。

---

**验收标准**：把这份 diff 交给不了解上下文的同事，他能看懂改了什么、为什么这么改、有没有风险。做不到就还没改完。
