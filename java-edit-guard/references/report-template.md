# 交付说明模板

> 每次改完 Java 代码，用这四段说明收尾。用户不需要追问「改了哪儿」「能不能跑」，就能判断这份改动能不能合。

---

## 模板

````markdown
**改了什么**
- `OrderService.java:88` 新增 `cancel(Long orderId, String reason)`，校验订单状态后置为已取消
- `OrderService.java:120` `sendMessage` 增加 `tag` 参数，日志中带上标签

**依据**
- `01-(七)4【强制】`：原线程池用 `Executors.newFixedThreadPool`，改为 `ThreadPoolExecutor` 显式参数（核心 8 / 最大 16 / 队列 200 / CallerRuns）
- `02-(二)5【强制】`：取消流程中扣减库存失败需要回滚，改为重新抛出 + `setRollbackOnly`
- 无对应条文的决策：取消订单采用「先更新状态再发消息」，理由是避免消息先到导致状态不一致；这一顺序沿用项目现有 `OrderCreateService` 的写法

**验证**
- `mvn -q -o compile` 通过
- `mvn -q -o test -Dtest=OrderServiceTest` 通过（6 个用例，含新增 2 个）
- 未做：未跑集成测试（依赖测试库，本地不可用）

**没做/风险**
- `OrderController` 中另有 3 处 `printStackTrace()`，属存量问题且不在本次改动范围内，未修改
- `OrderMapper.selectByUserId` 缺 `is_deleted = 0` 条件，可能导致已删订单被查出，建议单独修复
- 需要确认：前端是否已在使用 `OrderVO.remark` 字段的旧含义
````

---

## 分场景示例

### A. 小改（R1~R2）：一句话也要给验证

````markdown
**改了什么**
- `UserService.java:56` 手机号输出改为脱敏

**依据**
- `04-2【强制】`：用户敏感数据必须脱敏展示

**验证**
- `mvn -q -o compile` 通过；`UserServiceTest` 通过

**没做/风险**
- 同一响应中的 `idCard` 也已透出，但该字段来自另一个 VO，改动会影响 3 个接口，需你确认是否一起改
````

### B. 大改（R4）：先方案后实施，报告要给影响面

````markdown
**改了什么**
- 新增 `StockManager`（3 个方法）：扣减、回滚、查询可用库存
- `OrderService.createOrder` 改为调用 `StockManager`，移除原有的两处直接 DAO 操作
- `InventoryMapper` 增加 `deductWithVersion`（乐观锁），`update_time` 同步更新

**依据**
- `06-(一)`：库存原子能力从 Service 下沉到 Manager，供订单、售后两条链路复用
- `01-(七)11【强制】`：并发扣减需防更新丢失，采用 version 乐观锁
- `05-(四)7【强制】`：更新记录同步 `update_time`
- `01-(四)4【强制】`：`OrderService.createOrder` 签名未变，仅替换内部实现，避免影响 7 处调用方

**验证**
- `mvn -q -o compile` 通过
- `mvn -q -o test -Dtest=OrderServiceTest,StockManagerTest` 通过（14 用例）
- 并发场景未压测：乐观锁失败重试逻辑仅单测覆盖，建议在预发做一次 50 并发验证

**没做/风险**
- `InventoryMapper.deduct` 旧方法保留未删，仍有 2 处在售后链路调用，待迁移后再删
- 回滚路径依赖 `@Transactional` 生效前提（同类调用不走代理），`afterSale` 链路的调用方式需要复核
````

---

## 不合格的报告长什么样

| 写法 | 问题 |
|---|---|
| 「已按要求完成修改」 | 没有位置、没有依据、没有验证 |
| 「代码已优化，性能提升」 | 无法验证的结论 |
| 「已验证通过」但没跑过 | 虚假陈述，比不写更糟 |
| 只贴完整文件代码，不说明改了哪几行 | 用户要自己 diff |
| 罗列 20 条规约条文 | 与本次改动无关的条文是噪声，只写真正影响决策的那几条 |

---

## 写「依据」的规则

1. **只写影响本次决策的条文**，一般 1~3 条。全部罗列等于没写。
2. 引用格式统一：`章节-(小节)序号【级别】`，如 `01-(七)4【强制】`、`04-2【强制】`。
3. 手册没覆盖的决策，单独一行写「无对应条文的决策：...，理由是...」，不要硬套条文编号。
4. 修复了存量红线，必须写清「原来是什么、为什么必须改、改成什么」。
