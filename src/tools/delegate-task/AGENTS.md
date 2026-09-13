# src/tools/delegate-task/ — Task Delegation Engine

**Generated:** 2026-04-11

## OVERVIEW

49 files. The `task` tool implementation — delegates work to subagents via background or sync sessions. Resolves categories, models, skills, and manages both async and synchronous execution flows. 8+ built-in categories.

## TWO EXECUTION MODES

| Mode | Flow | Use Case |
|------|------|----------|
| **Background** (`run_in_background=true`) | Launch → BackgroundManager → poll → notify parent | Explore, librarian, parallel work |
| **Sync** (`run_in_background=false`) | Create session → send prompt → poll until idle → return result | Sequential tasks needing immediate result |

## KEY FILES

| File | Purpose |
|------|---------|
| `tools.ts` | `createDelegateTask()` factory — main entry point |
| `executor.ts` | Route to background or sync execution |
| `types.ts` | `DelegateTaskArgs`, `DelegateTaskToolOptions`, `ToolContextWithMetadata` |
| `category-resolver.ts` | Map category name → model + config |
| `subagent-resolver.ts` | Map subagent_type → agent + model |
| `model-selection.ts` | Model availability checking + fallback |
| `skill-resolver.ts` | Resolve `load_skills[]` → skill content for injection |
| `prompt-builder.ts` | Build system/user prompt with skill content, categories |

## SYNC EXECUTION CHAIN

```
sync-task.ts → sync-session-creator.ts → sync-prompt-sender.ts → sync-session-poller.ts → sync-result-fetcher.ts
```

Each file handles one step. `sync-continuation.ts` handles session continuation (resume with task_id).

## BACKGROUND EXECUTION

```
background-task.ts → BackgroundManager.launch() → (async polling) → background-continuation.ts
```

`background-continuation.ts` handles `task_id` resume for existing background tasks.

## CATEGORY RESOLUTION

1. Check user-defined categories (`pluginConfig.categories`)
2. Fall back to built-in 8 categories
3. Resolve model from category config
4. Check model availability → fallback if unavailable

## MODEL STRING PARSER

`model-string-parser.ts` handles `"model variant"` format (e.g., `"gpt-5.3-codex medium"` → model=`gpt-5.3-codex`, variant=`medium`).

## UNSTABLE AGENT TRACKING

`unstable-agent-task.ts` marks tasks from categories/agents known to be unstable (e.g., free models). Enables `unstableAgentBabysitter` hook monitoring.



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
