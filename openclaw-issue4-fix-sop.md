# OpenClaw 问题4修复SOP（多 agent + Feishu 会话路径校验）

## 1. 问题定义
- 现象日志：
  - `feishu[<account>]: dispatching to agent (session=agent:<agentId>:main)`
  - `failed to dispatch message: Error: Session file path must be within sessions directory`
- 典型影响：
  - 飞书消息能进入网关，但 agent 不回复。
  - 多 agent 场景（如 `product-manager` / `pm` / `market-researcher`）更容易触发。

## 2. 适用范围
- OpenClaw `2026.2.12`（及同类打包结构版本）。
- Windows 本地部署（含计划任务启动）。
- Feishu 多账号 + bindings 路由到不同 agent。

## 3. 根因（核心）
- `sessionFile` 在历史会话中常为绝对路径（例如 `C:\Users\...\agents\product-manager\sessions\xxx.jsonl`）。
- 运行时代码在部分链路中用当前 `sessionsDir` 做严格路径校验，导致绝对路径被误判为越界。
- 若重启未真正替换旧进程，补丁会“看起来已改，实际未生效”。

## 4. Codex 标准修复流程

### Step A: 先确认是同一问题
1. 检查日志是否出现：
   - `Session file path must be within sessions directory`
2. 检查通道是否正常：
   - `openclaw channels status --probe`

### Step B: 修补两个层面
1. 修调用点（确保传 `agentId`）
- 将以下文件中：
  - `resolveSessionFilePath(sessionIdFinal, sessionEntry)`
- 替换为：
  - `resolveSessionFilePath(sessionIdFinal, sessionEntry, { agentId })`
- 目标文件：
  - `dist/reply-B5GoyKpI.js`
  - `dist/extensionAPI.js`
  - `dist/pi-embedded-DxwVpEx9.js`
  - `dist/loader-BpQdnOY1.js`

2. 修路径函数（兼容绝对路径）
- 目标文件：
  - `dist/paths-B49s6UZQ.js`
  - `dist/paths-C7j5gEli.js`
  - `dist/paths-CnE9bV4t.js`
  - `dist/paths-mF4iWwgm.js`
- 在 `resolveSessionFilePath(sessionId, entry, opts)` 中加入逻辑：
  - 若 `entry.sessionFile` 是绝对路径，且位于 `resolveStateDir(...)` 下，则直接返回该绝对路径。
  - 否则继续走原有 `resolvePathWithinSessionsDir` 校验。

### Step C: 语法校验 + 真重启
1. 语法检查：
- `node --check <每个被修改的文件>`
2. 彻底替换旧进程：
- `openclaw gateway stop`
- 结束残留 gateway 进程（若有）
- `openclaw gateway start`
3. 再检查：
- `openclaw channels status --probe`

## 5. 验收标准
- 飞书消息进入后日志应为：
  - `received message ...`
  - `dispatching to agent ...`
  - 不再出现 `Session file path must be within sessions directory`
- 机器人能正常回复。

## 6. 回滚方案
- 修改前先备份 `dist` 下目标文件。
- 若修复异常，直接用备份覆盖回原文件后重启网关。

## 7. 部署防复发清单（新机器）
1. 固化配置文件（含 agents、bindings、feishu accounts）。
2. 固化 session store（建议）：
- `"session": { "store": "C:\\Users\\<user>\\.openclaw\\agents\\{agentId}\\sessions\\sessions.json" }`
3. 计划任务只保留“登录时启动一次”，不要每分钟重复。
4. 部署后统一做健康检查：
- `openclaw channels status --probe`
- 确认只有一个 gateway 实例监听 `18789`。
5. 升级版本时复核该补丁是否已被官方吸收；若已修复，去掉本地补丁。

## 8. 快速诊断命令
- `openclaw channels status --probe`
- `openclaw agents list --bindings`
- `Get-Content C:\Users\xie\AppData\Local\Temp\openclaw\openclaw-YYYY-MM-DD.log -Tail 300`
- `rg -n "Session file path must be within sessions directory|received message from|dispatching to agent" C:\Users\xie\AppData\Local\Temp\openclaw\openclaw-YYYY-MM-DD.log`

---
维护说明：本SOP用于 Codex 复现并修复同类“问题4”。
