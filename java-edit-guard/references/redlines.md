# P0 红线：反例 → 正例对照

> 与 `SKILL.md` §5 的红线表一一对应。编辑时命中「反例」列的写法，改用「正例」。
> 每条都标注黄山版原文编号；编号可在 `manual/` 里查到原条文。

**判定口径**：只有会导致线上故障、安全漏洞、资源泄露、数据损坏的问题才进这份清单。风格类问题不在这里（见 `manual/01` 各小节）。

---

## 1. 线程池用 Executors 工厂创建

**反例**

```java
private final ExecutorService pool = Executors.newFixedThreadPool(10);
private final ExecutorService cache = Executors.newCachedThreadPool();
```

**正例**

```java
private static final int CORE_POOL_SIZE = 8;
private static final int MAX_POOL_SIZE = 16;
private static final int QUEUE_CAPACITY = 200;
private static final long KEEP_ALIVE_SECONDS = 60L;

private final ExecutorService pool = new ThreadPoolExecutor(
        CORE_POOL_SIZE,
        MAX_POOL_SIZE,
        KEEP_ALIVE_SECONDS,
        TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(QUEUE_CAPACITY),
        new ThreadFactoryBuilder().setNameFormat("order-export-%d").build(),
        new ThreadPoolExecutor.CallerRunsPolicy());
```

**为什么**：`newFixedThreadPool` 和 `newSingleThreadExecutor` 用无界 `LinkedBlockingQueue`，任务堆积会 OOM；`newCachedThreadPool` 最大线程数是 `Integer.MAX_VALUE`，高并发下会创建海量线程直至 OOM。工厂方法把队列容量、拒绝策略、线程命名都藏起来了，出问题时无从排查。

**出处**：`01-(七)4【强制】`；线程命名见 `01-(七)2【强制】`

**识别**：grep `Executors.new`

---

## 2. 裸建线程 / 用 Timer

**反例**

```java
new Thread(() -> syncOrders()).start();

Timer timer = new Timer();
timer.schedule(new TimerTask() { ... }, 1000L, 60_000L);
```

**正例**

```java
orderPool.execute(() -> syncOrders());

ScheduledExecutorService scheduler = new ScheduledThreadPoolExecutor(
        2, new ThreadFactoryBuilder().setNameFormat("order-sync-%d").build());
scheduler.scheduleWithFixedDelay(this::syncOrders, 0L, 60L, TimeUnit.SECONDS);
```

**为什么**：裸线程无法复用、无上限、异常直接终止且无人知晓；`Timer` 内任一 `TimerTask` 抛异常会导致整个 Timer 线程死亡，其余任务静默停止。

**出处**：`01-(七)3【强制】`、`01-(七)12【强制】`

**识别**：grep `new Thread(`, `new Timer(`

---

## 3. SimpleDateFormat 作静态字段

**反例**

```java
private static final SimpleDateFormat SDF = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

public String format(Date date) {
    return SDF.format(date);
}
```

**正例**

```java
private static final DateTimeFormatter FORMATTER =
        DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

public String format(LocalDateTime time) {
    return time.format(FORMATTER);
}
```

**为什么**：`SimpleDateFormat` 内部用 `Calendar` 存中间状态，多线程共用会串数据，表现为偶发的时间错乱或 `NumberFormatException`，且极难复现。`DateTimeFormatter` 不可变、线程安全。

**出处**：`01-(七)5【强制】`；日期类型见 `01-(五)4【强制】`

**识别**：grep `SimpleDateFormat`

---

## 4. ThreadLocal 用完不清理

**反例**

```java
private static final ThreadLocal<UserContext> HOLDER = new ThreadLocal<>();

public void handle() {
    HOLDER.set(currentUser());
    doBusiness();
    // 没有 remove
}
```

**正例**

```java
private static final ThreadLocal<UserContext> HOLDER = new ThreadLocal<>();

public void handle() {
    HOLDER.set(currentUser());
    try {
        doBusiness();
    } finally {
        HOLDER.remove();   // 线程池复用线程，不清理会串用户、且引用无法回收
    }
}
```

**为什么**：线程池里线程会被复用，残留值会导致下一个请求读到上一个请求的用户身份（越权风险），同时造成 `ClassLoader` 层面的内存泄露。

**出处**：`01-(七)6【强制】`

**识别**：grep `ThreadLocal` 后检查是否有配对的 `remove()`

---

## 5. 共享状态用非线程安全容器

**反例**

```java
private final Map<String, Order> cache = new HashMap<>();     // 单例 Bean 的字段
private final List<String> errors = new ArrayList<>();
```

**正例**

```java
private final Map<String, Order> cache = new ConcurrentHashMap<>();
private final List<String> errors = new CopyOnWriteArrayList<>();
```

**为什么**：`HashMap` 并发扩容会丢数据甚至死循环（JDK8 前）；`ArrayList` 并发 `add` 会丢元素或抛 `ArrayIndexOutOfBoundsException`。

**坑**：`ConcurrentHashMap` 不接受 null 键和 null 值，`map.put(k, null)` 直接 NPE（`01-(六)` 集合对照表）。需要存 null 时改存 `Optional` 或哨兵对象。

**出处**：`01-(七)1【强制】`、`01-(六)` 集合对照表

**识别**：检查单例 Bean 的成员变量里出现 `HashMap` / `ArrayList` / `HashSet`

---

## 6. 事务异常被 catch 后继续提交

**反例**

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder(OrderDTO dto) {
    orderMapper.insert(dto);
    try {
        stockService.deduct(dto.getSkuId());
    } catch (Exception e) {
        log.error("扣库存失败", e);   // 吞掉，事务照常提交
    }
}
```

**正例**

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder(OrderDTO dto) {
    orderMapper.insert(dto);
    try {
        stockService.deduct(dto.getSkuId());
    } catch (Exception e) {
        log.error("扣库存失败, orderId={}", dto.getOrderId(), e);
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
        throw new BizException(ErrorCode.STOCK_DEDUCT_FAILED, e);
    }
}
```

**为什么**：异常被吞后 Spring 感知不到失败，事务提交，产生「有订单无库存扣减」的脏数据。注意：`@Transactional` 默认只对 `RuntimeException` 回滚，受检异常需显式 `rollbackFor`。

**出处**：`02-(二)5【强制】`

**识别**：在 `@Transactional` 方法里找 `try-catch`，看 catch 块是否只有日志

---

## 7. catch 后吞异常 / 只打印堆栈

**反例**

```java
try {
    paymentClient.pay(req);
} catch (Exception e) {
    e.printStackTrace();
}

try {
    paymentClient.pay(req);
} catch (Exception e) {
}
```

**正例**

```java
try {
    paymentClient.pay(req);
} catch (PayTimeoutException e) {
    log.warn("支付超时, orderId={}, 进入对账补偿", req.getOrderId(), e);
    compensationService.schedule(req.getOrderId());
} catch (Exception e) {
    log.error("支付调用失败, orderId={}", req.getOrderId(), e);
    throw new BizException(ErrorCode.PAY_FAILED, e);
}
```

**为什么**：吞异常等于把故障隐藏到下一次更难排查的地方。`printStackTrace()` 输出到 stdout，容器环境下多数采集不到，且丢失 traceId。

**出处**：`02-(二)4【强制】`、`02-(二)3【强制】`（区分异常类型）

**识别**：grep `catch`，检查空块与 `printStackTrace`

---

## 8. 资源未关闭

**反例**

```java
FileInputStream in = new FileInputStream(file);
byte[] bytes = in.readAllBytes();
in.close();
```

**正例**

```java
try (InputStream in = new FileInputStream(file)) {
    return in.readAllBytes();
}
```

**为什么**：`readAllBytes()` 抛异常时 `close()` 不执行，句柄泄露，最终 `Too many open files`。`finally` 里关闭时自身也可能抛异常，需要再包 try-catch。

**出处**：`02-(二)6【强制】`

**识别**：grep `new FileInputStream`, `new BufferedReader`, `getConnection()` 等，看是否在 try-with-resources 里

---

## 9. finally 中 return

**反例**

```java
public int parse(String s) {
    try {
        return Integer.parseInt(s);
    } catch (NumberFormatException e) {
        return -1;
    } finally {
        return 0;      // 永远返回 0，吞掉上面的结果和异常
    }
}
```

**正例**

```java
public int parse(String s) {
    try {
        return Integer.parseInt(s);
    } catch (NumberFormatException e) {
        log.warn("非法数字: {}", s);
        return -1;
    }
}
```

**为什么**：`finally` 里的 `return` 会丢弃 try 块的返回值，甚至吞掉异常，是最隐蔽的逻辑错误之一。

**出处**：`02-(二)7【强制】`

**识别**：grep `finally` 块内是否出现 `return` / `throw`

---

## 10. SQL 拼接与 `${}`

**反例**

```java
String sql = "select * from t_order where user_id = " + userId;

String name = "张";  // 来自用户输入
// Mapper XML：order by ${column}
```

**正例**

```java
// JdbcTemplate
jdbcTemplate.query("select * from t_order where user_id = ?", rowMapper, userId);

// MyBatis
@Select("select * from t_order where user_id = #{userId}")
List<OrderDO> selectByUserId(@Param("userId") Long userId);
```

**为什么**：拼接让输入变成 SQL 语法的一部分，`1 or 1=1` 即可拖库；`${}` 是文本替换，同样可注入。`#{}` 走 `PreparedStatement` 占位符，参数只当值。

**注意**：`order by ${}`, `like '%${}%'` 这类「必须动态」的场景，必须用白名单校验列名/排序方向，不能直接透传。

**出处**：`05-(四)4【强制】`、`04-3【强制】`

**识别**：grep `\$\{`, `+ " where`, `"select .* +`

---

## 11. 越权：按 id 直接操作

**反例**

```java
@GetMapping("/orders/{id}")
public OrderVO detail(@PathVariable Long id) {
    return orderService.getById(id);      // 任何人都能查任意订单
}
```

**正例**

```java
@GetMapping("/orders/{id}")
public OrderVO detail(@PathVariable Long id) {
    Long userId = UserContext.getUserId();
    OrderDO order = orderService.getByIdAndUserId(id, userId);
    if (order == null) {
        throw new BizException(ErrorCode.RESOURCE_NOT_FOUND);
    }
    return OrderConverter.toVO(order);
}
```

**为什么**：只校验「登录了」不校验「是不是你的」，就是水平越权。凡是带用户维度数据的查询、修改、删除都要带归属条件。

**出处**：`04-1【强制】`

**识别**：检查 `getById` / `updateById` / `deleteById` 调用是否带用户或租户条件

---

## 12. 敏感数据未脱敏

**反例**

```java
log.info("用户注册成功, phone={}, idCard={}", phone, idCard);
vo.setPhone(user.getPhone());
```

**正例**

```java
log.info("用户注册成功, phone={}, idCard={}", DesensitizeUtil.phone(phone), DesensitizeUtil.idCard(idCard));
vo.setPhone(DesensitizeUtil.phone(user.getPhone()));   // 139****1219
```

**为什么**：日志和接口响应是泄露面最大的两个出口，手机号、身份证、银行卡、地址都需要脱敏；日志还会被多方采集。

**出处**：`04-2【强制】`

**识别**：grep `log.info` 与 `setPhone` / `setIdCard` / `setBankCard`

---

## 13. 物理删除

**反例**

```java
@Delete("delete from t_order where id = #{id}")
int deleteById(@Param("id") Long id);
```

**正例**

```java
@Update("update t_order set is_deleted = 1, update_time = now() where id = #{id} and is_deleted = 0")
int logicDeleteById(@Param("id") Long id);
```

**为什么**：物理删除不可追溯，误删无法恢复，也无法满足审计要求。

**出处**：`05-(一)10【强制】`

**识别**：grep `delete from`

---

## 14. 数据订正不先确认范围

**反例**

```sql
update t_order set status = 3 where create_time > '2026-01-01';
```

**正例**

```sql
-- 1. 先查影响面
select count(*), min(id), max(id) from t_order where create_time > '2026-01-01' and status <> 3;

-- 2. 小批量执行，带明确 id 范围
update t_order set status = 3, update_time = now()
where id between 10001 and 20000 and create_time > '2026-01-01' and status <> 3;
```

**为什么**：订正语句写错条件就是全表事故。先 select 确认条数与范围，再分批执行，出错时损失可控。

**出处**：`05-(三)8【强制】`

**识别**：任何 `update` / `delete` 脚本执行前，先跑等价的 `select count(*)`

---

## 15. 更新记录不带 update_time

**反例**

```java
orderMapper.updateById(order);   // 只改了 status
```

**正例**

```java
order.setUpdateTime(LocalDateTime.now());
orderMapper.updateById(order);

// 或交给 MyBatis-Plus 自动填充
@TableField(fill = FieldFill.INSERT_UPDATE)
private LocalDateTime updateTime;
```

**为什么**：`update_time` 是排查变更、增量同步、缓存失效的依据，漏更新会导致对账和排障失去时间锚点。

**出处**：`05-(四)7【强制】`；建表三字段见 `05-(一)9【强制】`

**识别**：检查 `updateById` / `update` 语句是否带 `update_time`

---

## 16. 金额用浮点 / BigDecimal 用 equals

**反例**

```java
double price = 0.1 + 0.2;                         // 0.30000000000000004
BigDecimal a = new BigDecimal(0.1);                // 0.1000000000000000055511151231257827
if (amount.equals(BigDecimal.valueOf(100))) { }    // 100.0 与 100 不 equals
```

**正例**

```java
BigDecimal amount = new BigDecimal("0.1");
if (amount.compareTo(new BigDecimal("100")) == 0) { }

// 金额存储：最小货币单位 + 整型
private Long amountCent;
```

**为什么**：`double` 二进制表示无法精确表达十进制小数；`BigDecimal.equals()` 比较的是 scale 和 value，`100` 与 `100.00` 不相等，必须用 `compareTo`。

**出处**：`01-(四)9【强制】`、`01-(四)10【强制】`、`01-(四)12【强制】`、`01-(四)8【强制】`

**识别**：grep `new BigDecimal(`, `.equals(BigDecimal`, 以及 `double` 类型的金额字段

---

## 17. 包装类用 == 比较

**反例**

```java
Integer a = 128;
Integer b = 128;
if (a == b) { }        // false：超出 -128~127 缓存区

Long userId = 1000L;
if (userId == 1000L) { }   // 拆箱后比较，看似对，实际脆弱
```

**正例**

```java
if (Objects.equals(a, b)) { }
if (userId != null && userId.equals(1000L)) { }
```

**为什么**：`Integer` 只有 -128~127 走缓存，超出范围 `==` 比的是引用；`Long` 与字面量比较会触发拆箱，`null` 时直接 NPE。

**出处**：`01-(四)7【强制】`

**识别**：grep `== null` 是合法的，重点找 `== 数字` 且左边是包装类型

---

## 18. 过时的日期类型

**反例**

```java
java.sql.Timestamp ts = new java.sql.Timestamp(System.currentTimeMillis());
long now = new Date().getTime();
```

**正例**

```java
LocalDateTime now = LocalDateTime.now();
long millis = System.currentTimeMillis();
```

**为什么**：`java.sql.Date` / `Time` / `Timestamp` 混杂了日期与时间语义，与时区、纳秒精度处理不一致；`new Date()` 只是为取毫秒而创建对象，没有必要。

**出处**：`01-(五)4【强制】`、`01-(五)3【强制】`

**识别**：grep `java.sql.Timestamp`, `java.sql.Date`, `new Date()`

---

## 附录：快速扫描命令

编辑完成后可对改动文件跑一遍关键字检查（结果需人工判断，不是自动判定）：

```bash
grep -nE "Executors\.new|new Thread\(|new Timer\(|SimpleDateFormat|\$\{|delete from|printStackTrace|e\.getMessage\(\)|System\.out\.println" $(git diff --name-only)
```

`git diff --name-only` 只看本次改动的文件，避免对全仓扫描产生噪声。
