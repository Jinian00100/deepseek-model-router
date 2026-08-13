# deepseek-model-router

> **[English](./README.en.md)**

> Codex 模型路由技能：**日常用 Flash，重活自动上 Pro**，Pro 干完自动回到 Flash。

一个为 DeepSeek V4 用户设计的 Codex Skill。它让 Codex 每次接到新任务时先做一次低成本判定：任务够“重”才升级到 DeepSeek-V4-Pro，否则全程使用 DeepSeek-V4-Flash。既能保住日常任务的响应速度，又能把真正的硬骨头交给更强的模型，不浪费 token。

## 它解决什么问题

- 全用 Flash：重活质量不够，返工更贵；
- 全用 Pro：日常简单任务白白花 3 倍钱；
- 手动切换：忘了切、切错、来回切，成本失控。

本技能把“该不该用 Pro”变成一条可执行的规则，并内置执行、验收、回退的完整闭环。

## 工作原理

1. 主对话始终是 Flash（`deepseek-v4-flash`），不做热切换。
2. 每个新任务判定一次；同一任务链内的追问/修改/返修沿用首次判定。
3. 命中 **Pro 清单**，或**返工风险评分 ≥30** → 调用 `deepseek-v4-pro`。
4. Pro 执行完 → 回到 Flash，等待下一个任务重新判定。
5. 用户手动说“用 Pro / 用 Flash”时，永远听用户的。

### Pro 清单（命中任一即上 Pro）

- 仓库级/跨模块重构（牵一发动全身、依赖关系不清）
- 高难度编码或疑难 bug（跨文件深链路、性能/并发/安全类）
- 核心架构、算法设计（需要长链推理）
- 复杂多步 Agent 任务（自动化、安全、终端操作，完成率要求高）

### 返工风险评分表（可叠加，≥30 上 Pro）

| 场景 | 分值 |
| --- | --- |
| 改动跨 3+ 文件/模块且依赖关系不明 | +30 |
| 无法快速本地验证（无测试/无法运行/结果难自查） | +25 |
| 一次交付（对外/生产/质量要求高） | +20 |
| 逻辑链长（多步骤/多条件/状态多） | +20 |
| 长分镜/长剧本批量产出（>50 镜、跨多集、强连续性） | +15 |
| 结构清晰、可迭代、影响面小 | 0（Flash） |

## 安装

### 方式一：skill-installer（推荐）

```text
install-skill --repo Jinian00100/deepseek-model-router --path model-router
```

### 方式二：手动

1. 克隆或下载本仓库；
2. 把 `model-router/` 整个文件夹放进 `~/.codex/skills/`（Windows 示例：`C:\Users\<你>\.codex\skills\`）；
3. 完全重启 Codex。

## 环境要求

- Codex（桌面版或 CLI）
- DeepSeek API，自定义 provider 配置：

```toml
[model_providers.deepseek]
name = "deepseek"
base_url = "https://api.deepseek.com"
wire_api = "responses"
experimental_bearer_token = "<YOUR_API_KEY>"
```

- 模型目录中登记 `deepseek-v4-flash` 与 `deepseek-v4-pro`

## 执行通道与已知限制（重要）

- **上游 bug**：非 OpenAI 提供方（DeepSeek）下，`spawn_agent` / `followup_task` 的任务正文会被 `encrypted_content` 机制丢弃（openai/codex #37237 / #36493 / #37822）。截至 2026-08-13，社区 v1 workaround 经完全重启实测无效。
- 因此本技能 Pro 执行**默认走 API 直调 `deepseek-v4-pro` + Flash 验收**；上游修复后会切回 `spawn_agent` 通道。
- Pro 文档/长文任务：自动给足输出预算（`max_tokens ≥32768`）；空输出且 `finish_reason=length` 说明思考吃满预算，自动加大预算重试一次。
- 官方基准为厂商自报，非独立复测；价格与高峰定价以 DeepSeek 官方为准，可能调整。

## 文件结构

```text
deepseek-model-router/
├── README.md
├── LICENSE
└── model-router/
    ├── SKILL.md              # 技能主文件：判定、评分、执行、回退
    ├── agents/openai.yaml    # 技能 UI 元数据
    └── references/
        └── evidence.md       # 官方规格/定价/基准 + 实测记录
```

## 致谢与参考

- 思路参考：`zx1160763849-hash/codex-cost-router-skills`、`jinweechen/codex-auto-router`
- 上游 issue：openai/codex #37237、#36493、#37822、#37197

## License

[MIT](./LICENSE)
