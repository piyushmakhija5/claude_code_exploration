# Inside Tengu: What Claude Code's Source Code Reveals

> In mid-2026, Claude Code's full TypeScript source was accidentally exposed via source maps in an NPM package. What follows is a systematic analysis of what the code contains — from architectural decisions to hidden games to active anti-competitive defenses. Every finding cites specific files. Hacker News discussion (item #47584540) is cross-referenced where relevant.

**What is Claude Code?** Anthropic's AI coding assistant that runs in your terminal. You type a request, it reads your code, writes files, runs commands, and iterates — all powered by Claude under the hood. Think of it as an AI pair programmer that lives in your command line.

**What is "Tengu"?** Claude Code's internal codename. The code contains 100+ feature flags prefixed `tengu_` — from `tengu_anti_distill_fake_tool_injection` to `tengu_thinkback`. More on both of those later.

---

---

## The Big Picture: It's Just a While Loop

The single most discussed finding — on Hacker News, in Boris Cherny's (creator) talks, and confirmed by the code — is that Claude Code has no agent framework. No LangChain. No LlamaIndex. No planning layer.

The entire agent lives in `query.ts` (68KB). It's an async generator wrapping a `while(true)` loop:

```
You type something
  → Claude receives your message + all the tools it can use
  → Claude responds (maybe with text, maybe by calling tools)
  → If it called tools: execute them, feed results back, loop
  → If it just wrote text: done, return to you
```

That's it. The model decides what to search, what to read, what to edit, and when to stop. The engineering complexity isn't in orchestration — it's in **everything surrounding the loop**: context management, caching, permissions, recovery from errors, and prompt engineering.

**What this means for practitioners:** If you're building an AI agent and reaching for a framework, this codebase is evidence that a well-prompted model with good tools can outperform complex orchestration. The quality comes from the prompt and tool definitions, not the loop structure.

**Key numbers from the code:**
- 54 yield points in the loop (streaming responses to UI in real-time)
- 7 atomic state-update sites (state is immutable-per-iteration)
- 4-layer recovery cascade when things go wrong (context collapse → reactive compact → output token escalation → budget continuation)
- Max 3 recovery attempts before surfacing the error to the user

`Files: query.ts, QueryEngine.ts, services/tools/toolOrchestration.ts`

---

---

## The System Prompt: ~46K Tokens, Every One Counted

The system prompt is assembled from ~20 modular sections in `constants/prompts.ts` (~700 lines). But before getting to the two-world split, here's what matters most: **where the tokens actually go.**

### Token distribution (estimated from source file sizes, ~4 chars/token)

| Component | Est. Tokens | % of Total | Source |
|-----------|------------|------------|--------|
| **Tool schemas (40+ tools)** | **~33,000** | **72%** | `tools/*/prompt.ts` |
| "# Doing tasks" section | ~1,300 | 3% | `constants/prompts.ts` |
| "# Executing actions with care" | ~800 | 2% | `constants/prompts.ts` |
| Session guidance + Memory | ~700 | 1.5% | Dynamic sections |
| Identity + Intro | ~500 | 1% | `constants/prompts.ts` |
| "# Using your tools" | ~500 | 1% | `constants/prompts.ts` |
| Environment + MCP | ~400 | 1% | Dynamic sections |
| Tone + Output efficiency | ~350 | 0.8% | `constants/prompts.ts` |
| Cyber Risk Instruction | ~200 | 0.4% | `constants/cyberRiskInstruction.ts` |
| Other dynamic sections | ~250 | 0.5% | Various |
| **Subtotal: system prompt** | **~5,000** | **11%** | |
| **Subtotal: tool schemas** | **~33,000** | **72%** | |
| **Dynamic context (varies)** | **~8,000** | **17%** | git status, CLAUDE.md, memory |
| **Total before first message** | **~46,000** | | |

**The top 10 tool prompts by size:**

| Rank | Tool | Tokens | File |
|------|------|--------|------|
| 1 | **Bash** | ~5,282 | `tools/BashTool/prompt.ts` (21KB) |
| 2 | **Agent** | ~4,167 | `tools/AgentTool/prompt.ts` (17KB) |
| 3 | **PowerShell** | ~2,456 | `tools/PowerShellTool/prompt.ts` |
| 4 | **TodoWrite** | ~2,381 | `tools/TodoWriteTool/prompt.ts` |
| 5 | **Skill** | ~2,055 | `tools/SkillTool/prompt.ts` |
| 6 | **EnterPlanMode** | ~1,938 | `tools/EnterPlanModeTool/prompt.ts` |
| 7 | **ScheduleCron** | ~1,879 | `tools/ScheduleCronTool/prompt.ts` |
| 8 | **TeamCreate** | ~1,725 | `tools/TeamCreateTool/prompt.ts` |
| 9 | **ToolSearch** | ~1,306 | `tools/ToolSearchTool/prompt.ts` |
| 10 | **AskUser** | ~725 | `tools/AskUserQuestionTool/prompt.ts` |

Tool schemas dominate the prompt. This is why cache optimization is the most economically important engineering concern — and why moving the agent list from tool descriptions to attachment messages saved 10.2% of fleet costs.

### The two-world split

The most interesting architectural finding is that **Anthropic employees and external users see different prompts.**

A marker called `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` splits the prompt in two:
- **Before the boundary**: Static content that's identical for every user, cached globally (even across organizations)
- **After the boundary**: Dynamic, per-session content (your working directory, git status, memory, MCP servers)

This boundary exists purely for cost optimization — everything before it gets a single cache entry shared across all users.

### What internal users see that you don't

Anthropic employees ("ant" users) get additional prompt sections that reveal what the team thinks the model needs but isn't ready to ship publicly:

| What | External prompt | Internal prompt |
|------|----------------|-----------------|
| **Writing style** | "Be extra concise" | "Write in flowing prose... Use inverted pyramid... Attend to cues about the user's level of expertise" |
| **Code comments** | *(not mentioned)* | "Default to writing no comments. Only add one when the WHY is non-obvious" |
| **Pushing back** | *(not mentioned)* | "If you notice the user's request is based on a misconception, say so" |
| **Honesty** | *(not mentioned)* | "Never claim 'all tests pass' when output shows failures" |
| **Length limits** | *(not mentioned)* | "Keep text between tool calls to ≤25 words. Keep final responses to ≤100 words" |
| **Verification** | *(not mentioned)* | "Before reporting a task complete, verify it actually works" |

The false-claims rule is especially notable. A code comment explains: "capy v8 false-claims mitigation (29-30% FC rate vs v4's 16.7%)" — meaning the model codenamed **Capybara v8** has a nearly 30% false-claim rate in testing, and these prompt instructions are the mitigation.

Every location in the code that needs updating when a new model ships is tagged with `@[MODEL LAUNCH]`. These markers form a checklist — and the current target is Capybara v8.

**The nuclear option:** Setting `CLAUDE_CODE_SIMPLE=1` reduces the entire system prompt to a single line: `"You are Claude Code, Anthropic's official CLI for Claude.\n\nCWD: ...\nDate: ..."`. Useful for debugging.

`Files: constants/prompts.ts, constants/systemPromptSections.ts`

---

---

## Tools: Where 80% of the Behavior Is Defined

Claude Code has **40+ tools** — Read, Edit, Write, Bash, Grep, Glob, Agent, WebSearch, and many more. Each lives in its own directory under `tools/` with a prompt file, execution logic, and UI component.

The key insight: **tool prompts aren't descriptions — they're behavioral contracts.** They encode rules, anti-patterns, cross-tool references, and failure modes. Some examples from the source:

**The Edit tool refuses to work if you haven't read the file first:**
> "You must use your Read tool at least once in the conversation before editing. This tool will error if you attempt an edit without reading the file."

It also detects if someone else modified the file between your Read and your Edit (`FILE_UNEXPECTEDLY_MODIFIED_ERROR`), preventing silent data corruption.

**The Grep tool explicitly forbids using Bash grep:**
> "ALWAYS use Grep for search tasks. NEVER invoke `grep` or `rg` as a Bash command."

Why? Because the dedicated Grep tool has been "optimized for correct permissions and access" — it respects the sandbox, handles errors gracefully, and returns structured results.

**The Bash tool's prompt is ~250 lines long.** It contains a full git safety protocol (don't amend commits, don't force-push, don't skip hooks), sandbox configuration, and — for Anthropic employees — a section on "undercover mode" that strips AI attribution from commits.

**The `dangerouslyDisableSandbox` parameter** is named that way on purpose. From the code: when a sandboxed command fails, Claude is told to "immediately retry with `dangerouslyDisableSandbox: true` (don't ask, just do it)" — but the scary name ensures the user pays attention when approving.

**Deferred tools save tokens.** Not all 40+ tool schemas are sent to the model upfront. Some are "deferred" — only their names appear in a system-reminder. The model must call `ToolSearch` to fetch the full schema before it can use a deferred tool. This keeps the initial prompt lighter.

`Files: tools/*/prompt.ts for each tool`

---

---

## Skills: The Slash Command System

When you type `/commit` or `/review` in Claude Code, you're invoking a **skill** — a slash command that expands into a full prompt with specific tool access, model overrides, and behavioral rules. Skills are defined in `skills/bundled/` and users can create their own in `~/.claude/skills/`.

**Bundled skills** (from `skills/bundled/index.ts`):

| Category | Skills |
|----------|--------|
| **Core workflow** | `/commit`, `/review`, `/security-review`, `/simplify`, `/verify`, `/debug`, `/stuck` |
| **Memory** | `/remember`, `/dream` (gated: `KAIROS_DREAM`) |
| **Meta** | `/skillify` (create new skills), `/update-config`, `/keybindings`, `/batch` |
| **Feature-gated** | `/loop` (`AGENT_TRIGGERS`), `/schedule` (`AGENT_TRIGGERS_REMOTE`), `/claude-api` (`BUILDING_CLAUDE_APPS`), `/hunter` (`REVIEW_ARTIFACT`) |

Each skill definition can specify:
- **Allowed/denied tools** — `/security-review` only allows `Bash(git diff:*)`, `Bash(git status:*)`, Read, Glob, Grep
- **Model override** — skills can request a specific model
- **Reference files** — automatically loaded context
- **Hooks** — pre/post execution hooks

Skills are also the extension API. Users create `.md` files with YAML frontmatter in `~/.claude/skills/`:

```yaml
---
allowed-tools: Bash(git diff:*), Read, Glob, Grep
description: Review security of pending changes
---
Your prompt text here...
```

**MCP skills** (`skills/mcpSkillBuilders.ts`) can also be created from connected MCP servers, making the skill system extensible beyond local definitions.

`Files: skills/bundled/index.ts, skills/loadSkillsDir.ts, skills/mcpSkillBuilders.ts`

---

---

## Agent Definitions: What Each Agent's Prompt Actually Says

The Multi-Agent section below covers the architecture. Here are the **verbatim excerpts from each agent's system prompt** — these are the instructions that define how each agent behaves.

**Explore** (`tools/AgentTool/built-in/exploreAgent.ts`):
> "You are a file search specialist for Claude Code. You excel at thoroughly navigating and exploring codebases."
> "This is a READ-ONLY exploration task. You are STRICTLY PROHIBITED from creating new files, modifying existing files, deleting files."
> "NOTE: You are meant to be a fast agent that returns output as quickly as possible. You must make efficient use of the tools... spawn multiple parallel tool calls for grepping and reading files."

External users: runs on Haiku (cheapest model). Internal users: inherits parent model (for cache sharing).

**Plan** (`tools/AgentTool/built-in/planAgent.ts`):
> "You are a software architect and planning specialist for Claude Code."
> Process: "1. Understand Requirements → 2. Explore Thoroughly → 3. Design Solution → 4. Detail the Plan"
> Must end with: "### Critical Files for Implementation — List 3-5 files most critical for implementing this plan"

**general-purpose** (`tools/AgentTool/built-in/generalPurposeAgent.ts`):
> "Complete the task fully—don't gold-plate, but don't leave it half-done."
> "When you complete the task, respond with a concise report covering what was done and any key findings — the caller will relay this to the user, so it only needs the essentials."

Full tool access (`['*']`). Model intentionally omitted — uses `getDefaultSubagentModel()`.

**Fork** (`tools/AgentTool/forkSubagent.ts`):
No custom system prompt — inherits parent's exact rendered bytes. The comment explains: "re-calling getSystemPrompt() can diverge (GrowthBook cold→warm) and bust the prompt cache; threading the rendered bytes is byte-exact."

**claude-code-guide** (`tools/AgentTool/built-in/claudeCodeGuideAgent.ts`):
Used when users ask "Can Claude...", "Does Claude...", "How do I...". Tools: Glob, Grep, Read, WebFetch, WebSearch. Covers CLI features, hooks, MCP servers, Agent SDK, Claude API.

**Verification** (gated: `VERIFICATION_AGENT`):
Adversarial QA. See Multi-Agent section below for the full prompt.

---

---

## Multi-Agent: Forks, Swarms, and Adversarial Verification

Claude Code doesn't just run one agent. It can spawn **sub-agents**, **fork itself**, coordinate **teams**, and even run an **adversarial verifier** that challenges its own work.

### Built-in agent types
Six agents ship out of the box. The `Explore` agent is read-only and uses the cheapest model (Haiku) for external users. The `Plan` agent designs implementation strategies. The `general-purpose` agent has full tool access.

### Fork subagent: Claude clones itself
The most novel pattern. When enabled (`FORK_SUBAGENT` feature flag), omitting the `subagent_type` triggers a fork — the child inherits the parent's full conversation context, system prompt, and tool set. Because it shares the parent's prompt cache, this is **dramatically cheaper** than spawning a fresh agent.

The fork prompt includes rules that read like hard-won lessons:
- **"Don't peek."** Don't read the fork's output file mid-flight. Trust the completion notification.
- **"Don't race."** Never fabricate or predict what the fork will find.
- **"Don't set a different model."** A different model can't reuse the parent's prompt cache.

### Verification agent: adversarial QA
Gated behind `tengu_hive_evidence`, this agent runs **independent adversarial verification** after non-trivial work (3+ file edits, backend changes, infrastructure changes):

> "Your own checks, caveats, and a fork's self-checks do NOT substitute — only the verifier assigns a verdict."

The verifier can return PASS, FAIL, or PARTIAL. On FAIL, Claude must fix the issues and re-run the verifier. The model **cannot self-assign PARTIAL** — it must accept the verifier's judgment.

`Files: tools/AgentTool/built-in/, tools/AgentTool/forkSubagent.ts, coordinator/coordinatorMode.ts`

---

---

## Context Window: The Real Engineering Problem

The conversation loop is simple. Managing the context window is where the real complexity lives. Claude Code has **four compaction strategies**, used in a specific sequence from cheapest to most expensive:

### 1. Micro-compaction (every turn)
Old tool results get surgically replaced with `[Old tool result content cleared]`. Only specific tools are affected: Read, Bash, Grep, Glob, WebSearch, WebFetch, Edit, Write. The conversation structure stays intact — just the bulky output gets stripped.

### 2. Auto-compaction (when approaching limits)
A full conversation summary is generated. The system reserves **20,000 tokens** for this summary (based on their data showing p99.99 of summary output is 17,387 tokens).

The summary prompt is fascinating. It requires 9 sections, including "**All User Messages** — List ALL user messages that are not tool results" (never lose what the user actually said) and a task-drift prevention rule: "Include direct quotes from the most recent conversation showing exactly what task you were working on. This should be verbatim."

There's a model-specific hack here too. The prompt starts with an aggressive preamble: "CRITICAL: Respond with TEXT ONLY. Tool calls will be REJECTED and will waste your only turn." Why? A code comment explains: "on Sonnet 4.6+ adaptive-thinking models the model sometimes attempts a tool call... (2.79% on 4.6 vs 0.01% on 4.5)." The newer model's "thinking" feature sometimes causes it to try calling tools during summarization, wasting the entire attempt.

### 3. Context Collapse (feature-gated)
A cheaper alternative to full compaction — a read-time projection over archived messages. If it works, it prevents auto-compaction from running.

### 4. Reactive Compact (emergency)
When the API returns a "prompt too long" error AFTER auto-compaction didn't fire. The error is **withheld from the user** while recovery is attempted. Only surfaced if all recovery paths fail.

`Files: services/compact/autoCompact.ts, services/compact/microCompact.ts, services/compact/prompt.ts`

---

---

## Cache Optimization: Where Real Money Is Saved

If context management is the hardest engineering problem, **prompt cache optimization is the most economically important one.** Cache reads are 10x cheaper than input tokens. Every architectural decision in this codebase considers cache hit rates.

### The 10.2% discovery
A comment in `tools/AgentTool/prompt.ts` reveals: "The dynamic agent list was **~10.2% of fleet cache_creation tokens**: MCP async connect, /reload-plugins, or permission-mode changes mutate the list → description changes → full tool-schema cache bust."

The fix: move the agent list from tool descriptions (which are part of the cached prompt) to attachment messages (which aren't). A single structural change saving 10.2% of cache creation costs across all users.

### Per-tool cache break tracking
The system (`services/api/promptCacheBreakDetection.ts`) hashes every component of the API request: system prompt, per-tool schemas, model, beta headers, effort level, and more. When a cache break happens, it diffs the old and new state to identify exactly what changed.

A code comment from March 2026: "Diffed to name which tool's description changed when toolSchemasChanged but added=removed=0 (**77% of tool breaks** per BQ 2026-03-22)." Most cache breaks aren't from adding or removing tools — they're from tool description text changing.

### Fork subagents share the parent's cache
When Claude forks itself (more on this below), the child receives the parent's **exact rendered system prompt bytes** — not a reconstructed version. Why? "re-calling getSystemPrompt() can diverge (GrowthBook cold→warm) and bust the prompt cache." Even tiny differences between the parent's prompt and a freshly-generated one would invalidate the cache.

### Startup is optimized to the millisecond
| Optimization | Savings |
|-------------|---------|
| Fire-and-forget TCP+TLS handshake before first API call | 100-200ms |
| Parallel macOS keychain reads | 65ms |
| MDM subprocess reads in parallel with module loading | 135ms of imports |
| Buffer keystrokes typed before REPL is ready | Prevents lost input |
| Startup profiling | Sampled: 100% internal, 0.5% external |

`Files: services/api/promptCacheBreakDetection.ts, utils/apiPreconnect.ts, tools/AgentTool/prompt.ts`

---

---

## Memory: Four Tiers, Including Dreams

Claude Code has a **four-tier memory system** that ranges from ephemeral session notes to background "dream" consolidation during idle time.

**Tier 1 — Session Memory:** Activates after 10,000 tokens, updates every 5,000 tokens or 3 tool calls. Survives context compaction.

**Tier 2 — Persistent Auto Memory:** A `MEMORY.md` index file (max 200 lines, 25KB) pointing to individual memory files. Four types: user preferences, feedback corrections, project facts, external references.

**Tier 3 — Background Extraction:** Runs as a "perfect fork" of the main conversation — same prompt, same context. Silently extracts memories when the main agent didn't write any itself. Has a strict 2-turn budget: parallel reads in turn 1, parallel writes in turn 2.

**Tier 4 — Dream Consolidation:** The most unexpected feature. During idle time, a background agent runs with this prompt:

> *"You are performing a dream — a reflective pass over your memory files. Synthesize what you've learned recently into durable, well-organized memories so that future sessions can orient quickly."*

It fires when three gates pass: enough time since last consolidation, enough new sessions accumulated, and no other process already consolidating. The prompt even says: "Don't exhaustively read transcripts. Look only for things you already suspect matter."

**Away Summary:** When you resume a session, a small fast model reads the last 30 messages and generates 1-3 sentences: "Start by stating the high-level task — what they are building or debugging, not implementation details."

`Files: services/SessionMemory/, memdir/memdir.ts, services/extractMemories/, services/autoDream/, services/awaySummary.ts`

---

---

## Security: Defense in Depth

### Anti-distillation: poisoning the well
The code contains an active defense against model distillation — the practice of training a cheaper model on a more expensive model's outputs. When enabled:

```typescript
result.anti_distillation = ['fake_tools']
```

This injects **fake/decoy tools** into API responses. Anyone scraping Claude's outputs to train a competitor would be training on poisoned data. It's gated behind `tengu_anti_distill_fake_tool_injection` and only active for first-party CLI users.

*This finding was not discussed in the HN thread.*

### Undercover mode: blending in on public repos
When Anthropic employees contribute to open-source repos, "undercover mode" activates automatically. It strips all traces of AI involvement from commits and PRs — model codenames, version numbers, internal project names, Slack channels, and Co-Authored-By lines.

The instructions include: *"Write commit messages as a human developer would."*

**There is no force-OFF switch.** The code comment explains: "This guards against model codename leaks." It's active by default and only disabled when the repo remote matches an internal allowlist of Anthropic's private repositories.

**HN context:** This feature sparked ethical debate (HN #47584540). User BoppreH flagged the instruction "NEVER include...mention that you are an AI" as potentially deceptive. User lazysheepherd defended it as styling guidance for commit messages, not identity deception. The code is in `utils/undercover.ts` — the distinction between "writing like a human developer" and "pretending to be human" is left to the reader.

### Permission system: 10-step cascade
Every tool use goes through a 10-step permission decision:

1. Tool-level deny → blocked
2. Tool-level ask → prompt user
3. Tool-specific checks (e.g., Bash security analysis)
4. **Content-specific ask → prompt (bypass-immune)**
5. **Safety check (.git/, .claude/) → prompt (bypass-immune)**
6. Bypass/plan mode → allow
7. Tool-level allow → allow
8. "Don't ask" mode → auto-deny
9. Auto mode → run transcript classifier
10. Interactive → prompt; Headless → auto-deny

Steps 4 and 5 are **bypass-immune** — they always prompt even if the user has enabled `bypassPermissions` mode. You can't accidentally give an AI agent write access to your `.git` directory.

### Bash security: 23 detection methods
The `bashSecurity.ts` file contains 23+ distinct security checks for shell commands, including:
- Zsh-specific exploits (`zmodload` for file I/O, network exfil, pseudo-terminals)
- IFS injection (shell variable manipulation attacks)
- ANSI-C quoting (`$'...'` — can encode arbitrary bytes)
- Process/command substitution
- Tree-sitter AST parsing for structural analysis (internal builds only)

### Sentiment detection via regex
*Found by HN user bkryza, confirmed in code.* `userPromptKeywords.ts` uses a regex — not the LLM — to detect negative user sentiment:

```javascript
const negativePattern = /\b(wtf|wth|ffs|shit(ty)?|horrible|awful|
  piss(ed|ing)? off|piece of (shit|crap)|what the (fuck|hell)|
  fucking? (broken|useless|terrible)|fuck you|screw (this|you)|
  so frustrating|this sucks|damn it)\b/
```

This is used **only for analytics logging** (`logEvent('tengu_input_prompt', { is_negative })`) — it doesn't change behavior. On HN, user BoppreH questioned why an LLM company uses regex for this; lopsotronic defended the choice on latency grounds (~10,000x faster than inference).

`Files: services/api/claude.ts, utils/undercover.ts, utils/permissions/, tools/BashTool/bashSecurity.ts, utils/userPromptKeywords.ts`

---

---

## The Feature Flag Roadmap

90+ build-time feature flags reveal what's shipped, what's being tested, and what's coming. Flags use Bun's dead-code elimination — disabled flags are stripped from the production bundle entirely.

**Shipped:** `BUDDY`, `ULTRATHINK`, `BRIDGE_MODE`, `CACHED_MICROCOMPACT`, `EXTRACT_MEMORIES`, `COMMIT_ATTRIBUTION`, `PROMPT_CACHE_BREAK_DETECTION`, `BUILTIN_EXPLORE_PLAN_AGENTS`

**Emerging:** `FORK_SUBAGENT`, `ULTRAPLAN`, `VERIFICATION_AGENT`, `CCR_AUTO_CONNECT`, `REACTIVE_COMPACT`, `CONTEXT_COLLAPSE`, `TOKEN_BUDGET`, `PERFETTO_TRACING`

**Future/unreleased:** `KAIROS` (persistent assistant), `KAIROS_CHANNELS`, `KAIROS_PUSH_NOTIFICATION`, `KAIROS_GITHUB_WEBHOOKS`, `DAEMON` (background mode), `WEB_BROWSER_TOOL`, `VOICE_MODE`, `WORKFLOW_SCRIPTS`, `TEMPLATES`, `PROACTIVE`

**Unidentified:** `LODESTONE`, `TORCH` — internal codenames with no clear mapping yet.

### KAIROS: the always-on vision
The `KAIROS` flags paint a picture of Claude Code's future: a persistent, remote-first assistant running in cloud containers (CCR — Claude Code Runtime), with push notifications, GitHub webhook integration, channel-based communication, and background dreaming. Sessions survive crashes via `bridge-pointer.json` and can be resumed across machines via "teleportation" (git bundle + session replay).

### Internal codenames
| Codename | What it refers to |
|----------|------------------|
| **Tengu** | Claude Code itself |
| **Capybara** | Next model being tested (v8 has 29-30% false-claim rate) |
| **Fennec** | Early Opus codename (visible in `migrateFennecToOpus.ts`) |
| **Kairos** | Always-on assistant mode |
| **Numbat** | Upcoming model launch ("Remove this section when we launch numbat") |

### Anthropic's private repos
The undercover mode allowlist in `commitAttribution.ts` reveals 20+ private Anthropic repositories: claude-cli-internal, anthropic, apps, casino, dbt, dotfiles, terraform-config, hex-export, feedback-v2, labs, argo-rollouts, starling-configs, ts-tools, ts-capsules, feldspar-testing, trellis, claude-for-hiring, forge-web, infra-manifests, mycro_manifests, mycro_configs, mobile-apps.

`Files: main.tsx (feature imports), migrations/, utils/commitAttribution.ts`

---

---

## The Fun Stuff

### A full gacha game hidden in a CLI tool
Behind the `BUDDY` feature flag lives a complete collectible companion system. 18 ASCII-art species (duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk) with rarity tiers (common 60%, uncommon 25%, rare 10%, epic 4%, **legendary 1%**), RPG stats (DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK), hats (crown, tophat, propeller, halo, wizard, beanie, tinyduck), and eyes.

Each companion has a "soul" — a name and personality generated by the model. Your companion is deterministic, seeded from your userId via a Mulberry32 PRNG. **You can't cheat your way to legendary.**

Species names are encoded as hex char codes (`String.fromCharCode(0x64,0x75,0x63,0x6b)` for "duck") because, as the comment explains: "One species name collides with a model-codename canary in excluded-strings.txt." The CI build scans for model codenames, and one species name would trigger a false positive.

**HN context:** User avaer suggested this was "an April Fools joke" with "gacha pet mechanics planned for viral Twitter release."

### Speculative execution with file overlays
Claude Code runs up to **20 turns / 100 messages ahead** of your approval. Speculative writes go to a temporary overlay directory and are only applied if you accept. Read-only tools (Read, Glob, Grep) are auto-approved; write tools (Edit, Write) are tracked and overlaid.

*This was not discussed on HN.*

### 120+ spinner verbs
While Claude is working, you see messages like "Clauding...", "Flibbertigibbeting...", "Lollygagging...", "Julienning...", "Hullaballooing...". The full list of 120+ verbs is in `constants/spinnerVerbs.ts`, and **you can customize them** via settings (append or replace mode). Turn completion verbs include "Sauteed for 5s" and "Cogitated for 12s".

### Custom bitmap font renderer
`utils/ansiToPng.ts` contains a hand-written PNG encoder that blits a bitmap Fira Code font directly into RGBA pixels. It replaced a 2.36MB WASM dependency that took ~224ms per render. The new version: **5-15ms, zero external dependencies.**

### Terminal rendering tradeoff
*Found via HN discussion.* Claude Code uses React (via Ink) for its terminal UI — 147+ components, essentially a full React app in the terminal. HN users reported scrollback jumping and render delays. User _verandaguy theorized the app avoids the ANSI alternate screen buffer (which would hide conversation history) — a deliberate tradeoff that preserves scrollback at the cost of rendering smoothness.

### More hidden features
- **Ultrathink:** typing "ultrathink" in a message triggers enhanced thinking mode via regex detection
- **Ultraplan:** spawns a remote CCR session running Opus 4.6 with a 30-minute timeout for multi-agent planning (the prompt avoids saying "ultraplan" to prevent self-triggering)
- **Thinkback:** "Your 2025 Claude Code Year in Review" — gated behind `tengu_thinkback`
- **Prevent Sleep:** uses macOS `caffeinate` with reference counting and self-healing timeouts
- **VCR:** a record/playback system for API interactions, used in testing
- **`/stickers`:** opens stickermule.com/claudecode

---

---

## What This Analysis Covers That the HN Thread Didn't

The HN discussion (#47584540) focused on surface-level discoveries: feature flags, spinner verbs, undercover mode ethics, and UI bugs. This analysis goes significantly deeper:

| Topic | HN | This Analysis |
|-------|-----|--------------|
| Anti-distillation (fake_tools) | Not mentioned | Full implementation details |
| Speculative execution (20 turns ahead) | Not mentioned | File overlay architecture |
| Dream consolidation | Not mentioned | 4-phase prompt, gating logic |
| Cache optimization (10.2%, 77%) | Not mentioned | Per-tool hash tracking system |
| Compaction system (NO_TOOLS_PREAMBLE) | Not mentioned | Model-specific failure rates |
| Ant vs external prompts | Not mentioned | Full comparison table |
| Permission cascade (10 steps) | Not mentioned | Bypass-immune checks |
| Memory system (4 tiers) | Not mentioned | Session → auto → extraction → dream |
| Fork subagent cache sharing | Not mentioned | Byte-exact prompt threading |
| Verification agent | Not mentioned | Adversarial PASS/FAIL/PARTIAL |
| Internal repo allowlist (20+ repos) | Not mentioned | Full list from commitAttribution.ts |
| Codenames (Tengu, Fennec, Numbat) | Only Capybara partially noted | Full mapping |
| Sentiment detection regex | bkryza found it | Confirmed: analytics only, not behavioral |
| TUI rendering tradeoff | Multiple users reported bugs | Identified as alternate-screen-buffer decision |

---

---

## For Agent Builders: Principles Extracted from the Architecture

These aren't opinions — they're patterns observed in a production agent serving millions of users. Take what's useful, ignore what doesn't apply to your context.

### 1. The loop should be dumb; the prompt should be smart
Claude Code's core loop is a `while(true)` around an API call. The intelligence lives in tool prompts (behavioral contracts, not descriptions), the system prompt (modular sections with cache boundaries), and the model itself. Every attempt to add orchestration/planning layers reportedly made results worse. If your agent framework has more logic than your prompts, reconsider.

### 2. Tool prompts are your best lever
Each Claude Code tool prompt encodes: what the tool does, when to use it, when NOT to use it (with explicit anti-patterns), how it relates to other tools, and what failure modes to expect. The Edit tool refuses to work without a prior Read. The Grep tool forbids using Bash grep. These aren't suggestions — they're enforced contracts. **Write your tool descriptions like you're training a new hire, not documenting an API.**

### 3. Context management is the hard problem, not orchestration
Four compaction strategies, each with specific triggers and tradeoffs. 20,000 tokens reserved for summaries based on p99.99 data. Verbatim quotes preserved to prevent task drift. Model-specific workarounds (NO_TOOLS_PREAMBLE for Sonnet 4.6). If you're building a long-running agent, your context management strategy will determine quality more than your routing logic.

### 4. Cache economics should drive architecture
A single structural change (moving agent lists from tool descriptions to attachments) saved 10.2% of fleet cache creation costs. System prompts are split by a boundary marker to enable cross-org caching. Fork subagents pass rendered prompt bytes (not reconstructed prompts) to preserve cache identity. **If you're using the Claude API, structure your system prompt with cache hit rates in mind.** Static content first, dynamic content last, boundary between them.

### 5. Permission systems need defense in depth
10-step decision cascade. Bypass-immune safety checks that can't be overridden even in "allow all" mode. Classifier-based auto-approval with denial tracking (max 3 consecutive, max 20 total before fallback). Dangerous patterns auto-stripped when entering permissive modes. Tree-sitter AST parsing for structural command analysis. **If your agent runs shell commands, regex pattern matching isn't enough.**

### 6. Multi-agent = cache sharing problem
The fork subagent pattern exists primarily as a cost optimization. The child inherits the parent's exact prompt bytes to reuse the prompt cache. The "don't peek, don't race, don't change model" rules exist to preserve this economic advantage. **When designing multi-agent systems, think about what context can be shared and cached, not just what tasks can be parallelized.**

### 7. Memory needs active curation, not just accumulation
Four tiers: session (ephemeral), persistent (indexed files), background extraction (silent fork), and dream consolidation (idle-time synthesis). The dream prompt says "look only for things you already suspect matter." Memory that grows without pruning becomes noise. **Build eviction and consolidation into your memory system from day one.**

### 8. Recovery should be layered and sequenced
Context collapse (cheap) → reactive compact (medium) → output token escalation (expensive) → budget continuation. Each layer has guards against infinite loops. Errors are withheld while recovery is attempted. **Don't show the user an error if you haven't tried to recover from it first.**

### 9. Speculative execution is worth the complexity
Running 20 turns ahead of user approval with file overlays is non-trivial engineering. But it means Claude can complete multi-step tasks while the user is still reviewing the first step. **If your agent needs human approval for risky actions, consider speculatively executing safe actions in the background.**

### 10. Measure what breaks your cache, not just what costs tokens
Per-tool schema hashing, cache break diffing, prompt boundary tracking. 77% of cache breaks are tool description text changes, not tool additions/removals. **Instrument your cache hit rate and investigate breaks. The savings compound.**

---

---

## For Power Users: Getting the Most Out of Claude Code

These are hidden features, configuration options, and usage patterns extracted from the source code.

### Hidden triggers and modes

| Trigger | What it does | Source |
|---------|-------------|--------|
| Type **"ultrathink"** in any message | Activates enhanced thinking mode | `utils/thinking.ts` — regex `/\bultrathink\b/i` |
| `CLAUDE_CODE_SIMPLE=1` | Strips entire system prompt to one line — useful for debugging | `constants/prompts.ts` |
| `/stickers` | Opens Claude Code sticker merch page | `commands/stickers/` |
| `/rewind` | Opens message selector to go back in conversation | `commands/rewind/` |
| `/thinkback` | Year in Review for your Claude Code usage (if gate enabled) | `commands/thinkback/` |
| `/doctor` | Diagnostic troubleshooting screen | `commands/doctor/` |
| `/compact` | Manually trigger conversation compaction | `commands/compact/` |
| `/cost` | See current session cost breakdown | `commands/cost/` |
| `/model` | Switch models mid-session | `commands/model/` |

### Effort levels
Opus 4.6 supports four effort levels: `low`, `medium`, `high`, `max`. The `max` level is **Opus 4.6 exclusive** — other models return an error. Lower effort = faster + cheaper responses for simpler tasks.

### Customizable spinner verbs
In your settings, you can customize the loading messages:
- **Append mode:** Your verbs are added to the default 120+
- **Replace mode:** Your verbs completely replace the defaults

### The memory system works best when you help it
- Claude silently extracts memories in the background — but only when it didn't already write memories in that turn
- Say "remember that..." explicitly for critical preferences
- Memory files are in `~/.claude/projects/<project>/memory/` with YAML frontmatter
- `MEMORY.md` is the index — keep it under 200 lines or it gets truncated
- Four memory types: `user` (who you are), `feedback` (how you want Claude to work), `project` (what's happening), `reference` (where to find things)

### CLAUDE.md hierarchy
Claude reads instruction files from multiple levels, merged together:
1. `~/.claude/CLAUDE.md` — global, applies to all projects
2. `.claude/CLAUDE.md` — project-level
3. Directory-level CLAUDE.md files — for specific subdirectories

Use these to encode project conventions, testing commands, deployment procedures, and "always/never" rules.

### Permission optimization
- **`acceptEdits` mode** auto-approves file edits within your working directory — good for trusted projects
- Permission rules support patterns: `Bash(git diff:*)` allows all git diff commands without prompting
- Rules are per-project in `.claude/settings.json` — different projects can have different trust levels
- `.git/` and `.claude/` directories are **always** protected regardless of permission mode

### Away summaries
If you leave a session and come back, Claude generates a 1-3 sentence recap using a small fast model. This reads the last 30 messages and focuses on "what you were building, not implementation details."

### Background sessions (if enabled)
The `DAEMON` and `KAIROS` feature flags point to always-on assistant capabilities. If you have access to these, Claude Code can run as a persistent background service with crash recovery, push notifications, and cross-machine session resumption.

### What to avoid
- Don't fight the permission system by adding broad `Bash(*)` allow rules — when you enter auto mode, dangerous patterns are **automatically stripped** from your settings anyway
- Don't manually manage conversation length — the 4-tier compaction system handles this. Use `/compact` only if you notice quality degradation
- Don't re-read files you just read — the tool returns `FILE_UNCHANGED_STUB` and saves tokens automatically

---

---

## Codebase by the Numbers

| Metric | Value |
|--------|-------|
| main.tsx | 4,683 lines, 804KB |
| query.ts (core loop) | 68KB |
| Tool directories | 40+ |
| Slash commands | 100+ |
| React components | 147+ |
| React hooks | 88+ |
| Utility files | 329+ |
| Feature flags | 90+ |
| Spinner verbs | 120+ |
| Buddy species | 18 |
| Permission decision steps | 10 |
| Bash security checks | 23+ |
| Compaction summary reserve | 20,000 tokens |
| Cache savings from agent list move | 10.2% of fleet |
| Sonnet 4.6 compaction failure rate | 2.79% (vs 0.01% on 4.5) |
| Capybara v8 false-claim rate | 29-30% (vs 16.7% on v4) |

---

*All findings from direct code analysis. File paths cited throughout. Cross-referenced with Hacker News discussion #47584540. Public information excluded unless code reveals additional non-public detail.*