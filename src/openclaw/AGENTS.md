# src/openclaw/ — Bidirectional External Integration

**Generated:** 2026-04-18

## OVERVIEW

18 files. Bidirectional integration system: **outbound** session event notifications (Discord/Telegram/HTTP webhook/shell command) AND **inbound** reply handling (daemon polls chat apps, injects replies back into tmux session). Named "claw" because it reaches out from OpenCode and pulls replies back in.

## BIDIRECTIONAL FLOW

### Outbound (OpenCode → External)
```
OpenCode session event → dispatchOpenClawEvent()
  → runtime-dispatch.ts: map event to OpenClaw event
  → dispatcher.ts: execute gateway (HTTP POST or shell command)
  → session-registry.ts: record message ID ↔ sessionID ↔ tmux pane
```

### Inbound (External → OpenCode)
```
Discord/Telegram API → reply-listener daemon (separate Bun process)
  → reply-listener-{discord,telegram}.ts: poll every 3s
  → session-registry.ts: look up target tmux session from message ID
  → reply-listener-injection.ts: send-keys into tmux pane (rate limited)
```

## KEY FILES

| File | Purpose |
|------|---------|
| `index.ts` | `wakeOpenClaw()`, `initializeOpenClaw()` — main entry |
| `types.ts` | `OpenClawConfig`, `OpenClawPayload`, `WakeResult` types |
| `config.ts` | Gateway resolution + URL validation (HTTPS required, localhost exception) |
| `dispatcher.ts` | HTTP POST + shell command execution with variable interpolation |
| `runtime-dispatch.ts` | Maps OpenCode events → OpenClaw events, orchestrates dispatch |
| `session-registry.ts` | JSONL registry correlating message IDs ↔ sessions ↔ panes (file-locked) |
| `reply-listener.ts` | Daemon lifecycle: start/stop, poll loop, state persistence |
| `reply-listener-discord.ts` | Discord API polling |
| `reply-listener-telegram.ts` | Telegram API polling |
| `reply-listener-injection.ts` | Inject received reply into tmux pane (rate limiting + user filtering) |
| `reply-listener-state.ts` | Daemon state: PID, config signature, poll tracking |
| `daemon.ts` | Daemon entry point (runs as detached Bun process) |
| `tmux.ts` | `capturePane()`, `sendToPane()` utilities |

## GATEWAY TYPES

| Type | Config | Execution |
|------|--------|-----------|
| **HTTP webhook** | `url` field | POST with JSON payload |
| **Shell command** | `command` field | Execute with env vars (OPENCLAW_*) |

## PAYLOAD VARIABLES (interpolation)

`{sessionId}`, `{projectPath}`, `{tmuxSession}`, `{timestamp}`, `{eventType}` (session.created/deleted/idle), `{messageContent}`, `{promptSummary}`

## INTEGRATION POINTS

- `src/index.ts` — calls `initializeOpenClaw(pluginConfig.openclaw)` at plugin startup (if `enabled`)
- `src/plugin/event.ts` — calls `dispatchOpenClawEvent()` for session.created/deleted/idle
- `src/config/schema/openclaw.ts` — Zod config schema

## DAEMON LIFECYCLE

```
initializeOpenClaw(config)
  → wakeOpenClaw() if reply_listener.enabled
  → spawn daemon.ts as detached process
  → daemon writes PID to .opencode/openclaw.state.json
  → daemon polls Discord/Telegram every 3s
  → on reply: lookup in session-registry → inject into tmux via send-keys
```

## SECURITY

- **URL validation**: HTTPS required except localhost (config.ts)
- **Authorized users**: Inbound replies filtered by allowed user ID list
- **Token redaction**: Secrets masked in logs and error messages
- **Rate limiting**: Reply injection throttled per pane

## TESTING NOTE

`reply-listener-discord.test.ts` is **always isolated** in CI (listed in `ALWAYS_ISOLATED_TEST_FILES` of `script/run-ci-tests.ts`). Reason: mocks `globalThis.fetch` for Discord API simulation — needs process isolation to avoid interference with shared test batch.



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
