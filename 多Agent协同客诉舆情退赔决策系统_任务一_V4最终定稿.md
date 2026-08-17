# 多Agent协同客诉舆情退赔决策系统

## 任务一：需求分析、架构设计与 WBS 拆解 V4 最终定稿

> 本文档是演示型 MVP 进入编码阶段前的需求与技术规格基线。前端、后端、Agent 编排、数据库、接口、测试与部署均以本文档为准。

## 1. 项目定位

本项目面向电商客诉退款场景，通过多 Agent 完成客诉理解、证据分析、欺诈风险计算和舆情风险识别，并由确定性规则完成退款决策。

基本原则：

1. Agent 负责理解、识别、提取和评分；
2. Decision Node 负责确定性决策；
3. Refund Worker 负责真实资金动作；
4. LLM、VLM 不得直接决定或执行退款；
5. 高金额、高风险、低置信度、`UNKNOWN` 和关键服务异常全部转人工；
6. 资金安全和异常兜底优先于自动化率。

## 2. 业务目标

- 自动解析客诉文本和多模态凭证；
- 识别重复凭证、异常退款和舆情升级信号；
- 低金额、低风险且证据充分的案件自动进入退款执行；
- 高风险或不确定案件挂起并由主管审核；
- 展示 Agent 执行轨迹、状态变化和决策理由；
- 防止重复审批、状态覆盖和重复退款；
- Worker 或 API 重启后，任务和 Graph 状态可恢复。

## 3. 用户角色与权限

| 角色 | 权限 |
|---|---|
| `CUSTOMER_SERVICE` | 登录、创建工单、查看授权范围内工单、上传或补传凭证 |
| `SUPERVISOR` | 客服权限、查看待审核工单、批准或拒绝、查看审计记录 |
| `SYSTEM_WORKER` | 执行 Agent、发布事件、创建退款任务；不得通过人工 API 登录 |

权限规则：

- 密码使用 Argon2id；MVP 无法使用时可退化为 bcrypt；
- Access Token 有效期 30 分钟，Refresh Token 有效期 7 天；
- 禁用账号的所有令牌立即失效；
- 连续 5 次登录失败锁定 15 分钟；
- 审批时校验角色、分配关系、工单状态和版本号；
- 审批、拒绝、退款及权限失败均写入审计日志。

## 4. MVP 范围

### 4.1 必须实现

- JWT、RBAC、账号禁用和登录限流；
- 创建、查询、筛选和查看工单；
- 上传、补传和分析凭证；
- Intake、Evidence、Fraud、Sentiment 四个 Agent；
- Pydantic 严格结构化输出；
- LangGraph 挂起、恢复和 Redis Checkpointer；
- 确定性 Decision Node；
- 人工批准和拒绝；
- 模拟支付适配器及完整退款状态；
- PostgreSQL、Redis Streams、Outbox、Worker；
- SSE 实时轨迹、断线重放和快照补偿；
- 幂等、分布式锁、乐观锁、数据库唯一约束和审计日志；
- Docker Compose 和核心自动化测试。

### 4.2 MVP 实现边界

- 支付渠道使用可配置的模拟适配器，不接真实资金渠道；
- Fraud Provider 使用 PostgreSQL 样例数据；
- OCR 重点支持发票、面单、聊天和物流截图；
- VLM 重点支持商品破损图；
- UI 以业务闭环和状态可视化为主；
- 压测只覆盖同步 REST API，不包含模型推理和 SSE 长连接。

## 5. 非 MVP 范围

不实现 Kubernetes、Kafka、复杂微服务、模型训练或微调、强化学习、动态 Planning、自动多级审批、自动拒绝、知识图谱、生产级支付渠道和大规模特征平台。

## 6. 统一术语与标识

| 名称 | 定义 |
|---|---|
| `id` | `cases` 表内部 UUID 主键，仅服务端内部使用 |
| `case_id` | 对外业务工单号，全局唯一，格式 `CASE-{yyyyMMdd}-{6位序列}` |
| `case_pk` | 关联表中引用 `cases.id` 的 UUID 外键字段 |
| `refund_request_id` | 一次退款业务请求的全局唯一标识 |
| `thread_id` | LangGraph 线程标识，值固定为 `case_id` |
| `event_id` | Redis Stream ID，用于 SSE 排序、重放和去重 |

`case_no` 正式废弃，数据库、代码、接口、日志和事件中禁止出现。

## 7. 总体架构

```text
React Dashboard
  ├─ REST API ───────────────┐
  └─ SSE                     │
                             ▼
                         FastAPI
                     ┌───────┴────────┐
                     ▼                ▼
                PostgreSQL          Redis
              Cases + Outbox   Streams/Lock/Cache
                     │                │
                     ▼                ▼
              Outbox Publisher     Workers
                                      │
                                      ▼
                                  LangGraph
                                      │
                                      ▼
                              Refund Task/Worker
                                      │
                                      ▼
                           Mock Payment Adapter
```

PostgreSQL 是业务状态的最终事实来源。Redis 用于队列、锁、缓存、Graph Checkpoint 和 SSE 事件，不作为审批或退款结果的唯一存储。

## 8. Agent 执行拓扑

```text
Intake Agent
    ↓
Evidence Agent
    ↓
    ├───────────────┐
    ▼               ▼
Fraud Agent   Sentiment Agent
    └───────┬───────┘
            ▼
       Decision Node
        ├───────────────┐
        ▼               ▼
 Human Review      Refund Execution
```

- Evidence 依赖 Intake 输出的 `complaint_type` 和 `evidence_required`；
- Fraud 读取 Evidence 生成的重复凭证风险；
- Evidence 完成后，Fraud 与 Sentiment 并行；
- Decision 必须等待所有必需 Agent 完成；
- 任一必需结果为 `UNKNOWN` 或 `FAILED` 时转人工。

## 9. Agent 职责与严格 Schema

### 9.1 Intake Agent

输出：

```json
{
  "schema_version": "1.0",
  "intent": "REFUND",
  "complaint_type": "PRODUCT_DAMAGE",
  "entities": {"product": "手机", "problem": "屏幕破损"},
  "requested_action": "REFUND",
  "evidence_required": true,
  "confidence": 0.96,
  "status": "SUCCESS"
}
```

`complaint_type` 枚举：`PRODUCT_DAMAGE`、`WRONG_ITEM`、`LOGISTICS_DAMAGE`、`NOT_RECEIVED`、`SERVICE_COMPLAINT`、`OTHER`。

### 9.2 Evidence Agent

单文件输出 `EvidenceItemResult`，聚合输出 `EvidenceAggregateResult`：

```json
{
  "schema_version": "1.0",
  "status": "EVIDENCE_VALID",
  "evidence_confidence": 0.91,
  "required_types_satisfied": true,
  "items": [
    {
      "evidence_type": "PRODUCT_IMAGE",
      "valid": true,
      "image_quality": "GOOD",
      "confidence": 0.92,
      "ocr_text": null,
      "structured_fields": {},
      "damage_detected": true,
      "damage_type": "SCREEN_BROKEN",
      "damage_regions": [{"label": "screen", "description": "右上角裂纹"}],
      "consistency_score": 0.91,
      "risk_flags": [],
      "reason": "图片与客诉描述一致"
    }
  ],
  "aggregate_risk_flags": [],
  "duplicate_evidence_risk": 0.0
}
```

Evidence 状态枚举：`EVIDENCE_MISSING`、`LOW_QUALITY`、`UNREADABLE`、`MISSING_REQUIRED_TYPE`、`EVIDENCE_CONFLICT`、`WAITING_EVIDENCE_SUPPLEMENT`、`EVIDENCE_VALID`、`EVIDENCE_INVALID`、`EVIDENCE_UNKNOWN`。

### 9.3 Fraud Agent

```json
{
  "schema_version": "1.0",
  "fraud_score": 0.68,
  "risk_level": "HIGH",
  "force_hit": true,
  "force_hit_rules": ["DUPLICATE_EVIDENCE_HIGH"],
  "factors": {"duplicate_evidence_risk": 0.95},
  "missing_factors": [],
  "reasons": ["发现跨订单近似重复凭证"],
  "status": "SUCCESS"
}
```

### 9.4 Sentiment Agent

```json
{
  "schema_version": "1.0",
  "sentiment": "NEGATIVE",
  "public_opinion_risk": "HIGH",
  "risk_score": 0.86,
  "keywords": ["曝光", "小红书"],
  "status": "SUCCESS"
}
```

所有 Schema 采用 `extra="forbid"`、枚举和数值范围校验。解析失败时同模型修复 1 次；仍失败则输出 `UNKNOWN` 并转人工。解释字段仅用于展示和审计，不直接驱动资金规则。

## 10. Evidence 边界规则

| 客诉类型 | 最低证据要求 |
|---|---|
| `PRODUCT_DAMAGE` | 至少 1 张商品图片 |
| `WRONG_ITEM` | 至少 1 张商品图片或 1 张面单 |
| `LOGISTICS_DAMAGE` | 至少 1 张商品破损图片，并且至少 1 张面单或物流截图 |
| `NOT_RECEIVED` | 非强制，可由物流查询数据补充 |
| `SERVICE_COMPLAINT` | 聊天截图可选 |
| `OTHER` | 无法确定时转人工 |

OCR 路由：

- `confidence >= 0.85`：直接进入结构化；
- `0.60 <= confidence < 0.85`：增强后重试 1 次，以第二次结果为准；
- 最终 `confidence < 0.60`：`UNREADABLE`，允许补传；
- 纯商品图跳过 OCR，进入 VLM。

最多补传 2 次。补传只重新执行 Evidence 及下游节点。超过次数仍不满足要求则转人工。

重复检测采用 SHA256 精确匹配和 pHash 近似匹配。`SCREENSHOT_REUSE`、`POSSIBLE_EDIT` 和重复分值传入 Fraud Agent；明确冲突不得被其他高置信度文件覆盖。

## 11. Fraud 特征与评分

| 特征 | 权重 | 数据源 | 缺失处理 |
|---|---:|---|---|
| `refund_frequency_30d` | 0.15 | `refund_history` | `UNKNOWN` |
| `refund_rate_90d` | 0.15 | `orders`、`refund_history` | 新用户规则 |
| `high_value_refund_rate` | 0.10 | `orders`、`refund_history` | 降低权重 |
| `device_account_risk` | 0.10 | 设备指纹表 | `UNKNOWN` |
| `address_risk` | 0.10 | 地址 Hash 历史表 | `UNKNOWN` |
| `duplicate_evidence_risk` | 0.20 | Evidence SHA256、pHash | 无文件时不计算 |
| `historical_fraud_flag` | 0.15 | `risk_labels` | 无命中为 0，并记录来源 |
| `account_age_risk` | 0.05 | 用户账户表 | `UNKNOWN` |

缺失不等于零风险。缺失特征不超过 2 个时，对有效特征重新归一化权重；超过 2 个时 Fraud 输出 `UNKNOWN`。

等级：`0.00–0.39 LOW`、`0.40–0.69 MEDIUM`、`0.70–1.00 HIGH`。

强规则：

```text
historical_fraud_flag == 1       → HISTORICAL_FRAUD
duplicate_evidence_risk >= 0.90  → DUPLICATE_EVIDENCE_HIGH
device_account_risk >= 0.95      → DEVICE_ACCOUNT_HIGH
```

任一强规则命中时必须设置 `risk_level=HIGH`、`force_hit=true`，不受加权总分影响。权重是 MVP 专家经验初始化值，不得描述为训练结果。

## 12. Sentiment 规则

识别曝光、微博、小红书、抖音、媒体、消协、黑猫投诉、记者、起诉、热搜和直播曝光等升级信号。输出等级与 Fraud 相同。`HIGH` 必须人工审核，避免舆情敏感案件由系统自动完成资金决策。

## 13. Decision 确定性规则

自动退款决策必须同时满足：

```text
refund_amount <= 300
AND evidence_status == EVIDENCE_VALID
AND evidence_confidence >= 0.60
AND fraud_score < 0.70
AND fraud_risk_level != HIGH
AND fraud_force_hit == false
AND public_opinion_risk != HIGH
AND all_required_agents_success == true
```

满足时设置 `decision_status=AUTO_REFUND`、`refund_status=PENDING`，随后创建退款任务；不得直接设置为退款成功或工单完成。

以下任一条件强制人工：金额大于 300、证据无效或不足、Fraud 为 HIGH、命中强规则、舆情为 HIGH、任一必需结果为 `UNKNOWN/FAILED`、解析失败或关键服务不可用。

第一版不实现 `AUTO_REJECT`。

## 14. Human-in-the-loop

Decision 进入 HumanReview 后调用 `interrupt()`，保存 `thread_id=case_id`。主管通过审批 API 提交 `APPROVE` 或 `REJECT`：

- `APPROVE`：`decision_status=APPROVED`，创建退款任务，Graph 从 HumanReview 恢复；
- `REJECT`：`decision_status=REJECTED`、`refund_status=NOT_REQUIRED`、`case_status=COMPLETED`；
- 恢复只从 HumanReview 后继续，不重新执行已完成 Agent。

## 15. 真实退款执行链路

```text
AUTO_REFUND / APPROVED
        ↓
refund_status=PENDING
        ↓
创建 refund_transactions + outbox_events
        ↓
Refund Worker
        ↓
refund_status=PROCESSING
        ↓
Payment Adapter
   ├─ 明确成功 → SUCCEEDED → case COMPLETED
   ├─ 明确失败 → FAILED → case SUSPENDED
   └─ 超时/未知 → UNKNOWN → 主动查询渠道结果
```

退款超时不得盲目重试。必须先以 `channel_idempotency_key` 查询渠道结果；仅在确认未受理时才允许重试。

## 16. 四维状态模型

| 维度 | 状态 |
|---|---|
| `case_status` | `RUNNING`、`SUSPENDED`、`COMPLETED`、`FAILED` |
| `graph_status` | `PENDING`、`RUNNING`、`INTERRUPTED`、`RESUMING`、`COMPLETED`、`FAILED` |
| `decision_status` | `PENDING`、`AUTO_REFUND`、`WAITING_HUMAN`、`APPROVED`、`REJECTED` |
| `refund_status` | `NOT_REQUIRED`、`PENDING`、`PROCESSING`、`SUCCEEDED`、`FAILED`、`UNKNOWN` |

关键合法迁移：

| 事件 | 迁移 |
|---|---|
| 创建工单 | Case `RUNNING`；Graph `PENDING`；Decision `PENDING`；Refund `NOT_REQUIRED` |
| 开始 Agent | Graph `PENDING → RUNNING` |
| 转人工 | Case `RUNNING → SUSPENDED`；Graph `RUNNING → INTERRUPTED`；Decision `PENDING → WAITING_HUMAN` |
| 人工批准 | Graph `INTERRUPTED → RESUMING`；Decision `WAITING_HUMAN → APPROVED`；Refund `NOT_REQUIRED → PENDING` |
| 人工拒绝 | Decision `WAITING_HUMAN → REJECTED`；Refund保持 `NOT_REQUIRED`；Case `SUSPENDED → COMPLETED` |
| 自动退款决策 | Decision `PENDING → AUTO_REFUND`；Refund `NOT_REQUIRED → PENDING` |
| 执行退款 | Refund `PENDING → PROCESSING` |
| 退款成功 | Refund `PROCESSING/UNKNOWN → SUCCEEDED`；Graph → `COMPLETED`；Case → `COMPLETED` |
| 退款明确失败 | Refund `PROCESSING → FAILED`；Case → `SUSPENDED` |
| 退款结果不明 | Refund `PROCESSING → UNKNOWN`；Case → `SUSPENDED` |

`APPROVED` 和 `AUTO_REFUND` 均不等于退款成功。系统 `FAILED` 不代表业务拒绝。

## 17. Redis Checkpointer

FastAPI 无状态，不得使用进程全局变量保存 Graph State。Checkpoint 至少保存 `case_id`、当前节点、四维状态、Agent 结果、`version` 和更新时间。数据库仍保存可查询的业务状态快照。

## 18. Redis Streams 可靠消费

- Stream：`complaint:tasks`；Consumer Group：`complaint-workers`；
- 成功后 `XACK`；Worker 崩溃时消息留在 PEL；
- Recovery Worker 使用 `XAUTOCLAIM` 领取超时消息；
- 领取后先查询数据库任务状态，已完成则直接 ACK；
- 重试次数持久化在任务表，不依赖原始 Stream 消息的不可变字段；
- 超过 2 次写入 `complaint:tasks:dlq`；
- 进入 DLQ 时工单转为 `SUSPENDED/WAITING_HUMAN` 并写审计日志；
- Worker 必须基于业务唯一键幂等消费。

## 19. Transactional Outbox

创建工单、审批产生退款任务时，在同一数据库事务内写入业务表和 `outbox_events`。Outbox Publisher 通过 `FOR UPDATE SKIP LOCKED` 批量领取未发布事件，写入 Redis Stream 后记录 `published_at`。发布失败保留记录并重试。

允许极端情况下重复发布，因此消费端仍必须幂等；不允许数据库事务中先写 Redis 再提交数据库。

## 20. SSE 重连与补偿

- Endpoint：`GET /api/v1/cases/{case_id}/events`；
- 事件使用 Redis Stream ID 作为 `event_id`；
- 前端保存最后已处理 ID，重连携带 `Last-Event-ID`；
- 退避间隔为 1、2、5、10、30 秒；
- 服务端每 15 秒发送 `ping`，前端 45 秒无消息则重连；
- 前端按 `event_id` 去重；
- Redis Stream 按 `MAXLEN≈1000` 裁剪，并由定时任务删除 24 小时前事件；任一条件触发均可清理；
- ID 超出保留范围时发送 `snapshot_required`，前端重新获取工单和 Timeline；
- SSE 仅用于通知和展示，PostgreSQL 业务状态为最终事实来源。

## 21. 幂等、锁和并发控制

审批存在三类键：

1. 客户端 `X-Idempotency-Key`，用于识别网络重试；
2. 服务端业务键：`SHA256(case_id + action + operator_id + expected_version)`；
3. 渠道退款键：`channel_idempotency_key`，由 `refund_request_id` 派生。

服务端不得只信任客户端字符串。审批最终结果和业务幂等键写入 PostgreSQL 唯一列，Redis 只做快速拦截。

Case 锁使用 `lock:case:{case_id}`，只保护状态检查、任务创建和短事务，不覆盖模型或支付调用。锁设置合理 TTL，并在确需长事务时续租；释放时使用 token 校验 Lua。数据库版本和唯一约束提供最终并发保护。

乐观锁：

```sql
UPDATE cases
SET decision_status = :new_status,
    version = version + 1,
    updated_at = NOW()
WHERE case_id = :case_id
  AND version = :expected_version;
```

影响行数为 0 时返回 `CASE_STATE_CONFLICT`。

## 22. 数据库设计

所有时间字段使用 `TIMESTAMPTZ`，状态使用数据库枚举或 `CHECK`，0～1 分值均增加范围约束。

### 22.1 核心表

```text
users
- id UUID PK
- username VARCHAR(64) UNIQUE NOT NULL
- password_hash VARCHAR(255) NOT NULL
- role VARCHAR(32) NOT NULL CHECK (...)
- status VARCHAR(16) NOT NULL CHECK (...)
- failed_login_count INT NOT NULL DEFAULT 0
- locked_until TIMESTAMPTZ NULL
- created_at/updated_at TIMESTAMPTZ NOT NULL

cases
- id UUID PK
- case_id VARCHAR(64) UNIQUE NOT NULL
- order_id/user_id VARCHAR(64) NOT NULL
- complaint_text TEXT NOT NULL
- refund_amount DECIMAL(12,2) NOT NULL CHECK (refund_amount >= 0)
- case_status/graph_status/decision_status/refund_status VARCHAR NOT NULL CHECK (...)
- fraud_score/sentiment_score/evidence_confidence DECIMAL(5,4) CHECK (value BETWEEN 0 AND 1)
- assigned_to UUID NULL REFERENCES users(id)
- version INT NOT NULL DEFAULT 1 CHECK (version > 0)
- created_at/updated_at/completed_at TIMESTAMPTZ

evidences
- id UUID PK
- case_pk UUID NOT NULL REFERENCES cases(id) ON DELETE RESTRICT
- storage_key TEXT NOT NULL
- file_hash VARCHAR(64) NOT NULL
- perceptual_hash VARCHAR(64) NULL
- file_type/status VARCHAR NOT NULL CHECK (...)
- ocr_text TEXT NULL
- ocr_confidence/evidence_confidence DECIMAL(5,4) CHECK (...)
- visual_result/structured_fields JSONB
- created_at TIMESTAMPTZ NOT NULL

fraud_features
- id UUID PK
- case_pk UUID UNIQUE NOT NULL REFERENCES cases(id) ON DELETE RESTRICT
- 八类特征 DECIMAL(5,4) NULL CHECK (...)
- missing_factors JSONB NOT NULL
- final_score DECIMAL(5,4) NULL CHECK (...)
- risk_level VARCHAR NOT NULL CHECK (...)
- force_hit BOOLEAN NOT NULL
- force_hit_rules JSONB NOT NULL
- created_at TIMESTAMPTZ NOT NULL

agent_executions
- id UUID PK
- case_pk UUID NOT NULL REFERENCES cases(id) ON DELETE RESTRICT
- agent_name/status VARCHAR NOT NULL
- input_json/output_json JSONB
- error_message TEXT NULL
- start_time/end_time TIMESTAMPTZ
- duration_ms/retry_count INT

human_reviews
- id UUID PK
- case_pk UUID NOT NULL REFERENCES cases(id) ON DELETE RESTRICT
- reviewer_id UUID NOT NULL REFERENCES users(id)
- action VARCHAR NOT NULL CHECK (action IN ('APPROVE','REJECT'))
- comment TEXT NULL
- state_version INT NOT NULL
- client_idempotency_key VARCHAR(128) NOT NULL
- business_idempotency_key VARCHAR(128) UNIQUE NOT NULL
- result_json JSONB NOT NULL
- created_at TIMESTAMPTZ NOT NULL

refund_transactions
- id UUID PK
- case_pk UUID NOT NULL REFERENCES cases(id) ON DELETE RESTRICT
- refund_request_id VARCHAR(64) UNIQUE NOT NULL
- provider_transaction_id VARCHAR(128) UNIQUE NULL
- channel_idempotency_key VARCHAR(128) UNIQUE NOT NULL
- status VARCHAR NOT NULL CHECK (...)
- amount DECIMAL(12,2) NOT NULL CHECK (amount >= 0)
- attempt_count INT NOT NULL DEFAULT 0
- request_payload/response_payload JSONB
- last_error TEXT NULL
- created_at/updated_at/completed_at TIMESTAMPTZ

outbox_events
- id UUID PK
- aggregate_type/aggregate_id/event_type VARCHAR NOT NULL
- payload JSONB NOT NULL
- retry_count INT NOT NULL DEFAULT 0
- created_at TIMESTAMPTZ NOT NULL
- published_at TIMESTAMPTZ NULL

audit_logs
- id UUID PK
- case_pk UUID NULL REFERENCES cases(id) ON DELETE RESTRICT
- operator_id UUID NULL
- action VARCHAR NOT NULL
- old_state/new_state JSONB
- request_id/ip VARCHAR NULL
- created_at TIMESTAMPTZ NOT NULL
```

关键索引包括：`cases(case_status, created_at)`、`cases(decision_status, assigned_to)`、`evidences(file_hash)`、`evidences(perceptual_hash)`、`agent_executions(case_pk, agent_name)`、`outbox_events(published_at, created_at)`。

业务和审计数据 MVP 保留 180 天，凭证保留 90 天后删除对象存储文件并保留脱敏摘要；涉及争议案件时暂停清理。数据库业务记录使用 `ON DELETE RESTRICT`，禁止级联删除审计和资金记录。

## 23. API 契约

统一响应：

```json
{"code":"SUCCESS","message":"ok","data":{},"request_id":"REQ001"}
```

核心接口：

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/v1/auth/login` | 登录 |
| POST | `/api/v1/auth/refresh` | 刷新令牌 |
| POST | `/api/v1/cases` | 创建工单，成功返回 202 |
| GET | `/api/v1/cases` | 分页查询 |
| GET | `/api/v1/cases/{case_id}` | 当前完整状态 |
| POST | `/api/v1/cases/{case_id}/evidences` | 补传凭证 |
| GET | `/api/v1/cases/{case_id}/timeline` | 执行轨迹 |
| GET | `/api/v1/cases/{case_id}/events` | SSE |
| POST | `/api/v1/cases/{case_id}/review` | 人工审批 |
| GET | `/api/v1/cases/{case_id}/refund` | 查询退款状态 |

审批请求：

```json
{"action":"APPROVE","comment":"凭证有效","expected_version":3}
```

批准响应只能返回 `refund_status=PENDING`，不得提前返回 `SUCCEEDED`。

错误码至少包括：`CASE_NOT_FOUND`、`CASE_STATE_CONFLICT`、`CASE_PROCESSING`、`EVIDENCE_INVALID`、`AGENT_TIMEOUT`、`AGENT_OUTPUT_INVALID`、`PERMISSION_DENIED`、`DUPLICATE_REQUEST`、`REFUND_RESULT_UNKNOWN`、`SYSTEM_TEMPORARILY_UNAVAILABLE`。

## 24. 文件上传与数据安全

- 单文件不超过 10 MB，每工单最多 8 个文件；
- 允许 JPEG、PNG、WEBP 和 PDF；
- 同时校验扩展名、MIME 和文件头；
- 接入恶意文件扫描，扫描失败或服务不可用时不进入自动决策；
- 服务端生成随机存储名，禁止使用用户路径，拒绝路径穿越；
- 对象存储默认私有，只通过短时签名 URL 访问；
- 删除 EXIF 定位信息；OCR、聊天截图和模型输入进行手机号、地址等脱敏；
- 日志禁止记录密码、JWT、完整地址、完整聊天原文和支付凭证；
- 模型供应商不可用或不符合数据策略时转人工，不上传敏感原图。

## 25. 异常处理

| 异常 | 处理 |
|---|---|
| OCR/VLM/LLM 超时 | 最多重试 2 次；仍失败输出 `UNKNOWN` 并转人工 |
| Schema 解析失败 | 修复 1 次；仍失败转人工 |
| Fraud Provider 缺失超过 2 项 | Fraud `UNKNOWN`，转人工 |
| Redis 不可用 | 停止审批恢复、任务发布和自动退款，返回暂不可用 |
| Worker 崩溃 | PEL + `XAUTOCLAIM` 恢复 |
| Outbox 发布失败 | 保留未发布状态并重试 |
| 支付明确失败 | Refund `FAILED`，工单挂起 |
| 支付超时 | Refund `UNKNOWN`，查询渠道结果，禁止盲目重试 |

## 26. 审计与可观测性

结构化日志至少包含 `request_id`、`case_id`、`agent_name`、`event_type`、`duration_ms`、`retry_count`、状态迁移和脱敏后的错误信息。指标至少包含 API P95、错误率、队列堆积、PEL 数量、Agent 成功率、人工转交率、退款成功率、`UNKNOWN` 数量和 Outbox 延迟。

## 27. 性能指标与压测基线

演示型 MVP 基线：4 vCPU、8 GB RAM；PostgreSQL 和 Redis 各单实例；预置 10 万工单；100 个并发用户；持续 15 分钟；读写比例 8:2；排除 Agent 推理、文件上传、SSE 和 Refund Worker。

- 创建工单同步阶段 P95 < 300 ms；
- 普通查询 P95 < 300 ms；
- REST 吞吐目标 QPS ≥ 200；
- 错误率 < 1%；
- 审批并发测试必须达到 0 Lost Update、0 Duplicate Refund。

目标未在上述基线实测前只能标记为“待验证”，不得宣称已达成。

## 28. 核心验收场景

1. 350 元、低风险、证据有效：转人工；批准后创建一次退款任务，退款成功后工单完成。
2. 128 元、低风险、证据有效：Decision 为 `AUTO_REFUND`；退款成功后才完成。
3. 重复凭证风险 0.95、总分低于 0.70：`force_hit=true`，必须转人工。
4. OCR 最终低于 0.60：允许补传；超过 2 次转人工。
5. 两名主管同时审批：只有一个更新成功，另一个得到状态冲突或重复结果。
6. 同一退款任务重复投递：数据库和渠道均只产生一次退款。
7. 支付请求超时但渠道实际成功：状态先为 `UNKNOWN`，查询后收敛为 `SUCCEEDED`，不重复退款。
8. 数据库提交成功、Redis 发布失败：Outbox 重试后任务最终送达。
9. Worker 崩溃：消息由 `XAUTOCLAIM` 恢复，已完成任务不重复执行。
10. SSE 断线且事件已被裁剪：前端收到 `snapshot_required` 并恢复完整状态。

## 29. 细粒度 WBS

### 29.1 任务一文档：1.5 人日

| WBS | 工作项 | 人日 | 验收物 |
|---|---|---:|---|
| 1.1 | 范围、角色、术语冻结 | 0.20 | 范围与术语表 |
| 1.2 | Agent Schema 与 Evidence 边界 | 0.25 | Agent Spec |
| 1.3 | Fraud、Decision 与退款链路 | 0.25 | 决策和资金规格 |
| 1.4 | 四维状态机与异常路径 | 0.20 | 状态迁移表 |
| 1.5 | DB、API、Outbox 与并发设计 | 0.30 | DB/API Spec |
| 1.6 | 安全、验收、压测与风险 | 0.20 | 验收清单 |
| 1.7 | 一致性检查与冻结 | 0.10 | V4 定稿 |

### 29.2 演示型 MVP：1 人 10 个工作日

| 日期 | 工作项 | 验收点 |
|---|---|---|
| Day 1 | 工程骨架、Compose、PostgreSQL、Redis | 健康检查可用 |
| Day 2 | 用户、JWT、RBAC、Case API | 登录和工单 CRUD |
| Day 3 | 上传安全、Evidence 存储、OCR/VLM Adapter | 文件路由通过 |
| Day 4 | Intake、Evidence Schema 和补传 | Evidence 边界用例通过 |
| Day 5 | Fraud Provider、Fraud、Sentiment | 评分可复现 |
| Day 6 | LangGraph、Decision、Checkpoint、HITL | 挂起恢复通过 |
| Day 7 | Outbox、Streams、Agent Worker、SSE | 重启和重连通过 |
| Day 8 | Refund Worker、模拟支付、幂等和对账 | 重复任务只退一次 |
| Day 9 | React Dashboard 和审批面板 | 业务闭环可操作 |
| Day 10 | 自动化测试、Locust、Compose、文档 | 10 类验收场景通过 |

如限制为 5 天，只交付单机演示子集，不承诺生产级安全、真实支付和完整模型效果。

## 30. 项目目录

```text
multi-agent-refund-system/
├── frontend/src/{pages,components,services,types}
├── backend/
│   ├── app/
│   │   ├── api/
│   │   ├── agents/
│   │   ├── graph/
│   │   ├── models/
│   │   ├── schemas/
│   │   ├── repositories/
│   │   ├── services/
│   │   └── core/
│   ├── workers/{agent_worker,refund_worker,outbox_publisher}.py
│   └── tests/
├── docs/
├── locust/
├── docker-compose.yml
└── README.md
```

## 31. 风险清单

| 风险 | 影响 | 缓解措施 |
|---|---|---|
| OCR/VLM 误判 | 错误自动化 | 严格 Schema、阈值、补传、人工兜底 |
| 强风险被总分稀释 | 高危案件自动退款 | `force_hit` 独立判断 |
| 消息与数据库不一致 | 丢任务 | Transactional Outbox |
| 重复消费 | 重复执行 | 业务唯一键、状态检查、幂等 Worker |
| 支付超时状态不明 | 重复退款 | `UNKNOWN`、渠道查询、禁止盲重试 |
| Redis 单点 | 队列和状态不可用 | MVP 安全停机；生产阶段使用高可用方案 |
| 敏感凭证泄露 | 合规风险 | 私有存储、签名 URL、脱敏、保留周期 |
| 单人排期过紧 | 质量下降 | 10 天演示计划，优先 P0 闭环 |

## 32. 原问题修复对照表

| 原问题 | 修复方案 | 文档位置 | 验收方法 |
|---|---|---|---|
| Fraud 强规则可被总分绕过 | 增加 `force_hit` 和风险等级联合判断 | 9.3、11、13 | 构造总分低但重复风险高用例 |
| `case_id` 被同时用作 UUID 和业务号 | 统一 `id/case_id/case_pk` | 6、22 | 全局搜索禁止 `case_id UUID FK` |
| 乐观锁参数混淆 | 按业务 `case_id` 更新 | 21 | 并发审批测试 |
| 决策等同退款成功 | 新增 Refund Worker 与退款状态 | 15、16、22 | 支付超时和重复投递测试 |
| 锁 TTL 可能提前失效 | 短事务锁、续租及多层兜底 | 21 | 延迟与并发测试 |
| 物流证据要求含糊 | 明确商品图且面单/物流图 | 10 | 缺少任一类型时验证 |
| Agent 拓扑不一致 | 固定 Intake→Evidence→并行→Decision | 8 | 执行轨迹检查 |
| 客户端幂等键可信度不足 | 三类幂等键分离 | 21 | 修改 Header 后重放请求 |
| Streams 恢复规则不完整 | PEL、XAUTOCLAIM、持久化重试 | 18 | 杀死 Worker 后恢复 |
| SSE 只重连不补偿 | Stream ID、重放、快照兜底 | 20 | 裁剪后断线重连 |
| 状态维度不足 | 四维状态及合法迁移 | 16 | 状态迁移单元测试 |
| DB 与消息双写不一致 | Transactional Outbox | 19、22 | 模拟 Redis 故障 |
| 上传和认证安全不足 | 明确限制、脱敏和认证策略 | 3、24 | 安全用例检查 |
| QPS 缺少测量基线 | 固定硬件、数据量和流量模型 | 27 | Locust 报告 |
| 一周排期不现实 | 调整为 10 日演示型 MVP | 29 | 按日验收产物 |

## 33. 定稿结论

本系统采用“Agent 分析、规则决策、独立退款执行、异常人工兜底”的技术路线。自动退款决策只代表允许发起退款，真实资金完成必须以 Refund Worker 和支付渠道确认结果为准。

V4 已统一标识、Agent 拓扑、四维状态、数据库和 API 语义，并通过 Transactional Outbox、消费幂等、短事务锁、乐观锁、退款唯一约束和渠道幂等构成资金安全链路。本文档冻结后可进入 MVP 编码阶段；任何字段、状态或资金规则变更必须同步更新 Schema、迁移、API、测试和本文档。
