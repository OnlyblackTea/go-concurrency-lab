# go-concurrency-lab

**「go-concurrency-bagu」Track 的骨架仓** —— 与发卷平台 [bagu-trainer](https://github.com/OnlyblackTea/bagu-trainer)
对接：平台按 `track.yaml` 渲染看板与体检环境，按 `stages/*/ticket.yaml` 发卷，
判据在**学员自己的机器上**跑。

> **当前状态：设计完成，骨架尚未落地。**
> 本仓目前只有文档（`docs/`）。骨架代码、canonical 测试、`track.yaml`、`stages/*/ticket.yaml`
> 与阶段分支链会随 Stage 1 一起落进来——顺序与判据形状见
> [`docs/设计规格.md`](docs/设计规格.md)，落地清单见
> [`docs/骨架落地清单.md`](docs/骨架落地清单.md)。


## 这是什么

一句话定位：给「面试要考 Go 并发、但只背过 channel 金句」的人一条能亲手跑出红绿的赛道——

| 项 | 值 |
|---|---|
| track.id | `go-concurrency-bagu` |
| 语言栈 | Go 1.22+（钉到具体补丁版）· `go build` / `go vet` · `go test` + `go test -race` · **只用标准库（零第三方依赖）** |
| 基础设施 | **无外部依赖，纯进程内**：不开端口、不起容器、不联网。唯一的宿主前置是 Go 工具链与一个可用的 C 编译器（`-race` 要 cgo）；可选档位用钉死摘要的 `golang:1.22.x` 镜像把工具链也变成可复现项 |
| stage 数 | 6 个（另有 1 个 board-only 的 `stage-00-onboarding`，不出 ticket、无分支） |
| 每 stage 规模 | 按 [`_SCALE.md`](./_SCALE.md) 的下限复核后（本轮「v1.1 规模复核」，见 §1 开头）：**逐 stage 改动面 ≥ 3 个文件且三类角色齐 · canonical 6~7 条（红 3 · 单线程基线 2~4 · 边界/拒绝 1）· 被破坏的不变量 3 个（= 红用例数，互不相同）· 考点族 ≥ 2 个**。全 Track 合计改动面 **9 → 19 个文件位置**（v1.1 新增 10 处：`quota/issuer.go`、`quota/ledger.go`、`dispatch/fanout.go`、`dispatch/closeguard.go`、`dispatch/tally.go`、`limit/accounting.go`、`sched/pool.go`、`sched/stats.go`、`pool/manager.go`、`pool/gate.go`；另有 1 处**判定入口**改造：`scripts/gate-exit.sh` + `smoke/gate_exit_test.go`，见 §6），canonical **18 → 37 条**（**红 18 · 单线程基线 13 · 边界/拒绝 6**）。**6 个 stage 全部「同 stage 扩面」、零合并**，理由见 §1 的 v1.1 块 |
| 依赖前置 | `go: "1.22"`、`build: go`、`race: "cgo + C 编译器（Windows 上需 gcc）"`、`cpu: "2"`、`mem: "2GiB"`、`docker: "24"`（仅容器档位需要，不装不影响任何 stage） |
| 与 database-bagu 的关系 | **互补而非续写**。它是 Java 17 + Maven + MySQL + Redis 的「重资产栈」，本 Track 是零基础设施的第二种语言栈：seckill 要起两个容器才开始做题，这条 Track 的冷启动是 0。考点也不重叠——JVM 侧的 JMM / `synchronized` / 线程池 / 异步编排归 `concurrency-lab`（同为 Java），Go 侧的 channel/select 通信模型、GMP 调度、context 取消传播、race detector 归本条。两边共享的「临界区与可见性」在两个语言里是**不同机制**，正好互为对照 |
| 预计落地工作量 | 骨架（6 条 lane + **10 处 v1.1 新增的状态 / 并发语义载体 / 编排边界** + 组合根 + 工具链脚手架）**10~13 人日** / canonical 测试与 `testkit`（判据的真正难点）**8~10 人日** / 题面·判据·八股地图 6 张 **6~8 人日** / 分支链 + digest pin + 实测证据报告 **3~4 人日**，**合计 27~35 人日**（旧口径 17~24 人日，已按 §7 的 v1.1 重新估算块上修） |

## stage 一览

| stage | 分支名 | 依赖 | canonical 测试路径 |
|---|---|---|---|
| 1 | `stage-01-data-race` | `[]`（**DAG 根**，发卷即可用） | acceptance/stage01/race_test.go |
| 2 | `stage-02-channel-semantics` | `["go-stage-01"]` | acceptance/stage02/channel_test.go |
| 3 | `stage-03-goroutine-leak-and-context` | `["go-stage-02"]` | acceptance/stage03/lifecycle_test.go |
| 4 | `stage-04-sync-primitives` | `["go-stage-03"]` | acceptance/stage04/primitive_test.go |
| 5 | `stage-05-gmp-and-pprof` | `["go-stage-04"]` | acceptance/stage05/budget_test.go |
| 6 | `stage-06-escape-and-gc`（选修）` | `["go-stage-04"]` | acceptance/stage06/alloc_test.go |

> 表里的**分支名就是 git 分支名**：平台的 Forge 在 provision 时用
> `base_ref = origin/<分支名>`（起始代码）与 `head_ref = origin/<分支名>-sol`（参考解），
> 两者缺一都会报 `workspace_ref_missing`。分支是**一条链**（每个阶段分支 = 上一阶段 + 本阶段测试），
> 不是兄弟分支——理由见骨架落地后的 `stages/README.md`。

## 目录

```
go-concurrency-lab/
├── README.md              ← 本文件
├── LICENSE                ← MIT（内容仓许可）
├── .gitattributes         ← eol=lf：digest_pin 要求工作树字节跨平台稳定
├── .gitignore
└── docs/
    ├── 设计规格.md          ← 本 Track 的设计规格快照（见下方提醒）
    └── 骨架落地清单.md       ← 落地前要逐条核对的清单
```

骨架落地后还会出现（形状见设计规格 §2）：

```
├── track.yaml             ← 与平台对接的清单（manifest 惰性声明 / runtime 可执行入口 + digest_pin）
├── pom.xml / go.mod / pyproject.toml
├── src/ 或 internal/      ← 业务实现（学员改动面）
├── src/test/... 或 tests/ ← canonical 验收判据【判定面】
├── stages/<stage-目录>/ticket.yaml   ← 权威工单契约（目录名 = 分支名）
├── stages/README.md       ← 分支链契约
├── scripts/               ← build / digest-pin / check-red-green【判定面】
└── TASKS/                 ← 学习者叙述视图与交付证据
```

## 与规格的关系（重要）

`docs/设计规格.md` 是**创建本仓时的规格快照**，权威副本在题库侧的
`bagu-track-specs/go-concurrency-lab.md`（设计仍在演进的阶段，改动先落那边，再同步到这里）。

两份文档的分工别搞混：

- **平台契约**（字段、准入门、分支链、锚点算法）由平台仓单方面定义，本仓只遵守、不复制规范正文；
- **题面怎么写**见平台仓 `docs/工单写作规范.md`；
- **每个 stage 该有多大**见题库侧 `_SCALE.md`（改动面 ≥3 文件、canonical ≥5 条、
  红 ≥2 条不同不变量、含边界路径）——本规格已按它复核过。

## 许可

MIT，见 [LICENSE](LICENSE)。
