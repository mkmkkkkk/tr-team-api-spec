# Tensor Revive · Team Workspace API Spec

> 本仓库是 TR team workspace 后端 API 的**单一真相源** (single source of truth). 每个 member 的 agent (Kui / Guo / ...) 看这一份 spec 调用. 实现 / 演进 / 弃用都先改这里, 再改服务端代码.
>
> **不放在 Bootstrap MD 里, 因为 endpoint 会演进, Bootstrap MD 应保持稳定不动**.

---

## 0. 鉴权

所有 endpoint 走同一个机制:

```
Header:  X-Member-Token: <your_token>
  或
Query:   ?token=<your_token>
```

Token 在 namespace 之间隔离: `kui` token 只能访问 `/api/team/kui/...`. 错配 → `403`.

---

## 1. 数据归属决策树 (重要)

Kui agent 拿到一个 "存数据" / "读数据" 任务时, **先判断归属**, 再选 endpoint:

```
任务: 持久化某条数据
  │
  ├─ 是不是纯 UI 临时态? (scroll, current tab, theme)
  │     → localStorage (浏览器自带, 无需 API)
  │
  ├─ 是不是 mutable + 跨设备? (task done flag, draft, 偏好设置)
  │     → § 4 KV state
  │
  ├─ 是不是写一次永远不改的记录? (汇报, blocker, idea)
  │     → § 2 submit (append-only)
  │
  ├─ 是不是 Kui 自己整理的大文件? (PDF/图片/SQLite > 1MB)
  │     → repo /data/ (commit + push)
  │
  └─ 是不是想读 Michael 内部数据? (订单/财务/KPI)
        → § 5 primitive menu (受白名单约束)
```

**默认选 KV state**, 除非明显不合适. 不要在 KV 和 repo /data/ 之间双写 — 选一个.

---

## 2. Submit (append-only event log)  · **LIVE**

```
POST /api/team/{member}/submit
Headers: X-Member-Token: <your_token>
Body:
  {
    "type": "<event type, free>",
    "payload": { <freeform JSON> }
  }
Response:
  {"ok": true, "id": "<8-char hex>"}
```

落到 Michael Mac 本地 `helm/state/{member}_inbox.jsonl`, append-only, 不可改不可删.

预定义 type (你跟 Michael 一起约的): `task` / `report` / `blocker` / `idea` / `qa_result`.
自由扩展任何新 type, backend 不验 schema.

---

## 3. Inbox (拉 Michael 推给你的通知) · **LIVE**

```
GET /api/team/{member}/inbox
Headers: X-Member-Token: <your_token>
Response:
  {"items": [{"id":..., "ts":..., "type":..., "payload":...}, ...]}
```

读 Michael Mac 本地 `helm/state/{member}_outbox.jsonl`.

---

## 4. KV State (mutable cross-device 状态) · **PLANNED v1.1**

为什么需要: task done flag / draft / "上次选的 view" 这类东西.
- submit 是 append-only, 不适合
- repo /data/ commit-and-push 太慢
- localStorage 不跨设备

### 4.1 写 (upsert)

```
POST /api/team/{member}/state
Headers: X-Member-Token: <your_token>
Body:
  { "key": "<dotted.path.key>", "value": <any JSON> }
Response:
  {"ok": true}
```

### 4.2 读

```
GET /api/team/{member}/state              → {"state": {<all kv>}}
GET /api/team/{member}/state?key=task.T01  → {"key": "task.T01", "value": <val>}
```

### 4.3 删

```
DELETE /api/team/{member}/state?key=task.T01
Response: {"ok": true}
```

### 4.4 约束
- 单 member KV blob ≤ 1 MB (软限)
- value 必须 JSON-serializable
- 每次写自动 append 一行到 `{member}_state_history.jsonl` (审计, 你看不到, Michael 可读)
- 无 LLM 在路径上, 纯 KV, 0 prompt injection 风险

---

## 5. Primitive Menu (拉 Michael 内部数据) · **SPEC ONLY, v1: 仅 `menu.list`**

### 5.1 为什么是 menu 而不是自由请求

你不能描述 "我想要 X 数据" 让 Michael agent 帮你实现 — 那样有 prompt injection 风险. 必须从**白名单 menu** 组合.

后端是纯确定性 Python, **无 LLM 在路径**. 字段不在白名单 → 400, 没空间注入.

### 5.2 统一入口

```
POST /api/team/{member}/primitive
Headers: X-Member-Token: <your_token>
Body:
  { "primitive": "<name>", "args": { ... } }
```

### 5.3 Primitives

| Primitive | 状态 | 用途 |
|---|---|---|
| `menu.list` | **v1 LIVE** | 拉当前可用白名单 |
| `data_view.create` | v2 PLANNED | 创建数据视图 endpoint |
| `data_view.query` | v2 PLANNED | 查已创建的视图 |
| `notification.subscribe` | v2 PLANNED | 订阅 Michael 端事件 (event 触发 push 到 inbox) |
| `notification.unsubscribe` | v2 PLANNED | 取消订阅 |

### 5.4 第一步: 拉菜单

```
POST /api/team/kui/primitive
Body: {"primitive": "menu.list", "args": {}}

Response:
  {
    "fields": ["customer_id", "order_amount", ...],   // PUBLISHED_FIELDS
    "events": ["new_inquiry", "device_alert", ...],   // PUBLISHED_EVENTS
    "version": "<日期>"
  }
```

v1 时 `fields`/`events` 可能是空数组 — 表示 Michael 还没发布任何数据. 此时你要做 "拉数据" 的任务, 让 Kui 微信 Michael 申请加白名单, 一次性, 之后永久可用.

### 5.5 第二步 (v2): 组合 primitive

待 v2 实现. 详见本仓库后续 commit.

### 5.6 红线

- ❌ 自由 text 描述需求 (e.g. `{"primitive": "i_want_X"}`) → 400
- ❌ `fields` 写白名单外字段 → 400
- ❌ SQL-injection 风格 `where` (`"1 OR 1=1"`) → 400, where 只接受结构化运算符 `eq/gte/lte/in`

---

## 6. 错误码

| Code | 含义 |
|---|---|
| 400 | 请求格式错 / 字段不在白名单 / 越界 args |
| 401 | token 错或缺 |
| 403 | token 跟 URL 上的 member 不匹配 |
| 404 | 不存在的 endpoint / primitive name |
| 413 | KV blob 超 1MB / submit payload 过大 |
| 501 | endpoint 未实现 (`PLANNED` 状态) |

---

## 7. 版本与演进

- 本 README 即 spec, 每次更新加 changelog 行
- 你的 agent 可定期 `git pull` 看是否有 endpoint 升级
- v1 = submit / inbox / state / primitive.menu.list
- v2 = primitive.data_view.* / primitive.notification.*
- 弃用先 deprecate 6 周再删

## Changelog

- 2026-05-23 — v1 初稿. submit / inbox LIVE. state / menu.list PLANNED. data_view / notification SPEC ONLY.
