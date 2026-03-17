# Security Sweep — Claude Code Skill

Pentagon-grade red team security scanning for any codebase. Two skills, 19+ agents, zero gaps.

## Skills

### `/security-sweep` — Full Codebase Audit
19 agents in two tiers scan every attack surface:
- **Tier 1 (13 agents):** Auth, DB, AI/Chat, Network/SSRF, Frontend TS, Tauri IPC, Infrastructure, Runtime/DoS, WebView/Cache, Crypto, Access Control, Diff Review, Dependency Audit + Secrets Scan, RLS Verification + OWASP Top 10
- **Tier 2 (6 agents, conditional):** Silent Failure Hunter, Test Coverage, Type Design, Code Reviewer, Code Simplifier, Endpoint Fuzzer — only runs on files flagged by Tier 1

### `/security-check` — Surgical Scan (Changed Files Only)
Same 100% quality, scoped to recent changes. 5 agents on the diff. Use after any code change.

## Features

- **Confidence scores** (0-100%) on every finding
- **Exploit PoCs** required for HIGH+ severity
- **Attack chain detection** — LOWs that combine into HIGHs
- **Defense depth scoring** — count protective layers per attack surface
- **Security score** — `100 - (CRIT×25) - (HIGH×10) - (MED×3) - (LOW×1)`
- **Baseline diffing** — NEW/RECURRING/FIXED tracking across sweeps
- **Auto-fix** with regression tests for every CRITICAL/HIGH
- **Cross-validation** — verifies findings before fixing (drops false positives)
- **Tiered execution** — Tier 2 skipped if Tier 1 is clean (saves ~40% tokens)
- **Haiku for command-parsing agents** — saves tokens where deep reasoning isn't needed
- **Post-fix `/simplify`** — cleans up security code after fixes
- **Results saved** to `.security/sweep-{date}.json` for historical tracking

## Install

Copy the skill folders into your Claude Code skills directory:

```bash
# security-sweep (full audit)
mkdir -p ~/.claude/skills/security-sweep
cp SKILL.md ~/.claude/skills/security-sweep/

# security-check (surgical scan)
mkdir -p ~/.claude/skills/security-check
cp SECURITY-CHECK.md ~/.claude/skills/security-check/SKILL.md
```

Or clone this repo and symlink:

```bash
git clone https://github.com/Josue7211/security-sweep.git
ln -s $(pwd)/security-sweep/SKILL.md ~/.claude/skills/security-sweep/SKILL.md
mkdir -p ~/.claude/skills/security-check
ln -s $(pwd)/security-sweep/SECURITY-CHECK.md ~/.claude/skills/security-check/SKILL.md
```

## Usage

```
/security-sweep    # Full 19-agent audit
/security-check    # Quick 5-agent scan on recent changes
```

## Requirements

- Claude Code with Opus model access (agents use Opus for reasoning, Haiku for command parsing)
- A codebase with a CLAUDE.md describing the project structure
- Optional: SSH access to database host for live RLS verification

## How It Works

1. **Step 0:** Reads CLAUDE.md + git history for project context
2. **Step 1:** Launches 13 Tier 1 agents in parallel
3. **Step 2:** Consolidates, deduplicates, detects attack chains, scores defense depth
4. **Step 3:** Launches 6 Tier 2 agents on flagged files (skipped if clean)
5. **Step 4:** Presents report with executive summary + technical details
6. **Step 5:** Cross-validates low-confidence findings
7. **Step 6:** Fixes everything (one agent per file, no conflicts)
8. **Step 7:** Verifies tests pass, generates regression tests, runs /simplify
9. **Step 8:** Saves results for baseline diffing on next sweep

## License

MIT
