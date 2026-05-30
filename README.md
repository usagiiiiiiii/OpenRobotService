# OpenRobotService · FacAssist

> 基于微信服务号的工业移动机器人项目服务平台
> 关注服务号 **FacAssist**，即可在线咨询、提交工单、协同解决现场问题，加速项目交付。

[![License: AGPL v3](https://img.shields.io/badge/License-AGPL_v3-blue.svg)](./LICENSE)
[![Python](https://img.shields.io/badge/Python-3.11+-blue.svg)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.111-009688.svg)](https://fastapi.tiangolo.com/)
[![Vue](https://img.shields.io/badge/Vue-3-42b883.svg)](https://vuejs.org/)

---

## 🚀 立即使用（推荐）

无需任何部署，**微信扫码关注服务号 FacAssist，即刻开始使用**：

<p align="center">
  <img src="./docs/assets/facassist-qrcode.jpg" alt="FacAssist 服务号二维码" width="200" />
  <br/>
  <b>微信扫一扫，关注「FacAssist」</b>
</p>

关注后即可：

- 🗣️ **在线咨询** —— 直接在服务号里咨询工业移动机器人（AGV / AMR 等）相关问题
- 📋 **提交工单** —— 把现场问题登记成工单，系统自动"摇人"对接处理
- 💬 **协同跟进** —— 在工单中与工程师、项目经理实时讨论，疑难问题一键上报
- 🔔 **消息通知** —— 处理进展通过微信实时推送，不错过任何回复

> **FacAssist 是本项目的官方公共服务实例**，由项目维护者运营，开箱即用、持续更新。
> 这就是大多数用户需要的全部——扫码关注即可，下方的部署内容仅面向想自建实例的开发者。

---

## 这是什么

**OpenRobotService**（公共实例：**FacAssist**）是一个面向工业移动机器人项目交付场景的微信服务号平台，目标是**高效"摇人"解决问题、加快项目交付**：现场遇到问题 → 微信里提交工单 → 自动派给对应工程师/项目经理 → 工单内协作讨论 → 疑难一键上报领导 → 闭环解决。

### 核心业务流程

```
用户(任意角色) ──微信服务号菜单 / H5──> 咨询问题 / 提交工单
                                          │
                                          ▼
                              工单进入系统，自动/手动派单
                                          │
                  ┌───────────────────────┼───────────────────────┐
                  ▼                        ▼                        ▼
            转发给处理人员           工单内多方讨论            上报上级领导
            (微信模板消息通知)        (评论时间线留痕)          (升级 escalation)
                  │                        │                        │
                  └───────────────────────┴───────────────────────┘
                                          ▼
                                    工单关闭 / 交付完成
```

### 角色与权限

> **提交工单、参与讨论是所有角色的通用能力**；角色差异体现在派单、上报接收范围、可见工单范围和管理权限上。

| 角色 | 说明 | 主要能力 |
|------|------|----------|
| `customer` | 客户 / 现场人员 | 提交工单、咨询、参与讨论、查看自己提交的工单 |
| `engineer` | 实施工程师 | 提交工单、接单处理、转派、讨论、上报、查看指派给自己的工单 |
| `manager` | 项目经理 | 提交工单、派单/转发、处理、讨论、上报、查看本项目工单 |
| `leader` | 上级领导 | 提交工单、接收上报、决策、讨论、查看全局工单 |
| `admin` | 系统管理员 | 全部权限、用户与项目管理 |

---

## 🛠️ 自建部署（进阶 / 自托管）

> 以下内容**仅面向开发者**——如果你想私有化部署、用自己的微信服务号运行、或二次开发。
> 普通使用者请直接关注上方的 **FacAssist** 服务号，无需阅读本节。

本项目以 [AGPL v3](./LICENSE) 开源，支持完全自托管。

### 技术栈

| 层次 | 技术 |
|------|------|
| 后端 | Python 3.11+ · FastAPI · SQLAlchemy 2.0 · Alembic |
| 数据库 | MySQL 8.x (utf8mb4) |
| 微信对接 | wechatpy（消息回调 / 自定义菜单 / OAuth 网页授权 / 模板消息） |
| 前端 | Vue 3 · Vite · Vant（移动端 H5，微信内打开） |
| 认证 | 微信 OAuth + JWT |

### 目录结构

```
OpenRobotService/
├── backend/                 # FastAPI 后端
│   ├── app/
│   │   ├── api/             # 路由层
│   │   ├── core/           # 配置、安全、依赖
│   │   ├── models/         # SQLAlchemy 模型
│   │   ├── schemas/        # Pydantic 模型
│   │   ├── services/       # 业务逻辑层
│   │   ├── wechat/         # 微信服务号对接
│   │   └── main.py
│   ├── alembic/            # 数据库迁移
│   ├── tests/              # pytest 测试
│   └── requirements.txt
├── frontend/               # Vue3 H5 前端
├── docs/                   # 文档（架构 / 微信配置 / 部署）
├── scripts/                # 运维脚本
└── README.md
```

### 快速开始

详细步骤见 [docs/SETUP.md](./docs/SETUP.md)。简要：

```bash
# 1. 后端
cd backend
python -m venv .venv && source .venv/Scripts/activate   # Windows Git Bash
pip install -r requirements.txt
cp .env.example .env          # 填写数据库与微信配置
alembic upgrade head          # 建表
uvicorn app.main:app --reload # 启动，访问 http://127.0.0.1:8000/docs

# 2. 前端
cd frontend
npm install
npm run dev
```

### 文档

- [架构设计](./docs/ARCHITECTURE.md)
- [本地搭建与部署](./docs/SETUP.md)
- [微信服务号配置](./docs/WECHAT.md)

---

## 贡献

欢迎贡献！请阅读 [CONTRIBUTING.md](./CONTRIBUTING.md)。

## 许可证

本项目采用 **[GNU AGPL v3](./LICENSE)** 协议开源。

任何基于本项目的修改和衍生作品都必须以 AGPL v3 协议开源——**即使只是把它部署成网络服务（SaaS）对外提供，也必须向用户公开完整源代码**。

[AGPL v3](./LICENSE) © 2026 dhualai
