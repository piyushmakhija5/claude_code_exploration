# Inside Tengu: Claude Code Source Analysis

Claude Code's full TypeScript source was accidentally exposed via source maps in an NPM package.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Claude code source code has been leaked via a map file in their npm registry!</p>&mdash; Chaofan Shou (@Fried_rice) <a href="https://twitter.com/Fried_rice/status/2038894956459290963">March 31, 2026</a></blockquote>

> 21.2M views. [HN discussion](https://news.ycombinator.com/item?id=47584540) covered surface-level findings. This analysis goes deeper.

## What we found

**Anti-distillation is live** — `anti_distillation: ['fake_tools']` injects decoy tools into API responses to poison anyone scraping Claude's outputs for training data. *Not discussed on HN.*

**Claude literally dreams** — Background memory consolidation fires during idle time. The prompt says: *"You are performing a dream — a reflective pass over your memory files."* *Not discussed on HN.*

**Speculative execution** — Runs up to 20 turns / 100 messages *ahead* of your approval. Writes go to a temp overlay, applied only if you accept. *Not discussed on HN.*

**A hidden gacha game** — 18 ASCII species, RPG stats (DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK), legendary at 1% drop rate. Behind the `BUDDY` feature flag.

**KAIROS: the always-on vision** — Feature flags reveal a persistent, remote-first assistant mode with push notifications, GitHub webhooks, crash recovery, and cross-machine session teleportation.

**Tool schemas eat 72% of the prompt** — ~33,000 of ~46,000 tokens before your first message. Bash tool alone is ~5,300 tokens. This is why cache optimization dominates the architecture.

**Internal employees see a different prompt** — Extra rules include "never claim all tests pass when output shows failures" and false-claim mitigation for a model codenamed **Capybara v8** (29-30% FC rate vs 16.7% on v4).

## What's here

| File | Description |
|------|-------------|
| **[ANALYSIS.md](ANALYSIS.md)** | Full technical reference (578 lines). 16 sections, file citations, HN cross-reference, 10 builder principles, power user guide. |
| **[index.html](index.html)** | Interactive guided tour. 3 tracks, expandable cards, token visualization, keyboard nav. Open in browser. |

## Interactive tour

Open `index.html` locally. Three tracks:

- **The Architecture** (12 stops) — System prompt, tools, skills, agents, context, cache, memory, models
- **The Secrets** (8 stops) — Anti-distillation, undercover mode, codenames, buddy game, dreams, speculative execution
- **The Playbook** (7 stops) — 10 principles for agent builders, power user tricks, HN comparison

Arrow keys to navigate. Click cards to expand. Hover dotted text for source refs.

## Source

Analysis via direct code inspection. All findings cite specific files. Cross-referenced with [HN #47584540](https://news.ycombinator.com/item?id=47584540). Codebase: `@anthropic-ai/claude-code` NPM package.
