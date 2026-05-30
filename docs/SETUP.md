# 本地搭建与部署

## 环境要求

- Python 3.11+
- MySQL 8.x
- Node.js 18+（前端）

## 1. 数据库准备

确保 MySQL 已启动，并创建数据库（utf8mb4）：

```sql
CREATE DATABASE IF NOT EXISTS openrobotservice
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

> Windows 用户可直接运行仓库内 `scripts/setup_mysql.ps1`（管理员 PowerShell）一键完成安装与建库。

## 2. 后端

```bash
cd backend

# 创建虚拟环境
python -m venv .venv
# 激活（Windows Git Bash）
source .venv/Scripts/activate
# 激活（Windows PowerShell）
# .venv\Scripts\Activate.ps1
# 激活（Linux/Mac）
# source .venv/bin/activate

# 安装依赖
pip install -r requirements.txt

# 配置环境变量
cp .env.example .env
# 编辑 .env，填写数据库密码、JWT 密钥、微信参数

# 执行数据库迁移（建表）
alembic upgrade head

# （可选）初始化种子数据：管理员、示例项目
python -m app.seed

# 启动开发服务器
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

启动后访问交互式 API 文档：http://127.0.0.1:8000/docs

## 3. 环境变量说明（.env）

| 变量 | 说明 | 示例 |
|------|------|------|
| `DATABASE_URL` | MySQL 连接串 | `mysql+pymysql://root:123456@127.0.0.1:3306/openrobotservice?charset=utf8mb4` |
| `JWT_SECRET` | JWT 签名密钥（务必改成随机长串） | `change-me-to-a-random-secret` |
| `JWT_EXPIRE_MINUTES` | Token 有效期（分钟） | `10080` |
| `WECHAT_APP_ID` | 服务号 AppID | `wx1234567890abcdef` |
| `WECHAT_APP_SECRET` | 服务号 AppSecret | `xxxxxxxx` |
| `WECHAT_TOKEN` | 服务器配置 Token | `your_token` |
| `WECHAT_AES_KEY` | 消息加解密 Key（可选） | `xxxx` |
| `WECHAT_OAUTH_REDIRECT` | OAuth 回调地址 | `https://your.domain/api/wechat/oauth/callback` |
| `FRONTEND_BASE_URL` | 前端 H5 地址 | `https://your.domain` |
| `TPL_ASSIGN_ID` | 派单通知模板 ID | `xxx` |
| `TPL_COMMENT_ID` | 讨论通知模板 ID | `xxx` |
| `TPL_ESCALATE_ID` | 上报通知模板 ID | `xxx` |

> 没有微信参数也能跑：微信相关功能会降级（通知打印到日志），核心工单流程不受影响，可用 `/api/auth/dev-login` 登录联调。

## 4. 前端

```bash
cd frontend
npm install
npm run dev    # 开发，默认 http://127.0.0.1:5173
npm run build  # 生产构建，产物在 dist/
```

前端通过 `VITE_API_BASE`（见 `frontend/.env`）指向后端地址。

## 5. 运行测试

```bash
cd backend
pytest -v
```

## 6. 本地开发联调

没有公网域名、没有微信环境也能完整开发本平台的核心业务：

### 降级模式（无微信参数）

若 `.env` 未填写微信参数，系统自动进入**开发降级模式**：

- 消息回调 / OAuth / 模板消息相关接口不会真正调用微信
- 通知内容打印到后端日志（而非真正推送）
- 可用 `POST /api/auth/dev-login` 以任意角色获取 JWT，联调全部业务流程

```bash
# 以工程师角色快速登录拿 token（仅开发环境可用）
curl -X POST http://127.0.0.1:8000/api/auth/dev-login \
  -H "Content-Type: application/json" \
  -d '{"nickname":"测试工程师","role":"engineer"}'
```

这样在没有微信服务号的情况下也能完整开发、演示工单全流程。

### 内网穿透（需要真机微信联调时）

要在微信里真机测试（消息回调、OAuth、菜单跳转），需把本机 8000 端口暴露到公网 HTTPS。可用 frp / ngrok / cpolar 等工具：

```bash
# 以 cpolar 为例
cpolar http 8000
# 得到形如 https://xxxx.cpolar.io 的公网地址，
# 填入微信公众平台「服务器配置 URL」和 .env 的 WECHAT_OAUTH_REDIRECT
```

微信服务号的正式对接步骤见 [WECHAT.md](./WECHAT.md)。

## 7. 生产部署建议

- 后端用 `gunicorn -k uvicorn.workers.UvicornWorker app.main:app` 多进程运行
- 前置 Nginx 反向代理，配置 HTTPS（微信服务号要求 HTTPS）
- 微信服务器配置 URL 指向 `https://your.domain/api/wechat/callback`
- 前端 `npm run build` 后由 Nginx 托管 `dist/`
