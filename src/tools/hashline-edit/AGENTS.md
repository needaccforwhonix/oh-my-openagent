# src/tools/hashline-edit/ — Hash-Anchored File Edit Tool

**Generated:** 2026-04-11

## OVERVIEW

24 files. Implements the `hashline_edit` tool — hash-anchored file editing where every line reference includes a content hash (`LINE#ID`). Validates hashes before applying edits, rejecting stale references.

## THREE-OP MODEL

All edits use exactly 3 operations:

| Op | pos | end | lines | Effect |
|----|-----|-----|-------|--------|
| `replace` | required | optional | required | Replace single line or range pos..end |
| `append` | optional | optional | required | Insert after anchor (or EOF if no anchor) |
| `prepend` | optional | optional | required | Insert before anchor (or BOF if no anchor) |

`lines: null` or `lines: []` with `replace` = delete. `delete: true` at tool level = delete file.

## EXECUTION PIPELINE

```
hashline-edit-executor.ts
  → normalize-edits.ts       # Parse RawHashlineEdit → HashlineEdit (validate op schema)
  → validation.ts            # Validate LINE#ID references (hash match, line exists)
  → edit-ordering.ts         # Sort bottom-up (by line number, descending)
  → edit-deduplication.ts    # Remove duplicate ops
  → edit-operations.ts       # Apply each op using edit-operation-primitives.ts
  → autocorrect-replacement-lines.ts  # Auto-fix indentation/formatting
  → hashline-edit-diff.ts    # Build diff output using diff-utils.ts
```

## KEY FILES

| File | Purpose |
|------|---------|
| `tools.ts` | `createHashlineEditTool()` factory — tool schema + entry point |
| `hashline-edit-executor.ts` | Main execution: normalize → validate → order → apply → diff |
| `normalize-edits.ts` | Parse `RawHashlineEdit[]` (allows string `op` variants) → typed `HashlineEdit[]` |
| `validation.ts` | Validate LINE#ID: parse hash, verify line content matches stored hash |
| `hash-computation.ts` | `computeLineHash(line)` → 2-char CID from set `ZPMQVRWSNKTXJBYH` |
| `edit-operations.ts` | Apply replace/append/prepend to file lines array |
| `edit-operation-primitives.ts` | Low-level line array mutation primitives |
| `edit-ordering.ts` | Sort edits bottom-up to preserve line numbers during multi-edit |
| `edit-deduplication.ts` | Deduplicate overlapping/identical operations |
| `edit-text-normalization.ts` | Normalize line content (CRLF, BOM, trailing whitespace) |
| `file-text-canonicalization.ts` | Canonicalize full file content before hashing |
| `autocorrect-replacement-lines.ts` | Auto-restore indentation from original lines |
| `hashline-edit-diff.ts` | Generate unified diff for error/success messages |
| `diff-utils.ts` | Thin wrapper around `diff` npm library |
| `hashline-chunk-formatter.ts` | Format line chunks with `LINE#ID` tags |
| `tool-description.ts` | `HASHLINE_EDIT_DESCRIPTION` constant |
| `types.ts` | `HashlineEdit`, `ReplaceEdit`, `AppendEdit`, `PrependEdit` |
| `constants.ts` | Hash alphabet, separator character (`#`), pipe separator (`|`) |

## LINE#ID FORMAT

```
{line_number}#{hash_id}
```

- `hash_id`: two chars from `ZPMQVRWSNKTXJBYH` (CID letters)
- Example: `42#VK` means line 42 with hash `VK`
- Validation: recompute hash of current line content → must match stored hash
- Content separator: `|` (pipe) between hash tag and content in read output

## AUTOCORRECT BEHAVIORS (built-in)

- Merged lines auto-expanded back to original count
- Indentation restored from original lines
- BOM and CRLF line endings preserved
- `>>>` prefix and diff markers in `lines` text auto-stripped

## ERROR CASES

- Hash mismatch → edit rejected, diff shown with current state
- Overlapping ranges → detected and rejected
- Missing `pos` for `replace` → schema error
- `lines: null` with `append`/`prepend` → schema error

## HOW LINE HASHES WORK

```typescript
// Reading: every line gets tagged
"42#VK| function hello() {"

// Editing: reference by tag
{ op: "replace", pos: "42#VK", lines: "function hello(name: string) {" }

// If file changed since read: hash won't match → rejected before corruption
```



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
