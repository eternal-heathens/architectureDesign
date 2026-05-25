# 模型结果消息广播与查询接口技术架构

## 1. 背景与目标

系统在接收模型处理结果后，需要广播处理完成消息，并提供结果查询接口。下游通过 MQ 获得任务终态通知，再按 `taskId` 查询状态或结果。

核心目标：

- 模型处理完成后可靠通知多个下游。
- 查询接口以持久化结果为事实来源。
- 避免“结果已落库但消息没发出”和“消息已发但结果不可查”。
- 支持重复回调、重复消息、乱序消息、发送失败、消费失败和查询重试。
- 明确一致性边界、安全边界、状态机、版本规则和补偿机制。

## 2. 架构原则

- **结果先持久化，事件后广播**：只有结果或失败信息写入事实存储后，才允许产生终态事件。
- **MQ 只广播元信息**：消息只包含任务、状态、版本、追踪和幂等字段，不承载完整结果。
- **查询以存储为准**：MQ 不是事实来源，查询接口从数据库或对象存储回源。
- **至少一次投递 + 幂等**：不追求跨 DB、MQ、下游的端到端 exactly-once。
- **Outbox 保证双写可靠性**：任务状态和待发布事件在同一数据库事务内提交。
- **首期优先简单可控**：先做稳定闭环，缓存、CDC、多级归档等能力后续演进。

## 3. 总体架构

```text
模型处理服务
  -> 结果接收服务 Result Receiver
  -> 任务结果表 / 结果存储
  -> Outbox 事件表
  -> Outbox Publisher
  -> MQ Topic / Exchange
  -> 下游消费者

查询方
  -> API Gateway
  -> 查询服务 Query Service
  -> 任务结果表 / 对象存储
```

推荐首期方案：

```text
关系型数据库任务表
+ 小结果 DB / 大结果对象存储
+ Transactional Outbox
+ MQ 终态事件广播
+ 状态查询接口
+ 结果查询接口
```

## 4. 系统边界

系统内部负责：

- 接收模型结果或失败回调。
- 校验任务、租户、签名、状态和版本。
- 持久化任务终态、结果摘要、结果内容或结果地址。
- 生成并发布 MQ 终态事件。
- 提供任务状态查询和结果查询。
- 提供重试、补偿、重放、审计和观测能力。

系统不负责：

- 保证所有下游一定消费成功。
- 替下游处理业务幂等。
- 在 MQ 中承载完整结果。
- 把 MQ 消息当成访问授权凭证。

一致性边界：

- 结果表更新与 Outbox 写入：同库事务强一致。
- Outbox 到 MQ 发布：最终一致。
- 下游收到消息到查询结果：最终一致，但终态事件发出前结果必须已经可查。

## 5. 任务状态机

首期建议状态：

```text
CREATED     已创建，尚未开始
PROCESSING  模型处理中
SUCCEEDED   处理成功，结果已持久化且可查询
FAILED      处理失败，失败信息已持久化且可查询
CANCELED    任务取消
EXPIRED     结果过期，不再返回完整结果
```

不建议首期同时保留 `SUCCEEDED` 和 `RESULT_AVAILABLE`。如果结果成功就代表已经可查，那么 `RESULT_AVAILABLE` 会增加语义重叠和测试复杂度。

允许流转：

```text
CREATED -> PROCESSING
PROCESSING -> SUCCEEDED
PROCESSING -> FAILED
PROCESSING -> CANCELED
SUCCEEDED -> EXPIRED
FAILED -> EXPIRED
```

版本规则：

- 如果任务只允许一次完成，`resultVersion = 1`。
- 如果允许重跑，由任务管理服务生成 `attemptNo/resultVersion`。
- 模型侧只能回传服务下发的 `resultVersion`，不能自行递增。
- `taskId + resultVersion + resultHash` 相同视为幂等。
- `taskId + resultVersion` 相同但 `resultHash` 不同视为冲突，进入异常处理。

## 6. 核心模块设计

### 6.1 结果接收服务

职责：

- 接收模型处理成功或失败回调。
- 校验 `taskId` 是否存在、是否属于当前租户、状态是否允许流转。
- 校验签名、token、时间戳、防重放参数。
- 处理重复回调、乱序回调和版本冲突。
- 小结果写 DB，大结果写对象存储。
- 在同一事务内更新任务结果并写入 Outbox 事件。

建议内部组件：

```text
ResultReceiverController
  -> ResultCallbackValidator
  -> TaskStateService
  -> ResultStorageService
  -> OutboxEventService
```

### 6.2 结果存储

结果大小策略：

```text
<= 32KB       DB JSON 字段存储并直接返回
32KB ~ 10MB   对象存储，查询接口返回短期签名 URL 或代理下载
> 10MB        只返回元数据和异步下载地址
> 100MB       默认拒绝或进入离线导出流程
```

数据库只保存必要索引、摘要和地址：

- `resultUri`
- `resultHash`
- `resultSize`
- `schemaVersion`
- `expiredAt`
- `traceId`

### 6.3 Outbox 事件模块

事件状态：

```text
NEW
PUBLISHING
PUBLISHED
FAILED_RETRY
DEAD
```

多实例发布建议：

```sql
SELECT *
FROM outbox_event
WHERE status IN ('NEW', 'FAILED_RETRY')
  AND next_retry_at <= NOW()
ORDER BY created_at ASC
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

如果数据库不支持 `SKIP LOCKED`，使用 `locked_by + locked_until` 租约字段。

重试策略：

```text
1s, 5s, 30s, 2min, 10min, 30min
```

超过最大次数进入 `DEAD`，触发告警和人工处理。

### 6.4 MQ Publisher

职责：

- 批量拉取待发布 Outbox 事件。
- 发送到 MQ Topic/Exchange。
- 发送成功后更新发布状态。
- 发送失败后写失败原因、重试次数和下一次重试时间。

关键约束：

- 发布成功但更新 Outbox 状态失败时，允许重复投递。
- 下游必须按 `eventId` 或 `idempotencyKey` 幂等。
- Publisher 自身无状态，可水平扩展。

### 6.5 查询服务

建议拆分两个接口：

```http
GET /api/v1/tasks/{taskId}/status
GET /api/v1/tasks/{taskId}/result
```

状态查询适合高频轮询，结果查询适合终态后拉取结果。

读一致性：

- 首期建议结果查询读主库。
- 如果读从库，发现版本小于消息中的 `resultVersion` 时必须回源主库。
- 下游收到消息后短暂查询失败，需要按指数退避重试。

## 7. 关键流程

### 7.1 成功链路

```text
1. 业务创建任务，生成 taskId/resultVersion
2. 任务状态进入 PROCESSING
3. 模型处理完成并回调结果
4. Result Receiver 校验任务、签名、状态、版本
5. 小结果写 DB 或大结果写对象存储
6. 同一事务内：
   - 更新 task_result 为 SUCCEEDED
   - 写入 outbox_event
   - 写入状态变更日志
7. Outbox Publisher 发布 MODEL_TASK_SUCCEEDED
8. 下游收到消息后按 taskId 查询状态或结果
```

### 7.2 失败链路

失败也应广播终态事件：

```text
MODEL_TASK_FAILED
```

否则下游可能一直等待成功事件，只能靠轮询发现失败。

### 7.3 重复回调

- 同一 `taskId + resultVersion + resultHash`：直接返回幂等成功，不重复写 Outbox。
- 同一 `taskId + resultVersion` 但 hash 不同：标记冲突，进入异常处理。
- 旧版本回调：拒绝或记录审计，不覆盖新版本终态。

### 7.4 MQ 发布失败

- Outbox 保留 `FAILED_RETRY` 状态。
- 后台按退避策略重试。
- 超过阈值进入 `DEAD`。
- 运维接口支持指定 `eventId` 重发。

### 7.5 下游查询不到结果

可能原因：

- 查询读从库存在复制延迟。
- 下游使用了错误租户或权限。
- 结果已过期。
- 对象存储写入异常。

处理策略：

- 查询服务优先读主库或版本不满足时回源主库。
- 下游按指数退避重试。
- 查询接口返回明确状态码和 `traceId`。

## 8. 接口设计

### 8.1 模型结果回调

```http
POST /api/v1/model-tasks/{taskId}/result-callback
```

请求示例：

```json
{
  "taskId": "task_123",
  "tenantId": "tenant_001",
  "resultVersion": 1,
  "status": "SUCCEEDED",
  "result": {},
  "resultHash": "sha256:xxx",
  "schemaVersion": "1.0",
  "traceId": "trace_xxx",
  "completedAt": "2026-05-25T10:00:00Z"
}
```

### 8.2 状态查询

```http
GET /api/v1/tasks/{taskId}/status
```

响应：

```json
{
  "taskId": "task_123",
  "status": "SUCCEEDED",
  "resultVersion": 1,
  "updatedAt": "2026-05-25T10:00:00Z",
  "traceId": "trace_xxx"
}
```

### 8.3 结果查询

```http
GET /api/v1/tasks/{taskId}/result
```

响应：

```json
{
  "taskId": "task_123",
  "status": "SUCCEEDED",
  "resultVersion": 1,
  "schemaVersion": "1.0",
  "result": {},
  "resultUri": null,
  "downloadUrl": null,
  "expiredAt": "2026-06-24T10:00:00Z",
  "traceId": "trace_xxx"
}
```

错误码：

- `202`：任务仍在处理中。
- `403`：无权限。
- `404`：任务不存在。
- `409`：版本冲突或状态非法。
- `410`：结果已过期。
- `429`：请求过于频繁。
- `500`：内部错误。

## 9. MQ 消息契约

建议去 URL 化、去敏感化：

```json
{
  "eventId": "evt_123",
  "eventType": "MODEL_TASK_SUCCEEDED",
  "eventVersion": "1.0",
  "tenantId": "tenant_001",
  "taskId": "task_123",
  "resultVersion": 1,
  "schemaVersion": "1.0",
  "idempotencyKey": "task_123:1:MODEL_TASK_SUCCEEDED",
  "occurredAt": "2026-05-25T10:00:00Z",
  "traceId": "trace_xxx"
}
```

不放：

- 完整模型结果。
- 签名 URL。
- token。
- 敏感业务字段。
- 永久查询地址。

终态事件建议：

```text
MODEL_TASK_SUCCEEDED
MODEL_TASK_FAILED
MODEL_TASK_CANCELED
MODEL_TASK_EXPIRED
```

或统一为：

```text
MODEL_TASK_TERMINATED
status = SUCCEEDED / FAILED / CANCELED / EXPIRED
```

## 10. 数据结构

### 10.1 任务结果表 `task_result`

```text
task_id
tenant_id
business_key
status
result_version
result_inline
result_uri
result_hash
result_size
schema_version
error_code
error_message
created_at
updated_at
completed_at
expired_at
trace_id
version
```

### 10.2 Outbox 表 `outbox_event`

```text
event_id
event_type
event_version
aggregate_type
aggregate_id
tenant_id
task_id
result_version
payload
status
retry_count
next_retry_at
locked_by
locked_until
mq_message_id
last_error
published_at
created_at
updated_at
trace_id
idempotency_key
```

### 10.3 状态变更日志 `task_state_log`

```text
id
task_id
tenant_id
status_before
status_after
result_version
reason
operator_type
trace_id
created_at
```

### 10.4 查询审计表 `result_access_log`

```text
id
task_id
tenant_id
caller_id
access_type
result_version
status_code
trace_id
created_at
```

## 11. 安全设计

- 查询接口必须鉴权。
- 查询按租户、业务方、调用方隔离。
- MQ Topic 根据安全等级做 ACL 或按租户/业务拆分。
- MQ 消息不作为授权凭证。
- 对象存储不暴露永久公开地址。
- 大结果下载使用短期签名 URL 或查询服务代理。
- 所有查询、下载、状态变更和事件重放写审计日志。
- 回调接口需要签名、时间戳和防重放校验。

## 12. 可观测性

核心指标：

- 模型结果接收 QPS、失败率。
- 任务状态流转耗时。
- Outbox 未发布数量。
- Outbox 最老未发布事件年龄。
- MQ 发布成功率、失败率、延迟。
- DLQ 数量。
- 查询接口 QPS、P95/P99、错误率。
- 对象存储上传/下载失败率。

核心日志字段：

- `taskId`
- `tenantId`
- `eventId`
- `traceId`
- `resultVersion`
- `idempotencyKey`
- `statusBefore`
- `statusAfter`
- `consumerGroup`

告警：

- Outbox 堆积超过阈值。
- `PUBLISHING` 超时数量增长。
- `DEAD` 事件出现。
- MQ 发布失败持续增长。
- 查询 5xx 或 404 异常升高。
- 对象存储异常。

## 13. 下游接入规范

下游必须遵守：

- 至少一次投递，消息可能重复。
- 不保证全局顺序。
- 必须按 `eventId` 或 `idempotencyKey` 去重。
- 必须调用查询接口获取结果。
- 查询失败必须指数退避。
- 不得把 MQ 消息作为授权凭证。
- 每个消费组登记负责人和告警方式。
- 消费失败进入下游自己的 DLQ 或补偿队列。

## 14. 性能与容量

主要瓶颈：

- 结果回调写 DB。
- 大结果上传对象存储。
- Outbox 扫描和索引膨胀。
- MQ 发布吞吐。
- 查询接口热点流量。

优化策略：

- Outbox 按时间分区。
- `PUBLISHED` 事件保留 7 到 30 天后归档。
- `DEAD` 事件长期保留或进入异常归档表。
- 查询接口拆分状态查询和结果查询。
- 结果内容首期不缓存，只可短 TTL 缓存状态摘要。
- 大结果走对象存储，查询接口不直接返回超大 JSON。

## 15. 首期实施范围

必须实现：

- 任务结果表。
- 模型结果接收接口。
- 状态机和幂等更新。
- Outbox 表。
- Outbox Publisher。
- MQ 成功/失败终态事件。
- 状态查询接口。
- 结果查询接口。
- 基础监控和告警。
- 人工重发 Outbox 事件能力。

暂不建议首期实现：

- CDC 替代 Outbox。
- 复杂缓存体系。
- 多级冷归档。
- 完整消费者管理平台。
- 复杂跨区域容灾。

## 16. 压测与验收指标

| 指标 | 建议目标 |
|---|---|
| 回调接口 P95 | < 200ms，不含大结果上传 |
| 小结果查询接口 P95 | < 100ms |
| Outbox 发布延迟 P95 | < 5s |
| Outbox 最老未发布年龄 | < 60s |
| MQ 消息体大小 | < 4KB |
| 重复回调幂等成功率 | 100% |
| MQ 重复消息下游去重 | 必须验证 |
| DB 故障恢复后 Outbox 补发 | 必须验证 |

## 17. 架构争议与裁决

| 争议 | 裁决 |
|---|---|
| 是否保留 `RESULT_AVAILABLE` | 首期不保留，`SUCCEEDED` 即结果可查 |
| MQ 是否携带结果 | 不携带，只携带元信息 |
| 是否追求 exactly-once | 不追求，采用至少一次 + 幂等 |
| 是否使用 CDC | 首期不用，先用 Transactional Outbox |
| 查询是否读从库 | 首期结果查询读主库，后续再做版本回源 |
| 消息是否带 `queryUrl` | 不带完整 URL，只带资源定位字段 |
| 失败是否广播 | 广播失败终态事件 |

## 18. 最终结论

推荐采用：

```text
任务服务生成 taskId/resultVersion
-> 模型处理
-> Result Receiver 幂等接收
-> 结果/失败信息持久化
-> 同事务写 Outbox
-> Publisher 广播终态事件
-> 下游按消息元信息查询状态/结果
```

这个方案能平衡可靠性、复杂度、安全和演进能力。首期成败关键不在 MQ 产品本身，而在状态机、版本规则、Outbox 锁定与补偿、查询一致性、下游接入规范这几个工程边界是否做实。
