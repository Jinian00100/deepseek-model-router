# deepseek-model-router

DeepSeek V4 模型路由技能（Codex）：**日常用 Flash，重活自动上 Pro**。

## 这是什么

一个 Codex Skill（`model-router/SKILL.md`）：每次接到新任务，先由 Flash 判定一次——命中 Pro 清单（仓库级/跨模块重构、高难度编码与疑难 bug、核心架构/算法设计、复杂多步 Agent 任务）或返工风险评分 ≥30 才调用 DeepSeek-V4-Pro；否则一直用 Flash。Pro 执行后自动回到 Flash，等待下一次任务判定。

## 特性

- 每个新任务只判定一次，同一任务链不反复切换模型
- 用户手动指定模型（“用 Pro / 用 Flash”）时优先
- Pro 文档/长文任务自动给足输出预算（max_tokens ≥32768）；空输出且 finish_reason=length 时自动加大预算重试一次
- 内置 DeepSeek V4 Pro-0813 / Flash-0731 官方规格、定价与基准数据（2026-08-13）

## 环境要求

- Codex（桌面版或 CLI）
- DeepSeek API 自定义 provider：`base_url = https://api.deepseek.com`，`wire_api = "responses"`
- 模型目录（`models.json` 或模型选择器）中登记 `deepseek-v4-flash` 与 `deepseek-v4-pro`

## 安装

1. 把 `model-router/` 放到 `~/.codex/skills/`（Windows 示例：`C:\Users\<你>\.codex\skills\`）
2. 完全重启 Codex

或使用 skill-installer：

```text
install-skill --repo <你的仓库> --path model-router
```

## 已知限制（重要）

- **上游 bug**：非 OpenAI 提供方（DeepSeek）下，`spawn_agent` / `followup_task` 的任务正文会被 `encrypted_content` 机制丢弃（openai/codex #37237 / #36493 / #37822）。社区 v1 workaround 经完全重启实测无效（2026-08-13）。因此本技能 Pro 执行默认走 **API 直调 deepseek-v4-pro + Flash 验收**；上游修复后恢复 `spawn_agent` 通道。
- 官方基准为厂商自报，非独立复测，数字可能随版本变化。
- 价格与高峰定价以 DeepSeek 官方为准，可能调整。

## 借鉴与致谢

- 清单与思路参考：`zx1160763849-hash/codex-cost-router-skills`、`jinweechen/codex-auto-router`
- 上游 issue：openai/codex #37237、#36493、#37822、#37197

## License

MIT
