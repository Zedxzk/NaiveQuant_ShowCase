# NaiveQuant 架构说明

这份文档介绍 NaiveQuant 当前（**0.36.34**）的分层架构和几条关键边界。平台的 Python 包在 0.30 由 `quantplatform` 更名为 `naivequant`（旧名保留为别名）。下面只讲分层结构和边界，不涉及实现细节。

## 1. 分层依赖模型

平台依赖自下而上组织：底层提供基础能力，上层组合业务流程。

```
┌─────────────────────────────────────────────────────────────────────┐
│ L5 交互层   ui/ · cli_commands/ · reporting/ · visualization/ ·        │
│             scheduler/                                                 │
├─────────────────────────────────────────────────────────────────────┤
│ L4 编排层   ml/ · backtest/ · workflows/ · selection/ · optimization/ ·│
│             archive/                                                   │
├─────────────────────────────────────────────────────────────────────┤
│ L3 领域层   factor/  ‖  strategy/·portfolio/·risk/·policy/            │
│                      ‖  execution/·account/·trading/                   │
│             └──────────── timing/ 时序择时领域 ───────────┘           │
├─────────────────────────────────────────────────────────────────────┤
│ L2 数据/模型 data/ · models/(rl·deep·determinism) · stats/ · auth/     │
├─────────────────────────────────────────────────────────────────────┤
│ L1 core 基础 schemas · protocols · artifacts · registry ·              │
│              pipeline_graph · secrets · settings_store · workspace ·   │
│              data_catalog · secure_bundle · events                     │
├─────────────────────────────────────────────────────────────────────┤
│ L0 零依赖层  constants(零依赖·无内部引用) · logger · config            │
└─────────────────────────────────────────────────────────────────────┘
```

各层职责（跟代码里的契约一致）：

- `core` 定义协议和编排基础，不依赖市场、模型、执行或 UI 实现。
- `data` 读取市场数据；`factor` 计算/读取因子并维护因子生命周期。
- `ml` 提供数据集、模型和通用 walk-forward，不含 timing 决策和执行规则。
- `timing` 把 routed dataset 转成 OOS 时序预测；市场和数据源不进入公共引擎名。
- `portfolio` / `strategy` / `risk` / `policy` 形成账户目标与约束。
- `execution` 消费决策，负责成交、成本和账户；`trading` 负责常驻模拟/实盘运行。
- `reporting` 和 `visualization` 只呈现已有结果；`archive` 保存运行上下文和产物。
- `cli`、`ui`、`workflows` 是入口/编排层，底层计算模块不应该反过来依赖它们。

被依赖最多的模块是 `core/workspace`、`core/constants`、`core/protocols`、`auth`、`core/logger` 和 `core/config`。它们在基础层，改一处影响面很大，所以接口尽量保持稳定。

### 依赖红线是可执行的

这些边界写成 `pyproject.toml` 中的 **20 条 import-linter 契约**，由 `lint-imports` 校验：

| 契约 | 含义 |
|---|---|
| `core 是最底层，不得向上依赖` | `core` → 任意业务包的 import 一律失败 |
| `auth 只依赖 core` | 安全基座可被独立部署的 `trading_server` 复用，不会被拖入 UI 层 |
| `backtest 不得依赖呈现/入口层` | 计算层不 import `reporting` / `visualization` / `ui` / `cli` |
| `factor 不得依赖 selection 及上层` | 保持 `selection → factor` 单向 |
| `data 不得依赖研究/呈现/入口层` | 数据层不感知因子、回测和界面 |

少量历史兼容 shim 以 `ignore_imports` 显式登记并注明原因（例如 PEP 562 惰性转发），实现代码不使用这些边。

## 2. `naivequant`、`trading_server` 与插件的边界

```
trading_server  ──单向──►  naivequant.*
plugins/*       ──单向──►  naivequant 公开 API
```

**运维/推理服务 `trading_server`** 是一个独立包：它可以调研究平台 `naivequant` 的公开接口，但 `naivequant` 不会反过来 import 它。服务端主要依赖几类稳定接口：records、broker/env、auth 和 core 常量。早期有些地方直接调内部实现，后来都收到公开的 facade 函数里，免得 `naivequant` 内部重构时波及服务端。

**插件**由一份白名单 YAML 管理，加载器按名字动态加载，平台代码里没有任何一处静态 import 插件。每个插件带 `plugin.yaml`，声明自身版本、兼容的平台版本区间、提供的能力（`market_data` / `live_signal` / `ops_service` / `event_sink` / `research` / `screener` …）以及可选的研究台或运维台页面挂载点。

插件体系有三条约定：

- **启停不用改代码**：只改白名单里的 `enabled`，`git diff` 里能看到，运维台也能看到。
- **fail-closed**：启用但未安装或版本不兼容时直接抛错并在运维台标红，不静默降级。
- **页面不混挂**：研究插件页由研究台承载，声明 `ops_ui` 的管理页面只由运维台承载，避免配置和服务操作绕过登录、RBAC 与 TOTP 边界。

## 3. 用 YAML 配置的流水线：显式 DAG 与 Legacy 平铺

平台有两种 pipeline 配置形态，共用同一个 runner 门面。

### 3.1 显式 DAG（0.16 起）

```yaml
pipeline:
  graph:
    nodes:
      - id: data
        layer: data
        type: market_data_route
      - id: timing
        layer: timing
        type: ts_ml_timing
        inputs: {dataset: data.dataset}
      - id: decisions
        layer: timing_policy
        type: entry_exit_threshold
        inputs: {predictions: timing.predictions}
      - id: execution
        layer: execution
        type: signal_sink
        inputs: {decisions: decisions.decisions}
```

节点 `inputs` 写成 `<node-id>.<output-channel>`。编译器在**加载期**依次完成：

1. 查 provider registry 并读取其节点契约；
2. 校验重复 ID、未知输入、必需 channel、artifact 类型和 schema；
3. 校验 `run` / `fold` / `date` 作用域连接，向上汇总必须有显式 reducer；
4. 拓扑排序并拒绝环；
5. 要求图中存在 execution sink，且 sink 显式消费决策或成交事件 artifact。

runner 不理解 timing、L2 或某个具体市场，只按拓扑顺序实例化 provider、解析输入、执行、校验输出并写入 artifact store。artifact cache 的 manifest 覆盖节点配置、上游 identity、字段、日期、股票池、标签、滞后、协议、代码版本和源指纹。

### 3.2 Legacy 平铺 YAML

普通非 timing 的平铺 YAML 在加载期被编译成 legacy graph，由统一 runner 门面的 lifecycle adapter 校验节点顺序后复用原有 fold / date 生命周期。新的显式 graph 不依赖任何内置层的物理顺序。

### 3.3 数据流全貌

```
   config/*.yaml
        │
        ▼
  ┌───────────┐  版本化artifact  ┌───────────┐  版本化artifact ┌───────────┐
  │  数据层    │ ──────────────► │ Alpha/ML  │ ─────────────► │  组合构建  │
  │ (含L2路由) │                 │  /timing  │                └─────┬─────┘
  └───────────┘                 └───────────┘                      ▼
  ┌───────────┐    账户决策      ┌───────────┐    目标权重     ┌───────────┐
  │  执行下单  │ ◄────────────── │   风控     │ ◄───────────── │   策略     │
  └───────────┘                 └───────────┘                └───────────┘
        │
        ▼
   券商适配器(Alpaca/Futu/IBKR/Longbridge/QMT) → 报告 → 运行归档 → 运维监控
```

执行层只接收最终的账户决策，具体交易规则不在这里。研究阶段产出的目标权重、风控后的账户决策，可以直接复用到纸账户或实盘，不用重写下单逻辑。

## 4. DuckDB 存储和并发

DuckDB 的写入模型更适合单进程分析，但采集服务和研究进程会长期并发访问同一批数据。这里的做法是：

- **写入器不长期持有锁**：`open → write → close`，每次写完即释放连接。
- **只读连接按库区分**：静态研究库在进程内复用只读句柄，降低磁盘 I/O 较慢时反复建连的开销；可更新的 canonical 库显式关闭句柄复用，避免研究服务长期持有只读锁导致采集任务无法写入。
- **记录层采用 append-only JSONL**：订单、对账和审计记录避免与 DuckDB 竞争写锁，跨进程追加文件即可。
- **schema 迁移原子执行**：版本升级时先在影子库构建，全部约束和抽样对账通过后才原子切换 canonical，并保留带时间戳的可恢复旧库。

## 5. Point-in-Time 与复权口径

为避免前视偏差，平台在采集层就标记数据属于哪个财报周期、何时对外可见：

- SEC EDGAR 财报同时保存 `period_end` 和 `filed`→`available_at`。
- 因子和回测回放到任一历史时点，只能读到当时已经公布的数据。
- 这条约束在数据层就卡死了，后面的因子、回测、组合都按同一套时间口径取数。

复权统一采用**仿射复权**：

```
adjusted_price = raw_price × adj_factor(A) + adj_offset(B)
```

港股的 A/B 由官方复权事件按倒序语义累计得到——新事件 `(a, b)` 加入旧链 `(A, B)` 得 `A' = aA`、`B' = bA + B`。现金分红体现在 B、拆并股体现在 A，两者组合在同一条链上；单比例方案只能表达 A，分红会留下伪跳空。A 股和美股的单比例数据以 `B = 0` 兼容读写。

因子请求失败的标的不写入，也不以 `A=1 / B=0` 代填。展示价与成本用原始价，技术指标与研究口径用后复权价。

## 6. 安全与鉴权基座

`naivequant/auth/` 是研究台（本地 FastAPI）和运维台（远程 FastAPI）**共用**的鉴权基座：它只依赖 `core`，所以独立部署的 `trading_server` 不会因此被拖进 UI 层。一个请求要依次过三道互不相干的检查：

| 层 | 关注点 | 失败 |
|---|---|---|
| 鉴权 Authentication | 凭证解析（token / 签名会话 cookie）、全局中间件校验 | 401 |
| 授权 Authorization | RBAC 三层角色 + 权限集合 + TOTP 二次验证 + 危险操作四眼审批 | 403 |
| 作用域 Scoping | 部署级设置 vs 个人偏好、per-user 密钥目录、交易凭证命名空间 | — |

所有落盘文件（用户表、待审批请求、访问审计、会话签名密钥、研究资源授权）在 POSIX 下一律原子写 `0600`，并按 OS 账户与 profile 隔离。

## 7. 部署拓扑

```
        本地开发机 (main 分支)
            │  git push + 增量 rsync 部署链
            ▼
   ┌────────────────────────────────────┐
   │  云服务器（受管列表内主机）           │
   │  systemd --user + linger            │
   │  ├ nquant-ui           研究台        │
   │  ├ trading-server      运维/推理服务  │
   │  ├ nquant-notify       通知投递       │
   │  ├ nquant-system-monitor 资源遥测     │
   │  ├ nquant-portfolio-watch 持仓研究    │
   │  ├ nquant-hk-broker-flow  经纪流采集  │
   │  ├ nquant-hk-market-regime 盘中方向   │
   │  └ recorder / enrich / daemon 等定时任务│
   └────────────────────────────────────┘
     服务绑定 127.0.0.1，经 SSH 隧道访问，不暴露公网
```

- systemd `--user` + linger：免登录常驻，`Restart=on-failure` 自动重启；共 16 个 unit。
- 凭证主口令仅保存在 systemd 用户管理器内存中，通过 `set-environment` 传递给服务，不写入磁盘。
- 部署以受管服务器列表为准，脚本里不内嵌任何公网 IP；名单外的主机在建立网络连接之前就被拒掉。
- 部署使用增量 rsync，排除 `.git`、数据、运行缓存和快照目录，安装后跑一遍冒烟测试和版本核对，再按固定顺序显式重启各 unit，让已经 active 的旧进程也加载到当前 revision。

## 8. 质量保障

| 手段 | 说明 |
|---|---|
| `pytest` | 229 个测试文件、2,170+ 个测试函数，按 data / factor / mining / models / brokers / backtest / timing / plugins / trading_server 等域拆分子集；显卡相关的用例有显卡时另外跑 |
| `ruff` | 统一 lint 与格式 |
| `import-linter` | 20 条分层契约，防止依赖方向回潮 |
| `scripts/check_docs.py` | 校验文档本地链接与 anchor、平台/插件版本是否一致、文档索引有没有漏、归档状态标识、环境变量和关键数字 |
| 运行归档 | 每次研究运行的配置、命令、标的、指标和报告统一落库，可按 `run_id` 复原 |

## 9. 因子研究子系统

0.22 以后，因子研究的计算都收拢到 `factor` 领域内的几个子包，研究脚本只负责“搜索什么、怎么入库”，不再各自实现计算：

```
factor/
├── panel/        面板计算引擎
│   ├── data/     原料面板：按 catalog 装载日线、分钟线、估值、事件、行业、宏观，带预热期
│   ├── lang/     公式语言：表达式树、算子注册、记忆长度、子式缓存、取值指纹
│   ├── search/   搜索算法：分层搜索、演化搜索、搜索空间（量纲与冗余规则）
│   ├── model/    模型因子：特征矩阵、截面注意力网络、梯度提升树、训练与推断
│   └── signal/   冻结信号：定义 + 权重 + 原料规格 + 哈希，可在任意面板上重算核对
├── mining/       逐年挖掘：评价口径、逐日成绩表、入围与入库、安慰剂校准、报告
├── scoring.py    唯一的打分实现（前 20% 净超额、上半截排序、换手、稳定性）
├── values.py     因子取值的唯一入口
└── factor_library.py   因子库：状态、分库口径、审计、样本外跟踪
```

几条约束：

- **取值只有一个入口**。因子一律是“平台可重算的冻结信号”；回测、组合、报告取因子值都走 `factor.values`，不各自重算。0.36 删除了旧的因子类引擎，不保留兼容。
- **结果与数据起点无关**。每个算子登记记忆长度，面板开头记忆长度不足的行不给值；入库前用另一块面板重算并逐位核对，不一致的不入库。
- **人工把关生命周期**。挖掘程序只能把因子登记为候选；升级、退役、删除由人执行，程序调用会被拒绝。
- **长任务可续跑、受资源约束**。逐年挖掘写成流程定义（年份 × 步骤），由 `scheduler` 执行；每一步向资源账本申领内存与显存，超出额度由守护进程终止并重试，状态与日志在界面的任务页可见。

