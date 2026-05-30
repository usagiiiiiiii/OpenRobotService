# 架构设计

## 总体架构

```
                    ┌─────────────────────────────────────┐
                    │          微信服务号 (公众号)          │
                    │   自定义菜单 / 消息 / 模板消息通知    │
                    └───────────────┬─────────────────────┘
                                    │ HTTPS
                    ┌───────────────▼─────────────────────┐
                    │            前端 H5 (Vue3)            │
                    │   工单创建/列表/详情/讨论 · 咨询页    │
                    └───────────────┬─────────────────────┘
                                    │ REST API (JSON)
                    ┌───────────────▼─────────────────────┐
                    │          后端 FastAPI                │
                    │  ┌────────┐ ┌────────┐ ┌──────────┐ │
                    │  │  api   │ │services│ │  wechat  │ │
                    │  └────────┘ └────────┘ └──────────┘ │
                    │  ┌────────┐ ┌────────┐ ┌──────────┐ │
                    │  │ models │ │schemas │ │   core   │ │
                    │  └────────┘ └────────┘ └──────────┘ │
                    └───────────────┬─────────────────────┘
                                    │ SQLAlchemy
                    ┌───────────────▼─────────────────────┐
                    │            MySQL 8.x                 │
                    └─────────────────────────────────────┘
```

## 分层说明

| 层 | 职责 |
|----|------|
| `api/` | HTTP 路由，参数校验，调用 service，返回响应 |
| `services/` | 业务逻辑（工单状态流转、派单、上报、通知触发） |
| `models/` | SQLAlchemy ORM 模型，对应数据库表 |
| `schemas/` | Pydantic 模型，请求/响应数据结构 |
| `core/` | 配置、数据库连接、安全（JWT）、依赖注入 |
| `wechat/` | 微信服务号对接（消息回调、菜单、OAuth、模板消息） |

## 数据模型（ER 概览）

```
users ──┬──< tickets (creator_id)
        ├──< tickets (assignee_id)
        ├──< ticket_comments (author_id)
        ├──< ticket_assignments (from_user_id / to_user_id)
        └──< consultations (user_id)

projects ──< tickets (project_id)

tickets ──┬──< ticket_comments
          └──< ticket_assignments
```

### 核心表

- **users** — 用户。`openid`(微信), `nickname`, `role`(角色), `phone`
- **projects** — 项目。`name`, `code`, `customer_name`, `status`
- **tickets** — 工单。`title`, `description`, `status`, `priority`, `creator_id`, `assignee_id`, `project_id`
- **ticket_comments** — 工单讨论。`ticket_id`, `author_id`, `content`, `comment_type`(普通/上报/系统)
- **ticket_assignments** — 派单记录。`ticket_id`, `from_user_id`, `to_user_id`, `note`
- **consultations** — 咨询。`user_id`, `question`, `answer`, `status`

## 工单状态机

```
  ┌────────┐  派单   ┌────────┐  开始处理 ┌────────┐
  │待派单  ├────────>│已派单  ├──────────>│处理中  │
  │pending │         │assigned│           │handling│
  └────────┘         └────────┘           └───┬────┘
                                              │
              ┌───────────────┬───────────────┼───────────────┐
              ▼               ▼               ▼               ▼
        ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
        │待讨论   │     │已上报   │     │已解决   │     │已关闭   │
        │discussing│    │escalated│     │resolved │     │closed   │
        └─────────┘     └─────────┘     └────┬────┘     └─────────┘
                                             │ 重开
                                             └──────> handling
```

合法状态流转在 `services/ticket_service.py` 的 `ALLOWED_TRANSITIONS` 中定义并强制校验。

## 认证流程

1. 用户在微信内打开 H5 → 后端 `/api/wechat/oauth` 发起网页授权
2. 微信回调带 `code` → 后端用 code 换取 `openid`
3. 后端按 openid 查/建用户 → 签发 JWT 返回前端
4. 前端后续请求携带 `Authorization: Bearer <jwt>`

开发环境可用 `/api/auth/dev-login` 直接以指定角色登录，便于在没有微信环境时联调。

## 通知机制

工单关键动作（派单、新讨论、上报）触发 `wechat/notify.py` 发送**模板消息**给相关人员：
- 派单 → 通知被指派人
- 新讨论 → 通知工单参与者
- 上报 → 通知上级领导
