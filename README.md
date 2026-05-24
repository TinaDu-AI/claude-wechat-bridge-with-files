# claude-wechat-bridge-with-files

让 Claude Code（CLI）通过微信 ClawBot 收发消息的桥接配置 skill，**带图片 / 文件 / 语音 / 视频补丁**。

基于第三方 npm 包 [`claude-wechat-channel`](https://www.npmjs.com/package/claude-wechat-channel)（fengliu222 维护）+ 微信官方 ClawBot 接口。原版只透传文本，本 skill 自带一个补丁包给 `monitor.js` 加上：

- 收图片 → 落 `~/.wechat-claude/media/`，在消息里注入 `[图片: <绝对路径>]`
- 收文件（PDF / DOCX / ...）→ 同上，注入 `[文件: <绝对路径>]`
- 收语音 → `[语音文件: <绝对路径>]`
- 收视频 → `[视频: <绝对路径>]`

Claude 拿到路径后用 Read tool 直接读：图片靠多模态看，PDF 用 `pypdf` 抽文本。

## 装一下

1. 把整个 repo 拖到 `~/.claude/skills/claude-wechat-bridge/`
2. 按 [`SKILL.md`](./SKILL.md) 走完 1-4 步装好原版 bridge
3. 第 5 步装图片文件补丁：
   ```bash
   bash ~/.claude/skills/claude-wechat-bridge/apply-media-patch.sh
   pkill -f claude-wechat-channel
   ```

## 适用

- 需要在不在电脑前的时候继续给 Claude 喂任务（语音、截图、PDF、文件）
- 让 Claude 处理一份手机微信里收到的 BP / 报告 / 名片
- ClawBot loop 类长任务的人机互动

## 不适用 / 限制

- 微信 ClawBot 接口本身**单向**：只能收，Claude 不能主动给你发图片/文件
- macOS / Linux / Windows 都行，Mac 测过最多
- 必须电脑常开

## License

MIT
