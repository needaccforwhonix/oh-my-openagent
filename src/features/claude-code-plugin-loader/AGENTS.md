# src/features/claude-code-plugin-loader/ — Unified Claude Code Plugin Loader

**Generated:** 2026-04-18

## OVERVIEW

16 files. Full Claude Code plugin compatibility layer. Discovers and loads ALL plugin components (commands, agents, skills, hooks, MCP servers, LSP servers) from `.opencode/plugins/` and `~/.claude/plugins/`.

## WHY IT EXISTS

Claude Code plugins ship commands/agents/skills as separate files with `plugin.json` manifest. OmO uses this loader to ingest them into its own registry so existing Claude Code plugins work unchanged under OmO.

## LOAD PIPELINE

```
loadAllPluginComponents(ctx)
  → discoverPlugins()                  # scan .opencode/plugins + ~/.claude/plugins
  → readPluginManifest(plugin.json)    # parse name/version/commands/agents/skills/hooks/mcpServers
  → loadPluginCommands()
  → loadPluginAgents()
  → loadPluginSkills()
  → loadPluginHooks()                  # register hook handlers
  → loadPluginMcpServers()             # feed into mcp-config-handler (tier 2)
  → loadPluginLspServers()
  → return LoadedPluginBundle
```

Called from `src/plugin-handlers/plugin-components-loader.ts` during Phase 2 of config handler (10s timeout with error isolation — one broken plugin does not sink the plugin load).

## KEY FILES

| File | Purpose |
|------|---------|
| `index.ts` | Barrel: `loadAllPluginComponents`, `PluginManifest`, `ClaudeSettings` types |
| `plugin-discovery.ts` | Find plugin directories across scopes |
| `plugin-manifest-parser.ts` | Parse `plugin.json` with Zod validation |
| `command-loader.ts` | Load commands from `commands/` or `COMMANDS.md` |
| `agent-loader.ts` | Load agents from `agents/` or `AGENTS.md` frontmatter |
| `skill-loader.ts` | Load skills from `skills/` or `SKILL.md` |
| `hook-loader.ts` | Load hooks config from `hooks/` or manifest |
| `mcp-loader.ts` | Extract MCP server configs |
| `lsp-loader.ts` | Extract LSP server configs |
| `settings-loader.ts` | Parse Claude Code `settings.json` |

## PLUGIN MANIFEST (plugin.json)

```jsonc
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "...",
  "commands": ["./commands"],       // or string[] of paths
  "agents": ["./agents"],
  "skills": ["./skills"],
  "hooks": "./hooks/config.json",
  "mcpServers": "./.mcp.json",
  "lspServers": "./lsp"
}
```

## SCOPES

| Scope | Path | Priority |
|-------|------|----------|
| `project` | `.opencode/plugins/` | Highest |
| `local` | `~/.opencode/plugins/` | Medium |
| `user` | `~/.claude/plugins/` | Medium |
| `managed` | Built-in | Lowest |

## ERROR ISOLATION

Each plugin loads in isolation — if one fails (bad manifest, missing file, syntax error), others still load. Errors surface as warnings in `bunx oh-my-opencode doctor`.

## RELATED

- Phase 2 loader: `src/plugin-handlers/plugin-components-loader.ts`
- Tier 2 MCP integration: `src/features/claude-code-mcp-loader/`
- Claude Code compat hooks: `src/hooks/claude-code-hooks/`



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
