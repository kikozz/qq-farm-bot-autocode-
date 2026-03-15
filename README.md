# QQ Code + Farm Bot 

## 1. 总体架构

本套系统分为 3 个部分：

1. `qq-code-api`
   - 接收 QQ 小程序上报的 `code`
   - 提供“查询最新 code”接口

2. `qq-farm-bot-ui`
   - QQ 农场 bot 面板和核心服务
   - 管理账号、运行 bot、记录下线日志

3. `qq-code-sync`
   - 每分钟检查一次 bot 状态
   - 只有当账号因“长时间未操作 / 被踢下线 / 登录失效”等异常停掉时
   - 才会从 `qq-code-api` 拉最新 `code`
   - 自动更新账号并重新启动

---

## 2. 域名和端口

### 对外域名

- `https://api.****.com`
  - 用途：小程序上报 `code`，服务器查询最新 `code`

- `https://bot.****.com`
  - 用途：QQ 农场 bot 面板

### 本机监听端口

- `qq-code-api`
  - `127.0.0.1:3001`

- `qq-farm-bot-ui`
  - `127.0.0.1:3000`

---

## 3. 当前工作流

### 3.1 小程序端

小程序脚本会：

1. 启动时立即执行一次 `qq.login()`
2. 获取到 `code` 后上传到：
   - `https://api.****.com/api/qq-login-callback`
3. 后续随机每 `3 ~ 4` 分钟刷新一次 `code`
4. 每次刷新后再次上报

### 3.2 服务器端

`qq-code-api` 会：

1. 接收小程序上报的 `code`
2. 保存最新 `code`
3. 提供接口：
   - `GET /api/latest-qq-code?token=...`

### 3.3 bot 恢复逻辑

`qq-code-sync` 每分钟执行一次：

1. 登录 `qq-farm-bot-ui`
2. 检查目标账号是否仍在运行
3. 如果账号仍在运行：
   - 不做任何处理
4. 如果账号已停止：
   - 检查最近日志是否包含以下异常：
     - 长时间未操作
     - 请重新登录
     - 请更新 Code
     - `kickout_stop`
     - `ws_400`
5. 如果命中异常：
   - 从 `api.****.com` 获取最新 `code`
   - 调用 `qq-farm-bot-ui` 更新账号 `code`
   - 调用 `qq-farm-bot-ui` 启动账号

---

## 4. 目录结构

```text
/
├─ srv/
│  └─ apps/
│     ├─ qq-code-api/
│     │  ├─ app.py
│     │  ├─ venv/
│     │  └─ data/
│     │     └─ latest_code.json
│     │
│     ├─ qq-code-sync/
│     │  ├─ sync_code.py
│     │  ├─ config/
│     │  │  └─ sync.env
│     │  └─ data/
│     │     └─ state.json
│     │
│     └─ qq-farm-bot-ui/
│        ├─ core/
│        ├─ web/
│        ├─ data/
│        └─ ...
│
├─ etc/
│  ├─ systemd/
│  │  └─ system/
│  │     ├─ qq-code-api.service
│  │     ├─ qq-code-sync.service
│  │     ├─ qq-code-sync.timer
│  │     └─ qq-farm-bot-ui.service
│  │
│  └─ nginx/
│     ├─ sites-available/
│     │  ├─ api.****.com
│     │  ├─ bot.****.com
│     │  └─ cpa.****.com
│     │
│     └─ sites-enabled/
│        ├─ api.****.com
│        └─ bot.****.com
│
└─ var/
   └─ log/
      └─ nginx/
