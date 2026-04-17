# CLAUDE.md — Ansible Baseline (Portfolio Project 2)

> Place at `~/dev/floorp/infra-tooling/ansible-baseline/CLAUDE.md`

## What this project is (SysAdmin lens)

The project that most directly proves richmuscle is a sysadmin. Takes a fresh Ubuntu 22/24 or RHEL 9 server and brings it to a hardened, monitored, production-ready state — idempotently, testably, documented. Every role follows standard Galaxy layout, every role has Molecule tests, every role has a real README. This is configuration management at the level a small IT team would actually use.

**DevOps toolkit angle:** idempotent automation, testing discipline, multi-OS CI matrix. Modern sysadmin.

**What this project proves about richmuscle as a sysadmin:**
- Understands CIS hardening (not just "I ran Lynis once")
- Knows the difference between idempotent and "works the first time"
- Thinks in terms of fleets, not individual servers
- Tests automation before trusting it (Molecule)
- Documents per role, not just per project
- Can support both Debian-family (Ubuntu) and Red Hat-family (RHEL) distros

## Scope (hard limits)

**In scope:**
- 7 Ansible roles, Galaxy-layout, Molecule-tested where marked:
  - `base-hardening` (Molecule ✓)
  - `ssh-hardening` (Molecule ✓)
  - `auditd` (Molecule ✓)
  - `firewall` (firewalld on RHEL, ufw on Ubuntu)
  - `node-exporter` (Prometheus exporter via systemd unit)
  - `podman` (rootless, lingering, systemd user services)
  - `log-forwarding` (rsyslog / journald)
- Inventories for `dev/` and `prod/` with `group_vars` and `host_vars`
- Ansible Vault — at least one real encrypted secret used by a real play
- CI with ansible-lint + yamllint on every PR
- Molecule matrix in CI: Ubuntu 22, Ubuntu 24, RHEL 9 (via Docker images)
- `site.yml` composing all roles
- `bootstrap.yml` for bare-minimum onboarding (create user, install python, enable sudo)
- `smoke-test.yml` validating post-deploy state
- Before/after Lynis scan recorded in README (target: ~60 → 85+)
- **systemd unit artifacts:** node-exporter deployed as systemd unit, auditd configuration reloaded via systemd, podman via user lingering
- **journalctl queries** in RUNBOOK for every service this configures

**Explicitly out of scope:**
- Windows targets (Linux only)
- Ansible Tower / AWX / Automation Platform (CLI only)
- Dynamic inventory from cloud APIs (static inventories; mention in ADR)
- Custom modules in Python (built-in modules only)
- Secrets rotation automation
- AAP-style RBAC

## Required file structure

```
ansible-baseline/
├── README.md
├── ARCHITECTURE.md
├── ROLES.md                           # index: role → purpose → vars → OS support
├── RUNBOOK.md
├── LICENSE
├── Makefile
├── ansible.cfg
├── requirements.yml                   # galaxy collections
├── .gitignore
├── .editorconfig
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                     # ansible-lint + yamllint
│   │   └── molecule.yml               # matrix: Ubuntu 22/24, RHEL 9
│   ├── CODEOWNERS
│   └── PULL_REQUEST_TEMPLATE.md
├── DECISIONS/
│   ├── README.md
│   ├── ADR-001-ansible-over-alternatives.md
│   ├── ADR-002-role-layout.md
│   ├── ADR-003-secrets-strategy.md
│   └── ADR-004-testing-with-molecule.md
├── inventories/
│   ├── dev/
│   │   ├── hosts.yml
│   │   ├── group_vars/all.yml
│   │   └── host_vars/example.yml
│   └── prod/
│       ├── hosts.yml
│       ├── group_vars/all.yml
│       └── host_vars/example.yml
├── roles/
│   ├── base-hardening/                # Galaxy layout × 7 roles
│   ├── ssh-hardening/
│   ├── auditd/
│   ├── firewall/
│   ├── node-exporter/
│   ├── podman/
│   └── log-forwarding/
├── playbooks/
│   ├── site.yml
│   ├── bootstrap.yml
│   ├── hardening-only.yml
│   └── smoke-test.yml
├── group_vars/all.yml                 # org-wide defaults
├── vault/secrets.yml                  # ansible-vault encrypted
└── scripts/
    ├── encrypt-vault.sh
    └── decrypt-vault.sh
```

Each role directory: `defaults/main.yml`, `handlers/main.yml`, `meta/main.yml`, `tasks/main.yml`, `templates/`, `vars/main.yml`, `README.md`, and for marked roles `molecule/default/{molecule.yml,converge.yml,verify.yml}`.

## Role responsibilities

- **base-hardening** — sysctl (IP forwarding, TCP SYN cookies, RPF, kernel.randomize_va_space), login.defs (PASS_MAX_DAYS, LOGIN_RETRIES), PAM pam_tally2 / pam_faillock, unattended-upgrades (Ubuntu) or dnf-automatic (RHEL), disable legacy services (avahi, cups if not needed)
- **ssh-hardening** — sshd_config per CIS: no root login, key-only auth, specific cipher/MAC/KEX algorithms, ClientAliveInterval, AllowUsers directive, banner from /etc/issue.net, fail2ban install + sshd jail
- **auditd** — install, CIS-aligned ruleset (watch privileged commands, config files, mount events), log rotation, optional remote forwarding stub
- **firewall** — firewalld on RHEL (zones + services), ufw on Ubuntu (policy-based), both default-deny inbound with explicit allow lists per host group
- **node-exporter** — install Prometheus node_exporter binary, systemd unit with restricted user, TLS optional
- **podman** — rootless install, `loginctl enable-linger <user>`, basic networking, user systemd services directory
- **log-forwarding** — rsyslog imfile for specific log paths → forwarded to configurable remote (Loki endpoint in Project 5, or syslog-ng target)

## ADR topics (minimum set)

1. **ADR-001: Ansible over alternatives** — vs Salt / Puppet / Chef / cloud-init; agent vs agentless; when I'd pick differently
2. **ADR-002: Role layout** — monorepo vs per-role repos; why monorepo for a small team, where it breaks at scale
3. **ADR-003: Secrets strategy** — Ansible Vault vs SOPS vs HashiCorp Vault; the single-encrypted-file pattern; rotation as a day-two problem
4. **ADR-004: Testing with Molecule** — why Docker driver vs Vagrant; what gets tested (idempotency, config file presence, service state), what doesn't (cross-role interactions)

## SysAdmin-specific README sections (required)

### `## Supported platforms`
Matrix: OS × role × tested-via-Molecule? Example:

| Role | Ubuntu 22 | Ubuntu 24 | RHEL 9 | Molecule tested |
|------|-----------|-----------|--------|-----------------|
| base-hardening | ✓ | ✓ | ✓ | ✓ |
| ssh-hardening | ✓ | ✓ | ✓ | ✓ |
| ... |

### `## Before/after hardening scores`
Lynis hardening index before running playbooks, after running playbooks. Include screenshot or raw output. This is the single most compelling evidence point for this project.

### `## Onboarding a new server (runbook)`
Step-by-step:
1. Ensure SSH key access to target
2. Add to `inventories/prod/hosts.yml`
3. Run `make bootstrap HOST=<name>`
4. Run `make apply HOST=<name>`
5. Run `make smoke-test HOST=<name>`
6. Verify Lynis score ≥ 85

### `## Limitations and what I'd do differently`
Examples:
- No dynamic inventory — hosts are static; cloud inventory plugin would be day-two
- Vault file is single-tenant; multi-tenant would use SOPS + age keys
- No secret rotation — manual re-encrypt; HashiCorp Vault would solve this
- Molecule tests use Docker; full VM testing (Vagrant) would catch systemd edge cases

## Acceptance criteria

- [ ] `make lint` passes (ansible-lint + yamllint clean)
- [ ] `make test` runs Molecule on both marked roles across 3 OS targets, passes
- [ ] `ansible-playbook -i inventories/dev playbooks/site.yml --check` clean
- [ ] CI green on fresh PR
- [ ] Lynis before/after scores documented (target: ~60 → 85+)
- [ ] ROLES.md complete — one section per role with purpose/OSes/vars/dependencies
- [ ] README "Onboarding a new server" runbook works end-to-end
- [ ] Vault has at least one real encrypted secret consumed by a real play
- [ ] 20+ commits over 6+ calendar days

## Build sequence (Week 2)

**Day 8:** Repo skeleton, `ansible.cfg`, inventory structure, `Makefile`, ADR-001 draft. **Checkpoint.**

**Day 9:** `roles/base-hardening/` — full role with defaults, handlers, tasks, templates, Molecule scenario. **Checkpoint.**

**Day 10:** `roles/ssh-hardening/` + Molecule scenario. **Checkpoint.**

**Day 11:** `roles/auditd/` + Molecule. `roles/firewall/` (firewalld + ufw branches). **Checkpoint.**

**Day 12:** `roles/node-exporter/` (systemd unit), `roles/podman/`, `roles/log-forwarding/`. **Checkpoint.**

**Day 13:** Vault setup, `scripts/encrypt-vault.sh`, real encrypted secret wired into a play. `playbooks/site.yml`, `playbooks/bootstrap.yml`, `playbooks/smoke-test.yml`. ADR-003 draft. **Checkpoint.**

**Day 14 (weekend):** CI workflows (ansible-lint, yamllint, molecule matrix). ADRs 002 and 004. **Checkpoint.**

**Day 15 (weekend):** Lynis before/after scans. README polish. ROLES.md finalized. Supported platforms matrix. **PR → merge.**

## What NOT to generate

- No `ansible-playbook` runs against real hosts without per-run approval
- No shell/command modules where native Ansible modules exist
- No skipping Molecule tests to move faster
- No non-idempotent tasks (no `creates:` escape hatches for bad tasks)
- No plaintext secrets — Vault or nothing
- No marketing copy

## SysAdmin voice (required tone)

> Manually hardening a new server means remembering 40+ sysctl tweaks, a dozen sshd_config changes, firewalld zones, and audit rules — every time, on every box. I've skipped steps when tired and regretted it later during audits. This project puts all of it in 7 idempotent Ansible roles, tested on Ubuntu 22/24 and RHEL 9, with before/after Lynis scans that prove the hardening actually lands. Adding a new server is one command.

That's sysadmin voice. Lived experience, specific numbers, honest about the tired-and-skipped-steps reality.

## Linux depth markers (required)

- [ ] At least 3 systemd unit files created by roles (node_exporter, a podman user service, auditd service override)
- [ ] At least 2 `journalctl` query examples in RUNBOOK (how to check node_exporter logs, how to check SSH auth failures)
- [ ] At least one reference in ARCHITECTURE.md to: PAM, cgroups, SELinux contexts (for RHEL), or nftables (underlying firewalld/ufw)

## Cross-project connections

- **Hardens Project 4's k3s node** (before k3s install — can be run on the k3s host)
- **`node-exporter` role** is what **Project 5's Prometheus** scrapes
- **`log-forwarding` role** ships to **Project 5's Loki**
- Could be used to harden the EC2 instance in **Project 3** (WireGuard server)
