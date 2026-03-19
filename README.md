# anki-remote-api

远程建卡 API 服务，为 Anki 提供标准化的 HTTP 接口，支持通过 Discord skill 等外部调用方远程创建和更新卡片。

## 目标

- 接收结构化词卡数据（词、释义、例句、音标等）
- 按用户路由到对应的 Anki container
- 查重（dedup）后决定新建或合并更新（upsert）
- 通过 AnkiConnect 写入 Anki Desktop

## 架构

```
Discord skill
     ↓
Binding DB / Registry
     ↓
Per-user API service  ← 本项目
     ↓
AnkiConnect
     ↓
Anki Desktop
```

每个用户对应一个独立 container，隔离 profile / media / config / token。

## 技术栈

- Python + FastAPI
- AnkiConnect（Anki addon）
- SQLite（template registry）
- Docker（per-user container）

## 快速开始

> WIP — Phase 1 进行中

## 文档

- [v0 实现方案](docs/v0-design.md)
- [TODO](docs/todo.md)
