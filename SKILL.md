---
name: security-sweep
description: Pentagon-grade red team security scan. 19 agents in two tiers scan auth, database, AI, network, frontend, IPC, infra, runtime, crypto, WebView, supply chain. Includes live dependency audits, SSH-based RLS verification, OWASP Top 10, secrets scanning, endpoint fuzzing, silent failure hunting, test coverage, type design, and code simplification. Confidence scores, exploit PoCs, attack chain detection, defense depth scoring, security score tracking, baseline diffing, and auto-fix with regression tests.
---

# Red Team Security Sweep

Pentagon-grade security audit. ZERO gaps acceptable.

## Step 0: Read Context

Before launching agents, read CLAUDE.md and `git log --oneline -5` to understand:
- Tech stack (Rust/Tauri/React/Supabase)
- File structure (where routes, crypto, auth, frontend live)
- Recent changes (what was just modified)
- Previous sweep results (check `.security/sweep-*.json` if exists)

If last 3 sweeps found only LOW findings, ask user before proceeding.

## Step 1: Launch Tier 1 (13 agents, parallel)

Launch ALL in a SINGLE message with `run_in_background: true`.

**CRITICAL: Generate DETAILED prompts for each agent.** Don't just say "check auth." Include:
- Exact file paths from the project structure
- Specific function names to examine
- Recent git changes that affect this area
- What was found in previous sweeps (so they don't re-report)

**Every agent MUST:**
- Include confidence % (0-100) on each finding
- Include exploit PoC (curl/code) for HIGH+
- Include fix effort estimate (Quick/Medium/Large)
- Report positive findings too (feeds defense depth scoring)

### Agent 1: Auth + Sessions (Opus, HIGH PRIORITY — gets extra context)
Probe: RequireAuth coverage, MFA bypass, session fixation, token TOCTOU, JWT expiry, OAuth PKCE, rate limit ordering, 24h expiry, password change re-encryption, WebAuthn flows.
Files: server.rs, routes/auth.rs, gotrue.rs, crypto.rs

### Agent 2: Database + SQLite + Sync (Opus)
Probe: SQLite permissions, sync injection, SQL format!() injection, cursor manipulation, RLS on ALL tables, WAL mode, cleanup jobs, dynamic column upsert safety.
Files: sync.rs, db.rs, all migrations, supabase.rs, server.rs (cache/cleanup)

### Agent 3: AI Chat + Prompt Injection (Opus)
Probe: DOMPurify config (img restricted? rel enforced?), image magic bytes + size + count limits, WS message size, path traversal, model validation, history redaction, upstream AI gateway credentials.
Files: routes/chat.rs, sanitize.ts, MarkdownBubble.tsx

### Agent 4: Network + SSRF (Opus)
Probe: Link preview (IPv6, DNS rebinding, redirects), ntfy SSRF blocklist, CalDAV redirect credentials, IMAP injection, set_secret URL injection, SSE cross-user filter.
Files: routes/messages.rs, notify.rs, email.rs, calendar.rs, events.rs, secrets.rs

### Agent 5a: Frontend TypeScript (Opus)
Probe: XSS via dangerouslySetInnerHTML, DOMPurify hooks, localStorage/sessionStorage, settings import validation, VITE_ env vars, postMessage, prototype pollution.
Files: sanitize.ts, api.ts, vault.ts, webauthn.ts, package.json, settings/*.tsx

### Agent 5b: Tauri IPC + Config (Opus)
Probe: IPC blocklists, CSP all directives, capabilities, shell permissions, devtools.
Files: secrets.rs, capabilities/default.json, tauri.conf.json

### Agent 6: Infrastructure + Git (Opus)
Probe: Git history secrets, hardcoded IPs, CI pinned SHA, pre-commit patterns, sidecar checksums, file permissions.
Files: .gitignore, ci.yml, scripts/, supabase/config.toml
Run: `git log --all -p -G 'password|secret|100\.' -- '*.rs' '*.ts' '*.json' 2>&1 | head -60`

### Agent 7: Runtime + DoS (Opus)
Probe: Rate limiter ordering, TimeoutLayer, ALL connection limits, WAL+busy_timeout, table growth caps, HashMap eviction, image limits, pipeline counter, log cap.
Files: server.rs, routes/chat.rs, events.rs, messages.rs, sync.rs, logging.rs, db.rs

### Agent 8: WebView + Cache + Headers (Opus)
Probe: no-store, nosniff, no-referrer, x-frame-options all global, worker-src none, PouchDB removed, note content not in localStorage, Supabase SDK removed.
Files: tauri.conf.json, server.rs, package.json, vault.ts

### Agent 9: Crypto + Key Management (Opus)
Probe: Argon2id params, random salt decoded, nonce random, Zeroize on Drop + intermediates, OAuth key guard, PKCE zeroized, constant-time, password dry-run, core dumps disabled, API key rotation.
Files: crypto.rs, server.rs, routes/auth.rs, secrets.rs, routes/user_secrets.rs

### Agent 10: Access Control (Opus, HIGH PRIORITY)
Probe: EVERY handler RequireAuth, EVERY query user_id filtered, EVERY Supabase *_as_user, RLS ALL tables, SSE strict filter, DLP allowlist, security_events scoped.
Files: EVERY file in routes/, server.rs, sync.rs, audit.rs, dlp.rs

### Agent 11: Diff Review (Opus)
Reviews ONLY recent diffs for bugs in new security code.
Run: `git log --oneline -10` then `git diff HEAD~10..HEAD`

### Agent 12: Dependency Audit + Secrets Scan (Haiku — saves tokens)
Run: `cargo audit`, `npm audit`, secrets grep on files + history.

### Agent 13: RLS Verification + OWASP (Opus)
Check CLAUDE.md for database access instructions. If SSH access to a Supabase/Postgres host is documented, use it to run live RLS queries (check rowsecurity on pg_tables, verify policies exist, run canary integrity checks). If no SSH access, fall back to reading migration SQL files and verifying RLS is declared for all tables.
Also score OWASP Top 10 categories PASS/PARTIAL/FAIL against actual code.

## Step 2: Consolidate Tier 1

After all 13 complete:

1. **Deduplicate**: Same file + line range (5 lines) = keep most detailed
2. **Attack chains**: Find LOWs that combine into HIGHs (e.g., SSRF + credential leak)
3. **Defense depth**: Count layers per attack surface (0=CRITICAL, 1=weak, 2+=strong)
4. **Security score**: `100 - (CRIT*25) - (HIGH*10) - (MED*3) - (LOW*1)` with per-OWASP breakdown
5. **Baseline diff**: Mark NEW/RECURRING/FIXED vs previous sweep

## Step 3: Launch Tier 2 (ONLY if Tier 1 found findings)

Launch 6 agents on ONLY the flagged files. Skip entirely if Tier 1 was clean.

### Agent 14: Silent Failure Hunter (`pr-review-toolkit:silent-failure-hunter`)
### Agent 15: Test Coverage (`pr-review-toolkit:pr-test-analyzer`)
### Agent 16: Type Design (`pr-review-toolkit:type-design-analyzer`)
### Agent 17: Code Reviewer (`feature-dev:code-reviewer`)
### Agent 18: Code Simplifier (`code-simplifier:code-simplifier`)
### Agent 19: Endpoint Fuzzer (Opus)

## Step 4: Present Report

**Executive Summary** (non-technical, 3 sentences):
> "Security Score: XX/100. N new issues found (X critical, Y high). Main risk: [area]."

**Technical Report:**
- Score + per-category OWASP bars
- Findings table: severity, confidence %, file:line, fix effort, NEW/RECURRING
- Attack chains identified
- Defense depth map
- Exploit PoCs for HIGH+

## Step 5: Cross-Validate

ONLY for findings with confidence < 90%. Read actual code, drop false positives.

## Step 6: Fix

**If subagents fail on permissions, apply fixes directly in the main context.** Don't launch a 4th fix agent for the same issue.

**File locking: one agent per file.** Group:
- A: server.rs | B: auth.rs | C: chat.rs | D: other routes | E: frontend | F: config

## Step 7: Verify + Commit

`cargo test` + `npx vitest run` must pass.

For each CRITICAL/HIGH fix, generate a regression test inline. Don't make this a separate step — write the test WITH the fix.

Check if pre-commit hook catches the pattern. If not, add it.

One commit per fix group. Push.

## Step 8: Save Results

Save to `.security/sweep-{date}.json` (gitignored):
```json
{
  "date": "YYYY-MM-DD",
  "commit": "hash",
  "score": 94,
  "findings": {"critical": 0, "high": 1, "medium": 3, "low": 5},
  "new": 2, "recurring": 3, "fixed": 4,
  "owasp": {"A01": "PASS", "A02": "PASS", ...}
}
```

## Rules

- Read actual code, not filenames. Exact file:line on every finding.
- Confidence % required. Exploit PoC required for HIGH+.
- Calibrate severity based on deployment context from CLAUDE.md (localhost, LAN, public).
- Dedup: same file+line = one finding. Don't re-report previous sweep accepted risks.
- Tier 2 only on flagged files. Haiku for command-parsing. Skip what's clean.
- Fix directly if subagents fail. Don't retry the same broken approach.
- Regression test WITH each fix, not as a separate phase.
- Positive findings matter — they feed defense depth and security score.
