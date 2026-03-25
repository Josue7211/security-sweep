---
name: security-sweep
description: Pentagon-grade red team security scan. Auto-detects project type (app vs infrastructure vs hybrid). For app projects: 19 agents in two tiers scan auth, database, AI, network, frontend, IPC, infra, runtime, crypto, WebView, supply chain. For infrastructure projects: 8 agents SSH into live machines to audit Docker, firewalls, CrowdSec, Tailscale, SSH configs, backups, DNS/TLS, secrets. Hybrid mode runs both. Includes live dependency audits, SSH-based RLS verification, OWASP Top 10, secrets scanning, endpoint fuzzing, silent failure hunting, test coverage, type design, and code simplification. Confidence scores, exploit PoCs, attack chain detection, defense depth scoring per network layer, blast radius assessment, security score tracking, baseline diffing, and auto-fix with regression tests. Integrates with .claude/agents/ team definitions when available.
---

# Red Team Security Sweep

Pentagon-grade security audit. ZERO gaps acceptable.

## Step 0: Read Context + Detect Project Type

Before launching agents, read CLAUDE.md and `git log --oneline -5` to understand:
- Tech stack (Rust/Tauri/React/Supabase OR infrastructure/Docker/firewall)
- File structure (where routes, crypto, auth, frontend live)
- Recent changes (what was just modified)
- Previous sweep results (check `.security/sweep-*.json` if exists)

**Project type detection:**

```
HAS_APP_CODE = exists(src-tauri/ OR src/ OR app/ OR frontend/) with .rs/.ts/.tsx/.js files
HAS_INFRA = exists(.claude/agents/ARCHITECTURE.md) OR exists(docker-compose*.yml) OR exists(terraform/) OR exists(ansible/)
HAS_TEAM = exists(.claude/agents/TEAM.md) with agent definitions
```

| HAS_APP_CODE | HAS_INFRA | Mode |
|---|---|---|
| true | false | **APP** — run Tier 1 code agents only |
| false | true | **INFRA** — run Tier 1 infra agents only |
| true | true | **HYBRID** — run both Tier 1 code + infra agents |

**If HAS_TEAM is true:** Read `.claude/agents/TEAM.md` and agent definitions. Reference team roles in agent prompts.

If last 3 sweeps found only LOW findings, ask user before proceeding.

---

## Step 1A: Launch Tier 1 Code Agents (APP or HYBRID mode only)

Launch ALL in a SINGLE message with `run_in_background: true`.

**CRITICAL: Generate DETAILED prompts for each agent.** Include exact file paths, function names, recent git changes, and previous sweep findings.

**Every agent MUST:**
- Include confidence % (0-100) on each finding
- Include exploit PoC (curl/code) for HIGH+
- Include fix effort estimate (Quick/Medium/Large)
- Report positive findings too (feeds defense depth scoring)

### Agent 1: Auth + Sessions (Opus, HIGH PRIORITY)
Probe: RequireAuth coverage, MFA bypass, session fixation, token TOCTOU, JWT expiry, OAuth PKCE, rate limit ordering, 24h expiry, password change re-encryption, WebAuthn flows.

### Agent 2: Database + SQLite + Sync (Opus)
Probe: SQLite permissions, sync injection, SQL format!() injection, cursor manipulation, RLS on ALL tables, WAL mode, cleanup jobs, dynamic column upsert safety.

### Agent 3: AI Chat + Prompt Injection (Opus)
Probe: DOMPurify config, image magic bytes + size + count limits, WS message size, path traversal, model validation, history redaction, upstream AI gateway credentials.

### Agent 4: Network + SSRF (Opus)
Probe: Link preview (IPv6, DNS rebinding, redirects), SSRF blocklist, redirect credentials, injection vectors, SSE cross-user filter.

### Agent 5a: Frontend TypeScript (Opus)
Probe: XSS via unsafe HTML rendering, sanitizer hooks, localStorage/sessionStorage, settings import validation, env vars, postMessage, prototype pollution.

### Agent 5b: Tauri IPC + Config (Opus)
Probe: IPC blocklists, CSP all directives, capabilities, shell permissions, devtools.

### Agent 6: Infrastructure + Git (Opus)
Probe: Git history secrets, hardcoded IPs, CI pinned SHA, pre-commit patterns, sidecar checksums, file permissions.

### Agent 7: Runtime + DoS (Opus)
Probe: Rate limiter ordering, TimeoutLayer, ALL connection limits, WAL+busy_timeout, table growth caps, HashMap eviction, image limits, pipeline counter, log cap.

### Agent 8: WebView + Cache + Headers (Opus)
Probe: no-store, nosniff, no-referrer, x-frame-options, worker-src none, sensitive data not in localStorage.

### Agent 9: Crypto + Key Management (Opus)
Probe: Argon2id params, random salt, nonce random, Zeroize on Drop, constant-time comparisons, core dumps disabled, API key rotation.

### Agent 10: Access Control (Opus, HIGH PRIORITY)
Probe: EVERY handler RequireAuth, EVERY query user_id filtered, RLS ALL tables, SSE strict filter, DLP allowlist.

### Agent 11: Diff Review (Opus)
Reviews ONLY recent diffs for bugs in new security code.

### Agent 12: Dependency Audit + Secrets Scan (Haiku — saves tokens)
Run: `cargo audit`, `npm audit`, secrets grep on files + history.

### Agent 13: RLS Verification + OWASP (Opus)
Live RLS queries if SSH access documented, else read migration SQL. Score OWASP Top 10 PASS/PARTIAL/FAIL.

---

## Step 1B: Launch Tier 1 Infrastructure Agents (INFRA or HYBRID mode only)

Launch ALL in a SINGLE message with `run_in_background: true`.

**Read `.claude/agents/ARCHITECTURE.md` first** — ground truth for topology, machines, Docker stacks, security stack.

**Every infra agent MUST:**
- SSH into target machines and gather LIVE evidence
- Include confidence % (0-100) on each finding
- Include blast radius assessment (what breaks if exploited)
- Include remediation with exact commands
- Report positive findings too

**SSH commands:** Read from `.claude/agents/` definitions or CLAUDE.md. Never guess credentials.

### Agent I1: SSH + Access Control (Opus, HIGH PRIORITY)
Probe: SSH config on every VM (PermitRootLogin, PasswordAuthentication, key-only), authorized_keys audit, fail2ban/CrowdSec SSH protection, sudo config.

### Agent I2: Docker + Container Security (Opus)
Probe: Port bindings (0.0.0.0 exposure), privileged containers, Docker socket access, image versions, container network isolation, volume mounts, health checks.

### Agent I3: Firewall + Network Segmentation (Opus)
Probe: VLAN isolation, egress filtering (catch-all enabled?), port forwards, firewall bouncer health, unexpected listeners on all VMs.

### Agent I4: CrowdSec + IDS Health (Opus)
Probe: Engine running, all bouncers connected, LAPI reachable, collections current, blocklist import running, IDS rules loaded, decision flow active.

### Agent I5: Tailscale + VPN (Opus)
Probe: ACL policy (restrictive or wide open?), device inventory (unexpected peers?), subnet routes, exit node, key expiry.

### Agent I6: Backup + Recovery (Opus)
Probe: Backup existence, age (stale?), encryption at rest, permissions (deletable by compromised VM?), restore tested, offsite copy.

### Agent I7: DNS + TLS + Certificates (Opus)
Probe: DNS resolution (host AND inside containers), DNSSEC, cert expiry on tunnel endpoints, tunnel health, access policies.

### Agent I8: Secrets + Credential Hygiene (Opus)
Probe: Hardcoded passwords in configs, plain text API tokens, SSH keys without passphrases, non-expiring tokens, shared credentials, secrets in git history.

---

## Step 2: Consolidate Tier 1

After ALL agents complete (code + infra):

1. **Deduplicate**: Same file + line range (5 lines) = keep most detailed. Same machine + finding = keep most detailed.
2. **Attack chains**: Find LOWs that combine into HIGHs (e.g., wide Tailscale ACL + unpatched VM + no egress filtering)
3. **Defense depth**: Count layers per attack surface:
   - Code: 0=CRITICAL, 1=weak, 2+=strong
   - Infra: count layers (edge WAF -> network firewall -> host firewall -> app auth -> IDS -> monitoring)
4. **Security score**:
   - Code: `100 - (CRIT*25) - (HIGH*10) - (MED*3) - (LOW*1)` with per-OWASP breakdown
   - Infra: same formula with per-layer breakdown (Edge/Network/Host/App/Monitoring)
   - Combined: weighted average (infra 60%, code 40%)
5. **Baseline diff**: Mark NEW/RECURRING/FIXED vs previous sweep
6. **Blast radius map** (infra): For each CRITICAL/HIGH, what does attacker gain?

## Step 3: Launch Tier 2 (ONLY if Tier 1 found findings)

Skip entirely if Tier 1 was clean.

### For APP/HYBRID mode:
- Agent 14: Silent Failure Hunter
- Agent 15: Test Coverage
- Agent 16: Type Design
- Agent 17: Code Reviewer
- Agent 18: Code Simplifier
- Agent 19: Endpoint Fuzzer

### For INFRA/HYBRID mode:
- Agent I9: Posture Validation — run existing posture check scripts, compare against expected
- Agent I10: Cross-VM Connectivity — verify each VM can ONLY reach what it should

## Step 4: Present Report

**Executive Summary** (3 sentences):
> "Security Score: XX/100 (code: XX, infra: XX). N new issues found (X critical, Y high). Main risk: [area]."

**Technical Report:**
- Score + per-category bars (OWASP for code, layer-based for infra)
- Findings table: severity, confidence %, location, fix effort, NEW/RECURRING
- Attack chains, defense depth map, blast radius map
- Exploit PoCs for HIGH+

## Step 5: Cross-Validate

ONLY for findings with confidence < 90%. Read actual code or SSH live. Drop false positives.

## Step 6: Fix

**Code fixes — file locking:** one agent per file.
**Infra fixes — machine locking:** one agent per VM.
**Docker:** Portainer API ONLY if CLAUDE.md says so.
**After ANY infra fix:** blast radius verification — SSH, password manager, Docker ps, IDS, tunnels.
**If subagents fail:** fix directly in main context.

## Step 7: Verify + Commit

APP: `cargo test` + `npx vitest run` must pass.
INFRA: All services respond, SSH works, IDS healthy, no friendly IPs banned.
Regression test WITH each CRITICAL/HIGH fix. One commit per fix group.

## Step 8: Save Results

Save to `.security/sweep-{date}.json` (gitignored):
```json
{
  "date": "YYYY-MM-DD",
  "commit": "hash",
  "mode": "app|infra|hybrid",
  "score": {"combined": 94, "code": 96, "infra": 92},
  "findings": {"critical": 0, "high": 1, "medium": 3, "low": 5},
  "new": 2, "recurring": 3, "fixed": 4,
  "owasp": {"A01": "PASS", "A02": "PASS"},
  "infra_layers": {"edge": "PASS", "network": "PARTIAL", "host": "PASS", "app": "PASS", "monitoring": "PASS"},
  "blast_radius": ["finding-1: lateral movement risk description"]
}
```

## Rules

- Exact file:line for code, exact machine:service for infra.
- Confidence % required. Exploit PoC required for HIGH+.
- Calibrate severity from CLAUDE.md deployment context (localhost, LAN, public, VPN).
- Dedup across sources. Don't re-report accepted risks.
- Tier 2 only on flagged files/machines. Skip what's clean.
- Fix directly if subagents fail.
- Regression test WITH each fix.
- Positive findings feed defense depth scoring.
- **Infra: SSH and check LIVE state. Config files lie.**
- **Infra: blast radius on every CRITICAL/HIGH.**
- **If `.claude/agents/` exists: use team roles.**
- **NEVER hardcode secrets, IPs, or credentials in this file.**
