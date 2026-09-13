# src/hooks/runtime-fallback/ — Reactive Provider Error Recovery

**Generated:** 2026-04-11

## OVERVIEW

32 files. Session Tier hook that **reactively** switches to fallback models when API providers return errors at runtime (429, 503, quota exhausted, cooldown signals). Distinct from `model-fallback` (which applies preemptively at chat.params).

## RUNTIME-FALLBACK vs MODEL-FALLBACK

| Aspect | runtime-fallback | model-fallback |
|--------|-----------------|----------------|
| **Trigger** | Reactive — after error occurs | Proactive — at request time |
| **Event** | session.error, message.updated, session.status | chat.params |
| **Config source** | `categories[].fallback_models`, `agents[].fallback_models` | `AGENT_MODEL_REQUIREMENTS` hardcoded chains |
| **State** | Per-session FallbackState + cooldown tracking | Module-global pendingModelFallbacks |
| **Use case** | Provider errors during execution | Pre-configured agent fallback chains |

They operate **independently** — no direct integration.

## ERROR DETECTION

### HTTP Status Codes (configurable)
Default retry codes: `429, 500, 502, 503, 504`

### Error Message Patterns (constants.ts)
```
/rate.?limit/i, /too.?many.?requests/i, /quota.*reset.*after/i,
/exhausted.*capacity/i, /all.*credentials.*for.*model/i,
/cool(?:ing)?.?down/i, /model.*not.*supported/i,
/service.?unavailable/i, /overloaded/i, /temporarily.?unavailable/i
```

### Error Type Classification (error-classifier.ts)
- `missing_api_key` — provider rejects auth
- `model_not_found` — model unavailable
- `quota_exceeded` — billing/quota hit
- Auto-retry signal detection via `auto-retry-signal.ts` — extracts "retrying in ~2 weeks" style signals, triggers immediate fallback

## FALLBACK STATE MACHINE

```typescript
interface FallbackState {
  originalModel: string
  currentModel: string
  fallbackIndex: number
  failedModels: Map<string, number>  // model → cooldown-until timestamp
  attemptCount: number
  pendingFallbackModel?: string
}
```

## FALLBACK CHAIN RESOLUTION (fallback-models.ts)

Priority order:
1. **Session category** (via SessionCategoryRegistry)
2. **Agent config** `fallback_models`
3. **Agent's category** `fallback_models`
4. **Session ID pattern match** (detect agent from session ID format)

## RETRY FLOW

```
session.error / message.updated (with error) / session.status (retry signal)
  → isRetryableError(error)?
  → getFallbackModelsForSession(sessionID, agent)
  → findNextAvailableFallback() — skip cooldown models
  → prepareFallback() — update state, mark current failed
  → dispatchFallbackRetry() — toast notification + promptAsync with new model
  → 30s timeout — abort and try next if exceeded
```

## COOLDOWN MECHANISM

Failed models enter 60s cooldown. `findNextAvailableFallback()` skips models in cooldown, preventing thrashing on persistently failing models.

## KEY FILES

| File | Purpose |
|------|---------|
| `hook.ts` | `createRuntimeFallbackHook()` — composes all handlers |
| `event-handler.ts` | Route session lifecycle (created, error, stop, idle) |
| `message-update-handler.ts` | Handle error parts in `message.updated` |
| `session-status-handler.ts` | Handle provider retry signals in session.status |
| `chat-message-handler.ts` | Apply fallback model override on chat.message |
| `error-classifier.ts` | `isRetryableError()`, `classifyErrorType()` |
| `auto-retry-signal.ts` | Extract "retrying in..." signals |
| `fallback-state.ts` | State machine: createFallbackState, prepareFallback, findNextAvailableFallback, isModelInCooldown |
| `fallback-models.ts` | Resolve chain from config hierarchy (strings + raw objects) |
| `fallback-bootstrap-model.ts` | Derive initial model when state missing |
| `fallback-retry-dispatcher.ts` | Toast + dispatch retry orchestration |
| `auto-retry.ts` | Abort, timeout scheduling, cleanup |
| `agent-resolver.ts` | Session → agent name normalization |
| `retry-model-payload.ts` | Build model payload (providerID/modelID/variant/reasoningEffort) |
| `visible-assistant-response.ts` | Detect if assistant produced real output vs just errors |
| `last-user-retry-parts.ts` | Extract last user message parts for retry |

## NOTES

- Cooldown and failure tracking are **per-session** — concurrent sessions don't share state
- `visible-assistant-response.ts` prevents retry if the assistant already produced a partial valid response
- Runtime-fallback is registered in the Session Tier via `create-session-hooks.ts`



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
