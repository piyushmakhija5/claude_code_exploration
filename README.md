# Inside Tengu: Claude Code Source Analysis

In June 2026, Claude Code's full TypeScript source was [accidentally exposed](https://x.com/Fried_rice/status/2038894956459290963) via source maps in an NPM package. This repo contains a systematic deep dive into what the code reveals — architecture, hidden features, security measures, and lessons for anyone building AI agents.

**[Hacker News discussion](https://news.ycombinator.com/item?id=47584540)** covered surface-level findings. This analysis goes significantly deeper.

## What's here

| File | Description |
|------|-------------|
| **[ANALYSIS.md](ANALYSIS.md)** | Full technical reference (578 lines). 16 sections with file citations, HN cross-reference, builder principles, power user guide. |
| **[index.html](index.html)** | Interactive guided tour. 3 tracks, expandable tool/skill cards, token visualizations, keyboard navigation. Open locally in a browser. |

## Key findings

**Architecture** — The entire agent is a `while(true)` loop in `query.ts` (68KB). No LangChain, no orchestration framework. Tool execution happens inline. 54 streaming yield points. 4-layer recovery cascade.

**Token budget** — ~46,000 tokens consumed before your first message. Tool schemas alone are ~33,000 tokens (72%). The Bash tool prompt is ~5,300 tokens. This is why cache optimization dominates the architecture.

**Two prompts** — Anthropic employees see different system prompt sections than external users. Internal additions include: numeric length anchors ("≤25 words between tool calls"), assertiveness rules, false-claim mitigation for a model codenamed Capybara v8 (29-30% false-claim rate vs 16.7% on v4).

**Anti-distillation** — Active defense: `anti_distillation: ['fake_tools']` injects decoy tools into API responses to poison training data extraction. Not discussed on HN.

**Speculative execution** — Runs up to 20 turns / 100 messages ahead of user approval. Writes go to a file overlay, applied only if you accept. Not discussed on HN.

**Claude dreams** — Background memory consolidation during idle time. Prompt: "You are performing a dream — a reflective pass over your memory files." Not discussed on HN.

**Hidden gacha game** — 18 ASCII species, rarity tiers (legendary 1%), RPG stats (DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK), model-generated "souls". Behind the `BUDDY` feature flag.

**90+ feature flags** reveal the roadmap: `KAIROS` (always-on assistant), `DAEMON` (background mode), `WEB_BROWSER_TOOL`, `VOICE_MODE`, `WORKFLOW_SCRIPTS`. Internal codenames: Tengu (Claude Code), Capybara (next model), Fennec (early Opus), Numbat (upcoming).

**Skills & agents** — 17+ bundled slash commands. 6 built-in agent types with restricted tool access. Fork subagent shares parent's exact prompt cache for cost efficiency.

**Security** — 10-step permission cascade with bypass-immune safety checks. 23 bash security detection methods including Tree-sitter AST parsing and Zsh exploit detection. Undercover mode (no force-OFF) strips AI attribution from public commits.

## Interactive tour

Open `index.html` in a browser. Three guided tracks:

- **The Architecture** (11 stops) — Loop, tools, skills, agents, system prompt with token visualization, context management, cache optimization, memory, verification, models
- **The Secrets** (8 stops) — Anti-distillation, undercover mode, internal prompts, codenames, buddy game, dreams, speculative execution, sentiment detection
- **The Playbook** (7 stops) — 10 principles for agent builders, power user cheat sheet, feature flag roadmap, HN comparison

Navigate with arrow keys. Click tool/skill cards to expand details. Hover dotted text for source file references.

## What this covers that HN didn't

Anti-distillation, speculative execution, dream consolidation, cache metrics (10.2% fleet savings, 77% of breaks are schema text changes), compaction strategies (2.79% failure rate fix for Sonnet 4.6), internal vs external prompt differences, 10-step permission cascade, 4-tier memory system, fork subagent cache sharing, adversarial verification agent, token budget breakdown, 10 builder principles, power user guide.

## Source

Analysis performed via direct code inspection. All findings cite specific files and code locations. Cross-referenced with [HN #47584540](https://news.ycombinator.com/item?id=47584540). The codebase analyzed is Claude Code as shipped in NPM package `@anthropic-ai/claude-code`.
