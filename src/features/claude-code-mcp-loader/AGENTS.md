# src/features/claude-code-mcp-loader/ — Tier 2 MCP Loader (.mcp.json)

**Generated:** 2026-04-18

## OVERVIEW

11 files. Loads `.mcp.json` files from project/user scopes and expands `${VAR}` env vars. Feeds Tier 2 of the 3-tier MCP system into `mcp-config-handler.ts` during Phase 5 of config loading.

## WHY IT EXISTS

Claude Code ecosystem ships MCPs via `.mcp.json` files with `${VAR}` env var placeholders. OmO consumes these unchanged so existing Claude Code MCP configs work.

## LOAD PIPELINE

```
loadMcpConfigs(ctx)
  → scope-filter.ts: discover .mcp.json at project + user scopes
  → loader.ts: parse JSON
  → env-expander.ts: replace ${VAR} with process.env[VAR]
  → transformer.ts: map Claude Code format → OpenCode McpLocal / McpRemote shape
  → return LoadedMcpServer[]
```

## MCP FORMAT

```jsonc
// .mcp.json
{
  "mcpServers": {
    "my-stdio": {
      "type": "stdio",
      "command": "node",
      "args": ["server.js"],
      "env": {
        "API_KEY": "${MY_API_KEY}"
      }
    },
    "my-http": {
      "type": "http",       // "sse" legacy → mapped to http
      "url": "https://example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${MY_TOKEN}"
      }
    }
  }
}
```

## KEY FILES

| File | Purpose |
|------|---------|
| `index.ts` | Barrel: `loadMcpConfigs`, types |
| `loader.ts` | `loadMcpConfigs()` main entry |
| `types.ts` | `ClaudeCodeMcpServer`, `LoadedMcpServer`, `McpScope` |
| `env-expander.ts` | `expandEnvVarsInObject()` — recursive `${VAR}` substitution |
| `transformer.ts` | Claude Code format → OpenCode `Mcp` shape |
| `scope-filter.ts` | Project vs user scope precedence |

## THREE-TIER MCP CONTEXT

| Tier | Loader | Scope |
|------|--------|-------|
| 1. Built-in | `src/mcp/` `createBuiltinMcps()` | Global, 3 remote HTTP MCPs |
| 2. **Claude Code** | **This module** | **From `.mcp.json`, project + user** |
| 3. Skill-embedded | `src/features/skill-mcp-manager/` | Per-session, from SKILL.md YAML |

## SECURITY

- **Env var allowlist**: `mcp_env_allowlist` config restricts which env vars can be expanded
- **No shell execution**: `${VAR}` is string replacement only, not shell `$()`
- **Secrets redaction**: `env-cleaner.ts` (in skill-mcp-manager) filters known secret patterns from logs

## RELATED

- Phase 5 integration: `src/plugin-handlers/mcp-config-handler.ts`
- Skill-embedded MCPs (Tier 3): `src/features/skill-mcp-manager/`
- Built-in MCPs (Tier 1): `src/mcp/`



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
