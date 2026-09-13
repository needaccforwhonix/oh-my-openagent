# src/features/mcp-oauth/ — OAuth 2.0 + PKCE + DCR for MCP Servers

**Generated:** 2026-04-11

## OVERVIEW

18 files. Full OAuth 2.0 authorization flow for MCP servers requiring authentication. Implements PKCE (RFC 7636), Dynamic Client Registration (DCR, RFC 7591), and resource indicators (RFC 8707). Used by `bunx oh-my-opencode mcp-oauth login`.

## AUTHORIZATION FLOW

```
1. discovery.ts → fetch /.well-known/oauth-authorization-server
2. dcr.ts → Dynamic Client Registration (if server supports it)
3. oauth-authorization-flow.ts → generate PKCE verifier/challenge
4. callback-server.ts → local HTTP server on random port for redirect
5. Open browser → authorization URL
6. callback-server.ts → receive code + state
7. provider.ts → exchange code for token (with PKCE verifier)
8. storage.ts → persist token to ~/.config/opencode/mcp-oauth/
9. step-up.ts → handle step-up auth if initial token insufficient
```

## KEY FILES

| File | Purpose |
|------|---------|
| `oauth-authorization-flow.ts` | PKCE helpers: `generateCodeVerifier()`, `generateCodeChallenge()`, `buildAuthorizationUrl()` |
| `callback-server.ts` | Local HTTP redirect server — listens for OAuth callback |
| `provider.ts` | `OAuthProvider` — token exchange, refresh, revoke |
| `discovery.ts` | Fetch + parse OAuth server metadata from well-known endpoint |
| `dcr.ts` | Dynamic Client Registration — register this app with OAuth server |
| `resource-indicator.ts` | RFC 8707 resource indicator handling |
| `step-up.ts` | Handle step-up authentication challenges |
| `storage.ts` | Persist tokens to `~/.config/opencode/mcp-oauth/{server-hash}.json` |
| `schema.ts` | Zod schemas for OAuth server metadata, token response, DCR |

## PKCE IMPLEMENTATION

- Code verifier: 32 random bytes → base64url (no padding)
- Code challenge: SHA-256(verifier) → base64url
- Method: `S256`

## TOKEN STORAGE

Location: `~/.config/opencode/mcp-oauth/` — one JSON file per MCP server (keyed by server URL hash).
Fields: `access_token`, `refresh_token`, `expires_at`, `client_id`.

## CLI COMMANDS

```bash
bunx oh-my-opencode mcp-oauth login <server-url>   # Full PKCE flow
bunx oh-my-opencode mcp-oauth logout <server-url>  # Revoke + delete token
bunx oh-my-opencode mcp-oauth status               # List stored tokens
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
