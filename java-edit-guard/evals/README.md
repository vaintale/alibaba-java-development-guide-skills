# 评测用例

## 来源与许可

用例与评分脚本来自 [Sxuan-Coder/alibaba-java-development-guide](https://github.com/Sxuan-Coder/alibaba-java-development-guide)，按 **MIT** 协议收录，未修改判定逻辑。

本仓库主体按 Apache-2.0 发布；MIT 与 Apache-2.0 兼容，但保留此来源说明。

## 结构

```
evals/
├── eval.yaml                  # 评测入口：用例列表、超时、benchmark 开关
├── cases/                     # 10 个用例，每个含 prompt / expect / judge
└── fixtures/scripts/          # 两个 PowerShell 评分脚本（按用例的 judge 规则调用）
```

## 用例覆盖

| 用例 | 验证点 |
|---|---|
| `naming-style` | 命名风格（小驼峰、常量全大写） |
| `null-safety-service` | NPE 防护与入参校验 |
| `sql-injection` | `${}` 与字符串拼接 SQL |
| `date-time-api` | `SimpleDateFormat` → `java.time` |
| `project-layering` | 分层职责、DO 不出 Service |
| `collection-concurrency` | 线程安全容器 |
| `review-bad-code` | 审查视角：能否按严重度汇总 |
| `review-money-calc` | 金额精度与 `BigDecimal` 用法 |
| `legacy-add-feature` | 存量加功能：不误改周边风格 |
| `legacy-threadpool-enforce` | 存量线程池红线：强制修复 |

## 怎么跑

`eval.yaml` 的 `engine` 当前是 `claude_code`，`benchmark.enabled: true` 表示每个用例跑 with_skill / without_skill 两组对比。

接入 skill-up 后按它的 benchmark 模式执行；换平台改 `engine` 一处即可。用例本身不引用 skill 名，目录改名后仍可运行。

## 注意

- 两个 `.ps1` 评分脚本写的是 PowerShell，Linux/macOS 上需要 `pwsh`。
- `legacy-threadpool-enforce` 的判定依赖 `new ThreadPoolExecutor`、`TimeUnit`、阻塞队列类名等字符串，改了正例写法要同步改用例。
