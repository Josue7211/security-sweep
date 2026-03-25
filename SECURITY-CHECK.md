---
name: security-check
description: Surgical security scan on recently changed files only. Same 100% quality as /security-sweep (confidence scores, exploit PoCs, attack chains, defense depth) but scoped to recent changes. 5 Opus agents run in parallel on the diff. For infrastructure projects, detects config/firewall/Docker changes and runs targeted infra checks with blast radius verification. Use after features, bug fixes, or any code change. Use /security-sweep for full codebase audits.
---

# Security Check (Surgical)

Same rigor as /security-sweep. Same confidence scores, exploit PoCs, attack chains. Just scoped to what changed.

## Step 0: Identify Scope + Detect Mode

```bash
git diff --name-only HEAD~3..HEAD  # files changed in last 3 commits
git diff --cached --name-only      # staged but uncommitted
git diff --name-only               # unstaged changes
```

Combine all three into the TARGET_FILES list. If no changes detected, tell the user "nothing to check" and stop.

Read CLAUDE.md for project context. Check `.security/sweep-*.json` for previous findings on these files.

**Mode detection from changed files:**

| Changed files contain | Mode |
|---|---|
| `.rs`, `.ts`, `.tsx`, `.js`, `.jsx` source files | **CODE** — run code agents |
| `docker-compose*`, `*.conf`, `firewall*`, `*.rules`, SSH configs, `infra/` | **INFRA** — run infra agents + blast radius check |
| Both | **HYBRID** — run both |

## Step 1: Launch 5 Code Agents (CODE or HYBRID, parallel, Opus)

All agents get the TARGET_FILES list. They read ONLY those files + their direct dependencies.

**Same quality requirements as full sweep:**
- Confidence % on every finding
- Exploit PoC (curl/code) for HIGH+
- Fix effort estimate (Quick/Medium/Large)
- Positive findings too

### Agent 1: Auth + Access Control (HIGH PRIORITY)
If any route file, server.rs, auth.rs, or middleware changed:
- RequireAuth present? MFA enforced? user_id filtered?
- New endpoints exposed without auth?
- Session handling correct?
Skip if no auth-related files changed.

### Agent 2: Input Validation + Injection
For every changed file:
- New format!() with user input? SQL injection?
- New unsafe HTML rendering? Sanitizer applied?
- New external requests? SSRF protected?
- New file operations? Path traversal blocked?

### Agent 3: Crypto + Secrets + Config
If crypto, secrets, auth, or config files changed:
- Key handling correct? Zeroized?
- CSP still tight? New connect-src needed?
- Secrets exposed? New env vars?
Skip if no crypto/config files changed.

### Agent 4: Logic + Edge Cases
For every changed file:
- New error paths that leak info?
- Race conditions in new async code?
- Silent failures on critical ops?
- Connection counters that don't decrement?
- Bounds that overflow?

### Agent 5: Dependencies + Diff Review
- If package.json or Cargo.toml changed: run cargo audit / npm audit
- Review the actual diff line-by-line for obvious issues
- Check for patterns that previous sweeps flagged as recurring

## Step 1B: Launch Infra Agents (INFRA or HYBRID)

### Agent I1: Changed Config Validator (Opus)
For each changed infra file, SSH into the affected machine and verify:
- Config syntax is valid (no parse errors on reload)
- Service still running after change
- Expected behavior matches intent

### Agent I2: Blast Radius Check (Opus)
After ANY infra change, verify nothing broke:
- SSH to all VMs still works
- Password manager accessible
- Docker containers all running (no Exited)
- IDS/IPS metrics flowing
- DNS resolution working (host + inside containers)
- Tunnel endpoints responding

## Step 2: Consolidate + Report

Same process as full sweep:
1. Deduplicate (same file + 5 lines = one finding)
2. Attack chains (combine LOWs that chain)
3. Security score delta: "This change moves the score from X to Y"
4. Present with confidence %, PoCs, effort estimates
5. Blast radius status (infra mode): all services green or list what broke

## Step 3: Fix

Fix CRITICAL/HIGH directly in main context (no subagents for a surgical check).
Write regression test WITH each fix.
For code: cargo test + npx vitest run.
For infra: verify all services respond after fix.
Commit.

## When to escalate to /security-sweep

If the check finds 3+ HIGH findings, or any CRITICAL, recommend running the full sweep:
> "Found [N] high-severity issues in changed files. Recommend running /security-sweep for a full codebase audit — these changes may have broader implications."
