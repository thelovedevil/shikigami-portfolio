# Shikigami

【 write up 】                                                                                      【 write up 】

https://open.substack.com/pub/aikikokurai/p/my-favorite-game-growing-up-was-hack?r=798iyi&utm_campaign=post&utm_medium=web

【 write up 】                                                                                      【 write up 】

Agentic web application penetration testing ecosystem — Japanese infrastructure specialist.

![Version](https://img.shields.io/badge/version-0.9.0-blue)
![Python](https://img.shields.io/badge/python-3.11%2B-blue)
![Nodes](https://img.shields.io/badge/LangGraph%20nodes-70%2B-orange)
![Stack](https://img.shields.io/badge/stack-LangGraph%20%7C%20llama.cpp%20%7C%20AWS-orange)

---

## What it is

Shikigami is a **multi-pipeline agentic penetration testing ecosystem** built for Japanese web infrastructure. Four LangGraph pipelines — recon, session capture, active exploitation, and findings analysis — share a common state bus, evidence propagation layer, and LLM routing backbone. 70+ graph nodes coordinate parallel execution, iterative self-improvement, and multi-turn tool-calling exploitation loops.

Built for bug bounty programmes on Japanese platforms (IssueHunt.jp, BugBounty.jp) where character encoding edge cases (Shift-JIS, EUC-JP, ISO-2022-JP) create attack surfaces that generic Western scanners miss entirely. All LLM inference runs locally via llama.cpp — no target data leaves the machine.

---

## Track record

Findings submitted to authorised bug bounty programmes on IssueHunt.jp:

| Programme | Finding class | Severity | Status |
|-----------|--------------|----------|--------|
| American financial platform | key exposure leading to uuid deobfuscation and refferal code fraud | Critical | Triaged |
| Japanese financial platform | SQL error disclosure via authentication endpoint | Critical | Triaged |
| Japanese financial platform | Unauthenticated session exposure via WebAuthn flow | Critical | Triaged |
| Japanese financial platform | CORS misconfiguration + reflected XSS chain | Critical | Triaged |
| Japanese financial platform | WAF bypass via multibyte encoding on login | Critical | Triaged |
| Japanese media platform | JWT expiry absent + HMAC key exposure | High | Submitted |
| Japanese media platform | CDN token disclosure via unauthenticated endpoint | Medium | Submitted |
| Japanese media platform | S3 stack trace disclosure | Medium | Submitted |
| Japanese media platform | Compound DRM chain vulnerability | High | Submitted |

All findings produced within authorised scope. Reports include bilingual (EN/JP) write-ups with curl reproduction scripts.

---

## Architecture overview

### Four pipelines, one state bus

Shikigami is not a single scanner — it is four LangGraph pipelines that share a 139-field `ReconState` and an evidence bus for cross-pipeline signal propagation.

```
                              ┌─────────────────────────┐
                              │    shinki (shared lib)   │
                              │  state · LLM · HTTP · JP │
                              └────────┬────────────────┘
          ┌────────────────────────────┼────────────────────────────┐
          ▼                            ▼                            ▼
   ┌──────────────┐           ┌──────────────┐            ┌──────────────┐
   │  Kitsuneki   │           │    Tsuki     │            │   Kurasu     │
   │  22 nodes    │──state──▶ │   6 nodes    │──session──▶│  35 nodes    │
   │  Passive +   │           │  Session     │            │  Active      │
   │  Active Recon│           │  Capture     │            │  Exploitation│
   └──────────────┘           └──────────────┘            └──────┬───────┘
                                                                 │
                                                                 ▼
                                                          ┌──────────────┐
                                                          │   Kagami     │
                                                          │  Analysis +  │
                                                          │  Reporting   │
                                                          └──────────────┘
```

---

### Kitsuneki — passive + active recon (22 nodes)

```
dns_enum → subdomain_enum → reputation_check → cloudflare_enum → origin_bypass
    │
asset_graph → waf_encode_bypass → passive_recon → fetch → baseline_calibration
    │
tech_fingerprint → surface_map → js_render → crawler → sourcemap_extract
    │
js_ast_harvester → js_reverse_engineer → api_discover → path_probe
    │
analyze → continuous_recon → kitsuneki_report
```

Outputs structured recon state: subdomains, resolved IPs, technology fingerprint, JS sinks, API endpoints, WAF detection, encoding anomalies. Hands off to Kurasu via `--state recon_state.json`.

---

### Tsuki — session capture (6 nodes)

```
raw_capture → token_classifier → auth_validator → gap_detector → sso_chain_tracer → session_writer
```

Chrome DevTools Protocol (CDP) extraction of HttpOnly cookies, bearer tokens, and CSRF artifacts that JavaScript cannot access. LLM-based token classifier separates signal from noise. SSO chain tracer maps OAuth/SAML flows. Outputs structured session file consumed by Kurasu's `session_loader`.

---

### Kurasu — active exploitation (35 nodes)

```
session_loader → authenticated_crawler → xss_sink_analyzer → login_probe → active_probe
    │
    ├── auth_bypass     ─┐
    ├── cors_probe       ├──────── PARALLEL PHASE A ─────────────────────┐
    ├── idor_probe       │                                                │
    └── http2_probe     ─┘                                                │
                                                                          ▼
oob_handler → mad_probe → param_fuzz → race_probe → stateful_race_probe
    │
rce_probe → ssrf_pivot_engine → smuggle_probe → upload_polyglot_probe
    │
prototype_pollution → graphql_depth
    │
    ├── jwt_analyze     ─┐
    ├── ws_probe         ├──────── PARALLEL PHASE B ─────────────────────┐
    └── oauth_probe     ─┘                                                │
                                                                          ▼
deep_probe → adaptive_probe ◄──────────────────────────┐
    │                                                    │ (loop up to 5×
    │                                                    │  on no-hit)
adaptive_loop_router                                     │
    │                                                    │
    ├── continue ─────────────────────────────────────────┘
    ├── generate_poc → generate → END
    └── end → END
    │
canary_sweep → logic_analyzer → verify_findings → token_theft_probe
    │
privilege_matrix → raw_findings_logger → chain_assembler
```

Two parallel fan-out phases with `Annotated[T, reducer]` merge logic on shared accumulators. The `adaptive_probe` node runs a multi-turn ReAct tool-calling loop — reads all accumulated evidence, selects and calls tools iteratively (`http_probe`, `fuzz_param`, `playwright_eval`), observes responses, and adapts strategy each turn.

---

### Neurosymbolic pipeline

Two-layer decision system eliminates ~40% of LLM calls:

**Symbolic layer** (~1ms, 100% precision on matched patterns)
- Definite pattern classifiers fire before any LLM call
- Regex/rule-based on unambiguous HTTP artifacts
- Zero cost — pure pattern matching

**Neural layer** (LLM via llama.cpp → Ollama → Bedrock fallback)
- Handles ambiguous evidence the symbolic layer cannot classify
- Adaptive exploitation via native tool-calling
- Three-level JSON parsing with graceful degradation

**Devil's Advocate verifier**
- Second LLM pass attempts to disprove every non-confirmed finding
- High-conviction benign explanations drop the finding; mid-conviction demotes severity
- Confirmed findings (pattern-level evidence) are never touched
- `verify_skipped` audit trail tracks every dropped/demoted finding

---

### Evidence bus

Cross-node typed signals propagate via a shared bus in state, enabling chained exploitation across previously independent findings. Signals include: `trusted_origin`, `jwt_alg_none`, `auth_bypass_path`, `cors_credentialed`, `sink_confirmed`, `chain_confirmed`. The `chain_assembler` node constructs multi-step attack chains from independent bus signals, producing compound CVE candidates with CVSS 4.0 estimates.

---

### Vulnerability coverage

| Category | Nodes | Techniques |
|----------|-------|------------|
| Injection | `active_probe`, `rce_probe`, `param_fuzz` | SQLi, SSTI, CSTI, deserialization, command injection |
| XSS | `xss_sink_analyzer`, `canary_sweep` | DOM sink analysis, MutationObserver-based payload verification, stored XSS via Playwright |
| Access control | `auth_bypass`, `idor_probe`, `privilege_matrix` | Per-endpoint access-control diff across accounts, path-based auth bypass |
| SSRF | `ssrf_pivot_engine` | Internal network mapping, OOB verification, redirect chain following |
| Race conditions | `race_probe`, `stateful_race_probe` | Parallel request races, stateful races with token refresh/resource locks |
| API security | `graphql_depth`, `oauth_probe`, `jwt_analyze`, `ws_probe`, `http2_probe` | GraphQL introspection/batching, OAuth state confusion, JWT algorithm confusion, WebSocket auth, HTTP/2 smuggling |
| File upload | `upload_polyglot_probe` | Polyglot files, dual-format, magic byte manipulation |
| Prototype pollution | `prototype_pollution` | `__proto__` / `constructor` injection |
| CORS | `cors_probe`, `cors_chain_validator` | Credentialed requests, origin reflection, wildcard validation |
| Mass assignment | `mad_probe` | Hidden parameter discovery and mass assignment |
| Request smuggling | `smuggle_probe` | CL.TE, TE.CL, TE.TE desync |
| Token theft | `token_theft_probe` | Token exfiltration via confirmed XSS sinks |

---

## Japanese encoding specialisation

This is the primary differentiator from generic pentest tools.

- **Encoding priority chain**: correct decoding on every response regardless of misconfigured `Content-Type` headers — covers Shift-JIS, CP932, EUC-JP, ISO-2022-JP, UTF-8-SIG
- **Regional encoding expansion**: Big5/HKSCS (Taiwan/Hong Kong), Windows-1252/ISO-8859-1 (German)
- **`Accept-Language: ja`** on every request — triggers Japanese-language error messages that expose internal stack structure unavailable in English responses
- **Keigo analysis**: response politeness level as a signal for privilege tier boundaries in Japanese enterprise applications
- **Full-width digit IDOR**: server-side normalisation bypasses numeric ID validation in JP frameworks
- **Multibyte WAF bypass**: charset confusion between WAF and application layer for filter evasion
- **Mojibake detection**: garbled text as encoding misconfiguration signal for injection surface
- **Payload mutation**: 8+ semantic variants per payload family (SQLi, XSS, path traversal) with JP-specific encoding artifacts

---

## False positive defence

Three specific false-positive vectors discovered during live scanning are defended against explicitly:

- **SSTI arithmetic oracle**: the `{{7*7}}=49` check is gated behind a `ssti_oracle=True` flag — never applied to arbitrary page bodies, only to probe responses where the arithmetic expression was actually injected
- **CloudFront SPA soft-404**: path probe and API discovery fingerprint the CDN's uniform-200 behaviour before scanning (two junk-path requests, body size comparison) and suppress results accordingly
- **X-Original-URL header ignored**: auth bypass records the root-path baseline size before testing rewrite headers — a bypass response within +/-1000 bytes of the homepage is classified as a non-event

Additional defences:
- **Hallucination registry**: LLM false-positive deduplication across nodes prevents the same hallucinated finding from appearing multiple times
- **Empirical flag**: CRITICAL severity requires at least one finding with `empirical=true` (canary fired, race succeeded N/N, OOB callback received)
- **Canary sweep**: MutationObserver-based DOM verification via Playwright confirms XSS before reporting

---

## LLM backend hierarchy

| Priority | Backend | Use case |
|----------|---------|----------|
| 1 | llama.cpp (in-process) | Primary — GBNF grammar-constrained tool-calling, per-node token caps (256-2048) |
| 2 | Ollama (HTTP) | Fallback when llama.cpp unavailable |
| 3 | Amazon Bedrock | Final fallback — Claude Haiku, Tokyo region |
| 4 | Anthropic API | Optional — Sonnet for PoC generation, ephemeral prompt caching |

VRAM budget management prevents mid-inference spill: `nvidia-smi` query → per-layer cost estimation → safe `n_gpu_layers` cap → headroom reservation for KV cache.

---

## Tech stack

| Layer | Technology |
|-------|-----------|
| Workflow orchestration | LangGraph (StateGraph, conditional edges, fan-out/fan-in, `Annotated` reducers) |
| Local LLM inference | llama.cpp (in-process, GGUF) with Ollama fallback |
| LLM fallback | Amazon Bedrock (Tokyo `ap-northeast-1`) |
| HTTP client | httpx + curl_cffi (JA4 fingerprinting, Cloudflare bypass) |
| Browser automation | Playwright (CDP extraction, MutationObserver canary, SPA routing) |
| CVSS scoring | CVSS 4.0 vectors, standalone + chained estimates |
| Reporting | JSON + Markdown, bilingual EN/JP, empirical flag enforcement |
| Testing | pytest — integration suites with zero mocked network |

---

## Design principles

**Privacy by default** — all LLM inference is local. Target responses, vulnerability details, and application behaviour never reach a third-party API unless every local backend is unavailable.

**Precision over recall** — the neurosymbolic gate and Devil's Advocate verifier exist to reduce false positives. A finding that reaches the report has survived pattern classification, empirical verification, and an adversarial disproof attempt.

**Resumability** — 70-node scans take 15-40 minutes on a live target. Checkpoint-based resume means a crash or network interruption loses at most one node of work.

**Encoding correctness first** — every HTTP response is decoded through the full JP encoding chain before any analysis runs. A finding based on a misread Shift-JIS response is worse than no finding.

**Modular pipelines** — each pipeline (Kitsuneki, Tsuki, Kurasu, Kagami) runs independently or chains via state handoff. New nodes slot into the graph without touching existing node logic.

---

## Status

Private repository — available for review during hiring or engagement discussions.
