# OpenClaw CC Pipeline

Multi-turn Claude Code orchestration via Discord Bot.

> 让 Discord Bot 通过多轮对话调度本地 Claude Code 执行复杂任务。

## What is this?

An orchestration pattern: **Discord Bot (dispatcher) → Task API (relay) → Worker (runner) → Claude Code (executor) → Callback (result delivery)**.

> 一个编排模式：Bot 传话 → API 中转 → Worker 跑腿 → CC 干活 → 回调传结果。

It exposes Claude Code's powerful capabilities (skill system, session persistence, local file access) to a Discord Bot, enabling multi-turn interactive tasks.

```
Discord User
    ↓ "Write me an article"
OpenClaw Bot (Docker)
    ↓ Format request with callbackChannel
Task API (Local Docker)
    ↓ Store task, auto-generate sessionId
Worker (Mac Node.js)
    ↓ Poll → fetch task → claude --print --session-id xxx
Claude Code CLI (Local)
    ↓ Run skill → produce output
Worker
    ↓ docker exec → OpenClaw CLI → send to Discord
Bot → Discord User
    ↓ Include 📎 sessionId, user confirms, next round begins
```

## Why?

| Scenario | Without Pipeline | With Pipeline |
|----------|-----------------|---------------|
| Bot writes article | One shot, no human review | 3 rounds, user confirms each stage |
| Bot runs Skill | Bot calls CC but can't get results back | Callback auto-pushes results to Discord |
| Multi-turn | Each round is isolated, no context | sessionId persists, CC remembers everything |

## Architecture

### Components

| Component | Location | Role | Repo |
|-----------|----------|------|------|
| **OpenClaw Bot** | Local Docker | Receive user commands, format API requests | [openclaw](https://github.com/openclaw/openclaw) |
| **Task API** | Local Docker | HTTP relay, store tasks and results | [openclaw-worker](https://github.com/AliceLJY/openclaw-worker) (server.js) |
| **Worker** | Mac launchd | Poll tasks, invoke CC CLI, report results | [openclaw-worker](https://github.com/AliceLJY/openclaw-worker) (worker.js) |
| **Claude Code** | Mac CLI | Execute skills, generate content | [Claude Code](https://claude.com/claude-code) |

### Data Flow

```
Round 1 (New task):
Bot → POST /claude {prompt, callbackChannel}
    → API auto-generates sessionId
    → Worker fetches task → claude --print --session-id <id> "prompt"
    → CC completes → Worker reports result
    → Worker calls OpenClaw CLI → sends Discord message (with 📎 sessionId)

Round 2+ (Resume):
Bot → POST /claude {prompt, sessionId, callbackChannel}
    → Worker fetches task → claude --print --resume --session-id <id> "prompt"
    → CC resumes with full context from previous rounds
    → Same callback flow
```

## Quick Start

### Prerequisites

- [OpenClaw](https://github.com/openclaw/openclaw) deployed (Discord Bot)
- [Claude Code](https://claude.com/claude-code) installed (local CLI)
- [openclaw-worker](https://github.com/AliceLJY/openclaw-worker) deployed (Task API + Worker)

### 1. Task API: Auto-generate sessionId

> server.js 需要支持 callbackChannel 和自动生成 sessionId。

```javascript
app.post('/claude', auth, (req, res) => {
  const { prompt, timeout = 120000, sessionId, callbackChannel } = req.body;

  const taskId = crypto.randomUUID();
  // Key: auto-generate sessionId so subsequent rounds can --resume
  // 关键：没传 sessionId 时自动生成，确保后续轮次能 --resume
  const effectiveSessionId = sessionId || crypto.randomUUID();

  const task = {
    id: taskId,
    type: 'claude-cli',
    prompt,
    timeout,
    sessionId: effectiveSessionId,
    callbackChannel: callbackChannel || null,
    status: 'pending',
    createdAt: Date.now()
  };

  tasks.set(taskId, task);
  res.json({ taskId, sessionId: effectiveSessionId });
});
```

### 2. Worker: Callback with sessionId

> Worker 在 CC 完成后通知 Discord，消息里带 sessionId。

```javascript
function notifyOpenClaw(task, result) {
  if (task.type !== 'claude-cli' || !task.callbackChannel) return;

  const summary = (result.stdout || '').slice(-1500) || '(no output)';
  const status = result.exitCode === 0 ? 'Done' : 'Failed';
  const duration = result.duration ? `${Math.round(result.duration / 1000)}s` : 'unknown';

  // Include sessionId for multi-turn tracking
  // 回调消息带上 sessionId，Bot 从中提取用于后续轮次
  const sessionId = result.metadata?.sessionId;
  const sessionInfo = sessionId ? `\n📎 sessionId: \`${sessionId}\`` : '';

  const message = `**CC Task ${status}** (${duration})${sessionInfo}\n\n${summary}`;

  // Use docker exec to call OpenClaw CLI, with retry
  const { execFile } = require('child_process');
  const maxRetries = 3;
  let attempt = 0;

  function trySend() {
    attempt++;
    execFile('docker', [
      'exec', 'openclaw-antigravity',
      'node', 'openclaw.mjs', 'message', 'send',
      '--channel', 'discord',
      '--target', `channel:${task.callbackChannel}`,
      '-m', message
    ], { timeout: 15000 }, (error) => {
      if (error && attempt < maxRetries) {
        setTimeout(trySend, 5000); // Retry after 5s
      }
    });
  }
  trySend();
}
```

### 3. Bot MEMORY.md Configuration

> Bot 需要知道怎么调 Pipeline。详见 [examples/bot-memory-snippet.md](examples/bot-memory-snippet.md)。

Key points for the Bot:
- Act as a **messenger**, don't make decisions for the user
- Extract `📎 sessionId` from callback messages for subsequent rounds
- Append user feedback to the next round's prompt
- Wait for user confirmation between rounds

### 4. Task Cleanup Strategy

> 已完成的结果不要太快清理，否则 Worker 取不到。

```javascript
setInterval(() => {
  const now = Date.now();
  const TASK_EXPIRE_MS = 15 * 60 * 1000;   // Incomplete tasks: 15 min
  const RESULT_EXPIRE_MS = 30 * 60 * 1000; // Completed results: 30 min

  for (const [taskId, task] of tasks) {
    const age = now - task.createdAt;
    if (results.has(taskId)) {
      if (age > RESULT_EXPIRE_MS) { tasks.delete(taskId); results.delete(taskId); }
    } else if (age > TASK_EXPIRE_MS) {
      tasks.delete(taskId);
    }
  }
}, 60000);
```

## Real-World Example: Content Alchemy in 3 Rounds

> 用这个 Pipeline 实现了分段写公众号文章。详见 [examples/content-alchemy-3-round.md](examples/content-alchemy-3-round.md)。

| Round | CC Executes | User Interaction |
|-------|-------------|-----------------|
| 1 | Stage 1: Topic mining | User picks angle |
| 2 | Stage 2-3.5: Source extraction + cross-reference verification | User confirms data accuracy |
| 3 | Stage 4-5: Write article + de-AI-ify | User reviews draft |

Each round preserves full CC context via sessionId. User has complete control between rounds.

## Pitfall Guide

> 以下所有坑都是一个晚上踩出来的。All bugs discovered in a single evening of testing.

### 1. Timeout needs buffer

**Symptom**: CC ran 300,322ms, timeout was 300,000ms. Killed for 322ms overshoot. Five minutes of work wasted.

> CC 跑了 300,322ms，超时设 300,000ms，差 322ms 被杀。

**Fix**: Add 30s buffer in Worker. Set default to 10 minutes.

```javascript
const effectiveTimeout = (timeout || CONFIG.defaultTimeout) + 30000;
```

### 2. SessionId must be auto-generated

**Symptom**: Round 2 `--resume` exits in 596ms. Session doesn't exist because Round 1 had no sessionId.

> 第一轮没 sessionId → session 不存在 → 第二轮秒退。

**Fix**: API auto-generates when not provided.

### 3. Callback must have retry

**Symptom**: CC finished, but bot container was restarting (Opus 4.6 upgrade in another terminal). `fetch failed`. User never gets the result.

> 回调发送时 bot 刚好在重启，用户永远收不到结果。

**Fix**: 3 retries, 5s apart.

### 4. DM channels don't support CLI callback

**Symptom**: Testing in Discord DM, callback always fails.

> DM 频道不支持 OpenClaw CLI 的 `message send` 命令。

**Fix**: Only use server channels for callback.

### 5. Sessions are scoped by working directory

**Symptom**: Worker creates session from `$HOME`, manual CLI from another directory can't find it.

> CC 的 session 按工作目录分组，换目录就找不到。

**Fix**: Always start CC from the same working directory.

### 6. Don't clean up completed results too fast

**Symptom**: CC result was ready, but the 5-minute cleanup timer deleted it before Worker could fetch.

> 结果 5 分钟就删，Worker 还没取走。

**Fix**: Separate timers — 15 min for incomplete tasks, 30 min for completed results.

## Related Projects

- [openclaw-worker](https://github.com/AliceLJY/openclaw-worker) — Task API + Worker implementation
- [openclaw-config](https://github.com/AliceLJY/openclaw-config) — Bot configuration backup (with patches)
- [content-alchemy](https://github.com/AliceLJY/content-alchemy) — WeChat article writing skill (real-world use case)
- [openclaw-mas-guide](https://github.com/AliceLJY/openclaw-mas-guide) — Multi-Agent System (MAS) configuration guide

## License

MIT
