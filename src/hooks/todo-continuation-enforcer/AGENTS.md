# src/hooks/todo-continuation-enforcer/ — Boulder Continuation Mechanism

**Generated:** 2026-04-11

## OVERVIEW

14 files (~2061 LOC). The "boulder" — Continuation Tier hook that forces Sisyphus to keep rolling when incomplete todos remain. Fires on `session.idle`, injects continuation prompt after 2s countdown toast.

## HOW IT WORKS

```
session.idle
  → Is main session (not prometheus/compaction)? (DEFAULT_SKIP_AGENTS)
  → No abort detected recently? (ABORT_WINDOW_MS = 3s)
  → Todos still incomplete? (todo.ts)
  → No background tasks running?
  → Cooldown passed? (CONTINUATION_COOLDOWN_MS = 30s)
  → Failure count < max? (MAX_CONSECUTIVE_FAILURES = 5)
  → Start 2s countdown toast → inject CONTINUATION_PROMPT
```

## KEY FILES

| File | Purpose |
|------|---------|
| `handler.ts` | `createTodoContinuationHandler()` — event router, delegates to idle/non-idle handlers |
| `idle-event.ts` | `handleSessionIdle()` — main decision gate for session.idle |
| `non-idle-events.ts` | `handleNonIdleEvent()` — handles session.error (abort detection) |
| `session-state.ts` | `SessionStateStore` — per-session failure/abort/cooldown state |
| `todo.ts` | Check todo completion status via session store |
| `countdown.ts` | 2s countdown toast before injection |
| `abort-detection.ts` | Detect MessageAbortedError / AbortError |
| `continuation-injection.ts` | Build + inject CONTINUATION_PROMPT into session |
| `message-directory.ts` | Temp dir for message injection exchange |
| `constants.ts` | Timing constants, CONTINUATION_PROMPT, skip agents |
| `types.ts` | `SessionState`, handler argument types |

## CONSTANTS

```typescript
DEFAULT_SKIP_AGENTS = ["prometheus", "compaction", "plan"]
CONTINUATION_COOLDOWN_MS = 30_000     // 30s between injections
MAX_CONSECUTIVE_FAILURES = 5          // Then 5min pause (exponential backoff)
FAILURE_RESET_WINDOW_MS = 5 * 60_000  // 5min window for failure reset
COUNTDOWN_SECONDS = 2
ABORT_WINDOW_MS = 3000                // Grace after abort signal
```

## STATE PER SESSION

```typescript
interface SessionState {
  failureCount: number       // Consecutive failures
  lastFailureAt?: number     // Timestamp
  abortDetectedAt?: number   // Reset after ABORT_WINDOW_MS
  cooldownUntil?: number     // Next injection allowed after
  countdownTimer?: Timer     // Active countdown reference
}
```

## RELATIONSHIP TO ATLAS

`todoContinuationEnforcer` handles **main Sisyphus sessions** only.
`atlasHook` handles **boulder/ralph/subagent sessions** with a different decision gate.
Both fire on `session.idle` but check session type first.



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
