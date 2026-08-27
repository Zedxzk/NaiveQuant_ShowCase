# NaiveQuant 架构说明

这份文档介绍 NaiveQuant 当前（**0.21.5**）的分层架构和几个关键边界。本文档从技术评审角度出发，仅说明公开结构，不展开专有实现细节。

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

职责边界（与代码中的契约一致）：

- `core` 定义协议和编排基础，不依赖市场、模型、执行或 UI 实现。
- `data` 读取市场数据；`factor` 计算/读取因子并维护因子生命周期。
- `ml` 提供数据集、模型和通用 walk-forward，不持有 timing 决策或执行规则。
- `timing` 把 routed dataset 转成 OOS 时序预测；市场和数据源不进入公共引擎名。
- `portfolio` / `strategy` / `risk` / `policy` 形成账户目标与约束。
- `execution` 消费决策，负责成交、成本和账户；`trading` 负责常驻模拟/实盘运行。
- `reporting` 和 `visualization` 只呈现已有结果；`archive` 保存运行上下文和产物。
- `cli`、`ui`、`workflows` 是入口/编排层，不应被底层计算模块反向依赖。

被依赖最多的模块是 `core/workspace`、`core/constants`、`core/protocols`、`auth`、`core/logger` 和 `core/config`。这些模块位于基础层，改动影响范围较大，因此接口设计力求稳定。

### 依赖红线是可执行的

这些边界写成 `pyproject.toml` 中的 **19 条 import-linter 契约**，由 `lint-imports` 校验：

| 契约 | 含义 |
|---|---|
| `core 是最底层，不得向上依赖` | `core` → 任意业务包的 import 一律失败 |
| `auth 只依赖 core` | 安全基座可被独立部署的 `trading_server` 复用，不会被拖入 UI 层 |
| `backtest 不得依赖呈现/入口层` | 计算层不 import `reporting` / `visualization` / `ui` / `cli` |
| `factor 不得依赖 selection 及上层` | 保持 `selection → factor` 单向 |
| `data 不得依赖研究/呈现/入口层` | 数据层不感知因子、回测和界面 |

少量历史兼容 shim 以 `ignore_imports` 显式登记并注明原因（例如 PEP 562 惰性转发），实现代码不使用这些边。

## 2. `quantplatform`、`trading_server` 与插件的边界

```
trading_server  ──单向──►  quantplatform.*
plugins/*       ──单向──►  quantplatform 公开 API
```

**运维/推理服务 `trading_server`** 是一个独立包。它可以调用研究平台 `quantplatform` 的公开接口，但 `quantplatform` 不反向导入它。服务端主要依赖几类稳定接口：records、broker/env、auth 和 core 常量。早期曾有一些直接调用内部实现的代码，后来统一收敛到公开 facade 函数，避免 `quantplatform` 内部重构时波及服务端。

**插件**由一份白名单 YAML 治理，加载器按名动态加载，平台代码中不存在对任何插件的静态 import。每个插件带 `plugin.yaml`，声明自身版本、兼容的平台版本区间、提供的能力（`market_data` / `live_signal` / `ops_service` / `event_sink` / `research` / `screener` …）以及可选的研究台或运维台页面挂载点。

治理上的三条约束：

- **启停零代码改动**：改白名单里的 `enabled` 即可，`git diff` 可查，运维台可见。
- **fail-closed**：启用但未安装或版本不兼容时直接抛错并在运维台标红，不静默降级。
- **页面不混挂**：研究插件页由研究台承载，声明 `ops_ui` 的管理页面只由运维台承载，避免配置和服务操作绕过登录、RBAC 与 TOTP 边界。

## 3. YAML 驱动的流水线：显式 DAG 与 Legacy 平铺

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

执行层仅接收最终账户决策，不包含具体交易规则。研究阶段产出的目标权重和风控后的账户决策，可直接复用于纸账户或实盘路径，无需重写下单逻辑。

## 4. DuckDB 存储和并发

DuckDB 的写入模型更适合单进程分析场景，但长期运行的采集服务与研究进程会并发访问数据。具体处理方式如下：

- **写入器不长期持有锁**：`open → write → close`，每次写完即释放连接。
- **只读连接按库区分**：静态研究库在进程内复用只读句柄，降低磁盘 I/O 较慢时反复建连的开销；可更新的 canonical 库显式关闭句柄复用，避免研究服务长期持有只读锁导致采集任务无法写入。
- **记录层采用 append-only JSONL**：订单、对账和审计记录避免与 DuckDB 竞争写锁，跨进程追加文件即可。
- **schema 迁移原子执行**：版本升级时先在影子库构建，全部约束和抽样对账通过后才原子切换 canonical，并保留带时间戳的可恢复旧库。

## 5. Point-in-Time 与复权口径

为避免前视偏差，平台在采集层就标记数据属于哪个财报周期、何时对外可见：

- SEC EDGAR 财报同时保存 `period_end` 和 `filed`→`available_at`。
- 因子和回测行至任一历史时点，只能读取当时已经公布的数据。
- 这一约束在数据层就已落地，后续的因子、回测和组合模块均按同一套时间口径取数。

复权统一采用**仿射复权**：

```
adjusted_price = raw_price × adj_factor(A) + adj_offset(B)
```

港股的 A/B 由官方复权事件按倒序语义累计得到——新事件 `(a, b)` 加入旧链 `(A, B)` 得 `A' = aA`、`B' = bA + B`。现金分红体现在 B、拆并股体现在 A，两者组合在同一条链上；单比例方案只能表达 A，分红会留下伪跳空。A 股和美股的单比例数据以 `B = 0` 兼容读写。

因子请求失败的标的不写入，也不以 `A=1 / B=0` 代填。展示价与成本用原始价，技术指标与研究口径用后复权价。

## 6. 安全与鉴权基座

`quantplatform/auth/` 是研究台（本地 FastAPI）与运维台（远程 FastAPI）**共用**的完整鉴权基础设施，它只依赖 `core`，所以 `trading_server` 独立部署不会被拖入 UI 层。同一请求依次经过三层正交能力：

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
- 部署主机以受管服务器列表为唯一事实源，脚本中不内嵌任何公网 IP；列表外的主机在建立网络连接前即被拒绝。
- 部署使用增量 rsync，排除 `.git`、数据、运行缓存和快照目录，安装后进行冒烟测试和版本核对，并按固定顺序显式重启各 unit，保证已 active 的旧进程也加载当前 revision。

## 8. 质量保障

| 手段 | 说明 |
|---|---|
| `pytest` | 163 个测试文件、1,570+ 用例，按 data / factor / models / brokers / backtest / timing / plugins / trading_server 等域拆分子集 |
| `ruff` | 统一 lint 与格式 |
| `import-linter` | 19 条分层契约，防止依赖方向回潮 |
| `scripts/check_docs.py` | 校验文档本地链接与 anchor、平台/插件版本一致性、文档索引覆盖、归档状态标识、环境变量与关键数量事实 |
| 运行归档 | 每次研究运行的配置、命令、标的、指标和报告统一落库，可按 `run_id` 复原 |
