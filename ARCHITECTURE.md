# Architecture — Shikigami v0.9.0

## Multi-pipeline topology (70+ nodes)

Shikigami is four LangGraph pipelines sharing one `ReconState` (139 fields). Each pipeline is a separate `StateGraph` compiled independently; they chain via state serialisation (`--state recon_state.json`), not import-time coupling.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        shinki (shared library)                          │
│  state.py · llm_router.py · http_client.py · jp_encoding.py · react.py │
│  symbolic_rules.py · finding_bus.py · hallucination_registry.py         │
│  checkpoint.py · payload_mutator.py · polyglot_files.py                 │
└──────────────────────────────────────────────────────────────────────────┘
         │                    │                    │                │
         ▼                    ▼                    ▼                ▼
   ┌───────────┐       ┌───────────┐        ┌───────────┐   ┌───────────┐
   │ Kitsuneki │       │   Tsuki   │        │  Kurasu   │   │  Kagami   │
   │ 22 nodes  │──────▶│  6 nodes  │───────▶│ 35 nodes  │──▶│ Analysis  │
   │   Recon   │ state │  Session  │ session│  Exploit  │   │ Reporting │
   └───────────┘       └───────────┘        └───────────┘   └───────────┘
```

---

## Kitsuneki — recon pipeline (22 nodes)

```
dns_enum                DNS A/AAAA/CNAME/MX/TXT + zone transfer attempt
    │
subdomain_enum          CT log query, DNS wordlist brute, permutation
    │
reputation_check        IP reputation, ASN mapping, abuse contact
    │
cloudflare_enum         CF detection, WAF fingerprint, cfuvid pool init
    │
origin_bypass           Direct-IP discovery behind CDN (certificate, DNS history)
    │
asset_graph             Map relationships: domains ↔ IPs ↔ ASNs ↔ certificates
    │
waf_encode_bypass       Charset confusion probes against detected WAF
    │
passive_recon           robots.txt, sitemap.xml, security.txt, .well-known
    │
fetch                   Landing page, raw headers, encoding detection
    │
baseline_calibration    Soft-404 fingerprint, CDN uniform-200 detection
    │
tech_fingerprint        Server headers, meta tags, known framework patterns
    │
surface_map             HTML form/link/parameter discovery, hidden inputs
    │
js_render               Playwright DOM rendering, XHR/fetch interception,
    │                   WebSocket upgrade detection, SPA route extraction
crawler                 Follow internal links to depth N, scope enforcement
    │
sourcemap_extract       .map file discovery and source reconstruction
    │
js_ast_harvester        DOM sink extraction (innerHTML, eval, script.src)
    │
js_reverse_engineer     Webpack/Vite bundle reverse, API key extraction
    │
api_discover            Swagger/OpenAPI/GraphQL introspection, endpoint enum
    │
path_probe              JP-specific sensitive paths, admin panels, debug endpoints
    │
analyze                 Symbolic + neural classification of all collected signals
    │
continuous_recon        Delta detection: new subdomains, removed paths, changed headers
    │
kitsuneki_report        Serialise ReconState → JSON for Kurasu handoff
```

**Output**: `outputs/<target>/<scan_id>/recon_state.json` — consumed by `kurasu --state`.

---

## Tsuki — session capture pipeline (6 nodes)

```
raw_capture             Chrome DevTools Protocol — HttpOnly cookies, bearer tokens,
    │                   CSRF artifacts that JavaScript cannot access
token_classifier        LLM-based signal vs noise (session vs analytics cookies)
    │
auth_validator          Confirm session authenticity against authenticated endpoint
    │
gap_detector            Flag expected tokens missing from capture
    │
sso_chain_tracer        Map OAuth/SAML redirect chains, extract intermediate tokens
    │
session_writer          Structured session file → Kurasu session_loader
```

**Why CDP**: `document.cookie` cannot read `HttpOnly` flags. CDP's `Network.getAllCookies` sees everything the browser stores, including tokens set by redirect chains that never touch JS.

---

## Kurasu — exploitation pipeline (35 nodes)

```
session_loader          Load Tsuki session or manual cookie/header config
    │
authenticated_crawler   Crawl with active session — discover authed-only surfaces
    │
xss_sink_analyzer       DOM sink severity scoring (innerHTML=high, textContent=low)
    │
login_probe             JP default credentials, Basic Auth, form-based auth
    │
active_probe            Multi-technique injection suite (SQLi, XSS, path traversal)
    │
    ├── auth_bypass      ─┐
    ├── cors_probe        │
    │   └── cors_chain    ├──────── PARALLEL PHASE A ──────────────────┐
    ├── idor_probe        │                                             │
    └── http2_probe      ─┘                                             │
                                                                        ▼
oob_handler             Poll DynamoDB for blind callback confirmations
    │
mad_probe               Mass Assignment Discovery — hidden param injection
    │
param_fuzz              Parameter brute-force + type confusion
    │
race_probe              Parallel request race conditions
    │
stateful_race_probe     Races with token refresh / resource lock tracking
    │
rce_probe               SSTI, CSTI, deserialization, command injection
    │
ssrf_pivot_engine       Internal network mapping via confirmed SSRF
    │
smuggle_probe           CL.TE, TE.CL, TE.TE request desync
    │
upload_polyglot_probe   Dual-format file upload (GIF/JS, PDF/HTML, etc.)
    │
prototype_pollution     __proto__ / constructor.prototype injection
    │
graphql_depth           Introspection, depth/batch/alias abuse
    │
    ├── jwt_analyze      ─┐
    ├── ws_probe          ├──────── PARALLEL PHASE B ──────────────────┐
    └── oauth_probe       │                                             │
        └── oauth_state  ─┘                                             │
                                                                        ▼
deep_probe              Combine signals from both parallel phases
    │
adaptive_probe  ◄───────────────────────────────────────┐
    │                                                    │  loop up to 5×
adaptive_loop_router                                     │  on no-hit
    │                                                    │
    ├── continue ────────────────────────────────────────┘
    ├── generate_poc → generate → END
    └── end → END
    │
canary_sweep            MutationObserver-based XSS verification (Playwright)
    │
logic_analyzer          Business logic flaw detection
    │
verify_findings         Devil's Advocate adversarial disproof
    │
token_theft_probe       Token exfiltration via confirmed XSS sinks
    │
privilege_matrix        Per-endpoint access-control diff across 2+ accounts
    │
raw_findings_logger     Write individual finding JSON files
    │
chain_assembler         Multi-step attack chain construction, CVSS 4.0 estimation
```

---

## State contract

`ReconState` is a single `TypedDict` (139 fields) passed through every node. Nodes return partial dicts — LangGraph merges them with the previous checkpoint. Returning the full state from a parallel node causes `"Can receive only one value per step"` errors.

### Critical fields

| Field | Type | Purpose |
|-------|------|---------|
| `evidence_bus` | `List[Dict]` | Cross-node typed signals (`trusted_origin`, `jwt_alg_none`, `auth_bypass_path`, `cors_credentialed`, `sink_confirmed`) |
| `findings` | `List[Finding]` | Accumulated findings across all nodes — `Annotated` reducer deduplicates by `(vuln_type, description)` |
| `is_vulnerable` | `bool` | `Annotated[bool, _or_bool]` — any node setting True propagates |
| `verify_skipped` | `List[Dict]` | Audit trail of findings dropped/demoted by Devil's Advocate |
| `hallucination_registry` | `Dict` | LLM false-positive deduplication across nodes |
| `adaptive_iterations` | `int` | Loop counter for adaptive_probe (max 5) |
| `cve_candidates` | `List[Dict]` | Multi-step chain candidates with `chain_id`, `severity`, `path`, `components`, `narrative`, `cvss_estimate` |
| `canary_map` | `Dict` | Canary token → injection point mapping for MutationObserver verification |
| `api_rate_profiles` | `Dict[str, float]` | Per-path-prefix rate limiting delays |

### Finding schema

```
vuln_type:   sqli | xss | info_disclosure | encoding | path_traversal |
             broken_access | logic | oob | other
severity:    critical | high | medium | low | info
confidence:  confirmed | likely | possible
empirical:   bool   # True = exploitation path actually exercised
```

`empirical=True` requires: canary fired, race succeeded N/N, OOB callback received, or PoC generated and verified. CRITICAL severity is blocked unless at least one finding carries `empirical=True`.

### `state.py` design rule

`state.py` has **zero internal imports**. It is the base dependency for every node — internal imports here create circular dependency chains. All reducer functions (`_merge_findings`, `_or_bool`, `_merge_bus`) live in this file.

---

## Parallel execution and reducers

Each parallel node writes to its own exclusive output key (`rce_findings`, `idor_hits`, `auth_bypass_hits`, `cors_findings`, `jwt_findings`, `oauth_findings`, `ws_findings`). Three shared accumulators carry `Annotated[T, reducer]` type annotations:

| Accumulator | Reducer | Behaviour |
|-------------|---------|-----------|
| `findings` | `_merge_findings` | Dedup by `(vuln_type, description)` tuple |
| `is_vulnerable` | `_or_bool` | OR — any True wins |
| `evidence_bus` | `_merge_bus` | Dedup bus signals by `(signal_type, source)` |

Reducer logic lives entirely in `state.py`. Nodes are unaware of parallel topology — they append to shared keys without coordination.

---

## Neurosymbolic decision pipeline

### Symbolic layer (~1ms, 100% precision)

Pattern classifiers in `symbolic_rules.py` fire before any LLM call. Matched patterns produce `confirmed` findings directly. Unmatched evidence passes to the neural layer.

Eliminates ~40% of LLM invocations. On an RTX 4070 where each LLM call costs 2–8 seconds, this is the difference between a 20-minute and a 35-minute scan.

### Neural layer (LLM)

Handles ambiguous evidence. Three-level JSON parsing with graceful degradation:
1. Structured output via `with_structured_output()` (Pydantic schema)
2. Regex extraction of JSON block from completion text
3. Keyword scan fallback (field extraction from freeform text)

### Devil's Advocate verifier

Second LLM pass attempts to disprove every non-confirmed finding. `confirmed` findings (pattern-level, canary-level) are never touched — only `likely` and `possible` pass through adversarial review. Rejections and demotions are recorded in `verify_skipped` for audit.

---

## ReAct exploitation agent

The `adaptive_probe` node runs a multi-turn tool-calling loop:

```
1. Read all accumulated evidence (evidence_bus + findings + prior adaptive_results)
2. Select and call a tool:  http_probe | fuzz_param | playwright_eval
3. Observe response
4. Decide: call another tool, or report_confirmed
5. Repeat (capped by token budget per turn)
```

The graph loops the node up to 5 times via `adaptive_loop_router`. Each iteration provides fresh evidence context. The agent's tool calls are real HTTP requests — not simulated.

---

## Encoding priority chain

```
utf-8-sig → shift_jis → cp932 → euc-jp → iso-2022-jp → utf-8
```

`cp932` (Windows Shift-JIS extension) is tried before `euc-jp` because some byte sequences overlap. `utf-8` is tried last — many Shift-JIS byte sequences are technically valid UTF-8 but decode to mojibake. Trying UTF-8 first produces false confidence on legacy JP systems.

Regional extensions: Big5/HKSCS (Taiwan/Hong Kong), Windows-1252/ISO-8859-1 (German).

All charset logic is owned by `jp_encoding.py` and `regional_encoding.py`. No node imports charset detection directly.

---

## HTTP client stack

Two HTTP backends, selected per-request:

| Backend | When | Why |
|---------|------|-----|
| `httpx` | Default | Async-capable, encoding middleware, connection pooling |
| `curl_cffi` | Cloudflare-protected targets | JA4 TLS fingerprinting — browser-grade fingerprint avoids CF bot detection |

`multibyte_safe_transport.py` wraps both backends to guarantee correct encoding handling across the transport layer.

`cfuvid_pool` maintains a rotating pool of Cloudflare challenge-bypass tokens (`cf_clearance`). Playwright user-agent is pinned to match the JA4 fingerprint for session consistency.

---

## Payload mutation

`payload_mutator.py` generates 8+ semantic variants per payload across four families:

| Family | Techniques |
|--------|-----------|
| SQLi | Case mutation, comment injection (`/**/`), keyword obfuscation, hex literals, keyword splitting |
| XSS | Tag evasion, attribute breaking, event handler mutation, encoding artifacts |
| Path traversal | Null bytes, URL encoding, double encoding, unicode normalisation |
| Generic | NBSP/newline injection, charset confusion |

Mutations compose — a single injection point tests `original × N mutations × M encodings`.

---

## OOB infrastructure

```
Probe (shikigami)
    │
    └── HTTP/DNS callback ──► API Gateway ──► Lambda
                                                 │
                                            DynamoDB (NetNavi_OOB_Hits)
                                                 │
                                  oob_handler polls every 5s (timeout: 45s)
```

**Schema**: `TargetID` (hash), `TrackingID` (range), `source_ip`, `hit_at`, `request_path`, `is_japanese_ip`.

Japanese IP callbacks are flagged critical — they confirm server-side execution on JP infrastructure, the highest-value finding category on JP bug bounty platforms.

---

## False positive defences

| Defence | Mechanism |
|---------|-----------|
| SSTI arithmetic oracle | `{{7*7}}→49` gated behind `ssti_oracle=True` — only applied where the expression was actually injected |
| CloudFront soft-404 | `baseline_calibration` sends two junk-path requests, compares body size — suppresses uniform-200 targets |
| X-Original-URL bypass | Root-path baseline size recorded; bypass responses within ±1000 bytes classified as non-events |
| Hallucination registry | Cross-node deduplication prevents the same LLM-hallucinated finding from surfacing multiple times |
| Empirical gate | CRITICAL severity requires `empirical=True` on at least one finding |
| Canary verification | MutationObserver via Playwright confirms DOM modification before reporting XSS |

---

## LLM backend routing

```
llm_router.py
    │
    ├── 1. llama.cpp (in-process GGUF)
    │      GBNF grammar-constrained tool-calling
    │      Per-node token caps: 256–2048
    │      KV cache sharing across calls
    │
    ├── 2. Ollama (HTTP localhost:11434)
    │      Health-check gated: /api/tags ping before every call
    │      Transparent fallback on ConnectionRefusedError
    │
    ├── 3. Amazon Bedrock (ap-northeast-1)
    │      Claude Haiku — catches mid-call TimeoutError
    │
    └── 4. Anthropic API (optional)
           Sonnet for PoC generation
           Ephemeral prompt caching (~5min TTL, ~10% input cost)
```

`--no-llm` runs the full pipeline in symbolic-only mode (no GPU required).

---

## Checkpoint and resume

ask on interview 

---

## Rate limiting

1.0–1.5s jitter between probes. `api_rate_profiles` stores per-path-prefix delays learned during the scan — endpoints that return 429 get longer delays automatically.

Matches bug bounty platform etiquette and avoids rate-limit bans on shared infrastructure.

---

## Key design decisions

ask on interview 
