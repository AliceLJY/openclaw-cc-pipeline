# Bot MEMORY.md 多轮对话配置片段

> 把这段加到你 Bot 的 MEMORY.md 里，Bot 就知道怎么调度 CC 多轮对话。

```markdown
## CC 多轮对话调度（Pipeline 模式）

### 架构
- Bot 是传话人，不替用户做决定
- 每轮：Bot 发 API 请求 → CC 执行 → 回调带结果 + sessionId → 展示给用户 → 用户确认后下一轮

### 发起第1轮（新任务）
调用本地 Task API：
POST http://localhost:3456/claude
Headers: Authorization: Bearer <WORKER_TOKEN>
Body:
{
  "prompt": "/content-alchemy 话题：xxx\n\n只执行 Stage 1 话题挖掘，完成后停下等待用户确认。",
  "callbackChannel": "<当前 Discord 频道 ID>",
  "timeout": 600000
}

API 会返回 taskId 和自动生成的 sessionId。

### 等待回调
CC 跑完后，Worker 会自动发消息到 callbackChannel：
**CC 任务完成**（耗时 236s）
📎 sessionId: `d89e13e1-6de9-4fec-885b-a24bd1aad890`

<输出内容>

### 发起后续轮次
从回调消息提取 sessionId，加到请求里：
POST http://localhost:3456/claude
Body:
{
  "prompt": "继续 Stage 2-3.5 分析验证。\n\n用户确认：选角度A，继续推进。",
  "sessionId": "d89e13e1-6de9-4fec-885b-a24bd1aad890",
  "callbackChannel": "<频道 ID>",
  "timeout": 600000
}

### 规则
- 每轮结果展示给用户，等确认后才发起下一轮
- 用户的修改意见/选择直接拼到下一轮 prompt 开头
- 不要自作主张跳过轮次或替用户选择
- sessionId 格式：📎 sessionId: `xxx`，从回调消息正则提取
```
