# meeting_service — 三端会议实录服务

基于 MOSS-Transcribe-Diarize 的会议/对话实录系统，按「用户端 → 公网服务器 → 模型端」三端架构设计：

```
用户端（两种形式，均无公网IP）        公网服务器（有公网IP）                模型端（GPU，无公网IP）
┌──────────────────┐   上传录音    ┌──────────────────────┐   轮询认领   ┌──────────────────┐
│ 形式1: 网页        │ ───────────▶ │ 任务队列（SQLite）     │ ◀────────── │ worker 循环       │
│  （服务器直接挂载） │              │ 音频对象存储（磁盘）    │   下载音频   │ MOSS 0.9B 转写    │
│ 形式2: 安卓App     │ ◀─────────── │ LLM 纪要精炼（可选）   │ ──────────▶ │ +说话人分离       │
│  （后续开发）      │   拉取结果    └──────────────────────┘   上传结果    └──────────────────┘
└──────────────────┘
```

- **用户端形式 1（已实现）**：网页由公网服务器直接挂载在 `/`，浏览器打开即用，无需单独部署前端；录音/选文件、进度轮询、分段展示、SRT/JSON 导出全在浏览器完成。
- **用户端形式 2（规划中）**：独立安卓 App，消费同一套 REST API，服务器零改动；契约与功能清单见 [client/android/README.md](client/android/README.md)。
- 用户端与模型端**只发起出站连接**，无需公网 IP、无需端口映射。
- 任务状态机：`pending → processing → refining → ready`（含 `failed`）。
- 认领原子化 + 租约超时回收：worker 崩溃后任务自动重新入队。
- LLM 纪要可选：不配置时只返回说话人统计；配置任意 OpenAI 兼容接口后自动生成会议纪要。

## 目录结构

```
meeting_service/
├── run_server.py        # 公网服务器入口
├── server/              # FastAPI 应用：API + 状态机 + 精炼
│   ├── app.py           #   路由（/api 用户端、/worker 模型端）
│   ├── db.py            #   SQLite 任务状态机（原子认领、租约）
│   ├── auth.py          #   用户 PBKDF2 密码 + worker key
│   ├── storage.py       #   音频文件落盘
│   └── refine.py        #   说话人统计 + 可选 LLM 纪要
├── worker/main.py       # 模型端：轮询→下载→GPU转写→心跳→上传
├── client/
│   ├── web/index.html   # 用户端形式1：网页（服务器挂载在 /）
│   └── android/         # 用户端形式2：安卓App（规划，见其 README）
├── deploy/              # 公网服务器一键部署（PM2 托管）
│   ├── deploy.sh        #   一键：install/update/status/logs/restart/stop
│   ├── ecosystem.config.js  # PM2 进程定义（读 env/meeting.env）
│   ├── meeting.env.example  # 配置模板（worker key、端口、LLM 等）
│   ├── server-requirements.txt  # 服务端最小依赖（无 torch）
│   └── nginx-meeting.conf.example  # HTTPS 反代示例
├── test_e2e.py          # 端到端测试（假worker，无需GPU）
└── requirements.txt
```

## 快速开始（本机演示）

三个终端，均在仓库根目录执行（Windows 用 `.venv/Scripts/`，Linux 用 `.venv/bin/`）。

**1. 公网服务器**（有 GPU 的机器上跑也可以，只做协调不占显存）：

```bash
# 方式A：一键部署（PM2 托管，推荐用于公网 VPS）
npm install -g pm2                       # 未装 pm2 时先装
bash meeting_service/deploy/deploy.sh    # 幂等：建venv→装依赖→生成配置→启动

# 方式B：手动前台运行（本机调试用）
.venv/Scripts/python meeting_service/run_server.py --host 127.0.0.1 --port 8000 \
    --data-dir meeting_service_data --worker-key devkey123
```

一键部署的配置在 `deploy/env/meeting.env`（首次运行自动生成，含随机 worker
key），改完执行 `bash meeting_service/deploy/deploy.sh restart` 生效；其他子命令
`status / logs / stop / update`。开机自启执行一次 `pm2 save && pm2 startup`。
公网 HTTPS 反代示例见 `deploy/nginx-meeting.conf.example`。

**手机/平板录音必须 HTTPS**（浏览器只在安全上下文开放麦克风，明文 HTTP 下
`navigator.mediaDevices` 为 undefined）。免备案方案：DuckDNS 免费子域名 +
8443 端口 + DNS 验证签 Let's Encrypt 证书，完整步骤见
`meeting_service/deploy/nginx-meeting-https.example`；临时测试可用其中的
cloudflared 快速隧道。

**服务器性能要求极低**：无 torch 依赖（`deploy/server-requirements.txt` 仅
fastapi/uvicorn/requests），1核 1G VPS 即可；真正需要规划的是磁盘（16kHz WAV
约 115MB/小时音频）和带宽（每条音频约走 2 遍流量）。

启动日志会打印 worker key（不传 `--worker-key` 时自动生成并持久化到数据目录）。

**2. 模型端**（GPU 机器，首次需联网下模型；国内可加 `HF_ENDPOINT=https://hf-mirror.com`）：

```bash
.venv/Scripts/python -m meeting_service.worker.main \
    --server http://<服务器地址>:8000 \
    --worker-key devkey123 \
    --model OpenMOSS-Team/MOSS-Transcribe-Diarize
```

**3. 用户端**：浏览器打开 `http://<服务器地址>:8000/`，注册账号 → 录音或选择音频/视频文件 → 自动上传 → 等待状态变为「已完成」→ 查看说话人分段 / 导出 SRT / JSON。

浏览器端会把录音自动转成 16kHz 单声道 WAV 再上传（模型原生前端格式），服务器无需安装 ffmpeg。

## 端到端测试

```bash
.venv/Scripts/python meeting_service/test_e2e.py
```

启动真实服务器子进程 + 假 worker，覆盖 24 项检查：注册登录、上传鉴权、原子认领、
音频下载一致性、结果入库、说话人统计、越权访问拒绝、租约超时回收、失败上报。

## 配置项（环境变量）

| 变量 | 默认 | 说明 |
|---|---|---|
| `MTD_DATA_DIR` | `meeting_service_data` | 数据目录（SQLite + 音频文件） |
| `MTD_WORKER_KEY` | 自动生成 | 模型端认领任务的 API key |
| `MTD_LEASE_SECONDS` | `3600` | 任务租约时长，超时未完成自动回收 |
| `MTD_MAX_UPLOAD_MB` | `500` | 单个音频上传上限 |
| `MTD_LLM_BASE_URL` | 空 | OpenAI 兼容 API 地址（如 https://api.xxx.com/v1） |
| `MTD_LLM_API_KEY` | 空 | LLM API key |
| `MTD_LLM_MODEL` | 空 | 模型名，如 `gpt-4o-mini` / `glm-4.7` |

三项 LLM 变量同时配置才会生成 AI 纪要；LLM 失败不影响转写结果返回。

## API 概览

用户端（`Authorization: Bearer <token>`）：

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/health` | 连通性检查（两种客户端形式通用） |
| POST | `/api/auth/register` / `/api/auth/login` | `{username, password}` → `{token}` |
| POST | `/api/tasks?filename=x.wav&duration=6.0` | 音频二进制体 → `{task_id}` |
| GET | `/api/tasks` / `/api/tasks/{id}` | 任务列表 / 详情（ready 时含 transcript+summary） |
| GET | `/api/tasks/{id}/audio` | 下载原始音频 |

模型端（`X-Worker-Key: <key>`）：

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/worker/claim?worker_id=w1` | 原子认领一个任务（无任务返回 `{"task": null}`） |
| GET | `/worker/tasks/{id}/audio` | 下载音频 |
| POST | `/worker/tasks/{id}/heartbeat` | 续租 |
| POST | `/worker/tasks/{id}/result` | `{text, segments, duration_sec, worker_meta}` |
| POST | `/worker/tasks/{id}/failure` | `{error}` |

## 部署到公网的注意事项

- 用反向代理（Nginx/Caddy）终结 **HTTPS**，会议录音是高敏数据，明文 HTTP 不应出内网。
- `MTD_WORKER_KEY` 是模型端的唯一凭证，务必用强随机值并保密；泄露等于把所有用户录音开放下载。
- 多个模型端可并行：认领是原子的，天然支持按 GPU 分片扩容。
- 大流量场景建议把音频改为对象存储直传（S3/R2 预签名 URL），服务器只保留状态机。
- SQLite 支撑单机 thousands 级任务无压力；需要多服务器时再换 PostgreSQL。
