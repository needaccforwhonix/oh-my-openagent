# src/tools/background-task/ — Background Task Tool Wrappers

**Generated:** 2026-04-11

## OVERVIEW

18 files. Tool-layer wrappers for `background_output` and `background_cancel`. Does NOT implement the background execution engine — that lives in `src/features/background-agent/`. This directory provides the LLM-facing tool interface.

## THREE TOOLS

| Tool | Factory | Purpose |
|------|---------|---------|
| `background_output` | `createBackgroundOutput` | Get results from a running/completed background task |
| `background_cancel` | `createBackgroundCancel` | Cancel running task(s) |
| `createBackgroundTask` | internal | Shared factory used by both |

## KEY FILES

| File | Purpose |
|------|---------|
| `create-background-output.ts` | `background_output` tool: fetch task results by task_id |
| `create-background-cancel.ts` | `background_cancel` tool: cancel by taskId or all=true |
| `create-background-task.ts` | Shared tool factory with common params |
| `clients.ts` | Client interfaces for background output and cancel |
| `session-messages.ts` | Fetch session messages from OpenCode |
| `full-session-format.ts` | Format full session output (messages, thinking blocks) |
| `task-result-format.ts` | Format task result for LLM consumption |
| `task-status-format.ts` | Format task status (running/completed/error) |
| `message-dir.ts` | Temp directory for message exchange |
| `truncate-text.ts` | Truncate large output to fit context |
| `time-format.ts` | Human-readable duration formatting |
| `delay.ts` | Polling delay utility |
| `types.ts` | `BackgroundTaskOptions`, result/status types |
| `constants.ts` | Timeout defaults, polling intervals |

## BACKGROUND OUTPUT MODES

```
background_output(task_id, block=false)  → check current status/result
background_output(task_id, block=true)   → wait until complete (timeout default: 120s)
background_output(task_id, full_session=true) → return full session transcript
background_output(task_id, message_limit=N) → last N messages only
background_output(task_id, include_thinking=true) → include thinking blocks
```

## RELATIONSHIP TO BACKGROUND ENGINE

```
tools/background-task/  ← LLM tool interface
features/background-agent/  ← execution engine (BackgroundManager)
```

`createBackgroundOutput` queries `BackgroundManager.getTask(task_id)` — it does not manage task state.



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
