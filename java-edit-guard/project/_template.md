# 项目编辑画像：{{项目名称}}

> 本文件回答一个问题：**在这个仓库里改代码，有哪些事是「必须照做」的？**
> 读取优先级：`memory.md`（个人）→ 本文件（项目）→ 手册基线。
> 初始化方式：首次在该项目编辑时，扫描 `pom.xml` / `build.gradle` 与目录结构，向用户确认后填写；不要凭空猜。

---

## 1. 项目元信息

| 字段 | 值 | 来源 |
|---|---|---|
| 项目名称 | {{项目名称}} | 目录名 |
| JDK 版本 | | `pom.xml` `<java.version>` / `build.gradle` |
| 构建工具 | | Maven / Gradle |
| Spring Boot 版本 | | 父 POM |
| 主要模块 | | 目录结构 |
| 数据库 | | 配置文件 |
| 生效日期 | | |

## 2. 验证命令（编辑后必须执行的）

> 这是本文件最重要的一节。没有它，交付说明里的「验证」只能写「未验证」。

| 用途 | 命令 |
|---|---|
| 编译 | `mvn -q -o compile` |
| 单模块编译 | `mvn -q -o -pl <module> compile` |
| 单个测试类 | `mvn -q -o test -Dtest=<ClassName>` |
| 全量测试 | `mvn -q -o test` |
| 静态检查 | （如有：checkstyle / spotbugs / p3c） |
| 本地启动 | |

**离线约定**：{{是/否}}优先使用 `-o` 离线模式；若依赖不全，允许联网一次。

## 3. 分层与包结构

| 层 | 包路径 | 职责 | 禁止 |
|---|---|---|---|
| Controller | | 参数校验、调用 Service、组装返回 | 写业务逻辑、直接用 DAO |
| Service | | 业务编排、事务边界 | 返回 DO 给上层 |
| Manager | | 可复用原子能力 | 依赖 Controller |
| DAO / Mapper | | 数据访问 | 写业务判断 |
| DTO / VO / DO | | 入参 / 出参 / 持久化对象 | 跨层混用 |

## 4. 既有技术约定（新增代码必须跟随）

| 方面 | 项目做法 | 备注 |
|---|---|---|
| 日志 | SLF4J + `@Slf4j` / 自研封装 | |
| 异常 | 统一 `BizException` + 错误码枚举 | |
| 参数校验 | `@Valid` + JSR303 / 手写 `Assert` | |
| 对象转换 | MapStruct / BeanUtils / 手写 Converter | |
| ORM | MyBatis-Plus / MyBatis / JPA | |
| 分页 | PageHelper / MyBatis-Plus `Page` | |
| 缓存 | Redis + 自研 `CacheService` | key 规范： |
| 日期 | `LocalDateTime` + `DateTimeFormatter` | |
| 金额 | `BigDecimal` / `Long`（分） | |
| 权限 | `@PreAuthorize` / 自研注解 | |
| 返回值包装 | `Result<T>` / 直接返回 | |

## 5. 命名与代码习惯

| 元素 | 约定 | 依据 |
|---|---|---|
| Controller | `XxxController` | |
| Service 接口 | `XxxService` / `IXxxService` | |
| 实现类 | `XxxServiceImpl` | |
| DTO / VO | `XxxDTO` / `XxxVO` | |
| Mapper | `XxxMapper` | |
| 表名 / 列名 | 下划线 / 前缀 | |
| 包名 | | |

## 6. 存量代码卫生度

决定审查存量代码时风格问题（P1）的输出详略。

| 配置项 | 值 | 说明 |
|---|---|---|
| 卫生度 | 整洁 / 一般 / 混乱 | 初始化时抽样判断或直接问用户 |
| P1 输出模式 | 汇总（默认）/ 静默 / 全量 | 「汇总」= 一句话带过，不逐条列 |
| P0 红线 | 始终启用 | 不可通过本文件关闭 |

## 7. 红线豁免记录

仅记录用户点名批准的豁免，代码处须同步 `// JAVA-EDIT-P0-EXEMPT(编号): 原因` 注释。

| 条文编号 | 位置 | 原因 | 批准人 / 日期 | 计划修复 |
|---|---|---|---|---|
| | | | | |

## 8. 自定义规约（手册未覆盖）

| 规约 | 级别 | 说明 | 来源 |
|---|---|---|---|
| 接口必须幂等 | 强制 | 所有写接口按 `requestId` 去重 | 团队约定 |
| | | | |

## 9. 已知存量问题（避免重复报告）

| 位置 | 问题 | 状态 |
|---|---|---|
| | | 已知 / 计划修复 / 不修 |

## 10. 更新日志

| 日期 | 更新内容 |
|---|---|
| | |
