---
name: claude-wechat-bridge
description: >
  让 Claude Code 通过微信收发消息的桥接配置 skill。基于微信官方 ClawBot 接口 + 第三方 npm 包
  claude-wechat-channel（fengliu222 维护）。当用户说"把 Claude 连到微信" / "想用手机微信和
  Claude 聊" / "ClawBot 配 Claude" / "claude-wechat-channel 怎么装" 时触发。
version: 1.1.0
risk_level: 中（第三方维护 + 微信新号绑定 + dangerously flag）
tested_on: 2026-05-24, macOS 26.4, Claude Code 2.1.128, claude-wechat-channel@0.1.2（含图片文件补丁）
---

# Claude Code ↔ 微信 桥接配置 Skill

## 这是干啥用的

让 Claude Code（CLI）能：
- **收**：用户在手机微信「微信 ClawBot」对话发的消息，自动当成 prompt 进 Claude
- **发**：Claude 主动调 `mcp__wechat__reply` 工具回消息到那个对话

典型用例：
- 让 AI 帮你跟踪一个长流程，过程中你不在电脑前但需要随时给输入
- 把日常 todo / 知识库整理通过微信对话完成
- 跑 ClawBot loop 类任务（如批量数据搜集，AI 发关键词、人搜后贴回）

## 前置条件（必须，缺一不可）

| 项 | 要求 |
|---|---|
| Claude Code 版本 | ≥ 2.1.80 (`claude --version` 查) |
| Node.js | ≥ 18 (`node -v` 查) |
| 微信版本 | iOS ≥ 8.0.70 (设置→关于) |
| 操作系统 | Mac/Win/Linux 都行（Mac 测过）|
| 微信账号 | **强烈建议专用账号**，不要绑日常号 |

## 安装步骤

### 1. 在项目根目录创建 `.mcp.json`

```json
{
  "mcpServers": {
    "wechat": {
      "command": "npx",
      "args": ["claude-wechat-channel"]
    }
  }
}
```

### 2. 在终端启动 Claude Code（必须用 CLI，desktop app 暂不支持 channel-type MCP）

```bash
cd <你的项目根目录>
claude --dangerously-load-development-channels server:wechat
```

注意：`--dangerously-load-development-channels` 是必需的，因为 `claude-wechat-channel` 是第三方
channel-type MCP，不在官方 allowlist 里。

### 3. 终端输出二维码 → 手机扫码

首次启动会生成二维码，**用专用账号的手机微信扫码**，确认绑定。绑定后凭证自动保存在
`~/.npm/_npx/<hash>/`，下次启动无需再扫。

### 4. 验证

绑定成功后，手机微信里会出现「微信 ClawBot」对话入口。

测试通路：在 ClawBot 发一条 "hi" → Claude Code 终端应该出现以 `<channel source="wechat"
sender="..." user_id="...">` 包装的消息 turn。Claude 用 `mcp__wechat__reply` + 那个
user_id 回复就到你手机了。

### 5. （可选）图片文件补丁

原版 `claude-wechat-channel` 只透传文本。本 skill 自带一个补丁包，给 monitor.js 加上：
- 收到图片 → 自动下载到 `~/.wechat-claude/media/`，在消息里注入 `[图片: <绝对路径>]`
- 收到文件（PDF / DOCX / etc.）→ 同上，注入 `[文件: <绝对路径>]`
- 收到语音 → 注入 `[语音文件: <绝对路径>]`
- 收到视频 → 注入 `[视频: <绝对路径>]`

Claude 拿到 `[图片: ...]` 后用 Read tool 直接读那个路径就行（多模态 LLM 看图，PDF 用 pypdf 抽文本）。

**装补丁：**

```bash
# 1. 先确保 npx claude-wechat-channel 至少跑过一次（让 npm 把包下载到 ~/.npm/_npx/）
# 2. 跑补丁脚本
bash ~/.claude/skills/claude-wechat-bridge/apply-media-patch.sh
# 3. 重启 bridge（杀掉旧进程，下次 monitor 自动启）
pkill -f claude-wechat-channel
```

脚本会找所有 `~/.npm/_npx/*/node_modules/claude-wechat-channel/dist/monitor.js`、备份成
`.orig`、覆盖为 `patched-monitor.js`。已打过补丁的会跳过。npm 重新安装/升级包之后再跑一次即可。

**已验证（2026-05-24）：**
- ✓ 文字
- ✓ 图片（jpg/png）
- ✓ PDF（pypdf 抽文本；图片型 PDF 需要 `brew install poppler` 才能用 Read 直接看）

**已知限制：**
- 微信 ClawBot 接口本身不支持 Claude 主动给用户发图片/文件，**单向**（只能收，不能发）
- 文件名中如有特殊字符（中文括号等）落盘时会保留，path 传给工具需注意 quoting

## 在 Claude 这边用 `mcp__wechat__reply`

```python
mcp__wechat__reply(
  user_id="<从消息元数据里抽出来的 ID>",
  text="纯文本，不要 markdown。超 4000 字会自动分段。"
)
```

**关键约束：** `reply` 只能"响应"——必须先收到用户消息拿到 user_id，才能 reply 给同一个会话。
**不能主动发起对话**（除非缓存了之前的 user_id）。

## 已知坑 + 解决

| 现象 | 原因 | 解决 |
|---|---|---|
| Desktop app 加 `.mcp.json` 没用 | channel-type MCP 需要 `--dangerously-load-development-channels` flag，desktop app 启动时不带 | 用终端 CLI 启动；VSCode 内置终端也行 |
| 终端启动时弹出 `Newline followed by # inside a quoted argument` 等权限确认 | Claude Code 默认非 YOLO 模式，bash 命令含特殊字符会拦截 | 终端按 1 确认 / 让 Claude 用 Write tool 写文件而不是 heredoc / 启动时加 `--dangerously-skip-permissions`（不推荐）|
| ClawBot 单实例限制 | 一个微信账号同时只能绑一个 Claude Code 实例 | 第二台机器扫码会顶掉前一台，正常现象 |
| 电脑必须常开 | 微信只是遥控器，Claude 在你 Mac 上跑 | 设系统不睡眠 |
| 想在 desktop app 用 | 暂不支持 channel-type MCP | 等官方支持；或用 VSCode 内置终端体验近似 |

## 风险评估

| 风险 | 等级 | 说明 |
|---|---|---|
| 第三方维护 | 中 | `claude-wechat-channel` 是 fengliu222 维护，不是 Tencent / Anthropic 官方。每次 `npm update` 检视 changelog |
| `--dangerously-load-development-channels` | 中 | 标志名带 "dangerously" 是 Claude Code 官方提醒：绕过 channel allowlist 校验 |
| 微信封号 | 中-低 | 用的是腾讯 ClawBot 官方接口（不是反代/iPad 协议），相对安全；但**用专用账号**避免日常号风险 |
| 聊天数据流 | 中 | 内容路径：手机微信 → 腾讯服务器 → 此 npm 包（你 Mac）→ Anthropic API。三方都见明文，敏感内容慎发 |
| 凭证文件 | 低 | 存在 `~/.npm/_npx/<hash>/`，Mac 被入侵则泄漏 |

**强烈建议：**
1. 用**专门注册的新微信号**，不绑主号
2. 锁定 npm 版本：`.mcp.json` 里写 `claude-wechat-channel@0.1.2`（具体版本号）而不是 `latest`
3. 任务结束就 `pkill -f claude-wechat-channel` 停服务，不长跑
4. 默认**不开 YOLO 模式**

## 快速排查

```bash
# 看 channel 进程在不在
ps aux | grep claude-wechat-channel | grep -v grep

# 看绑定凭证
ls ~/.npm/_npx/*/node_modules/claude-wechat-channel/data/ 2>/dev/null

# 强制重新登录(用新二维码)
rm -rf ~/.npm/_npx/*/node_modules/claude-wechat-channel/data/
# 重启 claude --dangerously-load-development-channels server:wechat
```

## 参考链接

- npm 包：https://www.npmjs.com/package/claude-wechat-channel
- 作者 GitHub：https://github.com/fengliu222/claude-wechat-channel
- 微信 ClawBot 官方介绍（腾讯）：https://cloud.tencent.com/developer/article/2644003
