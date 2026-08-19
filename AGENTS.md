# QuickMemo — 注意事项（AGENTS.md）

Electron 备忘录应用，结构较复杂、牵一发而动全身。改动前先通读本文件与 `CLAUDE.md`。

## 1. Tokdash 统计服务端口（重要）

- 端口定义在 `main.js` 的 `TOKDASH_PORT`，当前为 **17523**（由 55423 改来，2026-08）。
- **根因**：旧端口 55423 落在 Windows 动态保留端口区间（Hyper-V/WSL 的 "excluded port range"，实测 `55338-55437`）内，导致 uvicorn 绑定失败 `[WinError 10013]` WSAEACCES。保留区间每次开机动态分配，所以服务"时好时坏"。
- 若再次遇到绑定失败：先 `netsh int ipv4 show excludedportrange protocol=tcp` 确认端口是否落在排除区间，再换一个远低于 49152 的端口。
- `startTokdash()` 用 `py -m tokdash.cli --port <port>` 启动；`start-tokdash-server` IPC 会调 `waitForTokdash()` 轮询 `/health` 确认就绪后才返回 `ok`，**不要改回"立即返回 ok"**（否则 UI 竞态：点了启动又显示未启动）。
- `tokdash/cli.py` 内部自带默认端口 55423（仅 `tokdash serve` 独立运行时用）；QuickMemo 启动时总是显式传 `--port`，所以应用不受影响，无需改 `cli.py`。

## 2. AI 生成标题功能已删除

- 旧的"AI 自动生成标题"（generateAITitle、AI 设置面板、aiSettings）已在 commit `8c706b0` 移除；残留死代码（styles.css 的 `#btn-ai-title` 动画、main.js 未使用的 `safeStorage` import）也已清理。
- `main.js` 里仍有一个 **AI CLI Named Pipe**（`\\.\pipe\QuickMemo_AI`，list_notes/get_note/create_note/...），是给外部 AI 工具用的，**不是**被删的"AI 标题"功能，别误删。

## 3. 当前工作区的未提交改动（别覆盖 / 别一并提交）

有一批**未提交**改动，是新的 getnote.top 云同步 + 大笔记预览截断优化，与 bug 修复无关：

- `main.js` / `preload.js`：新增 `getnote-get` / `getnote-put`（仿 note.ms 的另一个在线笔记服务）。
- `src/renderer.js` / `index.html` / `markdown-editor.js` / `styles.css`：对应 UI，及 `PREVIEW_CHAR_LIMIT = 60000` 的预览截断。

## 4. 文件编码 BOM 坑

- 多个文件带 UTF-8 BOM：`package.json`、`tokdash/requirements.txt`、`scripts/install-tokdash-deps.js` 等。
- `tokdash/requirements.txt` 第一行实际是 `﻿fastapi`；若 `pip install -r` 报 "Invalid requirement"，先去掉 BOM。

## 5. 环境备注

- 本机 Windows 11；`python` / `py` 可用（3.13），**没有** `python3` 命令（win32 下 `startTokdash` 的 cmds 里 `python3` 用不上）。
- fastapi / uvicorn 已装（fastapi 0.136.3 / uvicorn 0.49.0）。

## 6. 验证方式

- tokdash：`python -m tokdash.cli --bind 127.0.0.1 --port 17523 --no-open` 后 `curl http://127.0.0.1:17523/health` 应返回 `{"status":"ok"}`。
- main.js 改动后：`node -c main.js` 做语法检查。
- 这是 Electron GUI 应用，改动 UI 后实际 `npm start` 跑起来点一遍。
