# Fiserv 商户入驻（Merchant Boarding）流程 — 技术调研总结

> 编制日期：2026-03-19 | 基于 Fiserv 官方文档及公开资料整理
>
> ⚠️ 本文档所有信息均标注了来源。标记为「官方确认」的信息来自 Fiserv/CardPointe 官方文档；标记为「第三方参考」的信息来自 Infinicept 等合作伙伴公开文档；标记为「待确认」的信息需与 Fiserv 进一步核实。

---

## 1. Fiserv 北美处理平台体系

Fiserv（2019 年收购 First Data）在北美运营多个后端处理平台：

| 平台 | 说明 | 新商户入驻 | 技术支持电话 |
|------|------|-----------|-------------|
| **First Data North** (Cardnet North) | 主力平台，所有新商户统一入驻到此 | ✅ | 1-800-542-1894 |
| **Nashville** (Rapid Connect Nashville) | Fiserv 内部层级管理平台 | — 非独立入驻目标 | 1-800-542-1894 |
| **Omaha** | 旧平台，仅维护存量商户，不接受新入驻 | ❌ | 1-800-828-0210 |

> 📖 来源（官方确认）：CoPilot & CardPointe 过渡文档 — "Net new accounts will be boarded to the FirstData North platform exclusively." "There are currently no plans to allow boarding to the Omaha backend through CoPilot for Retail ISO."
> https://support.cardpointe.com/additional-pages/transitioning-to-copilot---cardpointe/

---

## 2. CoPilot — Fiserv 北美唯一的 ISO 商户入驻门户

### 2.1 CoPilot 是什么

CoPilot 是 Fiserv 面向 Retail ISO/代理商的**Web 门户**（portfolio management portal），是 AccessOne 等旧版入驻工具的统一替代品。

核心功能：
- 商户入驻（Boarding）— 创建新账户、提交 MPA、核保审批
- 设备订购与 TID 配置
- 佣金/残值（Residuals）查看
- 商户组合管理与绩效监控
- Ticket 工单系统（账户维护、费率变更等）

> 📖 来源（官方确认）：同上 CoPilot & CardPointe 过渡文档

### 2.2 CoPilot 没有公开的 REST Boarding API

**关键结论：CoPilot 不提供面向第三方集成商的公开 REST API 用于程序化提交商户入驻申请。**

CoPilot 内部使用 Bearer Token (JWT) 认证的 API 供其 Web 前端调用，但这不是面向外部开发者的公开接口。Fiserv 官方文档中没有任何 CoPilot Boarding API 的端点、参数或认证方式的公开说明。

> 📖 来源（官方确认）：遍历 support.cardpointe.com 和 developer.fiserv.com 均未找到 CoPilot REST API 文档。CoPilot 官方页面仅描述 Web 界面操作流程。

---

## 3. CoPilot 商户入驻完整流程

### 3.1 流程概览

```
ISO 在 CoPilot 创建新账户
        │
        ▼
选择 Application Template（预设定价/设备/默认字段）
        │
        ▼
填写商户信息（可预填银行信息，GIACT + Plaid 在线验证）
        │
        ▼
发送 MPA 给商户数字签名
  ├── 数字签名：商户需注册 CardPointe 账户，通过浏览器完成
  └── 纸质签名：下载 PDF，签署后上传至 CoPilot（无文件大小限制）
        │
        ▼
提交核保审批（Underwriting）
  ├── 自动审批：符合条件的申请通常数分钟内完成
  └── 人工审批：不符合自动审批条件的，通常 24 小时内完成
      （ISO 收到邮件通知 + CoPilot ticket 跟踪）
        │
        ▼
审批通过 → 分配 MID → 商户激活
```

> 📖 来源（官方确认）：CoPilot & CardPointe 过渡文档 — Application, Boarding, and Underwriting Questions 章节

### 3.2 Application Template（申请模板）

- 模板可预设定价、设备、字段默认值，减少重复填写
- ISO 可在提交前覆盖模板中的任何默认值
- 模板可随时编辑，无需审批流程即可生效
- 已使用过的模板也可以更新（AccessOne 不支持此功能）
- Standard 用户无权创建模板，需 Admin 或 Super Admin 权限

> 📖 来源（官方确认）：同上

### 3.3 MPA（Merchant Processing Application）

- CoPilot 基于入驻时选择的字段**动态生成** MPA，与 CoPilot 内部结构对齐
- MPA 支持 ISO 品牌定制
- 旧系统（AccessOne）的 MPA 模板**无法迁移**到 CoPilot，需重新创建
- 数字签名要求商户注册 CardPointe 账户（不要求实际使用 CardPointe 服务）
- 注册邮件使用 CardPointe 品牌发送，**不支持自定义品牌**

> 📖 来源（官方确认）：同上 — "CoPilot generates a dynamic MPA based on the fields selected at the time of boarding"

### 3.4 银行信息验证

- ISO 可在发送 MPA 前预填银行信息
- 银行信息通过 **GIACT + Plaid** 在线数字验证
- 验证通过则**无需提供 voided check**

> 📖 来源（官方确认）：同上 — "this data is verified digitally in CoPilot using GIACT + Plaid"

### 3.5 核保审批（Underwriting）

| 类型 | 说明 | 时效 |
|------|------|------|
| 自动审批 | 基于多因素自动评估，无明确的硬性条件 | 通常数分钟 |
| 人工审批 | 不符合自动审批条件时转人工 | 通常 24 小时 |

- 人工审批时 ISO 收到邮件通知，包含 CoPilot ticket 链接
- ISO 可在 CoPilot Tickets 中跟踪状态、提交评论、上传补充文档
- 多地点商户（10+ 账户）可提交 Multi-Location Form 批量处理

> 📖 来源（官方确认）：同上 — "Many accounts qualify for automated underwriting review and approval, which typically results in a decision within a matter of minutes."

---

## 4. MID 体系

### 4.1 CardPointe Gateway 使用的 MID

CardPointe Gateway API 中的 `merchid` 参数对应 **First Data North 平台的 MID**，用于交易授权、清算和资金结算。

> 📖 来源（官方确认）：CardPointe Gateway API 文档
> https://developer.fiserv.com/product/CardPointe

### 4.2 Fiserv 内部多层 MID 体系

通过 Fiserv Exchange 入驻完成后，Fiserv 后端会生成多个不同的 MID：

| MID 字段 | 类型 | 说明 | 用途 |
|----------|------|------|------|
| `NorthMid` | Integer | Fiserv North 平台分配 | 交易处理 & 资金清算（DFM 对账） |
| `NashvilleMid` | Integer | Fiserv Nashville 平台分配 | Fiserv 内部层级管理 |
| `ApplicationMid` | Integer | 顶层商户 MID | 标识申请，不用于资金清算 |
| `OutletInternalMid` | Integer | Fiserv 入驻流程内部标识 | 联系 Fiserv 技术支持时使用 |
| `SubgroupMid` | Integer | ApplicationMid 下的子组 | 层级管理 |
| `MerchantReference` | Integer | Fiserv 门户唯一商户 ID | 在 Fiserv 门户中定位商户 |
| `ApplicationReference` | Integer | Fiserv 唯一申请 ID | 在 Fiserv 门户中定位申请 |
| `TerminalId1-4` | Integer | 终端标识号（TID） | 配置终端/网关/移动应用 |

示例 payload（入驻成功）：
```json
{
  "NorthMid": "288965235213",
  "NashvilleMid": "208219377883",
  "ApplicationMid": "600100000118606",
  "OutletInternalMid": "609900000550681",
  "SubgroupMid": "600200000062310",
  "MerchantReference": "5043913",
  "ApplicationReference": "555000110859",
  "FiservStatus": "BOARDING_COMPLETE",
  "TerminalId1": "7731987"
}
```

> 📖 来源（第三方参考）：Infinicept Fiserv Exchange Boarding Service Results Webhook 文档
> https://developer.infinicept.com/docs/fiserv-exchange-boardingserviceresults-webhook
>
> ⚠️ 注意：以上字段规范来自 Infinicept（Fiserv 预集成合作伙伴）的公开文档，不是 Fiserv 官方直接发布的 API 文档。字段名和结构可能因集成方式不同而有差异。

### 4.3 Nashville MID 的获取方式

**Nashville MID 不能通过任何公开 API 主动查询获取。** 它是 Fiserv 内部系统在商户入驻完成后自动生成的，获取途径：

| 途径 | 说明 |
|------|------|
| Fiserv Exchange Boarding Results 回调 | 仅限 Infinicept 等预集成合作伙伴，回调中包含 NashvilleMid 字段 |
| CoPilot Web 界面 | 入驻完成后在商户详情页查看（待确认具体字段展示） |
| Fiserv 运营通知 | Fiserv 通过邮件或内部系统通知 ISO |
| ClientLine / AccessOne | Fiserv 报告系统中可查看完整 MID 层级 |
| 联系 Fiserv 支持 | 提供 North MID，拨打 1-800-542-1894 查询 |

> 📖 来源（第三方参考 + 待确认）：综合 Infinicept 文档及第三方操作指南

### 4.4 对日常业务的影响

| 场景 | 需要哪个 MID |
|------|-------------|
| CardPointe Gateway 交易处理 | North MID（CardPointe `merchid`） |
| 资金清算对账（DFM） | North MID |
| CoPilot 商户管理 | North MID（CoPilot 界面可搜索） |
| Fiserv 内部技术支持 | Nashville MID 或 OutletInternalMid |

**结论：对于 ISO 的日常交易处理和资金清算，只需要 North MID。Nashville MID 仅在与 Fiserv 内部系统交互时需要。**

---

## 5. Fiserv Exchange — 预集成合作伙伴的 Boarding 通道

### 5.1 什么是 Fiserv Exchange

"Fiserv Exchange" 不是一个面向外部开发者的公开 API 产品。它是 Fiserv 内部的商户入驻处理平台，是已下线的 "Fiserv Marketplace" 的替代品。只有与 Fiserv 建立了预集成合作关系的平台（如 Infinicept）才能通过专用通道对接。

> 📖 来源（第三方参考）：Infinicept 文档 — "The Fiserv Exchange BatchedBoardingResults webhook payload replaces the Fiserv Marketplace BatchedBoardingResults webhook payload. This is in accordance with Fiserv's decision to sunset the Marketplace platform."
> https://developer.infinicept.com/docs/fiserv-exchange-boardingserviceresults-webhook

### 5.2 Fiserv Exchange 的 Boarding Status 值

```
OPEN
APP_STATUS_AWAITING_CHECKS
AWAITING_SIGNATURE
AWAITING_REMOTE_SIGNATURE
AWAITING_PAPER_SIGNATURE
AWAITING_ON_SCREEN_SIGNATURE
BOARDING_COMPLETE
CANCELLED
APP_STATUS_READY_FOR_MARKETPLACE_SUBMISSION
APP_STATUS_AWAITING_MARKETPLACE_RESPONSE
APP_STATUS_MARKETPLACE_SUBMISSION_ERROR
APP_STATUS_MARKETPLACE_REJECTED
UNDERWRITING_DECLINED
```

> 📖 来源（第三方参考）：Infinicept Fiserv Exchange Boarding Service Results Webhook 文档

### 5.3 Infinicept 的角色

Infinicept 是一个 **Payment Facilitator (PayFac) 使能平台**，与 Fiserv 有预集成关系。它的工作方式：

```
ISV/ISO → Infinicept API → Fiserv Exchange（内部通道）→ Fiserv 后端系统
                                                              │
                                                    生成 NorthMid, NashvilleMid 等
                                                              │
                                              Boarding Results 回调 → Infinicept → ISV/ISO
```

Infinicept 并没有调用某个 Fiserv 公开 API 来获取 Nashville MID，而是 Fiserv 后端在完成入驻后通过专用通道将全部 MID 推送给 Infinicept。

> 📖 来源（第三方参考）：Infinicept Developer Portal
> https://developer.infinicept.com/docs/getting-started-with-infinicept

---

## 6. CardPointe Gateway API — 交易处理（非 Boarding）

CardPointe Gateway API 是 Fiserv 面向北美的公开交易处理 API，**不涉及商户入驻**。

| 项目 | 说明 |
|------|------|
| 用途 | 交易授权、捕获、退款、查询、对账 |
| 文档 | https://developer.fiserv.com/product/CardPointe |
| 支持文档 | https://support.cardpointe.com/cardpointe-gateway-api/ |
| 认证方式 | HTTP Basic Auth（每个 MID 最多 5 组 API 凭证） |
| 关键参数 | `merchid` = First Data North 平台 MID |
| 安全 | CardSecure 令牌化（专利技术），P2PE 加密 |

> 📖 来源（官方确认）：CardPointe Gateway API 文档

---

## 7. 账户维护与变更

入驻完成后的账户维护操作方式：

| 变更类型 | 操作方式 | 提交位置 | SLA |
|----------|---------|---------|-----|
| 银行账户变更 | CoPilot ticket（Account Updates → Bank Account Change） | CoPilot | 2 个工作日 |
| DBA/地址/联系方式变更 | CoPilot ticket（Account Updates → Demographic Change） | CoPilot | 2 个工作日 |
| MCC/SIC 更新 | CoPilot ticket（Account Updates → MCC/SIC Update） | CoPilot | 2 个工作日 |
| 费率/定价更新 | MSC 表单（Rates + Fees） | Merchant Service Center | 48 小时 |
| 取消/恢复账户 | MSC | Merchant Service Center | — |
| 季节性暂停 | MSC | Merchant Service Center | — |
| 礼品卡 | MSC | Merchant Service Center | — |
| 批量账户维护（BAM） | MSC 上传 | Merchant Service Center | — |

> 📖 来源（官方确认）：CoPilot Ticketing 101
> https://support.cardpointe.com/additional-pages/copilot-ticketing-101/
>
> 以及 CoPilot & CardPointe 过渡文档 Ticketing Questions 章节

---

## 8. 品牌定制支持

| 组件 | 支持自定义品牌 |
|------|--------------|
| CoPilot 门户 | ✅ |
| CardPointe 商户门户 | ✅ |
| MPA（商户处理申请） | ✅ |
| 月度对账单 | ✅ |
| 邮件通知（注册/签名/账户相关） | ❌ 使用 CardPointe 品牌发送 |

> 📖 来源（官方确认）：CoPilot & CardPointe 过渡文档 — "CoPilot, CardPointe, statements, and the MPA can be branded, but email communication cannot."

---

## 9. 数据保留策略

| 数据类型 | 保留期限 |
|---------|---------|
| 月度对账单 | 永久保留（账户取消后也不清除） |
| MPA | 永久保留（账户取消后也不清除） |
| 资金/结算数据 | 最近 12 个月 + 当月至今 |
| 历史迁移数据（从旧系统） | 最近 6 个月对账单 + 6 个月交易/资金/结算数据（仅 North） |

> 📖 来源（官方确认）：CoPilot & CardPointe 过渡文档

---

## 10. IRIS CRM 与 CoPilot/CardPointe 的数据流

Fiserv 官方确认：**使用 IRIS CRM 的合作伙伴，在迁移到 CoPilot 后数据仍会继续流向 IRIS。**

具体包括（通过 CoPilot 报告集成）：
- Deposits（存款）
- Batches（批次）
- Account Status（账户状态）
- Retrievals（检索/预扣款）
- Chargebacks（退单）
- Statements（对账单）

> 📖 来源（官方确认）：CoPilot & CardPointe 过渡文档 — "Data will continue to flow through to IRIS for partners that use it."

---

## 11. 联系方式

| 团队 | 电话 | 邮箱 | 用途 |
|------|------|------|------|
| Partner Solutions | 1-877-828-0730 | partnersolutions@fiserv.com | ISO 合作伙伴咨询 |
| CardPointe Support | 1-800-828-0720 | — | CardPointe 设备/网关支持 |
| ISV Support | 1-484-581-7690 | isvsupport@cardconnect.com | API/Bolt/CoPilot 集成支持 |
| North & Nashville Support | 1-800-542-1894 | — | North/Nashville 平台设备支持 |
| Partner Solutions Service Team | 1-800-337-1222 | — | CoPilot 密码重置等 |
| 系统状态监控 | — | — | https://status.cardconnect.com/ |

> 📖 来源（官方确认）：CoPilot & CardPointe 过渡文档

---

## 12. 对 SUNBAY 项目的关键影响

### 12.1 已确认的事实

| # | 事实 | 影响 |
|---|------|------|
| 1 | CoPilot 没有公开的 REST Boarding API | 中间件无法直接通过 API 自动提交商户入驻申请 |
| 2 | 新商户统一入驻到 First Data North 平台 | CardPointe Gateway `merchid` = North MID，可直接用于交易处理 |
| 3 | Nashville MID 无法通过公开 API 获取 | 如需存储，只能从 CoPilot 界面手动获取或通过预集成伙伴回调 |
| 4 | IRIS CRM 数据流在 CoPilot 迁移后继续保持 | IRIS CRM API 仍可用于状态同步和数据回写 |
| 5 | 账户维护需通过 CoPilot ticket 或 MSC 手动操作 | 费率变更、银行变更等无法程序化自动完成 |

### 12.2 可行的自动化方案选项

| 方案 | 描述 | 自动化程度 | 复杂度 | 风险 |
|------|------|-----------|--------|------|
| **A. 半自动方案** | IRIS CRM 自动采集数据 → 运营人员在 CoPilot 手动入驻 → 中间件自动同步审批结果回 IRIS CRM | 中 | 低 | 低 |
| **B. Infinicept 集成** | 通过 Infinicept PayFac 平台对接 Fiserv Exchange，实现全程序化入驻 | 高 | 高 | 中（需额外合作关系和费用） |
| **C. CoPilot Web 自动化（RPA）** | 使用 RPA 工具模拟 CoPilot Web 界面操作 | 中高 | 高 | 高（界面变更风险、违反 ToS 风险） |

---

## 附录：引用来源汇总

| # | 来源 | 类型 | URL |
|---|------|------|-----|
| 1 | CoPilot & CardPointe 过渡文档 | 官方确认 | https://support.cardpointe.com/additional-pages/transitioning-to-copilot---cardpointe/ |
| 2 | CoPilot Ticketing 101 | 官方确认 | https://support.cardpointe.com/additional-pages/copilot-ticketing-101/ |
| 3 | CardPointe Gateway API | 官方确认 | https://support.cardpointe.com/cardpointe-gateway-api/ |
| 4 | CardPointe Developer Docs | 官方确认 | https://developer.fiserv.com/product/CardPointe |
| 5 | Fiserv Developer Studio | 官方确认 | https://developer.fiserv.com/ |
| 6 | Infinicept Fiserv Exchange Boarding Webhook | 第三方参考 | https://developer.infinicept.com/docs/fiserv-exchange-boardingserviceresults-webhook |
| 7 | Infinicept Developer Portal | 第三方参考 | https://developer.infinicept.com/docs/getting-started-with-infinicept |
| 8 | CardPointe 系统状态 | 官方确认 | https://status.cardconnect.com/ |
