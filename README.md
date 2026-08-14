# deepseek-model-router

> **[English](./README.en.md)**

> Codex 模型路由技能：**日常用 Flash，重活建议手动切 Pro**，Pro 干完回到 Flash。

一个为 DeepSeek V4 用户设计的 Codex Skill。它让 Codex 每次接到新任务时先做一次低成本判定：命中 **Pro 清单**或**返工风险评分 ≥30** 才提示你切到 DeepSeek-V4-Pro，否则全程使用 DeepSeek-V4-Flash。既能保住日常任务的响应速度，又能把真正的硬骨头交给更强的模型，不浪费 token。

## 它解决什么问题

- 全用 Flash：重活质量不够，返工更贵；
- 全用 Pro：日常简单任务白白花约 3 倍钱；
- 手动切换凭感觉：忘了切、切错、来回切，成本失控。

本技能把“该不该用 Pro”变成一条可执行的规则，并内置切换纪律、执行、验收、回退的完整闭环。

## 工作原理

1. 默认使用 Flash（`deepseek-v4-flash`）；每个新任务判定一次，同一任务链内的追问/修改/返修沿用首次判定。
2. 命中 **Pro 清单**，或**返工风险评分 ≥30** → 先一句话提示“建议上 Pro，界面切一下”。
3. **首选通道**：你在 Codex 桌面端界面手动切到 Pro，当前对话直接以 Pro 执行。
4. **兜底通道**：你不切换但明确要求继续时，才走 API 直调 `deepseek-v4-pro` + Flash 验收。
5. Pro 交付验收后，建议切回 Flash；下一次新任务重新判定。
6. 用户手动说“用 Pro / 用 Flash”时，永远听用户的。
7. 切换纪律：一次任务链最多切换一次；中途升级必须先写状态交接块；任务完成 ≥70% 不中途抢救。

### Pro 清单（命中任一即建议上 Pro）

- 仓库级/跨模块重构（牵一发动全身、依赖关系不清）
- 高难度编码或疑难 bug（跨文件深链路、性能/并发/安全类）
- 核心架构、算法设计（需要长链推理）
- 复杂多步 Agent 任务（自动化、安全、终端操作，完成率要求高）
- 长分镜/长剧本批量产出（>50 镜或跨多集；≤20 镜单集先 Flash 小样）
- 网文长文（>3K 字，或跨多章/全文一致性修订）

### 返工风险评分表（可叠加，≥30 建议上 Pro）

| 场景 | 分值 |
| --- | --- |
| 改动跨 3+ 文件/模块且依赖关系不明 | +30 |
| 无法快速本地验证（无测试/无法运行/结果难自查） | +25 |
| 一次交付（对外/生产/质量要求高） | +20 |
| 逻辑链长（多步骤/多条件/状态多） | +20 |
| 结构清晰、可迭代、影响面小 | 0（Flash） |

### 创作任务量级清单（直接判定，不靠评分表）

- 短剧分镜 ≤20 镜/单集 → Flash 先出小样；质量不达标再升 Pro。
- 网文单章正文 ≤2K 字 → Flash；长文/多章/全文一致性修订 → Pro。
- 整理、总结、问答、拆解素材 → Flash。

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

- **首选**：Codex 桌面端界面手动切换（实测可用，最稳）。
- **兜底**：用户不切换但明确要求继续时，API 直调 `deepseek-v4-pro` + Flash 验收；Pro 文档/长文任务给足输出预算（`max_tokens ≥32768`），空输出且 `finish_reason=length` 说明思考吃满预算，自动加大预算重试一次；长输出任务默认关思考模式。
- **上游 bug**：非 OpenAI 提供方（DeepSeek）下，`spawn_agent` / `followup_task` 的任务正文会被 `encrypted_content` 机制丢弃（openai/codex #37237 / #36493 / #37822）。截至 2026-08-15，社区 workaround 经完全重启实测无效；上游修复后会切回 `spawn_agent` 通道。
- 官方基准为厂商自报，非独立复测；价格与高峰定价以 DeepSeek 官方为准，可能调整。

## 文件结构

```text
deepseek-model-router/
├── README.md
├── README.en.md
├── LICENSE
└── model-router/
    ├── SKILL.md                        # 技能主文件：判定、切换纪律、执行、验收
    ├── agents/openai.yaml              # 技能 UI 元数据
    └── references/
        ├── evidence.md                 # 官方规格/定价/基准 + 切换方式验证
        ├── experiment-2026-08-14.md    # 3 版路由实测（13 任务×2 轮，13/13 定案）
        └── 正向提示词清单.md           # 交付前自查清单（正向提示词版）
```

## 版本

- **v4.4（2026-08-15）**：13 任务×2 轮、104 次调用大样本实测，判定 13/13 且边界评分稳定；量级清单 + Pro 清单 + 评分表定案；长输出任务默认关思考、空输出自动兜底重跑。
- 完整变更记录见 `model-router/SKILL.md` 顶部「版本变化」。

## 致谢与参考

- 思路参考：`zx1160763849-hash/codex-cost-router-skills`、`jinweechen/codex-auto-router`
- 上游 issue：openai/codex #37237、#36493、#37822、#37197

## License

[MIT](./LICENSE)
