# src/shared/ — 100+ Utility Files

**Generated:** 2026-04-18

## OVERVIEW

Cross-cutting utilities used throughout the plugin. Barrel-exported from `index.ts`. Logger writes to `/tmp/oh-my-opencode.log`.

## CATEGORY MAP

| Category | Files | Key Exports |
|----------|-------|-------------|
| **Model Resolution** | ~22 | `resolveModel()`, `checkModelAvailability()`, `AGENT_MODEL_REQUIREMENTS` |
| **Tmux Integration** | 11 | `createTmuxSession()`, `spawnPane()`, `closePane()`, server health |
| **Configuration & Paths** | 10 | `resolveOpenCodeConfigDir()`, `getDataPath()`, `parseJSONC()` |
| **Session Management** | 8 | `SessionCursor`, `trackInjectedPath()`, `SessionToolsStore` |
| **Git Worktree** | 7 | `parseGitStatusPorcelain()`, `collectGitDiffStats()`, `formatFileChanges()` |
| **Command Execution** | 7 | `executeCommand()`, `executeHookCommand()`, embedded command registry |
| **Migration** | 6 | `migrateConfigFile()`, AGENT_NAME_MAP, HOOK_NAME_MAP, MODEL_VERSION_MAP |
| **String & Tool Utils** | 6 | `toSnakeCase()`, `normalizeToolName()`, `parseFrontmatter()` |
| **Agent Configuration** | 5 | `getAgentVariant()`, `AGENT_DISPLAY_NAMES`, `AGENT_TOOL_RESTRICTIONS` |
| **OpenCode Integration** | 5 | `injectServerAuth()`, `detectExternalPlugins()`, client accessors |
| **Type Helpers** | 4 | `deepMerge()`, `DynamicTruncator`, `matchPattern()`, `isRecord()` |
| **Misc** | 8 | `log()`, `readFile()`, `extractZip()`, `downloadBinary()`, `findAvailablePort()` |

## MODEL RESOLUTION PIPELINE

```
resolveModel(input)
  1. Override: UI-selected model (primary agents only)
  2. Category default: From category config
  3. Provider fallback: AGENT_MODEL_REQUIREMENTS chains
  4. System default: Ultimate fallback
```

Key files: `model-resolver.ts` (entry), `model-resolution-pipeline.ts` (orchestration), `model-requirements.ts` (fallback chains), `model-availability.ts` (fuzzy matching).

## MIGRATION SYSTEM

Automatically transforms legacy config on load:
- `agent-names.ts`: Old agent names → new (e.g., `junior` → `sisyphus-junior`)
- `hook-names.ts`: Old hook names → new
- `model-versions.ts`: Old model IDs → current
- `agent-category.ts`: Legacy agent configs → category system

## MOST IMPORTED

| Utility | Import Count | Purpose |
|---------|-------------|---------|
| `logger.ts` | 62 | `/tmp/oh-my-opencode.log` |
| `data-path.ts` | 11 | XDG storage resolution |
| `model-requirements.ts` | 11 | Agent fallback chains |
| `system-directive.ts` | 11 | System message filtering |
| `frontmatter.ts` | 10 | YAML metadata extraction |



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
