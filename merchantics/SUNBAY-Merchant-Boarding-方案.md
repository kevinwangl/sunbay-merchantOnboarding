# 《SUNBAY 商户入驻系统（Merchant Boarding）集成开发方案》

---

## 1. 背景及现状

### 项目背景

本方案为 SUNBAY 代客户（ISO/代理商）开发的商户 Boarding 集成方案。

客户当前使用 IRIS CRM（Merchantics 实例）管理销售线索和商户关系，使用 Fiserv CardPointe 作为支付网关处理交易。目前商户入驻流程依赖人工操作：销售人员在 IRIS CRM 中收集商户信息后，需手动登录 CardPointe 后台提交入驻申请，审批结果再手动回填到 CRM 系统。该流程存在以下痛点：

- **效率低下**：人工在两个系统间搬运数据，单个商户入驻耗时 2-3 天
- **易出错**：手动录入导致字段遗漏、格式错误，退回率高
- **状态不透明**：审批进度无法实时追踪，销售和商户体验差
- **无法规模化**：随着业务增长，人工处理成为瓶颈

### 需求依据

- IRIS CRM Open API v1.6.4 文档：https://merchantics.iriscrm.com/api [附录 #1]
- Fiserv CardPointe Gateway API 文档：https://developer.fiserv.com/product/CardPointe/apis?branch=active [附录 #4]
- Fiserv CardPointe CoPilot 商户入驻文档（Boarding API 待确认）[附录 #6]

### 改造/建设目标

由 SUNBAY 为客户构建 IRIS CRM 与 CardPointe 之间的自动化商户入驻中间件系统（OnBoarding Service Middleware），实现：
1. 从 IRIS CRM 自动采集商户申请数据并提交至 CardPointe Boarding
2. 入驻审批状态自动同步回 IRIS CRM
3. 全流程可追踪、可审计
4. 商户入驻周期从 2-3 天缩短至 4 小时内
5. 架构支持未来扩展至其他 Processor（TSYS、Elavon 等）

### 产品/系统范围

| 维度 | 内容 |
|------|------|
| 核心产品/系统 | IRIS CRM（Merchantics）、OnBoarding Service Middleware（新建）、Fiserv-CardConnect（当前阶段），架构预留 TSYS / Elavon 扩展 |
| 终端/客户端类型 | Web 端（IRIS CRM 界面）、API（中间件服务） |
| 业务渠道/方式 | ISO/代理商线上商户入驻、销售团队 CRM 操作 |

---

## 2. 业务需求

### 2.1 商户信息采集（IRIS CRM 侧）

- 在 IRIS CRM 中配置标准化的商户入驻自定义字段（Lead Fields），覆盖 CardPointe Boarding 所需全部信息
- 支持 KYC 文档上传（营业执照、身份证明、银行对账单、void check）
- Lead 状态机管理，支持从 "New Lead" 到 "Active Merchant" 的完整生命周期

### 2.2 自动化入驻流程（中间件）

- 监听 IRIS CRM Webhook（`lead.status.updated`），当 Lead 状态变为 "Ready for Boarding" 时自动触发
  > ⚠️ 待确认：Webhook 事件类型的精确命名需在 IRIS CRM Subscriptions API 文档中核实 [附录 #1, #2]
- 从 IRIS CRM 拉取完整商户数据并进行字段映射转换
- 提交前进行本地数据校验（必填项、格式、业务规则）
- 调用 CardPointe CoPilot Boarding API 创建商户申请
  > ⚠️ 待确认：CardPointe 商户入驻通过 CoPilot 平台完成，需与 Fiserv 确认是否提供 REST API 供程序化调用 [附录 #6]
- 支持失败重试（指数退避，最多 3 次）
- 超过重试次数后标记为异常，通知运营人员人工介入

### 2.3 CoPilot MPA 签名与核保流程

中间件提交商户申请至 CoPilot 后，由 CoPilot 平台接管签名与核保流程：

1. **MPA 生成**：CoPilot 基于 Application Template 和提交的商户数据，生成动态 MPA（Merchant Processing Application）[附录 #6]
2. **发送签名**：CoPilot 通过邮件向商户（Account Owner）发送数字签名链接，商户需注册 CardPointe 账户后通过浏览器完成 MPA 审阅与电子签名 [附录 #6]
   - ISO 可在发送前通过 CoPilot 预填部分字段（如银行信息），银行信息通过 GIACT + Plaid 在线验证，验证通过则无需提供 voided check [附录 #6]
   - 也支持下载 PDF 纸质签名后上传至 CoPilot 作为替代 [附录 #6]
3. **核保审批**：签名完成后自动进入核保（Underwriting）流程 [附录 #6]
   - 自动审批：符合条件的申请通常数分钟内完成
   - 人工审批：不符合自动审批条件的申请转人工审核，通常 24 小时内完成
4. **结果通知**：审批完成后 CoPilot 通过 Webhook 回调通知中间件

> 注：MPA 模板、签名邮件均支持 ISO 品牌定制（CoPilot/CardPointe/对账单/MPA 可定制，邮件暂不支持定制，使用 CardPointe 品牌发送）。[附录 #6]

### 2.4 状态同步与通知（双向）

- 接收 CardPointe Webhook 回调（审批通过/拒绝/更新）
  > ⚠️ 待确认：CoPilot Webhook 回调的事件类型及 payload 格式需从 Fiserv 获取实际规格文档
- 将审批结果同步回 IRIS CRM：
  - 更新 Lead 自定义字段（MID、审批状态、开通日期）
  - 添加 Lead Note（审批备注、拒绝原因）
  - 自动流转 Lead 状态
- 定时轮询兜底机制（每 15 分钟检查 pending 状态申请）
- 审批通过/拒绝时通过 IRIS CRM 发送邮件/短信通知商户

### 2.5 运营管理与审计

> 以下为业务层面的运营需求，具体技术实现随中间件各模块内置。

- 全链路操作日志（请求/响应/状态变更）
- 异常告警：API 调用失败、Webhook 投递失败、状态不一致
- 敏感数据脱敏存储（银行账号、税号等 PII 信息）

### 2.6 附加流程：协议/费率变更签名（IRIS CRM E-Signature，可选）

> 本流程不属于标准 Boarding 流程，仅在客户有额外的代理协议、费率变更协议等需要商户签署时启用。

- 通过 IRIS CRM E-Signature API 创建签名模板，配置 Lead 字段到 PDF 的映射 [附录 #2]
- 在特定业务节点（如费率调整、协议续签）触发签名流程
- 自动用 Lead 数据填充 PDF 模板，生成签名链接并发送给商户
- 签署完成后更新 Lead 状态/备注
- **CardPointe 侧变更为半自动流程**：CoPilot 未提供 ticket/maintenance REST API，涉及 CardPointe 侧的变更（费率更新、银行账户变更、DBA/地址变更等）需运营人员手动在 CoPilot Web 界面提交 ticket 或通过 MSC 提交表单完成 [附录 #7]

| 变更类型 | CardPointe 操作方式 | SLA |
|---|---|---|
| 银行账户变更 | CoPilot ticket（Account Updates → Bank Account Change） | 2 个工作日 |
| 费率/定价更新 | MSC 表单（Rates + Fees） | 48 小时 |
| DBA/地址/联系方式变更 | CoPilot ticket（Account Updates → Demographic Change） | 2 个工作日 |
| MCC/SIC 更新 | CoPilot ticket（Account Updates → MCC/SIC Update） | 2 个工作日 |

> 注：纯 ISO 代理协议/佣金协议仅涉及 IRIS CRM 记录更新，不需要同步到 CardPointe。

---

## 3. 技术方案

### 3.1 整体方案架构图

```
                                Subscribe              
                               Merchant Events         
  ┌──────────┐    ┌──────────┐ ──────────────▶ ┌─────────────────────┐
  │  Import  │───▶│ IRIS CRM │                 │  OnBoarding Service │
  │ Merchant │    │          │ ◀────────────── │     Middleware      │
  └──────────┘    └──────────┘  Sync Merchant  │                     │
                                    Data       │                     │
                                               │                     │     ┌──────────────────┐
                                               │                     │────▶│ Fiserv           │
                                               └─────────────────────┘     │  CardConnect     │
                                                      │                    │  Returns MID/TID │
                                               ┌──────┴──────┐            │  ┌─────────────┐ │
                                               │ Future      │            │  │ Omaha       │ │
                                               │ Expansion:  │            │  │ Nashville   │ │
                                               │ TSYS/Elavon │            │  │ North       │ │
                                               └─────────────┘            │  └─────────────┘ │
                                                                           └──────────────────┘
```

**关键数据流（当前阶段 — Fiserv CardConnect）：**

```
① Lead 创建/更新 → IRIS CRM 订阅商户事件 → OnBoarding Service Middleware
② OnBoarding Service Middleware → Field Mapper → Validator → Boarding Service
③ Boarding Service → CardPointe CoPilot Boarding API
④ Fiserv 返回关键数据 MID / TID
⑤ OnBoarding Service Middleware → 同步商户数据至 IRIS CRM（回写状态/MID/TID）
```

### 3.2 模块概览

> 模块划分、工作量评估及交付物详见 **SUNBAY-Merchant-Boarding-Proposal-SOW.md**。

本技术方案涉及以下模块，后续章节围绕其技术实现展开：

| # | Module | 核心职责 |
|---|--------|----------|
| 1 | IRIS CRM 配置 | Lead 自定义字段、Webhook 订阅、KYC 文档上传 |
| 2 | OnBoarding Service Middleware — Core | Webhook Receiver、Field Mapper、Data Validator、数据库 & 安全 |
| 3 | OnBoarding Service Middleware — Processor Routing | Fiserv-CardConnect CoPilot Boarding API 对接（架构预留 TSYS/Elavon 扩展） |
| 4 | OnBoarding Service Middleware — Sync & Reliability | 状态回写、Retry Queue、定时轮询兜底 |
| 5 | 集成测试 & UAT | 全链路测试、异常场景、性能测试、用户验收 |
| 6 | E-Signature 集成（可选） | IRIS CRM E-Signature 协议/费率变更签名 |

#### 技术风险点

| 模块 | 风险 | 影响 |
|------|------|------|
| Module 1 | IRIS CRM 自定义字段数量上限需确认；Webhook URL 需公网可访问 | 字段不足则需分批创建 |
| Module 2 | IRIS CRM Webhook 签名验证机制需确认；CardPointe 字段规格可能随版本变化 | 影响安全校验和映射准确性 |
| Module 3 | ⚠️ 需与 Fiserv 确认 CoPilot 是否提供可程序化调用的 REST API，若不支持需评估替代方案 | **关键阻塞风险** |
| Module 4 | IRIS CRM API 限流 120 次/分钟；Redis 持久化需确保任务不丢失 | 批量同步需控制速率 |

---

### 3.3 数据字段映射表

> ⚠️ 以下 CardPointe Boarding 字段名为预估，实际字段规格需待 CoPilot Boarding API 文档确认后更新（前置依赖 #2）。

| IRIS CRM Lead 字段 | CardPointe Boarding 字段 | 类型 | 必填 | 转换规则 |
|---|---|---|---|---|
| company_name | legal_name | String | ✅ | 直接映射 |
| dba_name | dba | String | ✅ | 直接映射 |
| contact_first_name | owner_first_name | String | ✅ | 直接映射 |
| contact_last_name | owner_last_name | String | ✅ | 直接映射 |
| email | email | String | ✅ | 格式校验 |
| phone | phone | String | ✅ | 去除格式符号，保留纯数字 |
| address_line1 | business_address1 | String | ✅ | 直接映射 |
| address_line2 | business_address2 | String | | 直接映射 |
| city | business_city | String | ✅ | 直接映射 |
| state | business_state | String | ✅ | 转为 2 位州代码 |
| zip_code | business_zip | String | ✅ | 5 位或 9 位格式 |
| ein_ssn | tax_id | String | ✅ | 去除连字符 |
| bank_name | bank_name | String | ✅ | 直接映射 |
| bank_routing | routing_number | String | ✅ | 9 位 ABA 校验 |
| bank_account | account_number | String | ✅ | 加密传输 |
| sic_code | mcc | String | ✅ | SIC → MCC 映射表 |
| business_type | ownership_type | String | ✅ | 枚举映射 |
| monthly_volume | avg_monthly_volume | Decimal | ✅ | 单位：美元 |
| avg_ticket | avg_ticket_amount | Decimal | ✅ | 单位：美元 |
| max_ticket | max_ticket_amount | Decimal | | 单位：美元 |
| website_url | website | String | | URL 格式校验 |
| business_start_date | date_established | Date | ✅ | 转为 YYYY-MM-DD |
| — | boarding_request_id | String | — | 提交后回写 |
| — | merchant_id (MID) | String | — | 审批通过后回写 |
| — | boarding_status | String | — | 状态同步回写 |
| — | boarding_date | Date | — | 审批通过日期回写 |

### 3.4 数据库设计

#### 3.4.1 ER 关系图

```
┌──────────────────┐       ┌──────────────────┐
│ boarding_requests │──1:N──│  webhook_events  │
└────────┬─────────┘       └──────────────────┘
         │
         │ 1:N
         ▼
┌──────────────────┐       ┌──────────────────┐
│   audit_logs     │       │  field_mappings   │
└──────────────────┘       └──────────────────┘
```

#### 3.4.2 表结构定义

**boarding_requests（核心业务表）**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | BIGINT AUTO_INCREMENT | PK | 主键 |
| iris_lead_id | VARCHAR(64) | ✅ | IRIS CRM Lead ID |
| processor | VARCHAR(32) | ✅ | 处理器标识：`fiserv_cardconnect` |
| boarding_request_id | VARCHAR(128) | | CoPilot 返回的申请 ID |
| mid | VARCHAR(64) | | 审批通过后的 Merchant ID |
| tid | VARCHAR(64) | | Terminal ID |
| status | VARCHAR(32) | ✅ | 状态枚举（见 3.4.3） |
| request_payload | JSON | | 提交请求体（脱敏后） |
| response_payload | JSON | | 响应体 |
| error_code | VARCHAR(32) | | 错误码 |
| error_message | TEXT | | 错误详情 |
| retry_count | INT DEFAULT 0 | | 已重试次数 |
| next_retry_at | DATETIME | | 下次重试时间 |
| submitted_at | DATETIME | | 提交时间 |
| approved_at | DATETIME | | 审批通过时间 |
| created_at | DATETIME | ✅ | 创建时间 |
| updated_at | DATETIME | ✅ | 更新时间 |

索引：
- `UNIQUE idx_iris_lead` ON (iris_lead_id, processor)
- `idx_status` ON (status)
- `idx_next_retry` ON (status, next_retry_at) — 重试队列查询
- `idx_submitted` ON (submitted_at) — 定时轮询查询

**webhook_events（事件日志表）**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | BIGINT AUTO_INCREMENT | PK | 主键 |
| event_id | VARCHAR(128) | ✅ | 事件唯一 ID（幂等键） |
| source | VARCHAR(32) | ✅ | 来源：`iris_crm` / `cardpointe` |
| event_type | VARCHAR(64) | ✅ | 事件类型 |
| payload | JSON | ✅ | 原始 payload |
| status | VARCHAR(16) | ✅ | `received` / `processing` / `processed` / `failed` |
| error_message | TEXT | | 处理失败原因 |
| processed_at | DATETIME | | 处理完成时间 |
| created_at | DATETIME | ✅ | 接收时间 |

索引：
- `UNIQUE idx_event_id` ON (event_id) — 幂等去重
- `idx_source_type` ON (source, event_type, created_at)

**field_mappings（映射配置表）**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | BIGINT AUTO_INCREMENT | PK | 主键 |
| processor | VARCHAR(32) | ✅ | 处理器标识 |
| version | VARCHAR(16) | ✅ | 映射版本号 |
| mapping_config | JSON | ✅ | 映射规则 JSON |
| is_active | BOOLEAN DEFAULT true | | 是否启用 |
| created_at | DATETIME | ✅ | 创建时间 |

索引：
- `UNIQUE idx_processor_version` ON (processor, version)

**audit_logs（审计日志表）**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | BIGINT AUTO_INCREMENT | PK | 主键 |
| boarding_request_id | BIGINT | | 关联 boarding_requests.id |
| action | VARCHAR(64) | ✅ | 操作类型 |
| actor | VARCHAR(32) | ✅ | 操作者：`system` / `webhook` / `scheduler` |
| detail | JSON | | 操作详情 |
| created_at | DATETIME | ✅ | 操作时间 |

索引：
- `idx_boarding_request` ON (boarding_request_id, created_at)

#### 3.4.3 状态枚举

**boarding_requests.status：**

| 状态值 | 说明 | 流转来源 |
|--------|------|----------|
| `pending_validation` | 待校验 | 接收 Webhook 后创建 |
| `validation_failed` | 校验失败 | Validator 校验不通过 |
| `pending_submission` | 待提交 | 校验通过，等待提交 |
| `submitted` | 已提交 | CoPilot API 调用成功 |
| `pending_signature` | 待签名 | CoPilot 已发送 MPA |
| `under_review` | 核保审批中 | 商户签名完成 |
| `approved` | 审批通过 | CoPilot 回调 |
| `declined` | 审批拒绝 | CoPilot 回调 |
| `need_info` | 需补充资料 | CoPilot 回调 |
| `submission_failed` | 提交失败 | API 调用失败 |
| `retry_pending` | 等待重试 | 失败后进入重试队列 |
| `dead_letter` | 死信 | 超过最大重试次数 |

#### 3.4.4 数据生命周期

| 表 | 保留策略 |
|----|----------|
| boarding_requests | 永久保留（业务核心数据） |
| webhook_events | 90 天后归档至冷存储 |
| audit_logs | 1 年后归档至冷存储 |
| field_mappings | 永久保留（配置数据） |

---

### 3.5 接口设计

#### 3.5.1 OnBoarding Service Middleware 暴露接口

| Method | Path | 说明 |
|--------|------|------|
| POST | `/webhooks/iris-crm` | 接收 IRIS CRM Webhook |
| POST | `/webhooks/cardpointe` | 接收 CardPointe Webhook |
| GET | `/health` | 健康检查 |
| GET | `/api/boarding-requests/:id` | 查询入驻申请状态（运营用） |
| POST | `/api/boarding-requests/:id/retry` | 手动触发重试（运营用） |

#### 3.5.2 调用外部接口清单

| 目标系统 | 接口 | 用途 |
|----------|------|------|
| IRIS CRM | `GET /leads/{leadId}` | 拉取 Lead 完整数据 |
| IRIS CRM | `PATCH /leads/{leadId}` | 回写状态/MID/TID |
| IRIS CRM | `POST /leads/{leadId}/notes` | 添加 Lead Note |
| IRIS CRM | `GET /leads/{leadId}/documents` | 获取 KYC 文档 |
| CardPointe CoPilot | Boarding API（待确认） | 提交商户入驻申请 |
| CardPointe CoPilot | Status API（待确认） | 轮询申请状态 |

---

### 3.6 错误码设计

| 错误码 | 分类 | 说明 | 处理策略 |
|--------|------|------|----------|
| `V001` | 校验错误 | 必填字段缺失 | 不可重试，回写 Lead Note |
| `V002` | 校验错误 | 字段格式不合法 | 不可重试，回写 Lead Note |
| `V003` | 校验错误 | 业务规则不满足 | 不可重试，回写 Lead Note |
| `M001` | 映射错误 | 映射配置缺失 | 不可重试，告警运营 |
| `B001` | 提交错误 | CoPilot API 超时 | 可重试 |
| `B002` | 提交错误 | CoPilot API 返回 4xx | 不可重试，回写错误详情 |
| `B003` | 提交错误 | CoPilot API 返回 5xx | 可重试 |
| `B004` | 提交错误 | 认证失败 | 不可重试，告警运营 |
| `S001` | 同步错误 | IRIS CRM API 超时 | 可重试 |
| `S002` | 同步错误 | IRIS CRM API 限流 | 可重试（延迟） |
| `S003` | 同步错误 | Lead 状态不一致 | 告警运营，人工介入 |

**异常分级：**

| 级别 | 处理策略 | 示例 |
|------|----------|------|
| L1 — 可重试 | 自动进入 Retry Queue | B001, B003, S001, S002 |
| L2 — 不可重试 | 回写错误至 IRIS CRM，标记失败 | V001-V003, B002 |
| L3 — 需人工介入 | 进入死信队列 + 告警运营 | M001, B004, S003 |

---

### 3.7 配置管理

#### 环境变量清单

| 变量名 | 说明 | 示例 |
|--------|------|------|
| `IRIS_CRM_API_BASE_URL` | IRIS CRM API 地址 | `https://merchantics.iriscrm.com/api/v1` |
| `IRIS_CRM_API_TOKEN` | IRIS CRM API Token | `***` |
| `CARDPOINTE_API_BASE_URL` | CoPilot Boarding API 地址 | 待确认 |
| `CARDPOINTE_API_KEY` | CoPilot API 认证密钥 | `***` |
| `CARDPOINTE_WEBHOOK_SECRET` | Webhook 签名密钥 | `***` |
| `DATABASE_URL` | Aurora 连接串 | `mysql://...` |
| `REDIS_URL` | Redis 连接串 | `redis://...` |
| `RETRY_MAX_ATTEMPTS` | 最大重试次数 | `3` |
| `RETRY_DELAYS_MS` | 重试间隔（毫秒） | `60000,300000,900000` |
| `POLLING_INTERVAL_MS` | 轮询间隔 | `900000` |
| `LOG_LEVEL` | 日志级别 | `info` |

#### Field Mapper 配置结构示例

```json
{
  "processor": "fiserv_cardconnect",
  "version": "1.0.0",
  "mappings": [
    {
      "source": "company_name",
      "target": "legal_name",
      "type": "string",
      "required": true,
      "transform": null
    },
    {
      "source": "state",
      "target": "business_state",
      "type": "string",
      "required": true,
      "transform": "state_to_code"
    },
    {
      "source": "sic_code",
      "target": "mcc",
      "type": "string",
      "required": true,
      "transform": "sic_to_mcc"
    }
  ]
}
```

---

### 3.8 Lead 状态机设计

```
New Lead
  │
  ▼
Info Collecting ◀──(资料不全)── Pending Info
  │                                ▲
  │                                │
  ▼                                │
Documents Uploaded ──(缺少文档)────┘
  │
  ▼
Ready for Boarding ★ 触发自动入驻
  │
  │ (OnBoarding Service Middleware 提交 CoPilot Boarding API)
  ▼
Boarding Submitted
  │
  │ (CoPilot 发送 MPA 给商户数字签名)
  ▼
Pending Signature
  │
  │ (商户完成 MPA 签名 → 进入核保审批)
  ▼
Under Review
  │
  ├──(boarding.approved)──▶ Approved → Active Merchant
  │
  ├──(boarding.declined)──▶ Declined → Closed / Retry
  │
  └──(boarding.updated: need_info)──▶ Need More Info → Pending Info
```

---

## 4. 项目启动条件

### 4.1 确定项目 UI/UX 交互设计

- 目标版本号：SUNBAY Merchant Boarding v1.0.0
- UI/UX 设计稿确认状态：
  - IRIS CRM Lead 字段布局（Tab 分组）：需确认

### 4.2 研发团队配置

| 角色 | 人数 | 职责 |
|------|------|------|
| 后端工程师 | 2 | 中间件核心开发（Webhook/Mapper/Boarding/Sync） |
| DevOps 工程师 | 1（兼职） | 部署、CI/CD、监控配置 |
| QA 工程师 | 1 | 集成测试、异常场景测试 |
| 产品/业务 | 1（兼职） | IRIS CRM 字段配置、业务规则确认、UAT |

### 4.3 项目实施计划

| Items | Description | W1 | W2 | W3 | W4 | W5 | W6 | W7 | W8 |
|-------|-------------|----|----|----|----|----|----|----|----|
| IRIS CRM 配置 | 自定义字段/Webhook/KYC | ■ | ■ | | | | | | |
| OnBoarding Service — Core | Webhook Receiver/Field Mapper/Validator/DB 设计 | | ■ | ■ | ■ | | | | |
| OnBoarding Service — Routing | CardPointe CoPilot Boarding API 对接 | | | | ■ | ■ | | | |
| OnBoarding Service — Sync | 状态回写/Retry Queue/定时轮询 | | | | | ■ | ■ | | |
| 集成测试 & UAT | 全链路/异常/性能/用户验收 | | | | | | | ■ | ■ |

**关键路径**：IRIS CRM 配置 → Core（Webhook/Mapper/Validator）→ Routing（Boarding Service）→ Sync → 集成测试

### 4.3.1 版本发布计划

| 时间节点 | 版本号 | 发布内容 |
|----------|--------|---------|
| W4 结束 | v0.1.0-alpha | IRIS CRM 配置完成 + OnBoarding Service Core（Webhook Receiver + Field Mapper + Validator） |
| W6 结束 | v0.5.0-beta | Processor Routing（CardPointe API 对接）+ Sync & Reliability（完整入驻流程可跑通，沙箱环境） |
| W7 结束 | v0.9.0-rc | 集成测试通过，全链路验证完成 |
| W8 结束 | v1.0.0 | UAT 通过，生产环境上线 |

---

## 5. 前置依赖与待确认事项

| # | 事项 | 负责方 | 状态 |
|---|------|--------|------|
| 1 | CardPointe CoPilot Boarding API 可用性确认及沙箱环境账号申请 | 客户 + Fiserv | ⚠️ 高优先级待确认 |
| 2 | CardPointe Boarding API 详细字段规格及 Webhook 事件规格文档 | Fiserv | ⚠️ 高优先级待获取 |
| 3 | IRIS CRM API Token 权限范围确认 | 客户 | 待确认 |
| 4 | IRIS CRM Subscriptions API Webhook 事件类型精确命名确认 | 客户 | 待确认 |
| 5 | IRIS CRM 自定义字段数量上限 | 客户 | 待确认 |
| 6 | Webhook 接收端公网域名/IP + SSL 证书 | SUNBAY DevOps | 待配置 |
| 7 | 商户协议/费率变更签名模板（E-Signature 用，可选）| 客户业务/法务 | 待提供 |
| 8 | SIC Code → MCC 映射表 | 客户业务 | 待提供 |
| 9 | 生产环境服务器/云资源 | SUNBAY DevOps | 待采购 |

---

> 文档版本：v1.3 | 编制日期：2026-03-19 | 编制人：SUNBAY 技术团队 | 更新说明：统一架构为 OnBoarding Service Middleware；当前阶段 Fiserv-CardConnect，架构预留 TSYS/Elavon 扩展；开发项合并为 6 个 Module，总工时调整为 33 人天；技术实现方案与 Module 对齐

---

## 附录：引用依据

| # | 引用内容 | 链接 |
|---|---------|------|
| 1 | IRIS CRM Open API 文档（Merchantics 实例） | https://merchantics.iriscrm.com/api |
| 2 | IRIS CRM Open API 概述（Leads/Merchants/E-Signature/Subscriptions） | https://merchantics.iriscrm.com/api#section/Open-API |
| 3 | IRIS CRM PHP SDK（API 功能参考） | https://packagist.org/packages/iris-crm/php-sdk |
| 4 | CardPointe Gateway API 文档 | https://developer.fiserv.com/product/CardPointe/apis?branch=active |
| 5 | CardPointe Gateway API 介绍 | https://support.cardpointe.com/cardpointe-gateway-api/ |
| 6 | CoPilot & CardPointe 平台概述（Boarding/MPA 签名/核保/品牌定制） | https://support.cardpointe.com/additional-pages/transitioning-to-copilot---cardpointe/ |
| 7 | CoPilot Ticketing 101（账户维护/费率变更/银行变更操作方式及 SLA） | https://support.cardpointe.com/additional-pages/copilot-ticketing-101/ |
