# src/tools/lsp/ — LSP Tool Implementations

**Generated:** 2026-04-11

## OVERVIEW

33 files. Full LSP (Language Server Protocol) client stack exposed as 6 tools. Custom implementation that manages server processes, opens files, and forwards requests — does NOT delegate to OpenCode's built-in LSP.

## TOOL EXPOSURE

| Tool | File | What It Does |
|------|------|--------------|
| `lsp_goto_definition` | `goto-definition-tool.ts` | Jump to symbol definition |
| `lsp_find_references` | `find-references-tool.ts` | All usages of a symbol |
| `lsp_symbols` | `symbols-tool.ts` | Document outline or workspace symbol search |
| `lsp_diagnostics` | `diagnostics-tool.ts` | Errors/warnings from language server |
| `lsp_prepare_rename` | `rename-tools.ts` | Validate rename before applying |
| `lsp_rename` | `rename-tools.ts` | Apply safe rename across workspace |

All 6 are direct `ToolDefinition` objects (not factory functions) — registered directly in `tool-registry.ts`.

## ARCHITECTURE

```
tools.ts (6 ToolDefinition exports)
  ↓ uses
LspClientWrapper (lsp-client-wrapper.ts)
  ↓ wraps
LSPClient (lsp-client.ts) extends LSPClientConnection (lsp-client-connection.ts)
  ↓ communicates via
LSPClientTransport (lsp-client-transport.ts)
  ↓ talks to
LSPProcess (lsp-process.ts) — spawns server binary
```

## KEY FILES

| File | Purpose |
|------|---------|
| `lsp-client-wrapper.ts` | High-level entry: resolves server, opens file, runs request |
| `lsp-client.ts` | `LSPClient` — file tracking, document sync (`didOpen`/`didChange`) |
| `lsp-client-connection.ts` | JSON-RPC request/response/notification layer |
| `lsp-client-transport.ts` | stdin/stdout byte-stream framing |
| `lsp-process.ts` | Spawn + cleanup of LSP server process |
| `lsp-manager-process-cleanup.ts` | Reap orphan LSP processes on exit |
| `lsp-manager-temp-directory-cleanup.ts` | Clean temp dirs used by some servers |
| `server-definitions.ts` | 40+ builtin servers synced from OpenCode's `server.ts` |
| `server-config-loader.ts` | Load custom server config from `.opencode/lsp.json` |
| `server-resolution.ts` | Resolve which server handles a file extension |
| `server-installation.ts` | Detect missing binaries, surface install hints |
| `language-mappings.ts` | Extension → language ID mapping |
| `lsp-formatters.ts` | Format LSP responses into human-readable strings |
| `workspace-edit.ts` | Apply `WorkspaceEdit` results to disk (for rename) |
| `types.ts` | `LSPServerConfig`, `Position`, `Range`, `Location`, `Diagnostic` etc. |

## SERVER RESOLUTION

```
file.ts → extension (.ts) → language-mappings → server ID (typescript)
  → server-resolution: check user config (.opencode/lsp.json) → fall back to server-definitions.ts
  → server-installation: verify binary exists (warn with install hint if not)
  → LSPProcess.spawn(command[])
```

## NOTES

- File must be opened via `didOpen` before any LSP request — `LSPClient.openFile()` handles this
- 1s delay after `didOpen` for server initialization before sending requests
- `lsp_servers` tool was removed — duplicates OpenCode's built-in `LspServers` tool
- Synced with OpenCode's `server.ts` — when adding servers, check upstream first



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
