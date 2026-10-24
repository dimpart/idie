# 子任务分解与评审状态（Teddy 维护）

> 分解原则：按需求域拆分、按依赖排序；每个子任务有明确的输入需求 ID、产出文件与验收标准。
> 评审流程：Jessica 完成 → Teddy 对照需求清单检查 → 不符合则指出问题重做 → 通过后进入下一项。

## 子任务清单（8/8 全部通过）

| # | 子任务 | 输入需求 | 产出文件 | 状态 |
|---|---|---|---|---|
| 1 | 积分经济系统详细设计 | E-001 ~ E-008 | `docs/01-积分系统设计.md` | ✅ 已通过 |
| 2 | 体力系统与 VIP 详细设计 | I-001 ~ I-009、E-006 | `docs/02-体力与VIP设计.md` | ✅ 已通过 |
| 3 | 格子生命周期状态机设计 | L-001 ~ L-007 | `docs/03-生命周期状态机设计.md` | ✅ 已通过 |
| 4 | 核心交互与信息架构设计 | U-001 ~ U-005、G-001 ~ G-008 | `docs/04-交互体验设计.md` | ✅ 已通过 |
| 5 | 数据模型与技术方案 | P/G/T/L 相关 | `docs/05-数据模型设计.md` | ✅ 已通过 |
| 6 | 内容治理与隐私设计 | T-004 ~ T-006、I-009 | `docs/06-内容治理设计.md` | ✅ 已通过 |
| 7 | 验证计划细化 | V-001 ~ V-007 | `docs/07-验证计划细化.md` | ✅ 已通过 |
| 8 | 数值体系与定价策略 | 用户三轮定价意见（20 条）、O/P 系列参数 | `docs/08-数值体系建议.md` | ✅ 已通过 + 用户已拍板 |

## 产品设计阶段结论（2026-10-24）

- 8/8 子任务全部通过评审；需求评审会 8/8 模块通过（见 `评审纪要-2026-10-24.md`）。
- **数值体系已由用户拍板采纳**（"按你们的建议来更新"）：货币「灵石/Stone」、K=100,000、注册赠送 150,000、签到 +100/连签 7 天 +500、邀请 +50,000/人·年上限 2·14 天窗口、F=1,000 回馈 30%、爱心 10/次不封顶（单日 >150 风控）、体力免费 30/VIP 120（4 小时回满，8/2 分钟每点）、VIP 单档 2.99/6.99/10.99/15.99 美元、双首充（¥6 档限量充值首充 3 折 + 0.99 美元破冰礼包）、缓冲期 30 天、网格 ±128、内测 30 人 3 周（iOS 首发）。
- 全模块一致性审计：✅ 全部自洽——requirements/ 02-09 已加注定稿值，docs/01-08 数值对齐，术语统一"灵石"，docs/08 英文名 Stone，飞书主文档同步 v4.3。

## 技术研发设计阶段（2026-10-24）

- **任务**：先读 dimpart 三个 repo 理解 DIMP 协议与 SDK，再输出数据库、API、双端架构设计；Balon=前端（Flutter，`product/`，AS 可编译）、Toby=后端（Python，`services/`，PyCharm 工程）；不架服务器，服务端 = Service Bots；通信走 DIM Message content（`type=CUSTOMIZED`，`app=chat.dim.idie` 恒定，每 bot 一个 mod、act 区分动作，其余参数放 content）。
- **SDK 决策**：客户端基于 pub.dev `dim_client: 1.7.1`（dim_flutter / tarsier 仅参考）；后端基于本地 `dimples`（service-py 仅参考）。协议对象：后端 `AppCustomizedContent.create(app,mod,act)`（官方格式 `{type:i2s(0xCC), sn, app, mod, act}`）；前端 dimp `ContentType.CUSTOMIZED = 0xCC('204')`。
- **产出**：
  1. `docs/09-技术架构设计.md` — 总体架构（mermaid）、应用协议契约、7 个 Bot/mod 拆分、消息时序（sequenceDiagram）、双端工程规划、错误码、安全合规；
  2. `docs/10-API接口设计.md` — 7 mod 全量 act（account/grid/tomb/economy/interact/govern/system）+ 请求/应答 content 契约 + 事务红线 + 幂等键清单；
  3. `docs/11-数据库设计.md` — 12 张表 DDL（user/cell/placement/tombstone/interaction/ledger/checkin/invite/purchase/caretaker_auth/report/config）+ 部分唯一索引 + Redis key 规范 + 一致性约束。
- **Teddy 校验**：数值全部对齐已拍板值；评审决策全量落地（占格条件更新 + 部分唯一索引、体力 Redis CAS、VIP 实时判定、16×16 block_key、MQ 缩略图、IAP 唯一入账锚、首充幂等、成品价下发、回馈即时、UTC 签到、14 天窗口、caretaker ≤5、匿名化注销）；补 `account.deregister`（docs/10，含 7 天冷静期取消）。
- **未拍板（不阻塞联调）**：献花频率、献爱心同墓碑 24h≤5、占格 24h≤2、姓名 ≤8 字、抽查 20% 分层、举报 24h 首响、Bot ID 域名、头像存储位置。
- **评审结论**：✅ 技术设计自洽。

## 工程搭建阶段（2026-10-24）

**任务**：双端工程脚手架落盘（Balon=前端 Flutter / Toby=后端 Python），基于 dim_client 与 dimples，通信走 DIMP CustomizedContent（app=chat.dim.idie、每 bot 一 mod、act 区分动作、应答回显 sn）。

**后端（Toby）**：
- `services/`（PyCharm 工程）：`.venv`（python3.11 + pip install -e demo-py）、`config/bots.ini`（全量已拍板数值）、`bots/shared.py`（bot_main 统一入口）、`database/{redis_store,business}.py`（Redis CAS + SQLite WAL 12 表）、`engine/{base_service,bot}.py` + 7 service + 7 入口、`start_all.sh`、`tests/smoke_test.py`。
- 冒烟测试 **11 项全部通过**：注册赠送、签到、占格扣减、格子占用保护(1002)、余额不足(1001)、cells_get、续费、献花/爱心计数、献花扣款、流水 5 条、释放。
- `cells_get` SQL 参数序已修正（BETWEEN 需 (x0,x1,y0,y1)）。

**前端（Balon）**：
- `product/`（Android Studio Flutter 工程）：`flutter create` + `dim_client: 1.7.1` + `object_key`（1.7.1 为解析链兼容最终版本）。
- `lib/core/`：`idie_database.dart`（IdieMemoryDB implements AccountDBI/MessageDBI/SessionDBI + CipherKeyDelegate）、`dim_session.dart`（IdieTerminal/IdiePacker/IdieMessenger/IdieProcessor + DeviceMixin + mockLogin）、`idie_api.dart`（sn 匹配通信层 + 9 个 act 封装）。
- `lib/models/grid.dart`（±128 边界、步长 2、视图窗）、`lib/views/grid_page.dart`（网格首页：平移交互 + 墓碑渲染 + DEV 降级 mock）、`lib/main.dart`。
- 验证全过：`flutter analyze` No issues、`flutter test` All tests passed、`flutter build apk --debug` 成功。

**Teddy 双端契约校验**：
- mod/act：后端 7 mod 全量 act 注册 vs 前端 9 个 act 封装（cells_get/cell_status/cell_occupy/register/checkin/balance_get/flower_send/heart_send/config_get）全部对齐；
- app='chat.dim.idie' 两端恒等；应答外壳 ok/code/msg/data 与 sn 回显两端一致；错误码 1001/1002 语义对齐。
- **评审结论**：✅ 双端工程可编译可运行（骨架），契约自洽。

**遗留（不阻塞骨架，联调前处理）**：Bot ID 最终域名（现占位 `idie-<mod>@chat.dim.idie`）；真实注册流程（前端 mockLogin → station 注册）；sqflite DB 替换内存实现；头像/FileContent 上传链路；IAP 回执验证 stub；redis 服务启动。

## design/ 全量复审（2026-10-24）

- **范围**：README.md、tasks.md、评审纪要-2026-10-24.md、requirements/ 01-09 全部 12 个文件；对照基线 = docs/08 已拍板数值 + 评审会决策 + DIMP 双端架构 + 工程搭建实况。
- **修正**：评审纪要接口/字段对齐 DIMP 最终契约（`system.config_get`、`stamina/stamina_max/vip_expiry`）；requirements 05/08 补注定稿值；README 更新目录与日期；requirements 02/04/06 定稿表述；requirements/09 移除旧值更新记录；tasks.md 全量重写为最新状态。
- **结论**：✅ design/ 全部自洽，无遗留矛盾，全部内容已统一至最新版（2026-10-24）。
