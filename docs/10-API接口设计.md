# 10 API 接口设计（Idie · 爱戴 — DIMP mod/act 契约）

> 作者：Toby｜审核：Teddy + Balon｜日期：2026-10-24
> 协议：所有接口均为 DIM CustomizedContent 消息；`app` 恒为 `chat.dim.idie`；请求由客户端发起、receiver = bot ID，应答由 Bot 发出、receiver = 请求者，**应答回显请求 sn**。
> 通用应答外壳：`{type, sn, time, app, mod, act:<act>_respond, ok:true|false, code:0|..., msg:"", data:{...}}`；`code=0` 成功，其余见 docs/09 §7。

## 0. 命名与通用约定

- mod/act 全小写 snake_case；act 命名 `<动词>_<对象>`；应答 act = 请求 act + `_respond`。
- 参数名全小写 snake_case；时间一律 Unix 秒（整数）；金额/灵石为整数；坐标 (x,y) 为整数，范围 [-128,128]（管理员可配置边界）。
- 客户端请求需携带 `sn`（每次递增）；同一请求重试必须复用同一 sn（幂等）。
- 未标注「可选」的参数为必填；「TBD」为待拍板次级参数，不阻塞联调（按文档默认值实现）。

## 1. mod = account（Bot: idie-account）

| act | 请求参数 | 应答 data | 说明 |
|---|---|---|---|
| `register` | `nickname`(可选) | `user_id, stone_balance, vip_expiry(0)` | 注册赠送 150,000 灵石（入账 ledgers，type=register）；重复注册返回已有 user_id |
| `login` | `device_id` | `user_id, profile, stone_balance, vip_expiry, stamina, config_ver` | 客户端登录态恢复；config_ver 用于比对拉取最新配置 |
| `profile_get` | `user_id`(可选，默认自己) | `profile{user_id,nickname,avatar_url,is_vip,vip_expiry,stone_balance,stamina,invite_code}` | 游客/公开场景不返回敏感字段 |
| `profile_update` | `nickname`(可选), `avatar_url`(可选) | `profile` | 头像需先经 tomb.avatar_set 上传校验 |
| `checkin` | — | `award, stone_balance, streak` | UTC 自然日判定；连签第 7 天 +500，其余 +100；已签返回 `code:2002, data:{already:true}` |
| `invite` | `invite_code` | `award, stone_balance, left_years` | 14 天内首次占格才结算 +50,000；年上限 2 人（超过返回 `code:1005`） |
| `invite_list` | — | `invites[{invitee_id, status, settled_at}]` | 邀请记录与结算状态 |
| `deregister` | `reason(可选)` | `ok, anonymized{user_id, tomb_count}` | 注销：user 置 deleted_at，墓碑匿名化保留（清昵称/头像等个人标识，保留纪念数据）；7 天冷静期内可 `cancel_deregister`（TBD 默认支持）；幂等 sn |

## 2. mod = grid（Bot: idie-grid）

| act | 请求参数 | 应答 data | 说明 |
|---|---|---|---|
| `cells_get` | `x0,y0,x1,y1` | `cells[{x,y,status,owner_tomb_id,revision}], etag` | 完整区块、无游标；边界 clamp；同坐标请求可携带 `if_none_match=etag` 返回 304（`code:0,data:{unchanged:true}`） |
| `cell_status` | `x,y` | `status{state,owner_tomb_id,expire_at,caretakers:[...]}` | 单格详情（含授权人） |
| `cell_occupy` | `x,y,years,renovate(可选)` | `placement{cell_id,expire_at,status}, balance, left_years` | 扣灵石 K×years（按促销折扣，见 system.config_get）；原墓碑自动进入未展示（docs/03 状态机）；24h 占格 ≤2 次（TBD 默认 2）；同坐标幂等（sn） |
| `cell_renew` | `cell_id,years` | `placement{expire_at}, balance` | 仅 OCCUPYING/GRACE 可续；幂等键 renew:{user_id}:{cell_id}:{sn} |
| `cell_re_place` | `tomb_id,x,y` | `placement{cell_id,expire_at}` | 墓碑重新放置到指定空/己方格子；原格子释放（若己方） |
| `cell_release` | `cell_id` | `placement{status:'RELEASED'}` | 主动释放（非 destroy；墓碑回未展示） |
| `caretaker_add` / `caretaker_remove` | `cell_id,user_id` | `caretakers` | 代续费授权 ≤5 人 |
| `deep_link` | `x,y` | `cell_status + tomb_summary` | 深链解析（未登录返回公开摘要，游客模式） |

## 3. mod = tomb（Bot: idie-tomb）

| act | 请求参数 | 应答 data | 说明 |
|---|---|---|---|
| `tomb_create` | `name, born_year, death_year, style_id, text_align(可选), avatar_url(可选), epitaph(可选)` | `tomb{id,revision}` | 校验：姓名 ≤8 字（TBD 默认）、生/离世年份合法、style_id 在样式表内；头像 ≤5MB 入抽查队列 |
| `tomb_update` | `tomb_id, 上述字段(可选)` | `tomb{id,revision}` | revision+1 触发 MQ 缩略图重生成（60s 生效） |
| `tomb_get` | `tomb_id, include_private(可选)` | `tomb{id,name,born_year,death_year,avatar_url,style_id,text_align,epitaph,revision,placement{cell_id,x,y,status},memorial{flower_count,heart_count},can_edit}` | 公开字段仅渲染参数（不下发成品图）；can_edit 按 owner/授权 |
| `avatar_set` | `tomb_id, avatar_url` | `tomb{revision}` | 走 DIM FileContent 上传后回调校验（≤5MB、格式白名单） |
| `visibility_set` | `tomb_id, visible(可选), epitaph_visible(可选)` | `tomb{revision}` | 墓碑可隐藏（保留数据） |
| `remember_set` | `tomb_id, enabled` | `ok` | 纪念日提醒订阅（VIP 权益，本人视角本地推送） |

## 4. mod = economy（Bot: idie-economy）

| act | 请求参数 | 应答 data | 说明 |
|---|---|---|---|
| `balance_get` | — | `stone_balance, pending{...}` | 余额 + 待结算（回馈即时入账，无 pending 一般） |
| `ledger_list` | `cursor, limit(≤50)` | `items[{ts,type,amount,balance_after,ref}], next_cursor` | 只增不改，时间倒序 |
| `packages_get` | `platform(ios/android)` | `packages[{id,amount,original_amount,final_amount,discount_label,vip_only}], first_charge{eligible}` | 成品价由后端统一计算下发（大促 5 折 / 普通 6 折 / 正常 8.3 折 / 节假日 7 折 / 首充 0.99 美元专用包；乘法叠加 + 2.5 折下限） |
| `purchase` | `package_id, receipt(苹果/谷歌回执)` | `order_id, stone_balance, vip_expiry, first_charge_granted` | **IAP 回调为唯一入账锚**：先验回执（服务端 IAP 校验）再入账；幂等 first_charge:{user_id} + order 幂等键 |
| `first_charge` | `receipt` | `order_id, grants{vip_1m, cell_6m_voucher, stone_5000}` | 首充原子发货：1 个月 VIP + 1 格 6 个月券 + 5,000 灵石；重复调用返回已发货 |
| `voucher_apply` | `voucher_id, cell_id` | `placement` | 6 个月占格券核销 |
| `vip_status` | — | `is_vip, vip_expiry, stamina_max, styles_unlocked` | VIP 权益实时判定（过期回落） |

## 5. mod = interact（Bot: idie-interact）

| act | 请求参数 | 应答 data | 说明 |
|---|---|---|---|
| `flower_send` | `tomb_id, count(1..3, TBD)` | `ok, stone_balance, tomb_flower_count` | 扣 1,000 灵石/束；同墓碑 24h ≤3、全站日 ≤20（TBD）；被献者回馈 30% 即时入账（type=flower_rebate） |
| `heart_send` | `tomb_id` | `ok, stamina_left, tomb_heart_count, reward_stone` | 免费，扣 1 点体力；回馈 10 灵石/次即时入账；单日献同墓碑 ≤5 次（TBD）、全站日 >150 触发风控（code:1006） |
| `interact_list` | `tomb_id, cursor` | `items[{type,ts,count}], next_cursor` | 互动记录（去重展示） |
| `my_contributions` | `cursor` | `items[{tomb_id,type,total,last_ts}]` | 我的献花/献爱心汇总 |

## 6. mod = govern（Bot: idie-govern）

| act | 请求参数 | 应答 data | 说明 |
|---|---|---|---|
| `report_submit` | `tomb_id, reason, detail(可选)` | `report{id,status:'PENDING'}` | 状态机 pending→processing→resolved|rejected；或先下架后申诉（48h 窗口） |
| `report_status` | `report_id` | `report{status,result,appeal_deadline}` | 申诉期可 `appeal_submit` |
| `appeal_submit` | `report_id, statement` | `report{status}` | 48h 内人工复核 |
| `risk_ping` | `tomb_id` | `risk{level,hit_rules}` | 客户端内容预检（本地 OCR/AI 初筛调用） |

## 7. mod = system（Bot: idie-system）

| act | 请求参数 | 应答 data | 说明 |
|---|---|---|---|
| `config_get` | `ver` | `config{ver, bounds, prices, discounts, stamina, vip, rates, limits}, changed` | 全量配置版本化；ver 一致返回 `changed:false`；60s 热更新生效 |
| `notice_get` | — | `notices[{id,title,body,ts}]` | 公告 |
| `config_update`(admin) | `key,value,ver` | `config_ver` | 运营后台调用（bot 内部白名单） |

## 8. 关键事务与一致性（联调红线）

1. **占格**：`cell_occupy` 与 `cell_renew` 走「灵石扣减 + Placement 条件更新」原子事务（`UPDATE placement SET ... WHERE cell_id=? AND status='EMPTY'` 受影响行=1 才提交；失败返回 code:1002）。灵石扣减用 Redis 原子 CAS（余额 key），与 DB 流水对账。
2. **体力**：Redis key `stamina:{user_id}` 原子扣减（CAS），VIP 过期实时判定回落 30；恢复进度存 `stamina_progress`。
3. **回馈入账**：献花/献爱心回馈即时入账（写 Ledger），不异步。
4. **幂等键清单**：占格/续费 `renew:{user}:{cell}:{sn}`、首充 `first_charge:{user}`、充值 `order:{order_id}`、邀请结算 `invite:{invitee}`（14 天窗口）。
5. **配置一致性**：所有价格/折扣/阈值只信 `system.config_get` 下发值；客户端不落任何定价常量。

## 9. 待确认（不阻塞联调）

- 次级参数拍板：献花频率（同墓碑 24h≤3 / 全站日≤20）、献爱心同墓碑 24h≤5、占格 24h≤2、姓名 ≤8 字、抽查默认 20% 分层、举报 24h 首响。
- Bot ID 域名（`idie-<mod>@<domain>` 的 domain 取值，建议与现有 sebot 一致）。
- 头像上传走 DIM FileContent 的具体接收/存储（Toby 落实现）。
