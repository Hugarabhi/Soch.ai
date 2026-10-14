# 11 — Security Considerations

**Threat model:** OWASP Top 10 + AI-specific (prompt injection, data
exfiltration via research agents, model abuse). Design goal: **zero-trust at
internal boundaries, PII minimized, abuse-costly for attackers.**

---

## 1. Assets & trust boundaries

| Asset | Protection |
| --- | --- |
| User PII (email, name, learning data) | Encrypted at rest (AWS KMS / app AES-GCM), TLS 1.3 in transit, RLS-shaped queries |
| Auth tokens / Clerk secrets | Vault-managed; never in repo; rotated quarterly |
| API ↔ AI gateway | mTLS (k8s service mesh — Istio/Linkerd) + per-call HMAC |
| Ollama pool tokens & prompts | Network-isolated VPC; request data scrubbed of PII by default (local-first) |
| Research cache | Treated as *untrusted data*; sanitized before LLM context |

## 2. Authentication & authorization
- **AuthN:** Clerk (OAuth/social + email magic link/passwordless). Session JWTs
  verified via Clerk JWKS at NestJS guard + middleware. No password storage.
- **AuthZ:** owner-scoped data via `userId` in every query (defense in depth:
  also enable Postgres RLS on multi-tenant tables). Feature/role tiers via
  metadata claims; staff endpoints behind `staff` claim + IP allowlist.
- **Clerk webhook:** verify `Clerk-Webhook-Signature` (HMAC SHA-256 over raw
  body, replay-window check).

## 3. Threat register (STRIDE) & controls
| Threat | Control |
| --- | --- |
| **IDOR / cross-user reads** | All queries carry `userId`; negative integration tests; RLS backup |
| **BOLA/parameter tampering** | CUID ids (unguessable), ownership checks in services (not just controllers) |
| **SSRF (research agents)** | Browser pool runs in isolated namespace with egress allowlist; URL scheme/literal-IP validation; DNS rebinding guards; no private-range targets |
| **Prompt injection via scraped content** | Tool output wrapped as data; judge gate; size caps; no system-prompt concatenation |
| **Malicious markdown/JS in portfolio/README output** | Output sanitized (DOMPurify), CSP `script-src 'self'`, sandboxed iframe preview |
| **Rate-limit/DOS** | Redis sliding-window per user; queue starvation guards; WAF on CDN |
| **Data exfiltration** | No PII to hosted LLMs by default; redaction regex pre-send; audit log of external model calls |
| **Secret leakage** | Scanning in CI (gitleaks/trivy), `pre-commit-hooks`, Vault injection only |
| **Dependency attacks** | `npm audit`/`pip-audit` gates, lockfile-sourced CI, SBOM generation, signature-enforced base images |
| **Cryptographic misuse** | Use managed KMS/Clerk; never hand-roll crypto; `crypto.subtle` for client-side |

## 4. AI-specific hardening
- **Research agent sandbox:** per-run ephemeral profile directory, cookie
  isolation, no user account login into scraped sites, request budget per run,
  concurrent pool cap, kill-switch feature flag.
- **Model router guardrails:** deny-list prompts/domains; per-user daily token
  cap; cost alerting SLO (docs/12 alerts) with auto-degrade to local/cached.
- **Output integrity:** every generated file (READMEs, resumes, portfolio code)
  passes structural validation + secret/hardcoded-credential scan before showing
  "download".
- **Eval-based safety net:** adversarial prompt-injection suite in CI (docs/10).

## 5. Data privacy & compliance
- **GDPR / India DPDP Act 2023:** lawful basis (performance of contract +
  legitimate interest); consent capture for profiling; **right to delete** =
  cascade delete of all user rows + Qdrant vectors + S3 objects + queue jobs
  (job-level `userId` routing makes this feasible).
- **Data residency:** single region v1 (ap-south); multi-region later (docs/12/14)
  with residency zones; research cache stored region-locally.
- **Retention:** research snapshots 72 h; chat messages 18 mo; progress 24 mo.
  Automated purger + backup encryption (AES-256, KMS).
- **Children:** platform is 13+; age-gate on signup; no marketing to minors.
- **DPO/Data notice:** published privacy policy + in-product "how your data
  trains nothing" (we do NOT train on user content).

## 6. Secrets & configuration
- `.env` never committed (pre-commit block). Secrets in AWS Secrets Manager /
  Doppler, injected as mounted secrets in k8s via ExternalSecrets.
- Rotation: Clerk keys 90 d; AI gateway key 90 d; S3 creds 90 d.
- Environment-specific (dev/staging/prod) isolated; prod secrets unreachable
  from CI except via deployment role.

## 7. Auditing & incident response
- Audit log table: auth events, AI external calls (model, tokens, no content),
  exports, admin actions.
- Incident response: runbook in `docs/12`; severity mapping; EU-SoC-adjacent
  time windows for compliance notifications (72 h for breach).
- Pen-test cadence: annual; quarterly dependency scan; monthly red-team on
  research agents.