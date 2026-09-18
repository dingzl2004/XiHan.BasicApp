# 零售模块边界与职责

本文把连锁零售 ERP 拆成可独立演进的工程边界。命名可微调，但 **覆盖面不能少**：主档、库存、统配、收银服务端、加盟、离线同步、Saas 底座、WPF 客户端、Vue 管理端。

总图与约束见 [总架构](./architecture)。POS 进程与协议见 [WPF POS](./pos-wpf)。

## 工程落点总表

| 逻辑模块 | 后端工程 | 前端（Vue） | 说明 |
| --- | --- | --- | --- |
| Retail.MasterData | `backend/src/modules/XiHan.BasicApp.Retail.MasterData` | `frontend/src/views/retail/master-data/` · `frontend/src/api/modules/retail/master-data/` | 商品主档 |
| Retail.Inventory | `...Retail.Inventory` | `views/retail/inventory/` · `api/modules/retail/inventory/` | 库存账 |
| Retail.Distribution | `...Retail.Distribution` | `views/retail/distribution/` · `api/modules/retail/distribution/` | 统配 |
| Retail.Pos | `...Retail.Pos` | `views/retail/pos/`（**查询/日结/班次**，不是收银界面） · `api/modules/retail/pos/` | 收银服务端 |
| Retail.Franchise | `...Retail.Franchise` | `views/retail/franchise/` · `api/modules/retail/franchise/` | 加盟 |
| Retail.Sync | `...Retail.Sync` | `views/retail/sync/`（设备、水位、失败队列） · `api/modules/retail/sync/` | 离线同步 |
| Saas（现有） | `XiHan.BasicApp.Saas` | 现有 `identity/` `tenant/` `setting/` | 账号权限组织租户审计 |
| WPF POS | **不在 backend/modules** | **不在 frontend**；见下方 `clients/pos-wpf/` | 收银桌面端 |

权限码前缀建议：`rtl:`，资源段按模块划分（`rtl:sku:read`、`rtl:requisition:submit`）。种子 `Order` 使用 **400–499**，与 Saas/AI/Workflow 错开。Dynamic API Group 建议 `BasicApp.Retail`，Tag 用模块名。

模块内部仍按 Domain / Application / Infrastructure 分层，仓储继承 `SaasRepository<T>`，写侧 AppService、读侧 QueryService。照 [二次开发 · 配方 B](../backend/development) 与 `XiHan.BasicApp.Workflow` 的接线方式做。

### 模块依赖（单向）

```text
                    ┌─────────────┐
                    │ Retail.Sync │  ← 可引用 Pos / MasterData 契约
                    └──────▲──────┘
         ┌─────────────────┼─────────────────┐
         │                 │                 │
  Retail.Pos        Retail.Distribution   Retail.Franchise
         │                 │                 │
         └────────┬────────┴────────┬────────┘
                  ▼                 ▼
           Retail.Inventory   Retail.MasterData
                  └────────┬────────┘
                           ▼
                         Saas
```

禁止环：Inventory 不得引用 Pos；MasterData 不得引用 Distribution。跨模块副作用用领域事件（例如 `DispatchPosted` → Inventory 记账 → Franchise 记应付）。

可选：再做一个空的聚合模块 `XiHan.BasicApp.Retail` 只 `[DependsOn]` 上述六个，供 `WebHost` 一行引用。不要把业务实体塞进这个空壳。

---

## WPF 客户端目录：`clients/pos-wpf/`

**选定目录：`clients/pos-wpf/`**（不要用仓库根上的 `pos-wpf/`，也不要放进 `frontend/` 或 `backend/src`）。

理由：

1. 现有根目录已经按运行时切开：`backend/`、`frontend/`、`docs/`。收银是第三类运行时（桌面），用 `clients/` 归类，以后若有手持盘点机、自助收银，可以并排放 `clients/handheld/`。
2. WPF 的 csproj、WiX/MSIX 安装包、驱动依赖与 Vue/Vite 工具链冲突；塞进 `frontend/` 会污染 pnpm 与文档站构建。
3. 放进 `backend/src/modules` 会让 WebHost 解决方案误引用 WinForms/WPF 目标框架，CI 的 `dotnet build backend/XiHan.BasicApp.slnx` 会变重、也容易把桌面程序当服务端发布。
4. `backend/src/business` 的 ERP 脚手架是空壳且未按零售拆模块，**不要**在那里堆 POS。

建议解决方案：`clients/pos-wpf/XiHan.BasicApp.Pos.slnx`（独立 slnx）。与后端的耦合仅限 **HTTP 契约**（DTO 可共享一个 `Retail.Contracts` 类库，或 POS 自己维护 client types；第一期允许重复 DTO，禁止 POS 直接引用 SqlSugar 实体项目）。

内部项目切分见 [WPF POS · 进程架构](./pos-wpf#进程架构)。

---

## Vue 管理端：`frontend` 的 retail 区

管理后台继续用现有 Vue 3 壳（Naive UI、Schema 列表、动态路由）。

```text
frontend/src/views/retail/
  master-data/product/index.vue
  master-data/sku/index.vue
  master-data/barcode/index.vue
  master-data/price-list/index.vue
  master-data/store/index.vue
  master-data/supplier/index.vue
  inventory/stock/index.vue
  inventory/lock/index.vue
  inventory/count/index.vue
  inventory/transfer/index.vue
  distribution/requisition/index.vue
  distribution/allocation/index.vue
  distribution/dispatch/index.vue
  distribution/receipt/index.vue
  pos/ticket/index.vue          # 小票查询，不是收银
  pos/shift/index.vue
  franchise/contract/index.vue
  franchise/settlement/index.vue
  sync/device/index.vue
  sync/queue/index.vue

frontend/src/api/modules/retail/
  master-data/*.ts
  inventory/*.ts
  ...
```

约定：

- `PageDescriptor.Component` = `retail/.../index`，与视图路径一致。
- 菜单 i18n 键加到 `frontend/packages/locales/langs/*/menu.ts`。
- **不要**在 Vue 里做购物车、开钱箱、听扫码枪；那些只属于 WPF。
- 门店人员用浏览器看要货/库存/日结即可；收银员不发 Vue 菜单。

---

## Retail.MasterData

**后端**：`XiHan.BasicApp.Retail.MasterData`  
**前端**：`views/retail/master-data/` · `api/modules/retail/master-data/`  
**权限示例**：`rtl:product:*` `rtl:sku:*` `rtl:barcode:*` `rtl:price-list:*` `rtl:store:*` `rtl:supplier:*`

### 职责

- 商品 SPU / SKU、计量单位、条码（一品多码）、销售方式（计件/计重）。
- 价目：标准零售价、门店覆盖价、（预留）配货价目；价格生效区间。
- 门店 `RtlStore`：编码、组织部门、直营/加盟、营业状态、默认仓。
- 仓库 `RtlWarehouse`：总仓/区域仓/门店仓，归属门店或总部物流部门。
- 供应商主数据（统配采购入库会用到；MVP 可只维护、不跑完整采购）。
- 向 POS 提供 **增量主档包**（SKU/条码/价目版本号），供 Sync 下发。

### 不负责

- 库存数量、锁定、盘点（Inventory）
- 要货/配货单据（Distribution）
- 小票与支付（Pos）
- 加盟合同金额（Franchise）
- 用户账号与角色（Saas）

### 主要实体 / 单据

| 实体 | 表（建议） | 说明 |
| --- | --- | --- |
| `RtlProduct` | `Rtl_Product` | SPU：名称、类目、品牌、状态 |
| `RtlSku` | `Rtl_Sku` | SKU：规格、单位、`SaleMode`、标准零售价、箱规 |
| `RtlBarcode` | `Rtl_Barcode` | 条码 → SKU，租户内条码唯一 |
| `RtlPriceList` / `RtlPriceListItem` | `Rtl_Price_List(_Item)` | 价目头行；维度：租户/门店/渠道（渠道 MVP 可只用 Default） |
| `RtlStore` | `Rtl_Store` | 门店；`DepartmentId`、`StoreMode`、`FranchiseeId` |
| `RtlWarehouse` | `Rtl_Warehouse` | 仓；`WarehouseType`、`StoreId` 可空（总仓） |
| `RtlSupplier` | `Rtl_Supplier` | 供应商 |
| `RtlMasterdataVersion` | `Rtl_Masterdata_Version` | 按租户+门店的主档版本水位，给 POS 增量 |

不是单据模块：没有「审核流」也可先用 `Status` 启停。价格变更建议记 `RtlPriceChangeLog`（流水，创建型实体）。

### 与谁集成

- **Saas**：门店挂 `SysDepartment`；启停可写操作日志
- **Inventory**：仓创建后由 Inventory 建零库存行（或首次入账时懒创建）
- **Pos / Sync**：下发 SKU 快照；POS 不得反向改主档
- **Distribution**：要货行校验 SKU 可订、箱规
- **Franchise**：加盟店必须已有 `FranchiseeId`（外键语义在领域层校验，不建跨模块 SQL FK 亦可）

---

## Retail.Inventory

**后端**：`XiHan.BasicApp.Retail.Inventory`  
**前端**：`views/retail/inventory/` · `api/modules/retail/inventory/`  
**权限示例**：`rtl:stock:read` `rtl:stock-lock:*` `rtl:stock-count:*` `rtl:stock-transfer:*`

### 职责

- 多仓库存：账面、可用、锁定、在途（在途数量可由 Distribution 回写或本模块维护 `InTransitQty`）。
- 库存流水（只增）：来源单据类型 + 单据 Id + 变动量。
- 锁定：配货占用、（可选）要货占用；过账释放。
- 盘点：盘点单 → 实盘 → 盘盈盘亏过账。
- 调拨：仓间/店间；第一期可要求总部审批。
- 库存校验 API：给 Pos 扣减、给 Distribution 出库；**服务端是库存对错的唯一裁判**。

### 不负责

- 定义 SKU/门店/仓（MasterData）
- 要货/配货状态机（Distribution 调本模块记账）
- 收银界面与支付（Pos 调本模块 `IssueForSale`）
- 加盟对账（可发领域事件，金额不在本模块算）
- 允许业务员在 SQL 里直接 `UPDATE` 库存列

### 主要实体 / 单据

| 实体 | 表 | 说明 |
| --- | --- | --- |
| `RtlStock` | `Rtl_Stock` | `(WarehouseId, SkuId)` 当前量：OnHand / Locked / Available 派生 |
| `RtlStockLedger` | `Rtl_Stock_Ledger_{yyyyMM01}` | 分月流水；`BasicAppCreationEntity` |
| `RtlStockLock` | `Rtl_Stock_Lock` | 锁定单/锁定行，关联配货单 |
| `RtlStockCount` / `Line` | `Rtl_Stock_Count(_Line)` | 盘点 |
| `RtlStockTransfer` / `Line` | `Rtl_Stock_Transfer(_Line)` | 调拨 |

建议量字段用 `decimal(18,4)`，计件 SKU 也走 decimal，避免称重切换时改列。

负库存：由仓或门店策略开关控制（待确认 P2），默认直营 `AllowNegative=false`。POS 离线超卖不在本模块「先改账面」，而在上线同步时走 `IssueForSale` 的失败/异常单路径。

### 与谁集成

- **MasterData**：校验仓、SKU 存在且启用
- **Distribution**：锁 / 出库 / 入库
- **Pos**：销售出库、退货入库（原单行）
- **Sync**：不直接给 POS 写 `RtlStock`；同步服务调本模块应用服务
- **Saas**：盘点过账权限、操作日志

---

## Retail.Distribution

**后端**：`XiHan.BasicApp.Retail.Distribution`  
**前端**：`views/retail/distribution/` · `api/modules/retail/distribution/`  
**权限示例**：`rtl:requisition:*` `rtl:allocation:*` `rtl:dispatch:*` `rtl:receipt:*`

### 职责

- 统配主链路：要货 → 配货 → 出库 → 在途 → 签收入库。
- 配额、箱规向上取整、缺货关行。
- 差异：实收 < 在途时生成差异行（丢失 / 拒收 / 待查）。
- 给加盟结算提供「已签收配货行」事件（含配货价快照）。

### 不负责

- SKU 与价目维护
- 库存数值存储（调用 Inventory）
- 门店零售开单
- 计算加盟账期利息、保证金（Franchise）
- 承运商 GPS 轨迹（可用备注字段，不做物流系统）

### 主要实体 / 单据

| 实体 | 表 | 说明 |
| --- | --- | --- |
| `RtlRequisition` / `Line` | `Rtl_Requisition(_Line)` | 要货 |
| `RtlAllocation` / `Line` | `Rtl_Allocation(_Line)` | 配货；可来自多张要货 |
| `RtlDispatch` / `Line` | `Rtl_Dispatch(_Line)` | 出库发运 |
| `RtlInTransit` | `Rtl_In_Transit` | 在途聚合（也可作为出库单状态，不必强拆） |
| `RtlReceipt` / `Line` | `Rtl_Receipt(_Line)` | 签收 |
| `RtlReceiptDiff` | `Rtl_Receipt_Diff` | 差异 |

状态机见 [总架构 · 统配主链路](./architecture#统配主链路)。

签收：**建议在线**。WPF 可以扫到货单号后调 API，不在本地库「先入账再补传」。若产品坚持离线签收，必须按库存冲突策略做成「本地暂存 + 服务端过账」，且可能产生负在途，复杂度放到第 ④ 期评估。

### 与谁集成

- **MasterData**：SKU、箱规、门店默认配货仓
- **Inventory**：锁定、出库、入库
- **Franchise**：`ReceiptPosted` 事件
- **Workflow（可选）**：超大要货审批
- **Printing**：Vue 打配货单/拣货单
- **Pos**：不直接依赖；门店到货不是收银流程

---

## Retail.Pos（服务端 API）

**后端**：`XiHan.BasicApp.Retail.Pos`  
**前端**：`views/retail/pos/` 仅管理查询（小票、班次、日结审核）  
**权限示例**：`rtl:ticket:create` `rtl:ticket:refund` `rtl:shift:close` `rtl:ticket:read`

这是 **收银领域的服务端**，不是 WPF 工程。WPF 通过 Dynamic API 调用本模块（及 Sync）。

### 职责

- 销售订单/小票：头、行、支付行、优惠行（MVP 优惠可空）。
- 支付登记：现金、扫码（在线）、其它；一票多支付。
- 退货：默认原单本店；改库存走 Inventory。
- 班次 / 日结：开班、缴款、长短款、关班。
- 幂等收单：按 `IdempotencyKey` 接受 POS 补传。
- 成交后写销售流水快照（价格、收银员、设备、会员号可选）。

### 不负责

- WPF UI、外设驱动、本机 SQLite
- 主档编辑、统配签收过账（签收属 Distribution）
- 加盟对账单
- 把「购物车」持久化到中心库（未成交购物车只在 POS 本地）

### 主要实体 / 单据

| 实体 | 表 | 说明 |
| --- | --- | --- |
| `RtlPosTicket` | `Rtl_Pos_Ticket_{yyyyMM01}` | 小票头；月分表；`LocalTicketNo`、`IdempotencyKey`、`DeviceId` |
| `RtlPosTicketLine` | `Rtl_Pos_Ticket_Line_{yyyyMM01}` | 行：SKU、数量、价目快照、实收价 |
| `RtlPosPayment` | `Rtl_Pos_Payment_{yyyyMM01}` | 支付行 |
| `RtlPosShift` | `Rtl_Pos_Shift` | 班次 |
| `RtlPosShiftCash` | `Rtl_Pos_Shift_Cash` | 日结缴款 |
| `RtlPosException` | `Rtl_Pos_Exception` | 价差/超卖/缺 SKU 等异常单，供总部处理 |

支付成功单：**中心库不得因库存失败而丢单**。库存校验失败 → 小票仍落 `PostedWithException`，出异常单，由总部补盘或改策略（见 Sync 冲突）。

### 与谁集成

- **MasterData**：校验 SKU/条码/价格版本；允许「快照成交」
- **Inventory**：`IssueForSale` / `ReceiptFromReturn`
- **Sync**：补传入口可放 Sync 再转调 Pos，避免 POS 模块依赖设备表细节
- **Saas**：收银员 `SysUser`、编号规则、审计
- **WPF**：唯一前台客户端

---

## Retail.Franchise

**后端**：`XiHan.BasicApp.Retail.Franchise`  
**前端**：`views/retail/franchise/` · `api/modules/retail/franchise/`  
**权限示例**：`rtl:franchisee:*` `rtl:franchise-contract:*` `rtl:settlement:*`

第 ③ 期才做满；第 ① 期至少要有 `RtlFranchisee` 与门店上的 `FranchiseeId`，否则直营/加盟枚举会悬空。

### 职责

- 加盟商档案、合同（期限、配货折扣/价目、是否允许改零售价、结算周期、是否允许负库存）。
- 费用项：加盟费、保证金、管理费（台账；自动计提规则待 P3）。
- 结算：按已签收配货行汇总应付、按周期出对账单、确认/付款登记。
- 限制加盟商账号的数据范围（与 Saas 数据范围配合）。

### 不负责

- 物流状态机（Distribution）
- 收银
- 把加盟商做成独立租户（默认不做，见假设 A2）
- 总账凭证导出以外的财务系统替代

### 主要实体 / 单据

| 实体 | 表 | 说明 |
| --- | --- | --- |
| `RtlFranchisee` | `Rtl_Franchisee` | 加盟商 |
| `RtlFranchiseContract` | `Rtl_Franchise_Contract` | 合同；可挂文件模块附件 |
| `RtlFranchiseFee` | `Rtl_Franchise_Fee` | 费用台账 |
| `RtlSettlement` / `Line` | `Rtl_Settlement(_Line)` | 对账单 |
| `RtlSettlementPayment` | `Rtl_Settlement_Payment` | 付款登记 |

### 与谁集成

- **MasterData**：加盟店绑定
- **Distribution**：签收事件 → 结算明细
- **Saas**：加盟商管理员用户、可选 Workflow 合同审批
- **Inventory**：只读该加盟门店仓；不直接改账

---

## Retail.Sync

**后端**：`XiHan.BasicApp.Retail.Sync`  
**前端**：`views/retail/sync/`（设备注册、水位、失败队列、强制全量）  
**权限示例**：`rtl:pos-device:*` `rtl:sync:replay` `rtl:sync:read`

协议细节见 [WPF POS](./pos-wpf)。本模块是 **服务端同步与设备身份**。

### 职责

- 设备注册、激活、吊销、密钥轮换。
- 下发主档增量（版本向量 / 水位）。
- 接收离线队列：小票、退货、交班；幂等。
- 冲突检测与异常单转交 Pos/Inventory。
- 记录每台设备的最后成功水位、失败原因、补传次数。

### 不负责

- 本机 SQLite schema（那是 WPF 的 `LocalData` 项目）
- 外设驱动
- 改价、改 SKU 属性（转 MasterData）
- 在 Sync 里手写一套库存账

### 主要实体 / 单据

| 实体 | 表 | 说明 |
| --- | --- | --- |
| `RtlPosDevice` | `Rtl_Pos_Device` | 设备；`StoreId`、状态、证书指纹/密钥哈希 |
| `RtlPosDeviceSecret` | `Rtl_Pos_Device_Secret` | 密钥材料；加密存储（Data Protection），不进查询 DTO |
| `RtlSyncWatermark` | `Rtl_Sync_Watermark` | 设备 × 数据集（MasterData / Tickets）水位 |
| `RtlSyncInbox` | `Rtl_Sync_Inbox` | 服务端收到的原始包（可选，便于重放）；注意体积，可按月分表 |
| `RtlSyncConflict` | `Rtl_Sync_Conflict` | 冲突记录 |

### 与谁集成

- **Pos**：提交已幂等的小票
- **MasterData**：拉版本包
- **Inventory**：仅通过 Pos 过账间接发生
- **Saas**：设备操作审计；不要用 `SysUserApiCredential` 冒充门店设备（那是开放 API 个人凭证）。设备是门店资产，单独表。

---

## 现有 Saas（继续做底座）

**后端**：`XiHan.BasicApp.Saas`（以及 Printing / Chat / Workflow 等可选模块）  
**前端**：现有 identity / tenant / setting / log …

### 职责（零售场景下）

- 账号、登录、JWT、会话、2FA
- 角色、权限码登记、数据范围、字段脱敏、ABAC
- `SysDepartment` 组织树与闭包
- `SysTenant` 多租户、Edition 功能门控
- 操作/登录/实体变更审计
- 编号规则、字典、文件、调度、消息

### 不负责（禁止把零售塞进来）

- 小票、要货、配货、库存账、价目、POS 设备、同步水位、加盟合同
- 在 `SysUser` 上增加 `CashDrawerId` 之类收银列——用扩展表或 `RtlPosShift`

零售模块通过 `[DependsOn(Saas)]` 复用仓储基类与权限基础设施。需要新权限时在 **零售模块自己的** `Domain/Permissions` 定义 `rtl:*`，种子登记到统一权限表，而不是改 `SaasPermissionCodes` 大杂烩（Saas 码只保留 `saas:*`）。

Printing / Chat / AI / CodeGeneration / Workflow：**按需依赖，不是零售主路径**。配货单打印可依赖 Printing；要货审批可依赖 Workflow。不要为了收银去改 Chat。

---

## 跨模块「一张小票」怎么走

便于对照边界是否被打破：

```text
收银员扫码
  WPF Peripherals → 本地 SKU 快照（MasterData 下发的副本）
  成交支付
  WPF LocalData 写离线单 + 同步队列
  在线或稍后：
    Sync API（设备+用户令牌）
      → Pos.CreateTicket(idempotencyKey)     -- 落小票
      → Inventory.IssueForSale               -- 扣门店仓
      → 失败则 Pos 记 Exception，票仍在
  Vue 总部
      → PosQuery 看票
      → InventoryQuery 看库存
      → 不能从 Vue 点「开钱箱」
```

若发现某次需求必须同时改 Saas 实体和 `RtlPosTicket` 才能完成，先停下来：多半是把收银状态误放到了用户表。
