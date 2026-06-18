# 考勤核对系统 · Web 部署文档

> 关联代码：`web/server.py`、`web/static/index.html`
> 入口应用：`web.server:app`（FastAPI 实例）

本文档覆盖三种场景：①本机自用 ②局域网给同事用 ③正式服务器部署。

---

## 0. 依赖

本项目是**标准 uv 项目**：依赖声明在 `pyproject.toml`、版本锁定在 `uv.lock`（两者都已入库）。包括：

```
pyside6               # 桌面 GUI（Web 用不到，但同仓库共用一套环境）
fastapi               # Web 后端
uvicorn[standard]     # Web 服务器
python-multipart      # 文件上传
openpyxl              # 读写 .xlsx（公司数据 / 输出）
xlrd                  # 读 .xls（客户数据）
```

首次/换机一键装齐（按 `uv.lock` 装到项目本地 `.venv/`，跨机器一致）：

```powershell
uv sync
```

国内网慢可临时换源（镜像是本机偏好，不写进仓库）：

```powershell
uv sync --default-index https://pypi.tuna.tsinghua.edu.cn/simple
```

> 规范：本项目 Python 一律经 **uv** 运行；装好后 `uv run python ...` 自动用 `.venv`，
> **不要加 `--no-project`**（加了会绕过 .venv、丢依赖）。机器没有 uv 先装 uv，不要退回裸 python。

---

## 1. 本机自用（最简）

### 方式 A：双击启动器（推荐给非技术用户）

桌面上的 **`启动考勤核对Web.bat`** 双击即可：
1. 弹出一个黑色命令窗口（这是服务进程，**别关**）；
2. 约 3 秒后浏览器自动打开 `http://127.0.0.1:8000`；
3. 用完关掉那个黑窗口即停止服务。

启动器内容（要点）：
```bat
:: 先把工作目录切到你本机 clone 本仓库的位置（每人不同，不写死在此）
cd /d 仓库根目录
start "" /min powershell -WindowStyle Hidden -Command "Start-Sleep 3; Start-Process 'http://127.0.0.1:8000'"
uv run python run_web.py
```

### 方式 B：命令行

```powershell
# 先 cd 到你本机 clone 本仓库的目录（路径因人而异，不写死在此）
uv run python -m uvicorn web.server:app --port 8000
```
然后浏览器访问 `http://127.0.0.1:8000`。

### 开发模式（改代码自动重载）

```powershell
uv run python -m uvicorn web.server:app --reload --port 8000
```
- 改 `web/server.py`（后端）需要 `--reload` 才生效；
- 改 `web/static/index.html`（前端）无需重载，后端每次请求实时读取，浏览器刷新即可。

### 自定义端口 / host（本机隔离，不入库）

端口和 host 是**因人而异的本机配置**——你可能用 Portmanager 分配的端口，同事用默认 8000。
这种差异不该写死在代码里、更不该提交进 git。本项目用一层「本机配置隔离」处理：

- `run_web.py`（入库，共享入口）：从 `.env.local` 和环境变量读 host/port，缺省 `127.0.0.1:8000`。
- `.env.local`（**不入库**，每人本机一份）：写自己的端口/host，只影响自己这台机器。
- `.env.local.example`（入库）：配置模板，复制为 `.env.local` 后改值即可。

```powershell
# 同事/默认：无需任何配置，直接默认 8000
uv run python run_web.py

# 本机要换端口（如配合 Portmanager）：先建 .env.local 再改 KQ_WEB_PORT
Copy-Item .env.local.example .env.local
uv run python run_web.py
```

> 仓库里共享的只有业务代码，端口/host 这类本机差异留在各自 `.env.local`，互不干扰。

### 常见问题

| 现象 | 原因 / 处理 |
|------|------------|
| 浏览器“无法访问/拒绝连接” | 服务没在跑（黑窗口被关了）。重新双击 bat。 |
| 打开是空白页 | 服务刚启动还没就绪，等 2-3 秒 **刷新**（Ctrl+R）。 |
| 启动器报 `address already in use` | 8000 端口被占（可能已有一个在跑）。关掉旧黑窗口，或换端口 `--port 8001`。 |
| 看不到最新界面 | 浏览器缓存。**Ctrl+Shift+R** 强刷。 |

---

## 2. 局域网给同事用（同一办公网访问）

让同一局域网内其他电脑/手机通过你的 IP 访问：

1. 启动时绑定所有网卡（注意是 `0.0.0.0`，不是 `127.0.0.1`）：
   ```powershell
   uv run python -m uvicorn web.server:app --host 0.0.0.0 --port 8000
   ```
2. 查本机局域网 IP：
   ```powershell
   ipconfig | Select-String IPv4
   ```
   假设是 `192.168.1.20`。
3. 放行 Windows 防火墙入站 8000（管理员 PowerShell）：
   ```powershell
   New-NetFirewallRule -DisplayName "KaoQin Web 8000" -Direction Inbound `
     -Protocol TCP -LocalPort 8000 -Action Allow
   ```
4. 同事浏览器访问 `http://192.168.1.20:8000`。

> 安全提醒：当前 **无登录鉴权**，绑 `0.0.0.0` 后局域网内任何人都能用并上传文件。
> 仅在可信内网使用；公网暴露前必须加鉴权（见第 3 节）。

---

## 3. 正式服务器部署（生产）

当前是 MVP，上生产前建议补齐以下几项：

### 3.1 进程托管
用专业进程管理常驻，不要靠黑窗口：
```bash
# Linux 示例：多 worker + 绑定
uvicorn web.server:app --host 0.0.0.0 --port 8000 --workers 4
```
配合 `systemd` / `supervisor` / `pm2` 守护与开机自启。

> 注意：当前下载令牌 `_DOWNLOADS` 存在**进程内存**里。多 worker 下，令牌可能落在
> A 进程而下载请求被路由到 B 进程导致 404。多 worker 前需把令牌→文件改为
> **共享存储**（如 Redis / 数据库 / 共享磁盘 + 文件名约定）。MVP 单 worker 无此问题。

### 3.2 反向代理 + HTTPS
前置 Nginx / Caddy 做 TLS 与转发：
```nginx
server {
    listen 443 ssl;
    server_name kaoqin.example.com;
    client_max_body_size 50m;          # 允许上传较大的考勤表
    location / { proxy_pass http://127.0.0.1:8000; }
}
```

### 3.3 必补的生产化项
- **鉴权**：加登录（最简可加 FastAPI 依赖做 Basic Auth / Token），否则任何人可上传。
- **临时文件清理**：`server.py` 当前把上传/输出写临时目录，输出保留供下载但**不自动清理**。
  生产需定时清理过期临时文件与令牌，避免磁盘膨胀。
- **持久化（可选）**：若要历史记录/多客户管理，需引入数据库（对应侧边栏「历史记录/客户管理」占位）。
- **并发与限流**：对账是 CPU/IO 任务，按需限制并发或排队。

### 3.4 打包交付（可选）
若要免装环境分发，可用 PyInstaller 把 `uvicorn` 启动脚本打成 exe（思路同桌面版），
随包带上 `web/static/`。注意 `hiddenimports` 需含 fastapi / uvicorn / starlette / pydantic
等子模块（同桌面版打包踩过的坑）。

---

## 4. 端口与地址速查

| 用途 | 命令片段 | 访问地址 |
|------|---------|---------|
| 本机自用 | `--port 8000`（默认 127.0.0.1） | `http://127.0.0.1:8000` |
| 局域网 | `--host 0.0.0.0 --port 8000` | `http://<本机IP>:8000` |
| 换端口 | `--port 8001` | `http://127.0.0.1:8001` |

## 5. 验证部署是否正常

```powershell
# 接口存活
curl http://127.0.0.1:8000/api/clients      # 应返回 [{"key":"client_a","display":"客户A"}]
# 首页
curl http://127.0.0.1:8000/                  # 应返回 HTML
```
浏览器打开后：选公司 `.xlsx` + 客户 `.xls` → 自动识别月份 → 开始核对 → 出 KPI/结果表 → 可下载处理后的 xlsx，即为正常。
