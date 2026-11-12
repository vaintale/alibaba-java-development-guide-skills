# 项目编辑画像（示例）：order-center

> **这是一份虚构示例**，用来展示 `_template.md` 填好之后长什么样。项目名、包名、命令、表名都是编的，可以直接改成你自己的。
>
> 真实项目画像放在同目录的 `<项目名>.md`，属于本地运行数据，**不会提交到公开仓库**（见 `README.md`）。

## 1. 项目元信息

| 字段 | 值 | 来源 |
|---|---|---|
| 项目名称 | order-center | 目录名 |
| JDK 版本 | 17 | `pom.xml` `<java.version>` |
| 构建工具 | Maven，3 模块聚合（api / service / dal） | 根 pom `<modules>` |
| Spring Boot 版本 | 3.2.5 | 父 POM |
| 数据库 | MySQL 8.0 | `application.yml` |
| 缓存 | Redis 6 | 同上 |

## 2. 验证命令

| 用途 | 命令 |
|---|---|
| 全量编译 | `mvn -q -o compile` |
| 单模块编译 | `mvn -q -o -pl order-service -am compile` |
| 单个测试类 | `mvn -q -o -pl order-service test -Dtest=OrderServiceImplTest` |
| 全量测试 | `mvn -q -o test` |
| 本地启动 | `mvn -o -pl order-api spring-boot:run -Dspring-boot.run.profiles=local` |

离线约定：优先 `-o`；首次拉依赖允许联网一次。

## 3. 分层与包结构

| 层 | 包路径 | 职责 | 禁止 |
|---|---|---|---|
| Controller | `com.example.order.api.controller` | 参数校验、调用 Service、组装 VO | 写业务、直接用 Mapper |
| Service | `com.example.order.service` | 业务编排、事务边界 | 返回 DO |
| Manager | `com.example.order.service.manager` | 可复用原子能力（库存、计价） | 依赖 Controller |
| Mapper | `com.example.order.dal.mapper` | 数据访问 | 写业务判断 |
| DTO / VO / DO | `...api.dto` / `...api.vo` / `...dal.entity` | 入参 / 出参 / 持久化 | 跨层混用 |

## 4. 既有技术约定

| 方面 | 项目做法 |
|---|---|
| 日志 | SLF4J + `@Slf4j`，关键链路打 `orderId` |
| 异常 | `BizException` + `ErrorCode` 枚举，全局 `@RestControllerAdvice` |
| 参数校验 | `@Valid` + JSR303，分页上限 100 |
| 对象转换 | MapStruct，Converter 放 `...service.convert` |
| ORM | MyBatis-Plus，`resultMap` 手写 |
| 分页 | MyBatis-Plus `Page` |
| 返回包装 | `Result<T>`（code / message / data） |
| 日期 | `LocalDateTime` + `DateTimeFormatter` 常量 |
| 金额 | `BigDecimal`，`compareTo` 比较 |

## 5. 命名约定

| 元素 | 约定 |
|---|---|
| Controller | `XxxController` |
| Service 接口 / 实现 | `XxxService` / `XxxServiceImpl` |
| DTO / VO / DO | `XxxDTO` / `XxxVO` / `XxxDO` |
| Mapper | `XxxMapper`（XML 同名） |
| 表名 / 列名 | `t_` 前缀 / 下划线 |
| 包名 | 单数，全小写 |

## 6. 存量代码卫生度

| 配置项 | 值 | 说明 |
|---|---|---|
| 卫生度 | 一般 | 核心链路整洁，边缘模块命名随意 |
| P1 输出模式 | 汇总（默认） | 存量风格问题一句话带过 |
| P0 红线 | 始终启用 | 不可关闭 |

## 7. 红线豁免记录

| 条文编号 | 位置 | 原因 | 批准人 / 日期 | 计划修复 |
|---|---|---|---|---|
| — | — | 暂无 | — | — |

## 8. 自定义规约（手册未覆盖）

| 规约 | 级别 | 说明 |
|---|---|---|
| 写接口必须幂等 | 强制 | 按 `requestId` 去重，Redis SETNX 24h |
| Mapper XML 必须手写 resultMap | 强制 | 禁止依赖自动映射 |
