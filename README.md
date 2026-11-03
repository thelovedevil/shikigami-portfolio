# Shikigami — Agentic Penetration Testing Ecosystem



**Ranked #3 on bitbank Inc. Bug Bounty Program (IssueHunt Japan) within 2 months using this tooling.**

![Version](https://img.shields.io/badge/version-1.0.0-blue)
![Python](https://img.shields.io/badge/python-3.11%2B-blue)
![LOC](https://img.shields.io/badge/LOC-34%2C000%2B-brightgreen)
![Nodes](https://img.shields.io/badge/LangGraph%20nodes-78%2B-orange)
![RAG](https://img.shields.io/badge/RAG%20vectors-20%2C158-purple)
![Stack](https://img.shields.io/badge/stack-LangGraph%20%7C%20llama.cpp%20%7C%20Burp%20Suite-orange)

---

## What it is

Shikigami is a **multi-pipeline agentic penetration testing ecosystem** — 34,000+ lines of Python and C++ across 128 files. Four LangGraph pipelines — recon, session capture, active exploitation, and findings analysis — share a common state bus, evidence propagation layer, and LLM routing backbone. 78+ graph nodes coordinate parallel execution, iterative self-improvement, and multi-turn tool-calling exploitation loops.

Built for bug bounty programmes on Japanese platforms (IssueHunt.jp, BugBounty.jp) where character encoding edge cases (Shift-JIS, EUC-JP, ISO-2022-JP) create attack surfaces that generic Western scanners miss entirely. All LLM inference runs locally via llama.cpp — no target data leaves the machine.

**Key results:**
- **#3 ranked researcher** on bitbank Inc. (Japan's largest crypto exchange) within 2 months
- **8 confirmed vulnerabilities** (5 Critical, 2 High) across authorised programmes
- **61% false positive reduction** via dual-LoRA fine-tuned triage model
- **~95% Cloudflare bypass rate** via unified CDN/WAF engine

---

## Track record

Findings submitted to authorised bug bounty programmes on IssueHunt.jp:

| Programme | Finding class | Severity | Status |
|-----------|--------------|----------|--------|
| American financial platform | key exposure leading to uuid deobfuscation and refferal code fraud | Critical | resolved |
| Japanese financial platform | SQL error disclosure via authentication endpoint | Critical | resolved |
| Japanese financial platform | Unauthenticated session exposure via WebAuthn flow | Critical | resolved |
| Japanese financial platform | CORS misconfiguration + reflected XSS chain | Critical | resolved |
| Japanese financial platform | WAF bypass via multibyte encoding on login | Critical | resolved |
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

### Tsuki — session capture + browser exploitation (9 nodes)

```
raw_capture → token_classifier → auth_validator → gap_detector → sso_chain_tracer → session_writer
                                                                                      │
web3_auth → metamask_setup ──────────────────────────────────────────────────────────────┘
```

Chrome DevTools Protocol (CDP) extraction of HttpOnly cookies, bearer tokens, and CSRF artifacts that JavaScript cannot access. LLM-based token classifier separates signal from noise. SSO chain tracer maps OAuth/SAML flows. Outputs structured session file consumed by Kurasu's `session_loader`.

Playwright browser-based exploitation backend for targets behind WAF fingerprinting (Cloudflare, Akamai). Auto-escalation: curl_cffi → Playwright on repeated 403s. CF clearance harvester with JA4 pin for session consistency. Web3/DeFi authentication via MetaMask Playwright integration for authenticated DeFi target scanning.

---

### Kurasu — active exploitation (40 nodes)

```
session_loader → authenticated_crawler → xss_sink_analyzer → login_probe → active_probe
    │
    ├── auth_bypass     ─┐
    ├── cors_probe       ├──────── PARALLEL PHASE A ─────────────────────┐
    ├── idor_probe       │                                                │
    ├── http2_probe      │                                                │
    ├── sqli_probe       │                                                │
    └── port_scan       ─┘                                                │
                                                                          ▼
oob_handler → mad_probe → param_fuzz → race_probe → stateful_race_probe
    │
rce_probe → ssrf_pivot_engine → smuggle_probe → upload_polyglot_probe
    │
prototype_pollution → graphql_depth → reflection_analyzer
    │
    ├── jwt_analyze     ─┐
    ├── ws_probe         ├──────── PARALLEL PHASE B ─────────────────────┐
    ├── oauth_probe      │                                                │
    └── context_detector─┘                                                │
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

**C++ HTTP/2 single-packet race engine** — native binary for microsecond-precision race conditions via HTTP/2 multiplexing. Multiple requests packed into a single TCP packet to eliminate network jitter. Called from `race_probe` and `stateful_race_probe`.

---

### Neurosymbolic pipeline

Two-layer decision system eliminates ~40% of LLM calls:

**Symbolic layer** (~1ms, 100% precision on matched patterns)
- Definite pattern classifiers fire before any LLM call
- Regex/rule-based on unambiguous HTTP artifacts
- Zero cost — pure pattern matching

**Neural layer** (LLM via llama.cpp, RAG-augmented)
- Handles ambiguous evidence the symbolic layer cannot classify
- RAG retrieval from vector corpus of historical findings enriches analysis context
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

### CDN/WAF bypass engine

Unified CDN detection and per-CDN exploitation strategy engine. Auto-detects 7 CDN/WAF providers and applies provider-specific bypass logic:

| Provider | Strategy | Detection |
|----------|----------|-----------|
| Cloudflare | curl_cffi JA4 fingerprint → Playwright escalation on 403 | `cf-ray`, `__cf_bm`, server header |
| Akamai | Playwright-first (browser before curl_cffi) | `akamai-x-cache`, `x-akamai-session-info` |
| DataDome | Playwright with sensor bypass | `datadome` cookie, `dd_s` |
| PerimeterX | Session token rotation | `_px3`, `_pxvid` |
| Imperva/Incapsula | Cookie challenge solver | `incap_ses_`, `visid_incap_` |
| Sucuri | Direct-origin fallback | `x-sucuri-id` |
| Fastly | Standard with cache-key manipulation | `x-served-by`, `fastly-` headers |

Anti-spoof safeguards: minimum confidence threshold of 3 independent signals before CDN classification. Session-blocked short-circuit prevents wasting requests against burned IPs.

---

### Vulnerability coverage

| Category | Nodes | Techniques |
|----------|-------|------------|
| Injection | `active_probe`, `rce_probe`, `sqli_probe`, `param_fuzz` | SQLi (error/blind/UNION), SSTI, CSTI, deserialization, command injection |
| XSS | `xss_sink_analyzer`, `canary_sweep`, `reflection_analyzer` | DOM sink analysis, MutationObserver-based payload verification, stored XSS via Playwright, mXSS payloads, reflection context analysis |
| Access control | `auth_bypass`, `idor_probe`, `privilege_matrix` | Per-endpoint access-control diff across accounts, path-based auth bypass, Autorize-style session replay |
| SSRF | `ssrf_pivot_engine` | Internal network mapping, OOB verification, redirect chain following, cloud metadata probing |
| Race conditions | `race_probe`, `stateful_race_probe` | C++ HTTP/2 single-packet races, stateful races with token refresh/resource locks |
| API security | `graphql_depth`, `oauth_probe`, `jwt_analyze`, `ws_probe`, `http2_probe` | GraphQL introspection/batching, OAuth state confusion, JWT algorithm confusion, WebSocket auth, HTTP/2 smuggling |
| Recon | `js_ast_harvester`, `js_reverse_engineer`, `port_scan` | JS secret scanning with validation, LinkFinder endpoint extraction, Wayback parameter mining, Wappalyzer tech detection, port scanning |
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
| 1 | llama-server (local, fine-tuned LoRA) | Primary — grammar-constrained tool-calling, per-node token caps (256-2048), RAG-augmented |
| 2 | Amazon Bedrock | Fallback — Claude Haiku, Tokyo region |
| 3 | Anthropic API | Optional — strategic reasoning escalation, PoC generation |

VRAM budget management prevents mid-inference spill: `nvidia-smi` query → per-layer cost estimation → safe `n_gpu_layers` cap → headroom reservation for KV cache.

---

## Tech stack

| Layer | Technology |
|-------|-----------|
| Workflow orchestration | LangGraph (StateGraph, conditional edges, fan-out/fan-in, `Annotated` reducers) |
| Local LLM inference | llama.cpp (in-process, GGUF) with Ollama fallback |
| LLM fallback | Amazon Bedrock (Tokyo `ap-northeast-1`) |
| HTTP client | httpx + curl_cffi (JA4 fingerprinting) + Playwright (WAF bypass escalation) |
| Browser automation | Playwright (CDP extraction, MutationObserver canary, SPA routing, WAF bypass) |
| Native binary | C++ HTTP/2 single-packet race engine |
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

**Aesop's fable** — follows legitimate philosophies of onmyodo and western fables to integrate the narrative of fox and crow in east-western syncretism. The fox gets the crow to speak.

---

## Kagami — RAG + QLoRA Training Pipeline

The fifth pipeline — a retrieval-augmented generation layer and fine-tuning infrastructure that powers Shikigami's LLM-driven triage.

### RAG knowledge base

| Metric | Value |
|--------|-------|
| Total vectors | 20,158 |
| Domains | Web2 exploitation + Web3/DeFi security |
| Embedding model | nomic-embed (local, 768-dim) |
| Vector store | LanceDB (local, zero-config) |
| Retrieval | Semantic similarity with domain-weighted scoring |

Corpus spans: OWASP methodologies, disclosed bug bounty reports, smart contract vulnerability patterns, WAF bypass techniques, Japanese-infrastructure-specific attack surfaces, and historical scan findings.

### QLoRA fine-tuning

Three iterations of adapter training on security-domain data:

| Iteration | Training examples | Loss | Purpose |
|-----------|------------------|------|---------|
| iter1 | Baseline corpus | — | Foundation |
| iter2 | 16,315 refined examples | 0.265 | Vulnerability classification + triage |
| iter3 (persona) | 492 examples | — | Communication style + report framing |

**Dual-LoRA production deployment:** Security adapter (0.85 weight) + persona adapter (0.15 weight) loaded simultaneously via llama.cpp `--lora-scaled`. Achieved **61% false positive reduction** on Anthropic scan targets vs single-adapter baseline.

### Training data pipeline

Automated extraction from scan outputs to training-ready JSONL:
- `unified_lora_extractor.py` — converts raw findings, LLM interactions, and verification outcomes into instruction-tuning format
- Value sanitisation: PII, credentials, and target-specific data replaced with placeholders before training
- `verify.py` — schema validation ensuring training data integrity
- 17 vulnerability types across 84 targets in production training set

---

## FindingBus — Cross-Pipeline Event Architecture

SQLite-backed event bus enabling real-time communication between all pipelines.

```
Kitsuneki publishes:        Kurasu subscribes:
  tech_detected       ──▶     selects relevant exploit nodes
  param_discovered    ──▶     prioritises parameter fuzzing
  cdn_identified      ──▶     applies CDN-specific bypass

Kurasu publishes:           Kagami subscribes:
  sink_confirmed      ──▶     includes in analysis
  chain_confirmed     ──▶     generates report section
  vuln_confirmed      ──▶     triggers verification pass
```

Persistent storage at `~/.shinki/finding_bus.db` — survives process restarts. Kurasu can start cold and still see all events Kitsuneki published in prior runs.

---

## Burp Suite Pro Integration

Six integration points between Shikigami and Burp Suite Professional:

| Integration | Direction | Purpose |
|-------------|-----------|---------|
| `--burp-proxy` flag | Shikigami → Burp | Route all scan traffic through Burp for passive analysis |
| Finding import | Burp → Shikigami | Import Burp Scanner findings into FindingBus for LLM triage |
| Session sharing | Tsuki → Burp | Export captured sessions as Burp cookie jar |
| Scope sync | Burp → Shikigami | Import Burp target scope as scan scope |
| XSS confirmation | Shikigami → Burp | Send XSS candidates to Burp Repeater for manual verification |
| Turbo Intruder | Shikigami → Burp | Delegate high-volume fuzzing to Turbo Intruder |

20+ Burp extensions configured: Autorize, Hackvertor, Turbo Intruder, Active Scan++, JSON Beautifier, JWT Editor, and others.

---

## Infrastructure

Deployed on a hardened Gentoo Linux workstation:

| Component | Detail |
|-----------|--------|
| OS | Custom Gentoo (source-compiled, PREEMPT_RT kernel) |
| GPU | NVIDIA RTX 4070 — 24 GPU layers for 12B model inference |
| MAC | TOMOYO Linux mandatory access control in learning mode |
| Network | pfSense VLAN segmentation with fail-closed VPN kill-switch |
| Encryption | LUKS2/LVM full disk encryption |
| LLM runtime | llama-server (Gemma 4 12B Q4_K_M), 8192 context window |
| Embedding | nomic-embed on dedicated port (8082) |

---

## Status

Private repository — available for review during hiring or engagement discussions.

Private repository — available for review during hiring or engagement discussions.

<img width="1914" height="588" alt="Screenshot 2026-06-12 123241" src="https://github.com/user-attachments/assets/6ee4e42c-aa45-4032-806e-d7e4ec8f38d8" /><img width="1668" height="842" alt="PHEOBUS" src="https://github.com/user-attachments/assets/b3899140-8289-4088-9de2-e4115ba7f1a3" />


