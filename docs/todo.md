# anki-remote-api v0 TODO

## A. 预研 / 基础

- [ ] 确认容器内运行 Anki + AnkiConnect 的最小可行方案
- [ ] 确认 API service 到 AnkiConnect 的调用方式与健康检查
- [ ] 确认 AnkiConnect 查询 note / 更新 note 所需接口

## B. API service 骨架

- [ ] 初始化 FastAPI 项目
- [ ] 实现 `/health`
- [ ] 增加 AnkiConnect client 封装
- [ ] 增加配置加载（token / anki 地址 / storage）
- [ ] 增加认证中间件

## C. Deck API

- [ ] `GET /v0/decks`
- [ ] `POST /v0/decks`
- [ ] `POST /v0/decks/ensure`

## D. Template API

- [ ] 定义 template 存储模型
- [ ] 实现 `vocab-basic` 初始模板
- [ ] `GET /v0/templates`
- [ ] `GET /v0/templates/{id}`
- [ ] `POST /v0/templates`
- [ ] `PATCH /v0/templates/{id}`
- [ ] （可选）`POST /v0/templates/{id}/render-preview`

## E. Note API

- [ ] 定义 note payload 模型
- [ ] 实现 canonical_term 归一化
- [ ] `POST /v0/notes/lookup`
- [ ] `POST /v0/notes`
- [ ] `PATCH /v0/notes/{note_id}`
- [ ] `POST /v0/notes/upsert`

## F. Merge 逻辑

- [ ] meanings 去重合并
- [ ] examples 去重合并
- [ ] tags 合并
- [ ] phonetic/audio replace 策略
- [ ] 渲染为 Anki fields 的 HTML 输出

## G. 容器化

- [ ] 设计 container 目录结构
- [ ] 编写 Dockerfile / compose 示例
- [ ] 挂载独立 profile / media / config
- [ ] 配置 API service 与 AnkiConnect 联通
- [ ] 容器启动健康检查测试

## H. Skill 接入

- [ ] 设计 binding DB 结构
- [ ] 提供 binding 查询接口或查询逻辑
- [ ] 在 skill 中按 `discord_user_id` 路由
- [ ] skill 调用 `/v0/notes/upsert`
- [ ] skill 返回用户可读结果

## I. 测试

- [ ] deck create/list 集成测试
- [ ] note create 集成测试
- [ ] note lookup/update 集成测试
- [ ] upsert created/updated 路径测试
- [ ] merge 去重测试

---

## 优先级

### P0
- 单用户 container 跑通
- deck API
- note create/find/update
- `vocab-basic` 模板
- upsert

### P1
- template CRUD
- render-preview
- binding DB 接入 skill

### P2
- 更完善的媒体策略
- 多模板支持
- 自动化部署/复制 container
