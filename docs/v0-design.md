# anki-remote-api v0 实现方案

## 1. 目标与范围

### 目标

实现一个面向 Anki 的远程建卡 v0 系统，支持：

- 由 Discord skill 生成结构化卡片数据
- 按用户路由到对应的 Anki container
- 在 container 内通过 `API service + AnkiConnect + Anki` 完成写卡
- 支持 deck 管理、template/schema 管理、note lookup/create/update/upsert

### v0 明确范围

- 单个 container 仅服务一个 Anki 用户
- 系统整体允许通过多个 container 扩展到多用户
- Discord skill 通过 DB 维护 `discord_user_id -> anki_user/container` 绑定
- container 内优先使用 **AnkiConnect addon**，不自研 addon
- 音频/媒体前期只接受外部 URL，不做 TTS 服务
- schema/template 由 API service 自己维护，不依赖 AnkiConnect 提供业务 schema

### v0 不做

- 单实例多租户
- 自研 Anki addon
- 复杂模板迁移
- 平台级自动调度与自动扩缩容
- 高级权限系统
- TTS 生成

---

## 2. 总体架构

```
Discord skill
     ↓
Binding DB / Registry
     ↓
Per-user API service
     ↓
AnkiConnect
     ↓
Anki Desktop
```

### 职责划分

#### Discord skill
- 读取当前 Discord user id
- 从 DB 查询对应的 Anki container/service 信息
- 调用对应 API service
- 把外部生成的结构化词卡数据提交过去

#### Binding DB / Registry
- 存储 `discord_user_id -> anki_user_id/service_url/token/status`
- 后续可扩展为 registry/service discovery

#### Per-user API service
- deck API
- business template/schema API
- lookup/create/update/upsert
- dedupe
- merge/update 规则
- 调用 AnkiConnect 执行底层写入

#### AnkiConnect
- deck list/create
- note create/find/update
- model/fields 基础查询
- tags / 媒体相关底层能力（v0 媒体先不深用）

#### Anki Desktop
- 持有该用户独立 collection/profile
- 最终存储卡片

---

## 3. 容器设计

每个用户一个 container，推荐最小组成：

- `anki`
- `ankiconnect`
- `anki-api-service`

### 推荐容器内组件

1. **Anki Desktop** — 使用独立 profile / collection，挂载独立数据目录
2. **AnkiConnect addon** — 暴露本地 HTTP 接口给 API service 调用
3. **API service** — 推荐 FastAPI，暴露业务层 HTTP API
4. **本地配置/存储** — template registry（JSON / SQLite）、service config

### 每个 container 需要隔离的内容

- collection/profile
- media 目录
- 配置文件
- token
- deck/model 使用状态

---

## 4. 数据模型

### 4.1 Binding DB

表：`anki_user_bindings`

| 字段 | 类型 | 说明 |
|------|------|------|
| `discord_user_id` | string, unique | Discord 用户 ID |
| `anki_user_id` | string | Anki 用户标识 |
| `service_base_url` | string | API service 地址 |
| `service_token` | string | Bearer token |
| `status` | string | `active\|disabled\|pending` |
| `created_at` | datetime | |
| `updated_at` | datetime | |

可扩展字段：`default_template_id`、`default_deck`、`notes`

### 4.2 Template（业务 schema）

这是 API service 维护的业务模板，不直接等于底层 Anki model。

建议字段：

- `id`
- `version`
- `name`
- `description`
- `defaults`：`deck`、`tags`
- `dedupe`：`by: canonical_term`
- `schema`（JSON Schema 风格）
- `mapping`：`anki_model`、`field_map`
- `render_rules`：`meanings -> html`、`examples -> html`

### 4.3 标准词卡模型（v0 主模板：vocab-basic）

```json
{
  "term": "abate",
  "term_type": "word",
  "phonetic": "/əˈbeɪt/",
  "meanings": [
    {
      "pos": "verb",
      "gloss_zh": "减轻；减弱",
      "gloss_en": "to become less strong"
    }
  ],
  "examples": [
    {
      "text": "The storm suddenly abated.",
      "translation": "暴风雨突然减弱了。"
    }
  ],
  "audio_url": "https://example.com/abate.mp3",
  "tags": ["discord", "vocab"],
  "source": {
    "app": "discord",
    "channel_id": "1480800595735482411"
  }
}
```

### 4.4 canonical_term 规则

- trim
- 小写化
- collapse whitespace
- 短语空格归一

例：`Abate`、` abate `、`ABATE` → 统一为 `abate`

---

## 5. API 设计

### 5.1 Health

#### `GET /health`

返回 service、AnkiConnect、Anki 状态。

---

### 5.2 Deck API

#### `GET /v0/decks`
列出可用 decks。

#### `POST /v0/decks`
创建 deck。

请求：
```json
{ "name": "English::Words" }
```

#### `POST /v0/decks/ensure`
确保 deck 存在（不存在则创建）。

返回：
```json
{
  "exists": true,
  "created": false,
  "deck": { "name": "English::Words" }
}
```

---

### 5.3 Template API

#### `GET /v0/templates`
#### `GET /v0/templates/{template_id}`
#### `POST /v0/templates`
#### `PATCH /v0/templates/{template_id}`
#### `POST /v0/templates/{template_id}/render-preview`（可选）

---

### 5.4 Note API

#### `POST /v0/notes/lookup`

按 `template_id + deck + canonical_term` 查重。

请求：
```json
{
  "template_id": "vocab-basic",
  "deck": "English::Words",
  "term": "abate"
}
```

返回：
```json
{
  "found": true,
  "note_id": "1712345678901",
  "fields": { "term": "abate" }
}
```

#### `POST /v0/notes`
新建 note。

#### `PATCH /v0/notes/{note_id}`
更新 note，支持 `replace` 和 `merge` 两种模式。

#### `POST /v0/notes/upsert`

skill 主要调用此接口。

请求：
```json
{
  "template_id": "vocab-basic",
  "deck": "English::Words",
  "update_mode": "merge",
  "note": { ... }
}
```

返回：
```json
{ "action": "created", "note_id": "1712345678901" }
```
或：
```json
{ "action": "updated", "note_id": "1712345678901", "updated_fields": ["meanings", "examples"] }
```

---

## 6. Merge / Update 规则

### replace
直接覆盖字段。适合：`phonetic`、gloss 修正、`audio_url`。

### merge
语义合并。适合：`meanings`、`examples`、`tags`、`source`。

| 字段 | 策略 |
|------|------|
| `meanings` | 按 `pos + gloss_zh + gloss_en` 去重 |
| `examples` | 按 `text + translation` 去重 |
| `tags` | 集合去重 |
| `audio_url` | v0 默认 replace |
| `phonetic` | 默认 replace，非空新值覆盖空旧值 |

---

## 7. 与 AnkiConnect 的边界

| 层 | 负责内容 |
|----|---------|
| AnkiConnect | deck list/create、note create/find/update、tags、model/field 查询 |
| API service | business template/schema、dedupe、canonical_term、merge 规则、render rules、upsert 语义 |

---

## 8. Skill 调用流程

1. skill 获取当前 `discord_user_id`
2. skill 查询 Binding DB，获得 `service_base_url` 和 `service_token`
3. skill 生成结构化词卡 payload
4. skill 调用目标 container 的 `/v0/notes/upsert`
5. service 返回 `created/updated`
6. skill 向用户反馈结果

---

## 9. 技术选型

| 组件 | 选型 | 原因 |
|------|------|------|
| API service | Python + FastAPI | 开发快，pydantic 支持好，易接词典/TTS |
| 存储 | SQLite（per-container templates）| v0 够用，无需额外依赖 |
| Binding DB | 独立，放 skill 所在系统侧 | 解耦 |
| 通信 | skill → API service：HTTP + Bearer token | 简单安全 |
| 通信 | API service → AnkiConnect：本地 HTTP | container 内部 |

---

## 10. 安全边界

- 不直接把 AnkiConnect 暴露给外部 skill
- skill 只访问 API service
- 每个 container 配独立 token
- API service 仅监听容器内部或受控网络

---

## 11. 分阶段实施计划

### Phase 1：底层执行通路
- 起单用户 container，跑通 Anki + AnkiConnect
- API service 实现：health、deck list/create、note create/find/update

### Phase 2：业务模板层
- 定义 `vocab-basic` 模板
- template CRUD
- render rules
- canonical_term

### Phase 3：upsert 能力
- lookup
- merge/update 规则
- `/v0/notes/upsert`

### Phase 4：Discord skill 接入
- Binding DB
- skill 读 binding 并路由
- 返回创建/更新结果

### Phase 5：容器复制与多用户准备
- 标准化 container 镜像与环境变量
- 支持多实例

---

## 12. 最终建议执行顺序

1. 单用户 container + API service + AnkiConnect 通路
2. 固化 `vocab-basic` 模板
3. 实现 `/v0/notes/upsert`
4. 接入 Discord skill 和 Binding DB
