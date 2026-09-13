# src/hooks/rules-injector/ — Conditional Rules Injection

**Generated:** 2026-04-11

## OVERVIEW

19 files (~1604 LOC). The `rulesInjectorHook` — Tool Guard Tier hook that auto-injects AGENTS.md (and similar rule files) into context when a file in a directory is read, written, or edited. Proximity-based: closest rule file to the target path wins.

## HOW IT WORKS

```
tool.execute.after (read/write/edit/multiedit)
  → Extract file path from tool output
  → Find rule files near that path (finder.ts)
  → Already injected this session? (cache.ts)
  → Inject rule content into tool output (injector.ts)
```

## TRACKED TOOLS

`["read", "write", "edit", "multiedit"]` — triggers only on file manipulation tools.

## KEY FILES

| File | Purpose |
|------|---------|
| `hook.ts` | `createRulesInjectorHook()` — wires cache + injector, handles tool events |
| `injector.ts` | `createRuleInjectionProcessor()` — orchestrates find → cache → inject |
| `finder.ts` | `findRuleFiles()` + `calculateDistance()` — locate AGENTS.md near target path |
| `rule-file-finder.ts` | Walk directory tree to find AGENTS.md / .rules files |
| `rule-file-scanner.ts` | Scan for rule files in a directory |
| `matcher.ts` | Match file paths against rule file scope |
| `rule-distance.ts` | Calculate path distance between file and rule file |
| `project-root-finder.ts` | Find project root (stops at .git, package.json) |
| `output-path.ts` | Extract file paths from tool output text |
| `cache.ts` | `createSessionCacheStore()` — per-session injection dedup |
| `storage.ts` | Persist injected paths across tool calls |
| `parser.ts` | Parse rule file content |
| `constants.ts` | Rule file names: `AGENTS.md`, `.rules`, `CLAUDE.md` |
| `types.ts` | `RuleFile`, `InjectionResult`, `RuleFileScope` |

## RULE FILE DISCOVERY

Priority (closest → farthest from target file):
1. Same directory as target file
2. Parent directories up to project root
3. Project root itself

Same-distance tie: all injected. Per-session dedup prevents re-injection.

## TRUNCATION

Uses `DynamicTruncator` — adapts injection size based on model context window (1M context models get full content, smaller models get truncated summaries).



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
