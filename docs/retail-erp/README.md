# 连锁零售 ERP

本文档描述如何在 **现有 XiHan.BasicApp 前后端分离 monorepo** 上扩展一套连锁零售 ERP，而不是另起一套系统。Saas（身份、RBAC、组织、审计、多租户）继续作为底座；零售能力以独立业务模块接入 `WebHost`。

**当前状态：文档先行，业务代码尚未落地。** 不要把 `backend/src/business` 里空的 ERP 脚手架当成已交付能力。

## 硬约束（写进设计，落地时不可改口）

| 项 | 约定 |
| --- | --- |
| 底座 | 复用 XiHan.BasicApp，不推翻 Saas / 权限 / 多租户 |
| 业态 | 连锁零售 ERP；约 **千家门店**；**直营 + 加盟**；**总部统配**；**离线收银** |
| 数据库 | **SQL Server 2019**（中心库）。SqlSugar 已支持 `DbType: SqlServer` |
| 收银前端 | **WPF 桌面客户端**，不是 Web POS；对接扫码枪、小票打印机、钱箱、客显、电子秤等外设 |
| 管理端 | 继续用现有 **Vue 3** 前端（总部 / 区域 / 门店管理后台） |
| 仓库形态 | `backend`（.NET 10，模块：Saas、CodeGeneration、AI、Workflow、Printing、Chat）+ `frontend`（Vue3）+ `docs`（VitePress）+ `AGENTS.md` / `.ai/skills` |

## 文档索引

| 文档 | 内容 |
| --- | --- |
| [总架构](./architecture) | 目标与非目标、逻辑架构、与 BasicApp 的关系、门店模型、数据与部署、统配主链路、安全、分期 |
| [模块边界](./modules) | 每个零售模块的职责、不负责什么、实体/单据、集成关系、工程落点 |
| [WPF POS](./pos-wpf) | 为何 WPF、进程架构、本地库、离线边界、同步协议、外设抽象、终端认证 |

阅读顺序：先 [总架构](./architecture) 对齐边界，再 [模块边界](./modules) 拆工程，收银与同步细节看 [WPF POS](./pos-wpf)。

## 分期路线（摘要）

细节与每期验收口径见 [总架构 · 分期](./architecture#分期)。

```text
MVP          主档 + 直营库存 + 在线收银 API + Vue 管理端最小闭环
   ↓
统配         要货 → 配货 → 出库 → 在途 → 签收入库
   ↓
加盟结算     合同 / 费用 / 对账；直营与加盟库存口径分开
   ↓
离线 POS 强化  WPF 本地库、断网收银、补传、冲突、外设适配
   ↓
规模化       千店索引/分表、汇总表、读写分离、设备与同步治理
```

| 期 | 必须能上线的 | 明确不做（留给后期） |
| --- | --- | --- |
| **MVP** | 租户内门店/仓/SKU/条码/价目；直营店进销存；在线 POS 开单/支付/退货/日结 API；总部 Vue 查询 | 加盟结算、完整离线、电商、会员、门店自采 |
| **统配** | 要货单、配货单、配送出库、在途、门店签收入库；区域仓可选 | 越库、第三方物流轨迹、智能补货算法 |
| **加盟结算** | 加盟合同、费用项、配货结算、对账单 | 复杂股权/多品牌加盟商独立租户（见待确认） |
| **离线 POS 强化** | WPF 客户端、SQLite、设备注册、幂等补传、外设适配层 | 全品类称重溯源、双机热备 |
| **规模化** | 流水按月分表、门店日汇总、只读副本、同步水位监控 | 默认一店一库（仅作为可选演进） |

## 仓库落点（规划，尚未建工程）

```text
XiHan.BasicApp/
├── backend/src/modules/          # 新增 XiHan.BasicApp.Retail.* 一等模块
├── frontend/src/views/retail/    # Vue 管理端页面（不是收银界面）
├── frontend/src/api/modules/retail/
├── clients/pos-wpf/              # WPF 收银客户端（独立目录，理由见模块文档）
└── docs/retail-erp/              # 本目录
```

## 相关内核文档

扩展零售模块时仍遵守现有约定，不要另搞一套分层：

- [目录结构与代码地图](../project-structure)
- [后端二次开发](../backend/development)（独立模块走「配方 B」）
- [多租户](../backend/multi-tenancy) · [组织架构](../backend/organization) · [数据权限](../backend/data-permission)
- [数据库配置](../backend/database)（SQL Server 连接、读写分离）
- [实体基类](../backend/entity) · [前端开发](../frontend/development)
