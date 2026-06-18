# 实现规格书（IMPLEMENTATION_SPEC）

> 由 `qiq-tech2spec` 产出。**这是后续编码的唯一真理来源**。任何含糊、暧昧、留待“工程师自行决定”的描述都不合格。
>
> 本模板共 **10 个主体章节 + 2 个附录**：1–10 为必填主体章节；附录 A/B 分别承载交付自检与待确认问题。任一主体章节缺失或必填表格为空 → 规格书交付不合格。
>
> **交付闭环（硬约束）**：方案中明确要求的每个功能 / 字段 / 错误码 / 流程 / 异常路径 / 参数都必须在本规格书内给出可直接编码的决策。**禁止**出现 `TODO` / `FIXME` / `待补充` / `待定` / `后续 PR` / `下一期补齐` / `留待工程师决定` / `先这样后面优化` / `建议后续补充` 等延迟决策措辞。信息缺失时只允许两种出口：(a) 基于仓库证据 + 合理默认值一次性给出确定性决策并在附录 B 记录取值来源；(b) 停下向用户提聚焦澄清问题。
>
> **总纲**：规格书运行在“最简充分方案 + 可扩展但不预先实现”原则下：在能完整满足方案中**功能与非功能需求**的前提下，选取**最少组件 / 最少中间件 / 最少状态 / 最少抽象层级**的实现路径；沿已明确的演进轴保留接口 / 配置口子，但不提前编码、不预留未使用的字段 / 枚举 / 错误码 / 接口 / 抽象层。违反本总纲的设计必须在 §10 中被明确需求举证驱动，否则交付不合格。

- 来源方案：{{原始方案路径/标题/版本}}
- 仓库证据：{{未提供 / 已只读查看，证据见各章节}}
- 覆盖率报告：`.qiqskills/<方案名>/SPEC_COVERAGE.md`（与本规格书同步产出）
- 产出时间：{{datetime}}

---

## 1. 项目结构定义

### 1.1 目录树

```text
{{完整目录树，每个目录附 1 行职责说明}}
project-root/
├── cmd/
│   └── server/
│       └── main.go              # HTTP 服务启动入口（复用 / 新增）
├── internal/
│   ├── domain/                  # 领域层（复用 / 新增）
│   │   ├── entity/
│   │   ├── valueobject/
│   │   └── repository/          # 仓储接口（仅接口）
│   ├── application/             # 应用服务层
│   ├── infrastructure/          # 基础设施层
│   │   ├── persistence/
│   │   ├── cache/
│   │   └── rpc/
│   └── interfaces/              # 接入层
│       ├── http/
│       └── consumer/
├── pkg/
├── configs/
├── scripts/sql/
└── ...
```

每个目录右侧标注「**复用已有** / **新增**」。如复用仓库现状，应给出证据路径；如无仓库证据，写“未提供仓库证据，按方案显式设计”。

### 1.2 命名规范

- 包名：{{...}}
- 文件名：{{...}}
- 接口/接收器：{{...}}
- 错误码：{{...}}
- API 路径：{{...}}

### 1.3 与仓库现状的偏离

> 列出所有偏离当前仓库可观察现状的设计决策与原因。无仓库证据时写“未提供仓库证据，无法判断偏离”。无偏离写“无”。

- 偏离 1：{{...}}（原因：{{...}}，证据：{{文件路径 / 未提供}}）

### 1.4 改动文件清单（**必填，不允许为空**）

> 把方案/规格书涉及的全部新增 / 修改文件逐行列出。空表 / 仅 1 行示意均视为不合格。该表用于让后续实现方理解落点，但本 skill 不继续拆任务或写代码。

| 文件路径 | 变更 | 一句话职责 | 关联规格书章节 |
|---|---|---|---|
| `internal/domain/entity/order.go` | ➕ 新增 | Order 实体定义、状态枚举、状态转换 | §3.Order |
| `internal/application/order_service.go` | ✏ 修改 | 新增 CreateOrder 方法 | §5.CreateOrder |
| `scripts/sql/0007_create_t_order.sql` | ➕ 新增 | t_order 建表 | §2.1 DDL |
| {{...}} | {{➕/✏}} | {{...}} | {{...}} |

---

## 2. 数据存储详设（**必填，不允许为空**）

### 2.1 关系数据库（DDL）

对方案涉及的**每张表**给出**完整 DDL**（必须可执行），并先完成本 skill 内置 SQL 规则子集自检与修正（规则见 `references/02-sql-self-check-rules.md`）。若不涉及关系数据库，写明“不涉及关系数据库，原因：{{...}}”。

```sql
-- t_xxx：xxx 主表
-- 对应方案 §9.x
CREATE TABLE IF NOT EXISTS `t_xxx` (
    `id`         BIGINT       UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `xxx_no`     VARCHAR(32)           NOT NULL DEFAULT '' COMMENT '业务唯一编号',
    `user_id`    BIGINT       UNSIGNED NOT NULL DEFAULT 0 COMMENT '用户ID',
    `status`     TINYINT      UNSIGNED NOT NULL DEFAULT 1 COMMENT '状态：1=已创建',
    `amount`     BIGINT                NOT NULL DEFAULT 0 COMMENT '金额，单位：分',
    `version`    INT                   NOT NULL DEFAULT 1 COMMENT '乐观锁版本号',
    `created_at` DATETIME(3)           NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
    `updated_at` DATETIME(3)           NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_xxx_no` (`xxx_no`(32)),
    KEY `idx_user_id_created_at` (`user_id`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='xxx 主表';
```

### 2.1.1 SQL 规范自检与修正记录（**必填，不允许为空**）

> 对规格书中所有 DDL/DML/查询 SQL 执行自检。规则来源为本 skill 自带的 `references/02-sql-self-check-rules.md`；MUST 必须全部修正，重要 RECOMMEND 未采纳必须写明原因。若不涉及 SQL / DB，写“不涉及 SQL / DB”。

| SQL/表/章节 | MUST 检查结果 | 重要 RECOMMEND 检查结果 | 修正动作 | 未采纳推荐项原因 |
|---|---|---|---|---|
| `t_xxx` / §2.1 | ✅ 保留字、命名、主键、索引列 NOT NULL、注释、禁止外键/ENUM/FLOAT/DOUBLE 均通过 | ✅ InnoDB、utf8mb4、默认值、必备字段、索引命名、字符序均通过 | 已补齐列 COMMENT、表 COMMENT、DEFAULT、字符序 | 无 |
| `{{SQL/表/章节}}` | {{✅/❌，逐项列出 MUST 未解决项；必须为 0}} | {{✅/⚠️，逐项列出重要推荐项}} | {{...}} | {{无 / 业务原因 + 风险接受人}} |

最小自检清单：

- **MUST**：禁止保留字；命名小写下划线；建库字符集显式且仅 `utf8`/`utf8mb4`；建表显式主键；主键 `BIGINT UNSIGNED`；索引列 `NOT NULL`；禁止 `FLOAT`/`DOUBLE`/`ENUM`；禁止外键；表和列都有 `COMMENT`；禁止 `DROP DATABASE`/`DROP TABLE`/`DROP COLUMN`；`INSERT` 显式字段；`UPDATE`/`DELETE` 带 `WHERE`；业务 `SELECT` 带筛选条件；JOIN 表数 ≤ 3；禁止存储过程/`CALL`；索引设计不走极端。
- **重要 RECOMMEND**：`ENGINE=InnoDB`；`DEFAULT CHARSET=utf8mb4`；字符序避免 `*_bin`；`NOT NULL` 列有 `DEFAULT`；时间列有默认值；必备字段 `id`/创建时间/更新时间；索引命名 `uk_`/`idx_`；库表列名长度 ≤ 32；时间字段后缀规范；避免 `TEXT`/`BLOB`；业务唯一字段建唯一索引；`VARCHAR` 索引指定前缀长度；避免 `SELECT *`、索引列函数、左/全模糊 `LIKE`、大 `IN`、`ORDER BY RAND()`；计数用 `count(*)`；超大分页优化；敏感字段加密；SQL 参数化。

### 2.2 Redis 键空间表

对方案涉及的**每一类 Redis key**给出。若不涉及 Redis，写明“不涉及 Redis，原因：{{...}}”。

| key 模板 | 数据类型 | TTL | 写入者 | 读取者 | 数据语义 | 一致性约束 |
|---|---|---|---|---|---|---|
| `bgagent:lease:{bot_id}` | Hash | 60s | reconcile loop | sweeper / scheduler | 当前持有者租约 | 真源；写顺序：先 DB 再 Redis |
| `bgagent:cred:{bot_id}` | String(JSON) | 300s | dispatcher | runtime | bot 凭证缓存 | 缓存非真源 |
| `bgagent:hot:{worker_id}` | Set | — | scheduler | sweeper | 节点 hot bot 集合 | 与 lease 双写最终一致 |
| {{...}} | {{...}} | {{...}} | {{...}} | {{...}} | {{...}} | {{...}} |

### 2.3 数据一致性约定

- **真源声明**：哪些数据 MySQL 是真源，哪些 Redis 是真源；不一致时以真源为准。
- **写顺序**：明确多源写入的顺序（先写 X 再写 Y）以及失败时的补偿策略。
- **降级模式**：Redis 不可用时降级到何种行为；DB 不可用时降级到何种行为。
- **幂等键**：跨服务/跨写入的幂等键命名与生命周期。

### 2.4 DDL/键空间演进策略

- 表结构变更使用 expand → migrate → contract 三段式，禁止破坏性 ALTER。
- Redis key 改名时必须双写期；TTL 调整要兼容旧消费者。

---

## 3. 核心数据结构定义

### 3.1 实体：{{EntityName}}

- **数据库表名**：`{{table_name}}`（与 §2.1 DDL 对应）
- **分表策略**：{{无 / 按 user_id % 64}}
- **字符集/引擎**：{{utf8mb4 / InnoDB}}

#### 字段表

| 字段名 | Go 类型 | DB 类型 | 必填 | 默认值 | 索引 | 说明 |
|---|---|---|---|---|---|---|
| id | int64 | BIGINT | 是 | 自增 | PK | 主键 |
| order_no | string | VARCHAR(32) | 是 | - | UNIQUE | 订单号 |
| user_id | int64 | BIGINT | 是 | - | idx_user_id | 用户ID |
| status | int8 | TINYINT | 是 | 1 | - | 订单状态 |
| amount | int64 | BIGINT | 是 | - | - | 金额（单位：分） |
| created_at | time.Time | DATETIME(3) | 是 | CURRENT_TIMESTAMP(3) | - | 创建时间（UTC） |
| updated_at | time.Time | DATETIME(3) | 是 | CURRENT_TIMESTAMP(3) | - | 更新时间（UTC） |
| version | int32 | INT | 是 | 1 | - | 乐观锁版本 |

#### 枚举

```go
type OrderStatus int8

const (
    OrderStatusCreated  OrderStatus = 1 // 已创建
    OrderStatusPaid     OrderStatus = 2 // 已支付
    OrderStatusShipped  OrderStatus = 3 // 已发货
    OrderStatusComplete OrderStatus = 4 // 已完成
    OrderStatusCanceled OrderStatus = 5 // 已取消
)
```

#### 状态机

合法转换：

```text
Created   → Paid       (支付成功)
Created   → Canceled   (用户/超时取消)
Paid      → Shipped    (商家发货)
Paid      → Canceled   (退款)
Shipped   → Complete   (确认收货)
```

非法转换：返回错误 `ErrInvalidStatusTransition`。

> 重复以上结构，为每个实体定义。

---

## 4. 接口契约定义

### 4.1 接口：{{接口名称}}

- **路径/方法**：`POST /api/v1/orders`
- **认证**：需要登录，从 Header `X-User-Id` 获取
- **限流**：单用户 10 次/秒

#### 请求体

| 字段 | 类型 | 必填 | 校验 | 说明 |
|---|---|---|---|---|
| product_id | int64 | 是 | >0 | 商品ID |
| quantity | int32 | 是 | [1,999] | 数量 |
| address_id | int64 | 是 | >0 | 收货地址ID |
| coupon_id | int64 | 否 | >=0 | 优惠券ID（0 表示不使用） |
| idempotency_key | string | 是 | UUID v4 | 幂等键 |

#### 响应体（成功）

```json
HTTP 200
{
  "code": 0,
  "message": "success",
  "data": {
    "order_no": "ORD20240101000001",
    "amount": 9900,
    "status": 1
  }
}
```

#### 错误码表

| code | HTTP | message | 说明 | 处理建议 |
|---|---|---|---|---|
| 10001 | 400 | invalid parameter | 参数校验失败 | 检查请求 |
| 10002 | 409 | duplicate request | 重复请求 | 查询订单 |
| 20001 | 400 | product not found | 商品不存在 | — |
| 20002 | 400 | insufficient stock | 库存不足 | 提示用户 |
| 50001 | 500 | internal error | 系统错误 | 重试 |

#### 幂等性

- 幂等键：`idempotency_key`
- 存储：Redis SETNX，key=`idempotent:{user_id}:{idempotency_key}`，TTL 24h
- 重复请求返回首次结果。

> 重复以上结构，为每个接口定义。

---

## 5. 核心流程伪代码（**至少 3 段；关键流程必须出现**）

> 关键流程指方案中带 mermaid 时序图、带状态机、或在故障表中被引用的流程。每段伪代码必须含：步骤、异常分支、事务边界、回滚、并发约束。若原始方案关键流程不足 3 段，本规格书必须从原方案明确提及但未展开的关键流程中闭环补齐伪代码并与 §2/§4/§6 一致；若某流程与本次交付范围无关，在本节首部一句说明“本方案仅包含 X 段关键流程，原因：...”。**不允许**出现“建议后续补充”等措辞。

### 5.1 流程：{{流程名}}

```text
func {{FuncName}}(ctx, req) (resp, err):
    // Step 1: 参数校验
    validate(req)

    // Step 2: 幂等检查
    existing = redis.GET("idempotent:{user_id}:{idempotency_key}")
    if existing != nil:
        return existing, ErrDuplicateRequest

    // Step 3: 查询商品（RPC，超时 1s，重试 2 次）
    product = productService.GetProduct(ctx, req.product_id)
    if product == nil:
        return nil, ErrProductNotFound

    // Step 4: 计算价格
    amount = product.price * req.quantity
    if req.coupon_id != 0:
        coupon = couponService.GetCoupon(ctx, req.coupon_id, user_id)
        if coupon == nil or coupon.expired:
            return nil, ErrCouponNotAvailable
        amount = applyCoupon(amount, coupon)

    // Step 5: 扣减库存（RPC 幂等，使用 order_no 作为幂等键）
    err = inventoryService.Deduct(ctx, req.product_id, req.quantity, order_no)
    if err != nil:
        return nil, ErrInsufficientStock

    // Step 6: 创建订单 + outbox（数据库事务）
    BEGIN TRANSACTION
        orderRepo.Create(ctx, order)
        outboxRepo.Create(ctx, outboxMsg)
    COMMIT
    // 失败时：调用 inventoryService.Rollback(ctx, order_no)；
    //        Rollback 失败时记录补偿日志，由定时任务兜底

    // Step 7: 设置幂等结果
    redis.SET("idempotent:{user_id}:{idempotency_key}", resp, TTL=24h)
    // 失败时不影响主流程，仅记日志（数据库唯一约束兜底）

    // Step 8: 返回结果
    return resp, nil
```

#### 异常分支

| 步骤 | 异常类型 | 处理方式 |
|---|---|---|
| Step 3 | RPC 超时 | 返回 50001，由调用方重试 |
| Step 5 | 库存不足 | 返回 20002，**不**回滚（未发生写） |
| Step 6 | DB 事务失败 | 调用 inventoryService.Rollback；失败则补偿日志 |
| Step 7 | Redis 失败 | 仅记日志，不影响主流程 |

> 重复以上结构，为每个核心流程定义。

---

## 6. 边界与异常路径（**必填，不允许为空**）

> 与方案“故障与异常路径汇总”一一映射。本表 < 1 行视为不合格。若原始方案未明示列出异常路径，本规格书必须从方案描述、伪代码、接口错误码表、仓库现有实践中推导并闭环补齐（超时 / 重试 / 依赖不可用 / 并发冲突 / 幂等冲突 / 反序列化失败 / 参数越界 等常见项）并在本表给出完整决策；**不允许**以“建议后续补充”代替。

| 异常项 | 触发条件 | 检测方式 | 处理动作 | 降级模式 | SLO 影响 | 对应方案章节 |
|---|---|---|---|---|---|---|
| 心跳超期重扫 | worker 心跳延迟 > 14s | reconcile loop 周期扫描 | 标记节点失联、抢占接管 | 重新分配热 bot | 接管延迟 ≤ 30s | §5.x / §故障表 |
| Pub/Sub 消息丢失 | redis pubsub 重连 | scheduler 周期 fallback 拉取 | 兜底拉取最新指令 | 5min 内补齐 | 指令延迟 ≤ 5min | §5.x |
| Redis 全挂 | 连接失败 N 次 | health check | 切只读模式，禁用调度 | 不接受新建 bot | 仅写降级 | §5.x |
| 心跳协程 panic | recover 捕获 | runtime recover | 重启协程，重新注册 lease | 自愈无外部干预 | 接管延迟一次 | §5.x |
| 凭证 -14 错误 | bot 凭证失效 | runtime 上报 | 触发凭证刷新 + bot 重扫 | 单 bot 阻塞 | 单 bot 影响 | §5.x |
| {{...}} | {{...}} | {{...}} | {{...}} | {{...}} | {{...}} | {{...}} |

---

## 7. 模块依赖关系图

```text
interfaces/http
    └── application/order_service
              ├── domain/entity
              ├── domain/repository (interface)
              ├── infrastructure/persistence (impl)
              ├── infrastructure/cache
              └── infrastructure/rpc/{product,inventory}_client
```

### 实现层次

| Layer | 内容 |
|---|---|
| Layer 0 | domain/entity, domain/valueobject |
| Layer 1 | domain/repository（接口） |
| Layer 2 | infrastructure/*（实现 repository 接口） |
| Layer 3 | application/* |
| Layer 4 | interfaces/* |

> 同 Layer 内可并行；跨 Layer 严格串行。

---

## 8. 配置与运行参数（**必填，不允许为空**）

### 8.1 配置 Schema

| 配置 key | 类型 | 必填 | 默认值 | 来源 | 说明 |
|---|---|---|---|---|---|
| `server.port` | int | 是 | 8080 | configs/config.yaml | HTTP 端口 |
| `mysql.dsn` | string | 是 | — | env `MYSQL_DSN` | DSN |
| `mysql.max_open` | int | 否 | 50 | configs/config.yaml | 连接池上限 |
| `redis.addr` | string | 是 | — | env `REDIS_ADDR` | Redis 地址 |
| `redis.db` | int | 否 | 0 | configs/config.yaml | Redis DB |
| `bgagent.heartbeat_ttl` | duration | 否 | 60s | configs/config.yaml | 租约 TTL |
| `bgagent.sweeper_interval` | duration | 否 | 30s | configs/config.yaml | sweeper 周期 |
| {{...}} | {{...}} | {{...}} | {{...}} | {{...}} | {{...}} |

### 8.2 环境变量映射

| ENV | 对应配置 key | 说明 |
|---|---|---|
| `MYSQL_DSN` | `mysql.dsn` | 优先于配置文件 |
| `REDIS_ADDR` | `redis.addr` | 同上 |
| {{...}} | {{...}} | {{...}} |

### 8.3 监控指标

| metric 名 | 类型 | 标签 | 语义 | SLO 关联 |
|---|---|---|---|---|
| `bgagent_takeover_latency_seconds` | Histogram | `worker_id` | 接管耗时 | §SLO.1 ≤ 30s |
| `bgagent_lease_active_total` | Gauge | `worker_id` | 活跃租约数 | 容量 |
| `bgagent_credential_refresh_total` | Counter | `reason` | 凭证刷新次数 | — |
| {{...}} | {{...}} | {{...}} | {{...}} | {{...}} |

---

## 9. 编码约束清单

> **核心约束**摘自 [@references/03-coding-constraints-common.md](../references/03-coding-constraints-common.md) 与 [@references/04-coding-constraints-go.md](../references/04-coding-constraints-go.md)；**风格类约束**优先沿用当前仓库可观察到的现状，仓库未提供证据时由本规格书显式声明。

### 9.1 类型与单位（核心）

- [ ] 金额字段使用 `int64`，单位“分”。
- [ ] 时间使用 `time.Time`（UTC），DB 列 `DATETIME(3)`。
- [ ] ID 使用 `int64`。
- [ ] 枚举使用 `type Foo int8` + 显式常量。

### 9.2 超时与外部调用（核心）

- [ ] DB 操作超时 {{3}} 秒。
- [ ] RPC/HTTP 调用超时 {{1}} 秒。
- [ ] 仅幂等下游允许重试；本方案需重试的下游：{{...}}。
- [ ] 所有 I/O 传 `context.Context`，生产路径无 `context.Background()` / `context.TODO()`。

### 9.3 幂等与并发（核心）

- [ ] 写操作幂等。
- [ ] 乐观锁使用 version 字段，更新时校验 `RowsAffected`。
- [ ] for 循环内禁止单条 DB/RPC 调用（必要的批量接口已在 §4 接口契约中声明）。

### 9.4 错误处理（核心）

- [ ] 错误使用 `fmt.Errorf("...: %w", err)` 包装。
- [ ] 不吞错；忽略必须注释说明。
- [ ] 业务错误使用 sentinel 或 typed error，与 §4 错误码表一一对应。

### 9.5 nil 安全（核心）

- [ ] 返回指针/接口/map 的函数，调用方必须先判 nil 再解引用。
- [ ] map 写入前必须确保已初始化。
- [ ] 类型断言必须使用 `v, ok := x.(T)` 形式。
- [ ] JSON / RPC 反序列化的指针字段使用前必须判 nil。
- [ ] `errors.As` 的目标变量必须先初始化。

### 9.6 风格沿用 / 显式选型

> **禁止凭空声明风格选型**。能从仓库观察到的项必须写证据；观察不到的项必须写“未发现仓库证据，规格书显式选型原因”。

| 维度 | 规格书决策 | 证据 / 引入原因 |
|---|---|---|
| 日志库 | {{zap / slog / logrus / ...}} | {{证据文件 / 未发现，原因...}} |
| 日志字段命名 | {{snake_case / camelCase / ...}} | {{...}} |
| trace_id 注入方式 | {{...}} | {{...}} |
| HTTP 框架 | {{gin / echo / 标准库 / ...}} | {{...}} |
| HTTP 响应格式 | {{...}} | {{...}} |
| 中间件顺序 | {{...}} | {{...}} |
| ORM/SQL 库 | {{gorm v2 / sqlx / ent / ...}} | {{...}} |
| 事务封装 | {{闭包 / 手写 Begin/Commit / ...}} | {{...}} |
| 错误返回风格 | {{`(T, error)` / `*AppError` / ...}} | {{...}} |
| 错误码风格 | {{int / iota typed / 字符串常量}} | {{...}} |
| 测试 mock 库 | {{gomock / testify mock / ...}} | {{...}} |
| 测试断言库 | {{testify / 标准库 / ...}} | {{...}} |
| 配置加载 | {{viper / envconfig / 自研 / ...}} | {{...}} |
| 包/文件/接收器命名 | {{...}} | {{...}} |

### 9.7 安全（核心）

- [ ] 所有外部输入参数校验。
- [ ] SQL 参数化。
- [ ] 错误消息对外脱敏。
- [ ] 敏感字段日志中脱敏。

### 9.8 测试（核心）

- [ ] 每个产出有单元测试。
- [ ] 覆盖正常 + 异常 + 边界。
- [ ] mock 在接口层（具体 mock 库见 §9.6）。

### 9.9 依赖版本约束

- Go {{1.21+}}
- MySQL {{8.0}}
- Redis {{7.0}}
- {{其他}}

---

## 10. 简洁性与可扩展性自检（**必填，不允许为空**）

> 本章是“最简充分方案 + 可扩展但不预先实现”原则在规格书里的落点。不允许以“不涉及”跳过；方案足够简单时也必须填写“本方案仅引入的组件 / 抽象层 与 需求驱动的对应关系”。
>
> 优先级：正确性 » 可验证性 » 简单性 » 可扩展性 » 性能极致。“可扩展”仅意味为未来保留**口子**（接口 / 配置 / DDL expand-migrate-contract / 错误码空间），**不**意味为未来提前**实现**。

### 10.1 复杂度驱动表（必填，≥ 1 行）

列出规格书相对于“极简仅可跑通功能”所**额外引入的每一个**组件 / 中间件 / 抽象层 / 并发原语 / 状态 / 预留字段 / 预留接口，并给出严格的需求举证。未被需求驱动者必须删除。

| 引入项 | 类别 | 被哪条功能 / 非功能需求驱动 | 需求出处 | 如果去掉会发生什么 | 更简单的替代方案及为何不选 |
|---|---|---|---|---|---|
| Redis 分布式锁 | 中间件 | 同一 bot 全局唯一接管者（强一致性需求） | 原始方案 §x.x | 双节点同时接管，违反 SLO§2 | DB 全局唯一索引；QPS 不够（已在 §8 记录设计容量） |
| {{引入项}} | {{组件/中间件/抽象层/并发原语/状态/预留字段/预留接口}} | {{需求一句话}} | {{原方案 §×.×}} | {{具体后果}} | {{替代方案 + 拒绝原因}} |

出现以下任一项即不合格：

- 表空、仅 1 行示意、驱动需求未能指向原始方案具体章节。
- 出现“为未来预留”“为后期可能需求预留”“额外预留 1～2 字段防止后期改动”“为了状态机可扩展多加中间状态”这类措辞。

### 10.2 YAGNI 自检清单（必填）

> 逐项勾选。任何一项为“是”都必须先删减后重新交付。

- [ ] 未出现未被 §3 / §4 / §5 / §6 引用的**预留字段 / 预留枚举值 / 预留错误码 / 预留接口 / 预留配置项**。
- [ ] 未引入本阶段 §8 SLO / 容量 / 一致性约束未要求的中间件（消息队列 / 分布式锁 / 缓存 / 调度器）。
- [ ] 未为**唯一实现**提前抽象出接口 / 工厂 / 策略（Rule of Three：不到三次不抽象）。
- [ ] 未在表结构中加入无当前使用场景的 `extra` / `ext` / `reserve` / `feature_flags` JSON 字段。
- [ ] 未加入与本方案 SLO 不相关的**重试 / 降级 / 熟成度开关 / 多级缓存**等机制。
- [ ] 未以“后续可能需要”为理由拆分本可一体实现的模块 / 服务。
- [ ] 依赖选型上，如仓库现状已能覆盖本方案需求，没有引入新依赖；确需引入的已在 §1.3 记录偏离原因。

### 10.3 可扩展轴声明（必填、但不提前实现）

> 仅以“接口契约 / 配置开关 / DDL expand-migrate-contract / 错误码空间” 4 种轻量形式声明未来可能的演进轴。**不允许**在本版本实现任何演进轴上的预留代码（包括预留抽象接口 / 预留实现类）。

| 可扩展轴 | 本版本仅做的事 | 将来报反时怎么动 |
|---|---|---|
| 付款渠道 | 只实现微信 | 新增渠道时：`payment.Channel` 枚举加值 + 新增适配器实现；表结构无需变更 |
| 存储后端 | 只实现 MySQL | 不预留 Repository 抽象；如需替换后端时再提取接口 |
| {{可扩展轴}} | {{本版本实现范围}} | {{未来报反时只动字段 / 只动配置 / 加列 / 新增错误码}} |

#### 10.4 复杂度 vs 需求 一轮决策纪要

> 必填。选型上出现过犹豫（例如“单库事务 vs Saga”、“DB 幂等 vs Redis SETNX”）的决策点都需交叉记录。若本方案所有选型都唯一且明显，写“本方案选型唯一且明显”。

| 决策点 | 选择 | 备选 | 拒绝备选的原因 |
|---|---|---|---|
| {{事务一致性}} | {{单库事务}} | {{Saga / 2PC / TCC}} | {{本版本 QPS / 一致性等级不需要}} |
| {{幂等实现}} | {{DB 唯一键}} | {{Redis SETNX}} | {{避免多组件占用}} |

---

## 附录 A. 规格书交付自检

- [ ] 10 个必填章节全部存在且非空。
- [ ] §1.4 改动文件清单不是空表，也不是示意占位。
- [ ] §2.1.1 SQL 自检完成；MUST 未解决项 = 0，或明确“不涉及 SQL / DB”。
- [ ] §5 核心流程覆盖方案中的关键流程，异常分支和事务边界明确。
- [ ] §6 边界与异常路径覆盖原始方案故障表；原始方案未详列部分已在本规格书内闭环补齐，未出现“建议后续补充”。
- [ ] §8 配置、环境变量、监控指标齐全。
- [ ] §9 编码约束均可机械验证。
- [ ] §10 复杂度驱动表 ≥ 1 行，每个额外引入都被方案具体需求举证驱动；YAGNI 清单逐项过关；未出现“为未来预留”代码。
- [ ] **闭环 grep**：规格书全文未出现 `TODO`/`FIXME`/`待补`/`待定`/`后续 PR`/`下一期`/`留待`/`先这样后面优化`/`建议后续补充` 等措辞；方案要求的全部功能都在本规格书内闭环。
- [ ] 已同步产出 `SPEC_COVERAGE.md`，未覆盖项 = 0 或已显式豁免（豁免类型仅限“风险接受”或“方案显式不实现”）。

## 附录 B. 待用户确认问题

> 本附录**仅**用于记录依赖业务侧人工决策、本规格书无法从方案 + 仓库证据 + 合理默认值推导出的关键参数（如业务阈值、授权人、安全合规选型）。
>
> 不允许在本附录记录“后续 PR 实现”“迭代补齐”“下一期优化”类任务型表项。每项都必须给出一个**本版本默认决策**（用于用户不响应时推进实现）、指出响应后需修改的具体章节，并表明仅依靠该项不会阻塞本规格书交付。若无问题写“无”。

| 问题 | 影响章节 | 本版本默认决策 | 需谁确认 | 确认后需改动的章节 |
|---|---|---|---|---|
| {{...}} | {{§X}} | {{默认选 A，原因：...}} | {{业务 / 安全 / 运维}} | {{§X.x}} |
