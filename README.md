# Security Sweep — Claude Code Skill

Pentagon-grade red team security scanning for any codebase or infrastructure. Auto-detects project type and adapts.

## Skills

### `/security-sweep` — Full Audit
Auto-detects project type and launches the right agents:

**App mode** (19 agents): Auth, DB, AI/Chat, Network/SSRF, Frontend, Tauri IPC, Infrastructure, Runtime/DoS, WebView/Cache, Crypto, Access Control, Diff Review, Dependency Audit, RLS/OWASP — plus 6 conditional Tier 2 agents.

**Infrastructure mode** (10 agents): SSH into live machines to audit SSH configs, Docker containers, firewalls, CrowdSec/IDS, Tailscale/VPN, backups, DNS/TLS, secrets — plus posture validation and cross-VM connectivity checks.

**Hybrid mode**: Runs both. Combined security score weighted 60% infra / 40% code.

### `/security-check` — Surgical Scan (Changed Files Only)
Same quality, scoped to recent changes. 5 code agents + 2 infra agents on the diff. Includes blast radius verification for infrastructure changes.

## Features

- **Auto-detection** — app vs infra vs hybrid based on project structure
- **Confidence scores** (0-100%) on every finding
- **Exploit PoCs** required for HIGH+ severity
- **Attack chain detection** — LOWs that combine into HIGHs
- **Defense depth scoring** — per attack surface (code) or per network layer (infra)
- **Blast radius assessment** — what an attacker gains from each CRITICAL/HIGH infra finding
- **Security score** — `100 - (CRIT×25) - (HIGH×10) - (MED×3) - (LOW×1)`
- **Baseline diffing** — NEW/RECURRING/FIXED tracking across sweeps
- **Auto-fix** with regression tests for every CRITICAL/HIGH
- **Team integration** — uses `.claude/agents/` definitions when available (Auditor=read-only, Hardener=fix, Red Team=validate)
- **Infrastructure: live SSH verification** — config files lie, running state is truth
- **Tiered execution** — Tier 2 skipped if Tier 1 is clean
- **Haiku for command-parsing** — saves tokens where deep reasoning isn't needed
- **Results saved** to `.security/sweep-{date}.json` for historical tracking

## Install

Copy the skill files into your Claude Code skills directory:

```bash
# security-sweep (full audit)
mkdir -p ~/.claude/skills/security-sweep
cp SKILL.md ~/.claude/skills/security-sweep/

# security-check (surgical scan)
mkdir -p ~/.claude/skills/security-check
cp SECURITY-CHECK.md ~/.claude/skills/security-check/SKILL.md
```

Or clone and symlink:

```bash
git clone https://github.com/Josue7211/security-sweep.git
ln -s $(pwd)/security-sweep/SKILL.md ~/.claude/skills/security-sweep/SKILL.md
mkdir -p ~/.claude/skills/security-check
ln -s $(pwd)/security-sweep/SECURITY-CHECK.md ~/.claude/skills/security-check/SKILL.md
```

## Usage

```
/security-sweep    # Full audit (auto-detects app/infra/hybrid)
/security-check    # Quick scan on recent changes only
```

## Project Type Detection

The skill automatically detects your project type:

| Project has | Mode | What runs |
|---|---|---|
| `src/`, `frontend/`, `.rs`/`.ts` files | **App** | 13 code agents + 6 conditional |
| `.claude/agents/ARCHITECTURE.md`, `docker-compose*`, `terraform/` | **Infra** | 8 SSH agents + 2 conditional |
| Both | **Hybrid** | All agents, combined scoring |

## Infrastructure Mode

For infra projects, agents SSH into live machines and check actual running state:

- **SSH + Access Control** — key-only auth, stale keys, sudo config
- **Docker + Containers** — 0.0.0.0 bindings, privileged containers, socket access
- **Firewall + VLANs** — inter-VLAN rules, egress filtering, unexpected listeners
- **CrowdSec + IDS** — engine health, bouncer connectivity, decision flow
- **Tailscale + VPN** — ACL policy, device inventory, subnet routes
- **Backups** — existence, age, encryption, permissions, restore testing
- **DNS + TLS** — resolution from host and containers, cert expiry, DNSSEC
- **Secrets** — hardcoded creds, plain text tokens, git history leaks

SSH commands are read from `.claude/agents/` definitions or `CLAUDE.md` — never hardcoded.

## Team Integration

If your project has `.claude/agents/TEAM.md` with agent definitions, the sweep integrates with your team model:
- **Auditor** agents run in read-only mode
- **Hardener** agents apply fixes
- **Red Team** agents validate defenses

## How It Works

1. **Step 0:** Read context + detect project type (app/infra/hybrid)
2. **Step 1A:** Launch Tier 1 code agents (if app/hybrid)
3. **Step 1B:** Launch Tier 1 infra agents (if infra/hybrid)
4. **Step 2:** Consolidate, dedup, attack chains, defense depth, blast radius
5. **Step 3:** Launch Tier 2 on flagged files/machines (skipped if clean)
6. **Step 4:** Present report with executive summary + technical details
7. **Step 5:** Cross-validate low-confidence findings
8. **Step 6:** Fix (file-locked for code, machine-locked for infra)
9. **Step 7:** Verify tests/services pass, regression tests, commit
10. **Step 8:** Save results for baseline diffing

## Requirements

- Claude Code with Opus model access
- A codebase with CLAUDE.md describing the project
- For infra mode: SSH access to target machines (documented in CLAUDE.md or `.claude/agents/`)
- Optional: `.claude/agents/` directory for team integration

## License

MIT
