# src/cli/ — CLI: install, run, doctor, mcp-oauth

**Generated:** 2026-04-18

## OVERVIEW

Commander.js CLI with 6 commands. Entry: `index.ts` → `runCli()` in `cli-program.ts`.

## COMMANDS

| Command | Purpose | Key Logic |
|---------|---------|-----------|
| `install` | Interactive/non-interactive setup | Provider selection → config gen → plugin registration |
| `run <message>` | Non-interactive session launcher | Agent resolution (flag → env → config → Sisyphus) |
| `doctor` | 4-category health checks | System, Config, Tools, Models |
| `get-local-version` | Version detection | Installed vs npm latest |
| `mcp-oauth` | OAuth token management | login (PKCE), logout, status |
| `refresh-model-capabilities` | Refresh models.dev cache | Model capabilities refresh |

## STRUCTURE

```
cli/
├── index.ts                     # Entry point → runCli()
├── cli-program.ts               # Commander.js program (5 commands)
├── install.ts                   # Routes to TUI or CLI installer
├── cli-installer.ts             # Non-interactive (console output)
├── tui-installer.ts             # Interactive (@clack/prompts)
├── model-fallback.ts            # Model config gen by provider availability
├── provider-availability.ts     # Provider detection
├── fallback-chain-resolution.ts # Fallback chain logic
├── config-manager/              # 20 config utilities
│   ├── plugin registration, provider config
│   ├── JSONC operations, auth plugins
│   └── npm dist-tags, binary detection
├── doctor/
│   ├── runner.ts                # Parallel check execution
│   ├── formatter.ts             # Output formatting
│   └── checks/                  # 15 check files in 4 categories
│       ├── system.ts            # Binary, plugin, version
│       ├── config.ts            # JSONC validity, Zod schema
│       ├── tools.ts             # AST-Grep, LSP, GH CLI, MCP
│       └── model-resolution.ts  # Cache, resolution, overrides (6 sub-files)
├── run/                         # Session launcher
│   ├── runner.ts                # Main orchestration
│   ├── agent-resolver.ts        # Flag → env → config → Sisyphus
│   ├── session-resolver.ts      # Create/resume sessions
│   ├── event-handlers.ts        # Event processing
│   └── poll-for-completion.ts   # Wait for todos/background tasks
└── mcp-oauth/                   # OAuth token management
```

## MODEL FALLBACK SYSTEM

No single global priority. CLI install-time resolution uses per-agent fallback chains from `model-fallback-requirements.ts`.

Common patterns: Claude/OpenAI/Gemini are preferred when an agent chain includes them, `librarian` prefers ZAI, `sisyphus` falls back through Kimi then GLM-5, and `hephaestus` requires OpenAI-compatible providers.

## DOCTOR CHECKS

| Category | Validates |
|----------|-----------|
| **System** | Binary found, version >=1.0.150, plugin registered, version match |
| **Config** | JSONC validity, Zod schema, model override syntax |
| **Tools** | AST-Grep, comment-checker, LSP servers, GH CLI, MCP servers |
| **Models** | Cache exists, model resolution, agent/category overrides, availability |

## HOW TO ADD A DOCTOR CHECK

1. Create `src/cli/doctor/checks/{name}.ts`
2. Export check function matching `DoctorCheck` interface
3. Register in `checks/index.ts`



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
