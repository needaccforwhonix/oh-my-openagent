# src/tools/call-omo-agent/ — Direct Agent Invocation Tool

**Generated:** 2026-04-11

## OVERVIEW

23 files. The `call_omo_agent` tool — direct invocation of named agents (explore, librarian only). Distinct from `delegate-task`: no category system, no skill loading, no model selection. Fixed agent set, same execution modes (background/sync).

## DISTINCTION FROM delegate-task

| Aspect | `call_omo_agent` | `delegate-task` (`task`) |
|--------|-----------------|--------------------------|
| Agent selection | Named agent (explore/librarian) | Category or subagent_type |
| Skill loading | None | `load_skills[]` supported |
| Model selection | From agent's fallback chain | From category config |
| Use case | Quick contextual grep | Full delegation with skills |

## ALLOWED AGENTS

Only `explore` and `librarian` — enforced via `ALLOWED_AGENTS` constant in `constants.ts`. Case-insensitive validation.

## EXECUTION MODES

Same two modes as delegate-task:

| Mode | File | Description |
|------|------|-------------|
| **Background** | `background-agent-executor.ts` | Async via `BackgroundManager` |
| **Sync** | `sync-executor.ts` | Create session → wait for idle → return result |

## KEY FILES

| File | Purpose |
|------|---------|
| `tools.ts` | `createCallOmoAgent()` factory — validates agent, routes to executor |
| `background-executor.ts` | Routes to background or sync based on `run_in_background` |
| `background-agent-executor.ts` | Launch via `BackgroundManager.launch()` |
| `sync-executor.ts` | Synchronous session: create → send prompt → poll → fetch result |
| `session-creator.ts` | Create OpenCode session for sync execution |
| `subagent-session-creator.ts` | Create session with agent-specific config |
| `subagent-session-prompter.ts` | Inject prompt into session |
| `completion-poller.ts` | Poll until session idle |
| `session-completion-poller.ts` | Session-specific completion check |
| `session-message-output-extractor.ts` | Extract last assistant message as result |
| `message-processor.ts` | Process raw message content |
| `message-dir.ts` + `message-storage-directory.ts` | Temp storage for message exchange |
| `types.ts` | `CallOmoAgentArgs`, `AllowedAgentType`, `ToolContextWithMetadata` |

## SESSION CONTINUATION

Pass `session_id` to resume an existing session rather than create a new one — handled in both executors.



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
