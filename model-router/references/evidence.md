# DeepSeek V4 官方数据与验证方法（2026-08-13）

## 1. 模型身份
- `deepseek-v4-flash` → 版本 DeepSeek-V4-Flash-0731
- `deepseek-v4-pro` → 版本 DeepSeek-V4-Pro-0813（2026-08-13 正式版，调用名不变）
- 来源：DeepSeek 官方 API 文档定价页（api-docs.deepseek.com/zh-cn/quick_start/pricing）

## 2. 规格（两者相同）
- 上下文 1M；最大输出 384K
- 思考/非思考模式（默认思考）；FIM 补全仅限非思考模式
- Responses API、Anthropic API、JSON Output、Tool Calls
- 并发限制：Flash 2500 / Pro 500

## 3. 定价（2026-08-13 官方定价页，元/百万 tokens）
| 项目 | Flash | Pro |
| --- | --- | --- |
| 输入（缓存命中） | 0.02 | 0.025 |
| 输入（缓存未命中） | 1 | 3 |
| 输出 | 2 | 6 |

- 官方已预告近期整体上调，并计划高峰定价（每日 9:00–12:00、14:00–18:00 翻倍）：Pro 输出高峰约 12 元/M，Flash 输出高峰约 4 元/M（财联社报道，以正式通知为准）。

## 4. 官方基准（厂商自报，DeepSeek 官方群放出，非独立复测）
| 基准 | Pro-0813 | Flash-0731 | Pro-Preview | Flash-Preview | Opus 4.8 | Fable 5 |
| --- | --- | --- | --- | --- | --- | --- |
| Terminal Bench 2.1 | 87.9 | 82.7 | 72.1 | 61.8 | 85.0 | 88.0 |
| Cybergym | 83.3 | 76.7 | 52.7 | 38.7 | 78.3 | 83.1 |
| AutomationBench (Public) | 31.8 | 25.1 | 12.8 | 10.8 | 27.2 | 29.1 |
| DeepSWE | 62.7 | 54.4 | 12.8 | 7.3 | 58.0 | 70.0 |
| Toolathlon-Verified | 74.1 | 70.3 | 55.9 | 49.7 | 未披露 | 未披露 |
| DSBench-FullStack | 71.1 | 68.7 | 41.8 | 37.0 | 71.6 | 77.2 |
| DSBench-Hard | 67.2 | 59.6 | 31.1 | 25.8 | 未披露 | 未披露 |

- 解读：Pro 在 Terminal Bench 2.1、Cybergym、DeepSWE、AutomationBench 超过 Opus 4.8；HLE/NL2Repo 上 Opus 领先。Flash 达 Pro 的 85–95%，Pro 优势集中在最难一档。

## 5. 本机 API 验证方法（2026-08-13 实测通过）
- API key 位置：`~/.codex/config.toml` 的 `[model_providers.deepseek]` → `experimental_bearer_token`
- `GET https://api.deepseek.com/models` → 应返回 `deepseek-v4-flash`、`deepseek-v4-pro`
- `POST https://api.deepseek.com/chat/completions`：body 必须 UTF-8；max_tokens 要给足，否则思考模式吃光配额会导致 content 为空
- `POST https://api.deepseek.com/responses`：Codex 实际走的端点；返回 200 且 status=completed 即通
- 沙箱内直连可能报“基础连接已关闭”（TLS），需用 require_escalated 提权后测试

## 6. 切换方式与限制（Codex 实测）
- CLI：`codex exec -m/--model deepseek-v4-pro|deepseek-v4-flash`；交互式会话用 `/model`
- 桌面端：每个线程在 `~/.codex/state_5.sqlite` 的 `threads.model` 独立记录；自定义 provider 的 UI 热切换不可靠（openai/codex#15364，2026-05 关闭，未实现）
- 多代理：spawn_agent 支持 `model=deepseek-v4-pro` / `deepseek-v4-flash` 覆盖，但必须 `fork_turns=none` 或少量轮次才生效；整段 fork 会继承父模型
- 当前本机默认：`config.toml` 的 `model = "deepseek-v4-flash"`

## 7. 使用注意
- Pro 输出价为 Flash 的 3 倍；高峰时段双双翻倍，避开高峰能省钱
- 简单任务用 Flash；评分 ≥30 或命中 Pro 清单才升级（流程见 SKILL.md）
## 8. 测试发现（2026-08-13）
- spawn_agent / followup 消息通道在非 OpenAI 提供方下曾不可用：任务文本被放进 encrypted_content 被 DeepSeek 丢弃（上游 openai/codex #37237 / #36493 / #37822）。已应用配置 workaround（2026-08-13）：models.json 两个 DeepSeek 模型 multi_agent_version v2→v1、supports_search_tool true→false；备份 models.json（本地）；重启后探针验证。验证前/失败时回退：API 直调 deepseek-v4-pro + Flash 验收。
- Pro 思考模式可能吃满 max_tokens 导致 content 为空（case-09 首次 16384 空输出，finish_reason=length）。判定方法：content 为空且 finish_reason=length → 思考吃满预算。文档类任务给足预算（max_tokens ≥32768）或改用非思考模式。
- 初测（未重启，2026-08-13）：探针 spawn 仍收不到任务；且 spawn_agent 模型覆盖列表变空（v1 下可能不再支持覆盖）。待完全重启后复测；若 v1 不支持 Pro 覆盖，则 Pro 任务继续走 API 直调回退。
- 二次探针（2026-08-13）：消息仍未送达（子代理回“等待任务”）。若确认已完全重启仍如此，则 v1 workaround 在本机/本构建无效，建议恢复 v2 备份并继续走 API 直调回退。
- 终测结论（2026-08-13）：完全重启后二次探针仍失败 → v1 workaround 在本机无效；已恢复 v2（models.json 与备份一致）。本机 Pro 子代理通道判定不可用，执行通道固定为 API 直调 + Flash 验收；待上游 #37197。
