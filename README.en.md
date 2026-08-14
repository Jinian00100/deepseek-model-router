# deepseek-model-router

> **[中文](./README.md)**

> A Codex model-routing skill: **use Flash for daily tasks, escalate to Pro for heavy lifting**, then return to Flash.

A Codex Skill built for DeepSeek V4 users. Every time Codex receives a new task, it runs a low-cost preflight decision: suggest switching to DeepSeek-V4-Pro only when the task hits the **Pro checklist** or the **rework-risk score ≥30**; otherwise stay on DeepSeek-V4-Flash. This keeps everyday responses fast and saves the hard problems for the stronger model — without burning tokens.

## What problem it solves

- All-Flash: heavy tasks suffer in quality, and rework costs more;
- All-Pro: simple daily tasks waste ~3x the money;
- Manual switching by feel: you forget, mis-switch, or flip-flop, and costs get out of control.

This skill turns "should I use Pro?" into an executable rule, with a complete loop of decision, switching discipline, execution, acceptance, and fallback.

## How it works

1. Default to Flash (`deepseek-v4-flash`); each new task is judged once; follow-ups, edits, and rework within the same task chain keep the first decision.
2. If it hits the **Pro checklist** or the **rework-risk score ≥30** → first prompt "suggest switching to Pro in the UI".
3. **Primary channel**: you manually switch to Pro in the Codex desktop UI, and the current conversation runs directly on Pro.
4. **Fallback channel**: only if you don't switch but explicitly ask to continue, the skill calls the `deepseek-v4-pro` API directly with Flash verification.
5. After Pro finishes and is accepted, it suggests switching back to Flash; the next new task is judged again.
6. When the user explicitly says "use Pro / use Flash", the user always wins.
7. Switching discipline: at most one switch per task chain; mid-task upgrades require a state-handoff block; do not rescue mid-task when ≥70% complete.

### Pro checklist (any hit → suggest Pro)

- Repository-level / cross-module refactoring (wide blast radius, unclear dependencies)
- Hard coding or elusive bugs (deep multi-file chains, performance / concurrency / security)
- Core architecture or algorithm design (long-chain reasoning)
- Complex multi-step agent tasks (automation, security, terminal operations, high completion requirements)
- Batch storyboard / script production (>50 shots or multi-episode; ≤20 shots per single episode starts with a Flash sample)
- Long webnovel text (>3K words, or cross-chapter / full-text consistency revision)

### Rework-risk scoring (stackable; ≥30 → suggest Pro)

| Scenario | Score |
| --- | --- |
| Changes span 3+ files/modules with unclear dependencies | +30 |
| Cannot be verified quickly (no tests / cannot run / hard to self-check) | +25 |
| One-shot delivery (external / production / high quality bar) | +20 |
| Long logic chain (many steps / conditions / states) | +20 |
| Clear structure, iterable, small blast radius | 0 (Flash) |

### Creative-task volume checklist (direct judgment, not scoring)

- Storyboard ≤20 shots / single episode → Flash sample first; upgrade to Pro if quality is insufficient.
- Webnovel single chapter ≤2K words → Flash; long chapters / multi-chapter / full-text consistency revision → Pro.
- Organizing, summarizing, Q&A, and material breakdowns → Flash.

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

## Execution channels & known limitations (important)

- **Primary**: manual switch in the Codex desktop UI (verified, most stable).
- **Fallback**: only when you don't switch but explicitly ask to continue — direct API call to `deepseek-v4-pro` + Flash verification. Pro document / long-form tasks set a high output budget (`max_tokens ≥32768`); empty output with `finish_reason=length` means thinking consumed the budget → retry once with a larger budget; long-output tasks default to non-thinking mode.
- **Upstream bug**: with non-OpenAI providers (e.g. DeepSeek), task payloads sent via `spawn_agent` / `followup_task` are dropped by the `encrypted_content` mechanism (openai/codex #37237 / #36493 / #37822). As of 2026-08-15, the community workaround was tested after a full restart and still does not work; we will switch back to `spawn_agent` once upstream fixes it.
- Official benchmarks are vendor-reported, not independently replicated; pricing and peak-hour pricing are subject to DeepSeek's official announcements.

## Repository structure

```text
deepseek-model-router/
├── README.md
├── README.en.md
├── LICENSE
└── model-router/
    ├── SKILL.md                        # Skill entry: decision, switching discipline, execution, acceptance
    ├── agents/openai.yaml              # Skill UI metadata
    └── references/
        ├── evidence.md                 # Official specs / pricing / benchmarks + channel verification
        ├── experiment-2026-08-14.md    # 3-round routing experiments (13 tasks × 2 rounds, 13/13 final)
        └── 正向提示词清单.md           # Pre-delivery self-check list (positive-prompt version)
```

## Version

- **v4.4 (2026-08-15)**: large-sample experiments (13 tasks × 2 rounds, 104 calls) finalized routing decisions 13/13 with stable boundary scores; volume checklist + Pro checklist + scoring table finalized; long-output tasks default to non-thinking mode with automatic retry on empty output.
- Full changelog: see "Version changes" at the top of `model-router/SKILL.md`.

## Credits & references

- Ideas: `zx1160763849-hash/codex-cost-router-skills`, `jinweechen/codex-auto-router`
- Upstream issues: openai/codex #37237, #36493, #37822, #37197

## License

[MIT](./LICENSE)
