# 考勤核对系统 · Web 架构文档

> 适用版本：Web v1.0（前后端分离 MVP）
> 关联代码：`web/server.py`、`web/static/index.html`、`core/service.py`

## 1. 总览

Web 版是在原桌面（PySide6）应用之上，**复用同一套对账业务逻辑**搭建的浏览器版本。
核心原则：**业务逻辑零重写**——前端只负责交互与展示，后端只负责文件收发与编排，
真正的对账计算全部由 `core/` + `clients/` 完成。

```
┌──────────────┐     HTTP/JSON      ┌──────────────┐     函数调用      ┌────────────────────┐
│   浏览器前端  │ ───────────────▶  │ FastAPI 后端  │ ──────────────▶  │  对账服务层         │
│ index.html   │ ◀───────────────  │ web/server.py │ ◀──────────────  │ core/service.py     │
│ (原生 JS)    │   结果JSON/文件     │              │   ReconcileReport │ run_reconcile(...)  │
└──────────────┘                    └──────────────┘                   └─────────┬──────────┘
                                                                                  │ 复用
                                                                    ┌─────────────▼─────────────┐
                                                                    │ core/ + clients/ 对账流水线 │
                                                                    │ parse_company → read_* →   │
                                                                    │ reconcile → write_output   │
                                                                    └────────────────────────────┘
```

桌面 GUI（`app/main_window.py`）与 Web 后端走的是**同一个** `core/service.py: run_reconcile`，
只是进度回调实现不同（GUI 用 Qt Signal，Web 用日志收集器）。

## 2. 分层职责

| 层 | 文件 | 职责 | 是否含业务逻辑 |
|----|------|------|---------------|
| 前端 | `web/static/index.html` | 上传文件、选月份/节假日、渲染结果表/KPI/日志、主题切换、触发下载 | 否（纯展示） |
| 后端 | `web/server.py` | 路由、文件收发、临时目录管理、下载令牌、调用服务层、组装 JSON | 否（仅编排） |
| 服务层 | `core/service.py` | 编排对账主循环：解析→读取→核对→写出，进度经 `ProgressReporter` 回调 | 否（仅编排，不依赖 GUI） |
| 业务核心 | `core/`、`clients/` | 公司/客户数据解析、年月识别、核对规则、罚款、写 xlsx | **是** |

## 3. HTTP 接口

| 方法 | 路径 | 入参 | 返回 |
|------|------|------|------|
| GET | `/` | — | 前端单页 `index.html` |
| GET | `/api/clients` | — | `[{key, display}]` 客户列表（驱动下拉框） |
| POST | `/api/months` | `client`(.xls 文件), `client_key` | `{months:[...]}` 内容化识别出的可用月份 |
| POST | `/api/reconcile` | `company`(.xlsx), `client`(.xls), `client_key`, `months`(逗号分隔), `makeup_dates`, `holiday_dates` | 见下方结果结构 |
| GET | `/api/download/{token}` | 路径中的下载令牌 | 处理后的 `.xlsx`（FileResponse） |
| 静态 | `/static/*` | — | 静态资源 |

### `/api/reconcile` 返回结构

```json
{
  "ok_total": 36,
  "err_total": 3,
  "highlight_total": 4,
  "months": [
    {
      "month": "2026-05",
      "ok": 36, "err": 3,
      "download_token": "bd304062...",
      "rows": [
        {"name": "蔡敏杰", "passed": true,  "highlight": false, "notes": "OK"},
        {"name": "宁锌婷", "passed": false, "highlight": false, "notes": "..."}
      ]
    }
  ],
  "logs": ["正在处理 2026-05 ...", "2026-05 完成：36 人OK，3 人异常", "..."]
}
```

## 4. 一次核对的请求流程

```
浏览器                          后端 server.py                      服务层 core/service.py
  │ 选公司.xlsx/客户.xls            │                                   │
  │ POST /api/months ──────────────▶ 存临时→get_available_months ──────▶ (xlrd + date_detect)
  │ ◀──────── {months:[...]} ───────│ 删临时                             │
  │ 勾月份/填节假日，点“开始核对”      │                                   │
  │ POST /api/reconcile(FormData) ─▶ 存两份上传到临时目录                  │
  │                                 │ run_reconcile(...) ───────────────▶ parse_company
  │                                 │  + _LogCollector 收集日志            read_monthly/read_detail
  │                                 │                                     reconcile（逐人）
  │                                 │                                     write_output→临时 xlsx
  │                                 │ 每月输出登记 token→路径(内存)         │
  │ ◀──── 结果JSON + logs + token ──│ 删上传临时（输出保留供下载）          │
  │ 渲染 KPI/结果表/日志             │                                   │
  │ 点下载 GET /api/download/{tok} ─▶ FileResponse(xlsx) ────────────────│
```

## 5. 关键设计点

- **服务层解耦**：`run_reconcile` 不 import 任何 PySide6；进度/逐行结果通过 `ProgressReporter`
  抽象暴露。GUI 注入 `_QtProgress`（发 Qt Signal），Web 注入 `_LogCollector`（攒日志随响应返回）。
- **年月内容化识别**：客户/公司表的年月列由 `core/date_detect.py` 扫描内容判定，
  不依赖文件名或固定列（详见根目录 `CLAUDE.md` 的「年月识别」）。
- **插件化客户**：`clients/__init__.py` 的 `REGISTRY` 决定 `/api/clients` 下拉项；
  新增客户无需改 Web 层。
- **下载令牌**：每月输出 xlsx 落在临时目录，用 uuid 令牌映射到内存字典 `_DOWNLOADS`，
  前端凭令牌下载。

## 6. MVP 边界（当前未做）

- **无持久化 / 无账号**：单次请求用临时目录，结果不入库；`_DOWNLOADS` 是进程内内存，
  重启即失效。
- **单客户**：仅 `client_a`。
- **并发/清理**：未做并发隔离与临时文件定期清理（MVP 单用户够用）。
- **仅本机**：默认绑 `127.0.0.1`，未做局域网/公网部署、鉴权、HTTPS（见部署文档）。

## 7. 技术栈

- 后端：FastAPI 0.137 + Uvicorn 0.49 + python-multipart（文件上传）
- 业务依赖：openpyxl（读写 .xlsx）、xlrd（读 .xls）
- 前端：原生 HTML + CSS + JavaScript（无构建步骤，无框架），10 套浅色主题用 CSS 变量切换
- 运行：`uv sync` 装依赖后 `uv run python run_web.py`（或 `uv run python -m uvicorn web.server:app --port 8000`）
