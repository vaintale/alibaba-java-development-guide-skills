# 项目编辑画像

> 每个项目一个文件，文件名 = 工作目录顶层目录名（如 `D:\code\物业系统\contract-service` → `物业系统.md`）。
> 目的：让下一次在本仓库编辑时，不必重新发现「怎么编译、怎么分层、什么风格」。

## 结构

```
project/
├── README.md          # 本文件（入库）
├── _template.md       # 模板（入库）
├── _example.md        # 虚构示例，展示填好后的样子（入库）
└── <项目名>.md         # 真实项目画像（本地，不入库）
```

## 入库约定（公开仓库必读）

**真实的 `<项目名>.md` 不要提交到公开仓库。** 它不是文档，是运行数据，里面有：

- 内部模块名、包结构、类名（`CachedStationRepository`、`JsonbTypeHandler` 这类）
- 构建与启动命令、私服地址、profile 名
- 表名、字段名、业务定位

这些拼起来就是一份内部架构说明。仓库根 `.gitignore` 已排除 `project/*.md`，只放行 `README.md`、`_template.md`、`_example.md`。

要共享给团队，放**私有**仓库或内部文档系统，不要跟着 skill 一起公开。

**真实画像放哪**：仓库根目录的 `.local/project-profiles/<项目名>.md`（已被 `.gitignore` 排除，也不会进入打包产物）。安装 skill 之后，把它复制到 `~/.workbuddy/skills/java-edit-guard/project/` 即可生效。

> 注意：仅靠 `.gitignore` 不够——`package_skill.py` 打包的是目录内容，会连未跟踪文件一起带走。所以真实画像必须放在 skill 目录之外。

## 读取时机

编辑代码前（`SKILL.md` §9）：先读 `memory.md`，再读 `project/<项目名>.md`，项目层覆盖个人层。

## 创建方式

1. 首次在某项目编辑时，扫描 `pom.xml` / `build.gradle`、目录结构、已有 Service 与 Controller。
2. 用 `AskUserQuestion` 确认 2~3 个关键项：验证命令、存量卫生度、是否有自研框架封装。
3. 复制 `_template.md` 为 `<项目名>.md` 并填写，告知用户已创建。
4. 不提交到公开仓库（见上）；团队共享走私有仓库。

## 维护

- 发现新的隐性约定（如「所有 Mapper 必须手写 XML」），问一次是否记录。
- 不要记录一次性的调试信息、临时路径。
- **密码、密钥、token 一律不写**；内网地址与私服地址也不要写，写「见内部文档」即可。
- 项目改名或模块拆分后，回来更新「分层与包结构」与「验证命令」——这两节过期比没有更糟。
