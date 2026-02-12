# OpenClaw CC Pipeline Skill

**Name**: openclaw-cc-pipeline
**Description**: Orchestrate multi-turn Claude Code tasks via Discord Bot, with sessionId persistence and callback delivery.
**When to use**: When a user asks you to run a multi-round pipeline through an OpenClaw Bot, dispatch tasks to a local Claude Code CLI via Task API, or execute the Content Alchemy 3-round article writing workflow.

> 当用户要求通过 Discord Bot 执行多轮 CC 任务时使用此 Skill。

---

## Prerequisites

Before starting, verify these components are running:

1. **Task API** (local Docker, port 3456) -- HTTP relay that stores tasks and auto-generates sessionIds.
   - Verify: `curl -s http://localhost:3456/health` should return OK.
2. **Worker** (Mac launchd service) -- Polls Task API, invokes `claude --print`, reports results.
   - Verify: `launchctl list | grep worker` should show the worker service.
3. **OpenClaw Bot** (local Docker, e.g. `openclaw-antigravity`) -- Receives user commands on Discord, formats API requests, displays callback results.
   - Verify: `docker ps | grep openclaw` should show the bot container running.
4. **Claude Code CLI** installed locally.
5. **WORKER_TOKEN** environment variable or known auth token for Task API.

If any prerequisite is missing, stop and inform the user which component needs to be set up. Point them to [openclaw-worker](https://github.com/AliceLJY/openclaw-worker) for Task API + Worker setup.

---

## Multi-Turn Orchestration Protocol

Follow this protocol exactly when orchestrating a multi-turn pipeline.

### Step 1: Initiate Round 1 (New Task)

Generate and send the first task to the Task API. The API auto-generates a `sessionId`.

```bash
curl -s -X POST http://localhost:3456/claude \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $WORKER_TOKEN" \
  -d '{
    "prompt": "<ROUND_1_PROMPT>",
    "callbackChannel": "<DISCORD_CHANNEL_ID>",
    "timeout": 600000
  }'
```

- **prompt**: The instruction for Claude Code to execute. Include the skill invocation (e.g., `/content-alchemy`) and explicit stop instruction (e.g., "Complete Stage 1 then stop, do not proceed to Stage 2.").
- **callbackChannel**: The Discord channel ID where results should be delivered. Must be a server channel (DM channels do NOT work with OpenClaw CLI callback).
- **timeout**: Set to 600000 (10 minutes). Never use 300000 -- real tasks routinely exceed 5 minutes by a few hundred milliseconds and get killed.

The API responds with:
```json
{ "taskId": "xxx", "sessionId": "auto-generated-uuid" }
```

Record the `sessionId`. You will need it for all subsequent rounds.

### Step 2: Wait for Callback

After Claude Code completes the task, the Worker automatically delivers results to the Discord channel via `docker exec` + OpenClaw CLI. The callback message includes:

```
**CC Task Done** (236s)
📎 sessionId: `d89e13e1-6de9-4fec-885b-a24bd1aad890`

<output content>
```

If you are the Bot: extract the `sessionId` from the callback message using regex pattern `📎 sessionId: \x60([^\x60]+)\x60`. Present the results to the user. **Do NOT auto-advance to the next round.** Wait for explicit user confirmation.

### Step 3: Collect User Feedback

Present the round's output to the user. Ask for:
- Confirmation to proceed
- Any modifications, angle choices, or corrections
- Whether to abort the pipeline

Record the user's feedback verbatim. It will be prepended to the next round's prompt.

### Step 4: Initiate Subsequent Rounds (Resume)

Send the next task with the same `sessionId` to resume CC's context:

```bash
curl -s -X POST http://localhost:3456/claude \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $WORKER_TOKEN" \
  -d '{
    "prompt": "<ROUND_N_PROMPT>\n\nUser feedback: <USER_FEEDBACK>",
    "sessionId": "<SESSION_ID_FROM_ROUND_1>",
    "callbackChannel": "<DISCORD_CHANNEL_ID>",
    "timeout": 600000
  }'
```

Key differences from Round 1:
- **sessionId** is included -- the Worker will invoke `claude --print --resume --session-id <id>`, so CC retains full context from all previous rounds.
- **User feedback** is prepended to the prompt so CC sees it immediately.

Repeat Steps 2-4 for each subsequent round until the pipeline completes.

### Step 5: Final Delivery

After the last round completes, present the final output to the user. If the output is an article or document, offer to:
- Copy to clipboard
- Save to a specific file path
- Proceed to publishing workflow (if applicable)

---

## Content Alchemy 3-Round Workflow

This is the primary real-world workflow. It compresses Content Alchemy's 6 checkpoints into 3 human-reviewed rounds.

> 把 Content Alchemy 的 6 个 checkpoint 压缩为 3 轮人工审核。

### Round 1: Stage 1 -- Topic Mining

**Prompt template:**
```
/content-alchemy Topic: {user's topic}

Execute Stage 1 topic mining only.
Target section: {AI Hands-On / AI Bug Diary / AI Human Observer / AI Gallery}
Target audience: Non-technical users interested in AI.

After completing mining-report.md, stop and output recommended angles for the user to choose.
Do NOT proceed to Stage 2.
```

**What CC does**: Searches multiple sources, generates `mining-report.md`, outputs angle recommendations.

**User interaction**: User picks an angle/direction, optionally provides additional guidance.

### Round 2: Stage 2-3.5 -- Analysis & Validation

**Prompt template:**
```
Continue to Stage 2-3.5.

User confirmed: {user's chosen angle and any modifications}

Execute source extraction, deep analysis, and cross-reference validation.
After completing cross-reference-report.md, stop and let the user confirm data accuracy.
```

**What CC does**: Extracts sources, analyzes, cross-validates claims. Outputs `cross-reference-report.md`.

**User interaction**: User confirms data accuracy, flags any corrections needed.

### Round 3: Stage 4-5 -- Article Writing

**Prompt template:**
```
Continue to Stage 4-5 article writing.

{user's corrections or notes, if any}

Write the article based on cross-reference results. Match the target section's style.
After completing article.md, output the full article text.
```

**What CC does**: Writes full article, applies de-AI-ification pass. Outputs `article.md`.

**User interaction**: User reviews the draft. Pipeline complete.

### Timeout Guidance

| Round | Recommended Timeout | Reason |
|-------|-------------------|--------|
| 1 | 600s (10 min) | Searches multiple sources |
| 2 | 600s (10 min) | Cross-validation is the most time-consuming step |
| 3 | 600s (10 min) | Writing + de-AI-ification pass |

> Never set timeout to 300s (5 min). Real-world tests show tasks exceeding 300s by a few hundred milliseconds, getting killed and wasting all work.

---

## Hook Configuration Reference

To enable callback support in your OpenClaw Bot, add this to `openclaw.json`:

```json
{
  "hooks": {
    "enabled": true,
    "token": "your-callback-token",
    "path": "/hooks",
    "internal": {
      "enabled": true,
      "entries": {
        "session-memory": {
          "enabled": true
        }
      }
    }
  }
}
```

The `session-memory` hook enables session persistence within the Bot, which complements the CC-side `--session-id` persistence.

---

## Bot Memory Configuration

Add the following to your Bot's MEMORY.md so it knows how to dispatch pipeline tasks. See [examples/bot-memory-snippet.md](examples/bot-memory-snippet.md) for a ready-to-paste version.

Core rules for the Bot:
1. **Act as a messenger.** Never make decisions for the user. Never skip rounds. Never auto-advance.
2. **Extract sessionId from callbacks.** Parse `📎 sessionId: \`xxx\`` from every callback message.
3. **Prepend user feedback.** When the user provides modifications or choices, prepend them to the next round's prompt.
4. **Wait for confirmation.** After each round's results are displayed, explicitly ask the user before starting the next round.

---

## Troubleshooting

### Timeout kills task by milliseconds
**Symptom**: CC ran 300,322ms but timeout was 300,000ms. Task killed, all progress lost.
**Fix**: Always add a 30s buffer in Worker config. Use 600s (10 min) as default timeout, not 300s.

### Round 2 --resume exits instantly (596ms)
**Symptom**: Second round finishes in under a second with no output.
**Cause**: Round 1 had no sessionId, so the session never existed. `--resume` finds nothing and exits.
**Fix**: Ensure Task API auto-generates sessionId when not provided. Verify the API response includes `sessionId` after Round 1.

### Callback never arrives
**Symptom**: CC completed successfully but no message appears in Discord.
**Cause**: Bot container was restarting during callback (e.g., upgrade in progress). `docker exec` failed.
**Fix**: Worker implements 3 retries with 5s intervals. Check `docker ps` to confirm bot container is up.

### Callback fails in DM channels
**Symptom**: Callback always errors when testing in Discord DMs.
**Cause**: OpenClaw CLI `message send` does not support DM channel targets.
**Fix**: Only use server (guild) channels for `callbackChannel`.

### Session not found across directories
**Symptom**: Worker created the session but manual `claude --resume` from a different directory cannot find it.
**Cause**: Claude Code sessions are scoped by working directory.
**Fix**: Always start CC from the same working directory. Configure Worker's `cwd` to a consistent path.

### Completed results disappear before Worker fetches them
**Symptom**: Task API shows task completed but Worker gets empty result.
**Cause**: Cleanup timer deleted completed results too quickly (was 5 min).
**Fix**: Use separate cleanup timers -- 15 min for incomplete tasks, 30 min for completed results.
