# oh-my-opencode — OpenCode Plugin

**Generated:** 2026-04-18 | **Commit:** 2892ca4a | **Branch:** dev

## OVERVIEW

OpenCode plugin (npm: `oh-my-opencode`, dual-published as `oh-my-openagent` during transition) extending Claude Code with 11 agents, 52 lifecycle hooks, 26 tools, 3-tier MCP system (built-in + .mcp.json + skill-embedded), Hashline LINE#ID edit tool, IntentGate classifier, and Claude Code compatibility. 1766 TypeScript source files, 377k LOC, 104 barrel index.ts files. Entry: `src/index.ts` → 5-step init (loadConfig → createManagers → createTools → createHooks → createPluginInterface).

## STRUCTURE

```
oh-my-opencode/
├── src/
│   ├── index.ts              # Plugin entry: default export `pluginModule`, shape `{ id, server }`
│   ├── plugin-config.ts      # JSONC multi-level config: user → project → defaults (Zod v4)
│   ├── agents/               # 11 agents (Sisyphus, Hephaestus, Oracle, Librarian, Explore, Atlas, Prometheus, Metis, Momus, Multimodal-Looker, Sisyphus-Junior)
│   ├── hooks/                # 52 lifecycle hooks across dedicated modules and standalone files
│   ├── tools/                # 26 tools across 16 directories (includes Hashline edit with LINE#ID content hashing)
│   ├── features/             # 19 feature modules (background-agent, skill-loader, tmux, MCP-OAuth, skill-mcp-manager, etc.)
│   ├── shared/               # 170+ utility files (barrel-exported, logger → /tmp/oh-my-opencode.log)
│   ├── config/               # Zod v4 schema system (32 files)
│   ├── cli/                  # CLI: install, run, doctor, mcp-oauth (Commander.js)
│   ├── mcp/                  # 3 built-in remote MCPs (websearch, context7, grep_app)
│   ├── plugin/               # 10 OpenCode hook handlers + 52 hook composition
│   ├── plugin-handlers/      # 6-phase config loading pipeline
│   └── openclaw/             # Bidirectional external integration (Discord/Telegram/webhook/command)
├── packages/                 # 11 platform-specific compiled binaries (darwin/linux/windows, AVX2 + baseline variants)
├── script/                   # Build/publish automation (singular, not scripts/)
├── .sisyphus/                # AI agent workspace (rules, plans, tasks, notepads)
└── .local-ignore/            # Dev-only test fixtures + PR worktrees
```

## INITIALIZATION FLOW

```
pluginModule.server(input, options)
  ├─→ loadPluginConfig()         # JSONC parse → project/user merge → Zod validate → migrate
  ├─→ createManagers()           # TmuxSessionManager, BackgroundManager, SkillMcpManager, ConfigHandler
  ├─→ createTools()              # SkillContext + AvailableCategories + ToolRegistry (26 tools)
  ├─→ createHooks()              # 3-tier: Core(43) + Continuation(7) + Skill(2) = 52 hooks
  └─→ createPluginInterface()    # 10 OpenCode hook handlers → PluginInterface
```

## 10 OPENCODE HOOK HANDLERS

| Handler | Purpose |
|---------|---------|
| `config` | 6-phase: provider → plugin-components → agents → tools → MCPs → commands |
| `tool` | 26 registered tools |
| `chat.message` | First-message variant, session setup, keyword detection (ultrawork/search/analyze) |
| `chat.params` | Anthropic effort level, think mode, runtime fallback override |
| `chat.headers` | Copilot x-initiator header injection |
| `event` | Session lifecycle (created, deleted, idle, error), openclaw dispatch, runtime fallback |
| `tool.execute.before` | Pre-tool hooks (file guard, label truncator, rules injector, prometheus md-only) |
| `tool.execute.after` | Post-tool hooks (output truncation, comment checker, hashline read enhancer) |
| `experimental.chat.messages.transform` | Context injection, thinking block validation, tool pair validation |
| `experimental.session.compacting` | Context + todo preservation during compaction |

## WHERE TO LOOK

| Task | Location | Notes |
|------|----------|-------|
| Add new agent | `src/agents/` + `src/agents/builtin-agents/` | Follow createXXXAgent factory pattern |
| Add new hook | `src/hooks/{name}/` + register in `src/plugin/hooks/create-*-hooks.ts` | Match event type to tier |
| Add new tool | `src/tools/{name}/` + register in `src/plugin/tool-registry.ts` | Follow createXXXTool factory |
| Add new feature module | `src/features/{name}/` | Standalone module, wire in plugin/ |
| Add new MCP | `src/mcp/` + register in `createBuiltinMcps()` | Remote HTTP only (tier 1 of 3) |
| Add new skill | `src/features/builtin-skills/skills/` | Implement BuiltinSkill interface |
| Add new command | `src/features/builtin-commands/` | Template in templates/ |
| Add new CLI command | `src/cli/cli-program.ts` | Commander.js subcommand |
| Add new doctor check | `src/cli/doctor/checks/` | Register in checks/index.ts |
| Modify config schema | `src/config/schema/` + update root schema | Zod v4, add to OhMyOpenCodeConfigSchema |
| Add new category | `src/tools/delegate-task/constants.ts` | DEFAULT_CATEGORIES + CATEGORY_MODEL_REQUIREMENTS |
| Debug provider errors | `src/hooks/runtime-fallback/` | Reactive error recovery (distinct from model-fallback) |
| External notifications | `src/openclaw/` | Bidirectional Discord/Telegram/webhook integration |
| Skill-embedded MCP | `src/features/skill-mcp-manager/` | Tier 3 MCPs (stdio + HTTP, per-session) |

## MULTI-LEVEL CONFIG

```
Project (.opencode/oh-my-opencode.jsonc)  →  User (~/.config/opencode/oh-my-opencode.jsonc)  →  Defaults
```

- `agents`, `categories`, `claude_code`: deep merged recursively (prototype-pollution-safe)
- `disabled_*` arrays: Set union (concatenated + deduplicated)
- All other fields: override replaces base value
- Zod `safeParse()` fills defaults for omitted fields; partial parsing as fallback
- `migrateConfigFile()` transforms legacy keys automatically (idempotent via `_migrations` tracking)

Fields: agents (14 overridable, 21 fields each), categories (8 built-in + custom), disabled_* arrays (agents, hooks, mcps, skills, commands, tools), 19 feature-specific configs.

## THREE-TIER MCP SYSTEM

| Tier | Source | Mechanism |
|------|--------|-----------|
| Built-in | `src/mcp/` | 3 remote HTTP: websearch (Exa/Tavily), context7, grep_app |
| Claude Code | `.mcp.json` | `${VAR}` env expansion via claude-code-mcp-loader |
| Skill-embedded | SKILL.md YAML | Managed by SkillMcpManager (stdio + HTTP) |

## CONVENTIONS

- **Runtime**: Bun only (1.3.11 in CI) -- never use npm/yarn
- **TypeScript**: strict mode, ESNext, bundler moduleResolution, `bun-types` (never `@types/node`)
- **Test pattern**: Bun test (`bun:test`), co-located `*.test.ts`, given/when/then style (nested describe with `#given`/`#when`/`#then` prefixes or inline `// given` / `// when` / `// then` comments)
- **CI test split**: `script/run-ci-tests.ts` auto-detects `mock.module()` usage, isolates those tests in separate processes
- **Factory pattern**: `createXXX()` for all tools, hooks, agents
- **Hook tiers**: Session (24) → Tool-Guard (14) → Transform (5) → Continuation (7) → Skill (2)
- **Agent modes**: `primary` (respects UI model) vs `subagent` (own fallback chain) vs `all`
- **Model resolution**: 4-step: override → category-default → provider-fallback → system-default
- **Config format**: JSONC with comments, Zod v4 validation, snake_case keys
- **File naming**: kebab-case for all files/directories
- **Module structure**: index.ts barrel exports, no catch-all files (utils.ts, helpers.ts banned), 200 LOC soft limit
- **Imports**: relative within module, barrel imports across modules (`import { log } from "./shared"`)
- **No path aliases**: no `@/` -- relative imports only
- **Dual package**: `oh-my-opencode` + `oh-my-openagent` published simultaneously (transition period)

## ANTI-PATTERNS

- Never use `as any`, `@ts-ignore`, `@ts-expect-error`
- Never suppress lint/type errors
- Never add emojis to code/comments unless user explicitly asks
- Never commit unless explicitly requested
- Never run `bun publish` directly -- use GitHub Actions
- Never modify `package.json` version locally
- Test: given/when/then -- never use Arrange-Act-Assert comments
- Comments: avoid AI-generated comment patterns (enforced by comment-checker hook)
- Never create catch-all files (`utils.ts`, `helpers.ts`, `service.ts`)
- Empty catch blocks `catch(e) {}` -- always handle errors
- Never use em dashes, en dashes, or AI filler phrases in generated content
- index.ts is entry point ONLY -- never dump business logic there

## COMMANDS

```bash
bun test                    # Bun test suite
bun run build              # Build plugin (ESM + declarations + schema)
bun run build:all          # Build + platform binaries
bun run typecheck           # tsc --noEmit
bunx oh-my-opencode install # Interactive setup
bunx oh-my-opencode doctor  # Health diagnostics
bunx oh-my-opencode run     # Non-interactive session
```

## CI/CD

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| ci.yml | push/PR to master/dev | Tests (split: mock-heavy isolated + batch), typecheck, build, schema auto-commit |
| publish.yml | manual dispatch | Version bump, dual npm publish (oh-my-opencode + oh-my-openagent), platform binaries, GitHub release |
| publish-platform.yml | called by publish | 11 platform binaries via bun compile (darwin/linux/windows) |
| sisyphus-agent.yml | @mention / dispatch | AI agent handles issues/PRs |
| refresh-model-capabilities.yml | weekly schedule / dispatch | Auto-refresh model capabilities from models.dev API |
| cla.yml | issue_comment/PR | CLA assistant for contributors |
| lint-workflows.yml | push to .github/ | actionlint + shellcheck on workflow files |

## NOTES

- Logger writes to `/tmp/oh-my-opencode.log` -- check there for debugging
- Background tasks: 5 concurrent per model/provider (configurable, circuit breaker support)
- Plugin load timeout: 10s for Claude Code plugins
- Model fallback: per-agent chains in `shared/model-requirements.ts`, not a single global priority
- Two fallback systems: `model-fallback` (proactive, chat.params) vs `runtime-fallback` (reactive, session.error)
- Config migration: idempotent via `_migrations` tracking, creates timestamped backups before atomic writes
- Build: bun build (ESM) + tsc --emitDeclarationOnly, externals: @ast-grep/napi
- Test setup: `test-setup.ts` preloaded via bunfig.toml, resets session/cache state between tests
- Test split: `script/run-ci-tests.ts` auto-isolates files using `mock.module()` (plus `src/openclaw/__tests__/reply-listener-discord.test.ts`)
- 104 barrel export files (index.ts) establish module boundaries
- Architecture rules enforced via `.sisyphus/rules/modular-code-enforcement.md`
- Windows builds run on `windows-latest` runner (not cross-compiled) to avoid Bun segfaults
- Platform binaries detect AVX2 + libc family at runtime, fallback to baseline if needed
- Hashline edit: every Read output tagged with `LINE#ID` content hashes; edits reject on hash mismatch
- IntentGate: classifies user intent (research/implementation/investigation/evaluation/fix) before routing



## 🛡️ Strict Embedding Separation, Zero-Fallback Law & 24/7 Dual-GPU Invariant
- **Reference**: `/home/m1st/.agents/rules/RULE_Strict_Embedding_Separation_And_Dual_Pipeline.md`

### 1. Das Absolute Fallback-Verbot (Zero-Fallback Law)
Unter keinen Umständen, zu keinem Zeitpunkt und aus keinem Grund darf ein Fallback zwischen verschiedenen Embedding-Modellen stattfinden.
* **Geltende Aktion:** Fällt ein Embedding-Modell aus oder ist überlastet, MUSS die Operation sofort hart fehlschlagen (`Fail-Fast`) oder die Payload transaktional in einer Queue (NATS/SQLite) verharren, bis das exakte Modell bereit ist.
* **Verboten:** Kein stiller oder dynamischer Modellwechsel (weder Jina -> Gemma noch umgekehrt).

### 2. Warum ein Embedding-Fallback mathematisch & informationstheoretisch unmöglich ist
* **Topologische Inkompatibilität heterogener Vektorräume (Non-Isomorphism):**
  Jedes Modell $f_\theta: \mathcal{X} \to \mathbb{R}^D$ projiziert Text in eine spezifische, gelernte Riemannsche Mannigfaltigkeit. Jina v5 ($D=256$) und EmbeddingGemma ($D=768$) spannen zwei völlig inkompatible geometrische Räume auf. Die Basisvektoren der semantischen Achsen sind ohne explizite Procrustes-Transformation nicht ausgerichtet.
* **Kollaps der Kosinus-Ähnlichkeit ($	ext{sim} \approx 0$):**
  Wird eine Suchanfrage mit Modell $B$ berechnet ($v_q = f_B(q)$), während der Dokumentenkorpus mit Modell $A$ indiziert wurde ($v_d = f_A(d)$), verhält sich das Skalarprodukt mathematisch wie das zweier rein zufälliger Vektoren auf einer hochdimensionalen Einheitssphäre:
  $$\mathbb{E}[\text{sim}(u, v)] = 0 \quad \text{mit Varianz} \quad \sigma^2 = \frac{1}{D}$$
  Der Nearest-Neighbor-Algorithmus (HNSW/k-NN) liefert stochastisches Rauschen. Das RAG-System erhält völlig falsche oder irrelevante Kontexte.
* **Irreversible Index-Vergiftung (Index Poisoning):**
  Wird auch nur ein einziger Vektor von Modell $B$ als "Fallback" in den Index von Modell $A$ geschrieben, verunreinigt er die Distanzgraphen und Clusterzentren dauerhaft.
* **Das Gesetz des Fail-Fast:**
  Ein Ausfall muss hart abbrechen (`HTTP 503 Service Unavailable / IngestionQueueBlocked`).

### 3. Duale 24/7 Erfassungspflicht (GPU-Only)
* **GPU-Only Mandat:** Es läuft absolut nichts auf der CPU — GPU ONLY (NVIDIA GB10 CUDA) für ausnahmslos jedes Embedding-Modell.
* **24/7 Parallelität:** Sowohl `jina-embeddings-v5-omni-nano-classification` (256D, ~4,1 GB VRAM) als auch `google/embeddinggemma-300m` (768D, ~1,2 GB VRAM) laufen dauerhaft 24/7 im VRAM (Summe ~5,3 GB VRAM).
* **Duale Erfassung:** Jeder zu indizierende Text/Chunk wird immer von beiden Modellen parallel eingebettet und getrennt persistiert.

### 4. Idioten- & Failsafe-Sicherung auf Datenbankebene
* **SQLite Schema CHECK-Constraints:**
  `model_signature TEXT NOT NULL CHECK(model_signature = '...')` und `dimension INTEGER NOT NULL CHECK(dimension = ...)` erzwingen atomare Abbrüche auf Engine-Ebene bei Modell-Mismatches.
* **Qdrant Collection Constraints:**
  Strikte Trennung in separate Collections (`dgx_text_embeddings_jina_256` vs `dgx_text_embeddings_gemma_768`) mit fixierter Vektordimension.


## ⚡ High-Quality Systems Programming Languages Priority (No-Python Policy)
- **Reference**: `/home/m1st/.agents/rules/RULE_High_Quality_Systems_Programming_Languages.md`
- **Rule**:
  1. **Bevorzugte Sprachen:** High Quality **Golang (Go), Rust, C++, Zig, PowerShell, C** sind IMMER und AUSNAHMSLOS die bevorzugten Programmiersprachen.
  2. **Kein Python:** Python ist für neue Daemons, Watcher, Automatisierungen, APIs, CLI-Tools und Dienste strikt untersagt (GIL-Bottlenecks, Dependency-Drift, Speicherineffizienz).
  3. **Natives Systems-Engineering:** Alle Hintergrunddienste, Caching-Ebenen und Task-Runner müssen als native, speichersichere und nebenläufige Binaries kompiliert werden.
