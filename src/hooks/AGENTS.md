# src/hooks/ — 52 Lifecycle Hooks

**Generated:** 2026-04-18

## OVERVIEW

52 hooks across dedicated modules and standalone files. Three-tier composition: Core(43) + Continuation(7) + Skill(2). All hooks follow `createXXXHook(deps) → HookFunction` factory pattern.

## HOOK TIERS

### Tier 1: Session Hooks (24) — `create-session-hooks.ts`
## STRUCTURE
```
hooks/
├── agent-usage-reminder/         # Reminds about available agents
├── atlas/                      # Main orchestration (757 lines)
├── anthropic-context-window-limit-recovery/ # Auto-summarize
├── anthropic-effort/            # Reasoning effort level adjustment
├── auto-slash-command/         # Detects /command patterns
├── auto-update-checker/        # Plugin update check
├── background-notification/    # OS notification
├── category-skill-reminder/    # Reminds of category skills
├── claude-code-hooks/          # settings.json compat layer
├── comment-checker/            # Prevents AI slop
├── compaction-context-injector/ # Injects context on compaction
├── compaction-todo-preserver/  # Preserves todos through compaction
├── delegate-task-retry/        # Retries failed delegations
├── directory-agents-injector/  # Auto-injects AGENTS.md
├── directory-readme-injector/  # Auto-injects README.md
├── edit-error-recovery/        # Recovers from failures
├── hashline-edit-diff-enhancer/ # Enhanced diff output for hashline edits
├── hashline-read-enhancer/     # Adds LINE#ID hashes to Read output
├── interactive-bash-session/   # Tmux session management
├── json-error-recovery/        # JSON parse error correction
├── keyword-detector/           # ultrawork/search/analyze modes
├── legacy-plugin-toast/        # Legacy plugin name migration toast
├── model-fallback/             # Provider-level model fallback
├── no-hephaestus-non-gpt/      # Block Hephaestus from non-GPT
├── no-sisyphus-gpt/            # Block Sisyphus from GPT
├── non-interactive-env/        # Non-TTY environment handling
├── prometheus-md-only/         # Planner read-only mode
├── question-label-truncator/   # Auto-truncates question labels
├── ralph-loop/                 # Self-referential dev loop
├── read-image-resizer/         # Resize images for context efficiency
├── rules-injector/             # Conditional rules
├── runtime-fallback/           # Auto-switch models on API errors
├── session-recovery/           # Auto-recovers from crashes
├── sisyphus-junior-notepad/    # Sisyphus Junior notepad
├── start-work/                 # Sisyphus work session starter
├── stop-continuation-guard/    # Guards stop continuation
├── task-reminder/              # Task system usage reminders
├── task-resume-info/           # Resume info for cancelled tasks
├── tasks-todowrite-disabler/   # Disable TodoWrite when task system active
├── think-mode/                 # Dynamic thinking budget
├── thinking-block-validator/   # Ensures valid <thinking>
├── todo-continuation-enforcer/ # Force TODO completion
├── todo-description-override/  # Override todo descriptions
├── tool-pair-validator/        # Validate tool pair usage
├── unstable-agent-babysitter/  # Monitor unstable agent behavior
├── webfetch-redirect-guard/    # Guard webfetch redirect behavior
├── write-existing-file-guard/  # Require Read before Write
└── index.ts                    # Hook aggregation + registration
```

| Hook | Event | Purpose |
|------|-------|---------|
| contextWindowMonitor | session.idle | Track context window usage |
| preemptiveCompaction | session.idle | Trigger compaction before limit |
| sessionRecovery | session.error | Auto-retry on recoverable errors |
| sessionNotification | session.idle | OS notifications on completion |
| thinkMode | chat.params | Model variant switching (extended thinking) |
| anthropicContextWindowLimitRecovery | session.error | Multi-strategy context recovery (truncation, compaction) |
| autoUpdateChecker | session.created | Check npm for plugin updates |
| agentUsageReminder | chat.message | Remind about available agents |
| nonInteractiveEnv | chat.message | Adjust behavior for `run` command |
| interactiveBashSession | tool.execute | Tmux session for interactive tools |
| ralphLoop | event | Self-referential dev loop (boulder continuation) |
| editErrorRecovery | tool.execute.after | Retry failed file edits |
| delegateTaskRetry | tool.execute.after | Retry failed task delegations |
| startWork | chat.message | `/start-work` command handler |
| prometheusMdOnly | tool.execute.before | Enforce .md-only writes for Prometheus |
| sisyphusJuniorNotepad | chat.message | Notepad injection for subagents |
| questionLabelTruncator | tool.execute.before | Truncate long question labels |
| taskResumeInfo | chat.message | Inject task context on resume |
| anthropicEffort | chat.params | Adjust reasoning effort level |
| modelFallback | chat.params | Provider-level model fallback on errors |
| noSisyphusGpt | chat.message | Block Sisyphus from using GPT models (toast warning) |
| noHephaestusNonGpt | chat.message | Block Hephaestus from using non-GPT models |
| runtimeFallback | event | Auto-switch models on API provider errors |
| legacyPluginToast | chat.message | Show toast when legacy plugin name detected |

### Tier 2: Tool Guard Hooks (14) — `create-tool-guard-hooks.ts`

| Hook | Event | Purpose |
|------|-------|---------|
| commentChecker | tool.execute.after | Block AI-generated comment patterns |
| toolOutputTruncator | tool.execute.after | Truncate oversized tool output |
| directoryAgentsInjector | tool.execute.before | Inject dir AGENTS.md into context |
| directoryReadmeInjector | tool.execute.before | Inject dir README.md into context |
| emptyTaskResponseDetector | tool.execute.after | Detect empty task responses |
| rulesInjector | tool.execute.before | Conditional rules injection (AGENTS.md, config) |
| tasksTodowriteDisabler | tool.execute.before | Disable TodoWrite when task system active |
| writeExistingFileGuard | tool.execute.before | Require Read before Write on existing files |
| bashFileReadGuard | tool.execute.before | Guard bash commands that read files |
| readImageResizer | tool.execute.after | Resize large images for context efficiency |
| todoDescriptionOverride | tool.execute.before | Override todo item descriptions |
| webfetchRedirectGuard | tool.execute.before | Guard webfetch redirect behavior |
| hashlineReadEnhancer | tool.execute.after | Enhance Read output with line hashes |
| jsonErrorRecovery | tool.execute.after | Detect JSON parse errors, inject correction reminder |

### Tier 3: Transform Hooks (5) — `create-transform-hooks.ts`

| Hook | Event | Purpose |
|------|-------|---------|
| claudeCodeHooks | messages.transform | Claude Code settings.json compatibility |
| keywordDetector | messages.transform | Detect ultrawork/search/analyze modes |
| contextInjectorMessagesTransform | messages.transform | Inject AGENTS.md/README.md into context |
| thinkingBlockValidator | messages.transform | Validate thinking block structure |
| toolPairValidator | messages.transform | Validate tool call/result pairs |

### Tier 4: Continuation Hooks (7) — `create-continuation-hooks.ts`

| Hook | Event | Purpose |
|------|-------|---------|
| stopContinuationGuard | chat.message | `/stop-continuation` command handler |
| compactionContextInjector | session.compacted | Re-inject context after compaction |
| compactionTodoPreserver | session.compacted | Preserve todos through compaction |
| todoContinuationEnforcer | session.idle | **Boulder**: force continuation on incomplete todos |
| unstableAgentBabysitter | session.idle | Monitor unstable agent behavior |
| backgroundNotificationHook | event | Background task completion notifications |
| atlasHook | event | Master orchestrator for boulder/background sessions |

### Tier 5: Skill Hooks (2) — `create-skill-hooks.ts`

| Hook | Event | Purpose |
|------|-------|---------|
| categorySkillReminder | chat.message | Remind about category+skill delegation |
| autoSlashCommand | chat.message | Auto-detect `/command` in user input |

## KEY HOOKS (COMPLEX)

### anthropic-context-window-limit-recovery (31 files, ~2232 LOC)
Multi-strategy recovery when hitting context limits. Strategies: truncation, compaction, summarization.

### atlas (17 files, ~1976 LOC)
Master orchestrator for boulder sessions. Decision gates: session type → abort check → failure count → background tasks → agent match → plan completeness → cooldown (5s). Injects continuation prompts on session.idle.

### ralph-loop (14 files, ~1687 LOC)
Self-referential dev loop via `/ralph-loop` command. State persisted in `.sisyphus/ralph-loop.local.md`. Detects `<promise>DONE</promise>` in AI output. Max 100 iterations default.

### todo-continuation-enforcer (13 files, ~2061 LOC)
"Boulder" mechanism. Forces agent to continue when todos remain incomplete. 2s countdown toast → continuation injection. Exponential backoff: 30s base, ×2 per failure, max 5 consecutive failures then 5min pause.

### keyword-detector (~1665 LOC)
Detects modes from user input: ultrawork, search, analyze, prove-yourself. Injects mode-specific system prompts.

### rules-injector (19 files, ~1604 LOC)
Conditional rules injection from AGENTS.md, config, skill rules. Evaluates conditions to determine which rules apply.

## STANDALONE HOOKS (in src/hooks/ root)

| File | Purpose |
|------|---------|
| context-window-monitor.ts | Track context window percentage |
| preemptive-compaction.ts | Trigger compaction before hard limit |
| tool-output-truncator.ts | Truncate tool output by token count |
| session-notification.ts + 4 helpers | OS notification on session completion |
| empty-task-response-detector.ts | Detect empty/failed task responses |
| session-todo-status.ts | Todo completion status tracking |

## HOW TO ADD A HOOK

1. Create `src/hooks/{name}/index.ts` with `createXXXHook(deps)` factory
2. Register in appropriate tier file (`src/plugin/hooks/create-{tier}-hooks.ts`)
3. Add hook name to `src/config/schema/hooks.ts` HookNameSchema
4. Hook receives `(event, ctx)` — return value depends on event type



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
