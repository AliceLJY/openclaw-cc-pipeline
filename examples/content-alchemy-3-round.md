# 分段 Content Alchemy：3 轮模式

> 把 Content Alchemy 的 6 个 checkpoint 压缩为 3 轮，适配 Discord 交互。

## 轮次设计

| 轮次 | Stage | CC 执行内容 | 用户交互点 |
|------|-------|-----------|-----------|
| 第1轮 | Stage 1 | 话题挖掘，输出 mining-report.md | 用户选角度/方向 |
| 第2轮 | Stage 2-3.5 | 素材提取 + 分析 + 交叉验证 | 用户确认数据准确性 |
| 第3轮 | Stage 4-5 | 写文章 + 去 AI 味 | 用户审稿 |

## 第1轮 Prompt 模板

```
/content-alchemy 话题：{用户的话题}

只执行 Stage 1 话题挖掘。
目标板块：{AI实操手账 / AI踩坑实录 / AI照见众生 / AI随心分享}
目标读者：关注 AI 的非技术用户

完成 mining-report.md 后停下，输出推荐角度让用户选择。
不要继续执行 Stage 2。
```

## 第2轮 Prompt 模板

```
继续 Stage 2-3.5，sessionId 是 {sessionId}

用户确认：{用户选择的角度和修改意见}

执行素材提取、深度分析和交叉验证。
完成 cross-reference-report.md 后停下，等用户确认数据准确性。
```

## 第3轮 Prompt 模板

```
继续 Stage 4-5 写文章，sessionId 是 {sessionId}

{用户的修改意见（如果有）}

按交叉验证结果写文章，板块风格匹配。
完成 article.md 后输出全文。
```

## Timeout 建议

| 轮次 | 推荐 Timeout | 原因 |
|------|-------------|------|
| 第1轮 | 600s (10min) | 搜索多个来源 |
| 第2轮 | 600s (10min) | 交叉验证耗时最长 |
| 第3轮 | 600s (10min) | 写作 + 去 AI 味检查 |

> 不要设 300s（5 分钟）——实测会差几百毫秒超时。
