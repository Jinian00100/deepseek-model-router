# deepseek-model-router

> **[中文](./README.md)**

> A Codex model-routing skill: **use Flash for daily tasks, escalate to Pro for heavy lifting**, then automatically return to Flash.

A Codex Skill built for DeepSeek V4 users. Every time Codex receives a new task, it runs a low-cost preflight decision: escalate to DeepSeek-V4-Pro only when the task is heavy enough; otherwise stay on DeepSeek-V4-Flash. This keeps everyday responses fast and saves the hard problems for the stronger model — without burning tokens.

## What problem it solves

- All-Flash: heavy tasks suffer in quality, and rework costs more;
- All-Pro: simple daily tasks waste 3x the money;
- Manual switching: you forget, mis-switch, or flip-flop, and costs get out of control.

This skill turns “should I use Pro?” into an executable rule, with a complete loop of execution, acceptance, and fallback.

## How it works

1. The main conversation always runs Flash (`deepseek-v4-flash`); no hot-swapping.
2. Each new task is judged once; follow-ups, edits, and rework within the same task chain keep the first decision.
3. If it hits the **Pro checklist** or the **rework-risk score ≥30** → call `deepseek-v4-pro`.
4. After Pro finishes → return to Flash, waiting for the next task.
5. When the user explicitly says “use Pro / use Flash”, the user always wins.

### Pro checklist (any hit → Pro)

- Repository-level / cross-module refactoring (wide blast radius, unclear dependencies)
- Hard coding or elusive bugs (deep multi-file chains, performance / concurrency / security)
- Core architecture or algorithm design (long-chain reasoning)
- Complex multi-step agent tasks (automation, security, terminal operations, high completion requirements)

### Rework-risk scoring (stackable; ≥30 → Pro)

| Scenario | Score |
| --- | --- |
| Changes span 3+ files/modules with unclear dependencies | +30 |
| Cannot be verified quickly (no tests / cannot run / hard to self-check) | +25 |
| One-shot delivery (external / production / high quality bar) | +20 |
| Long logic chain (many steps / conditions / states) | +20 |
| Batch storyboard / script production (>50 shots, multi-episode, strong continuity) | +15 |
| Clear structure, iterable, small blast radius | 0 (Flash) |

## Installation

### Option 1: skill-installer (recommended)

```text
install-skill --repo Jinian00100/deepseek-model-router --path model-router
```

### Option 2: manual

1. Clone or download this repository;
2. Put the whole `model-router/` folder into `~/.codex/skills/` (Windows example: `C:\Users\<you>\.codex\skills\`);
3. Fully restart Codex.

## Requirements

- Codex (desktop or CLI)
- DeepSeek API with a custom provider:

```toml
[model_providers.deepseek]
name = "deepseek"
base_url = "https://api.deepseek.com"
wire_api = "responses"
experimental_bearer_token = "<YOUR_API_KEY>"
```

- `deepseek-v4-flash` and `deepseek-v4-pro` registered in your model catalog

## Execution channel & known limitations (important)

- **Upstream bug**: with non-OpenAI providers (e.g. DeepSeek), task payloads sent via `spawn_agent` / `followup_task` are dropped by the `encrypted_content` mechanism (openai/codex #37237 / #36493 / #37822). As of 2026-08-13, the community v1 workaround was tested after a full restart and does not work.
- Therefore, Pro execution defaults to **direct API calls to `deepseek-v4-pro` + Flash verification**; we will switch back to `spawn_agent` once upstream fixes it.
- Pro document / long-form tasks: output budget is set high automatically (`max_tokens ≥32768`); empty output with `finish_reason=length` means thinking consumed the budget → retry once with a larger budget.
- Official benchmarks are vendor-reported, not independently replicated; pricing and peak-hour pricing are subject to DeepSeek's official announcements.

## Repository structure

```text
deepseek-model-router/
├── README.md
├── README.en.md
├── LICENSE
└── model-router/
    ├── SKILL.md              # Skill entry: decision, scoring, execution, fallback
    ├── agents/openai.yaml    # Skill UI metadata
    └── references/
        └── evidence.md       # Official specs / pricing / benchmarks + field notes
```

## Credits & references

- Ideas: `zx1160763849-hash/codex-cost-router-skills`, `jinweechen/codex-auto-router`
- Upstream issues: openai/codex #37237, #36493, #37822, #37197

## License

[MIT](./LICENSE)
