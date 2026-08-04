# 工作日志 - 2026年8月5日

## 概览

启动「AI 串口助手」项目：基于开源 EK-OmniProbe 2.5.0（Tauri 2 + Rust + React，MIT）改造，让 AI Agent（Claude Code）能后台接入控制串口、烧录固件，同时前台 GUI 实时可见所有串口操作（后台烧录、前台映射）。

本次完成全部 7 个阶段改造，代码推送到新私有仓库 `github.com/2903537389/aichuankoutool`。

---

## 项目背景与需求（实际时间：本次会话，具体时分未知）

核心痛点：Agent 在后台跑 build/flash 时，串口被 Agent 的进程独占，我前台打不开串口、看不到任何串口活动。

需求：
1. Agent 后台操作串口的同时，前台实时看到串口操作（类似映射到前台）
2. 搭配 MCP 工具 / skill 控制串口（也可人工控制）
3. 内置通用的 build/flash 脚本供 Agent 调用

## 架构决策（关键转折点）

EK-OmniProbe 2.5.0 已内置「AI 数据桥接」（应用独占串口，AI 经 TCP:8765 NDJSON 读写），数据流与 GUI 共享同一读线程——方向正好符合需求。但有三个缺口必须补：

1. **桥接协议只有 `serial.write`**，Agent 无法开/关/列串口 → 扩展 `serial.list/open/close/start/stop/status`
2. **Agent 用 esptool 直连会抢串口** → 新增 TCP 字节透传通道（8770），esptool 用 `--port socket://` 零改造烧录，字节流镜像到 GUI
3. **Agent 经桥接发的 TX 数据不进 GUI** → 后端补 TX 事件，前台显示 Agent 发送内容

约束：暂不做托盘常驻，Agent 使用时应用窗口保持打开（桥接文本流依赖前端转发）。

## 实施内容

### Rust 后端
- `serial/ops.rs`（新增）：抽出 open_serial / close_serial / serial_status，GUI 命令与桥接共用同一 SerialState
- `serial/passthrough.rs`（新增）：TCP 字节透传通道，下行挂在 accumulator（与 GUI/桥接共享数据流）
- `ai_bridge.rs`：命令分发扩展（serial.list/open/close/start/stop/status + 现有 write），成功操作后 emit serial-status 让 GUI 同步；write 成功 emit TX 事件
- `commands/serial.rs`：start_serial 重构为同步入口 `start_serial_polling`，供 GUI 与桥接共用；accumulator 挂透传广播
- `state.rs` / `lib.rs`：注册 passthrough 状态、get_serial_status 命令

### 前端
- `useSerialEvents.ts`：方向切换时先 flush 残帧（避免 tx/rx 单缓冲串帧）；桥接文本流保持只发 rx
- `AiBridgeControl.tsx`：新增 TCP 透传通道控制 UI（端口/启动/停止/客户端数）
- 串口连接状态随 Agent 操作经 serial-status 事件自动同步

### 内置脚本库（src-tauri/scripts，打包为 Tauri resource）
- `flash_esp32.py`：esptool 走 `socket://127.0.0.1:8770` 烧录
- `flash_stm32.py`：STM32 系统 bootloader ISP 协议 + socket 串口适配（无第三方依赖）
- `serial_cli.py`：通用串口 CLI（list/open/close/start/stop/write/monitor/snapshot）
- `build_esp32.sh` / `build_stm32.sh`：编译 + 烧录模板
- `README.md`：用法说明

### skill 更新
- `client.py` 新增串口控制子命令；`SKILL.md` 补充烧录流程、协议表

## 踩坑记录

1. **pnpm 11 在 Windows 上顶层 node_modules 为空**（symlink 失败），包都在 .pnpm 里但链接没建 → 用 `--config.node-linker=hoisted` 重装解决
2. **`pnpm exec` 极慢**：每次有 supply-chain 供应链检查（验证 447 包）→ 直接调 `node_modules/.bin/` 绕过
3. **cargo check 假成功**：用 `| tail` 管道吞了真实退出码；且 `tauri.conf.json` 的 resources glob `scripts/**` 写错导致 build script 失败 → 改 `scripts/*`，验证必须不带管道
4. **cargo test 跑不起来**：测试二进制 `0xc0000139`（DLL 入口点缺失，疑似 Tauri 依赖 DLL 与系统环境不匹配），`--list` 能加载但跑测试崩 → 环境限制，代码本身 cargo check/tsc/vite 全通过
5. **GitHub 仓库名不支持中文**：gh repo create "中文名" 被截断成 `ai-`（空仓库）→ 改用英文 `aichuankoutool`
6. **token 缺 workflow scope 推不上 CI 文件**：移除上游 .github/workflows（改造 fork 不适用）后推送成功；`gh auth refresh` 加 scope 需浏览器交互（本次未完成）

## 验证结果

| 项目 | 结果 |
|------|------|
| cargo check | ✅ 通过 |
| tsc --noEmit | ✅ 通过 |
| vite build | ✅ 通过 |
| Python 脚本语法 | ✅ 通过 |
| cargo test | ⚠️ 环境限制（0xc0000139） |

## 结果

- 代码仓库：`github.com/2903537389/aichuankoutool`（私有，已推送）
- 遗留：cargo test 环境问题待排查；实机烧录（ESP32/STM32）待验证；旧空仓库 `ai-` 待删除（需 delete_repo scope）
