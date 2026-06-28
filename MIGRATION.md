# MIGRATION — Move opencode from root to assistant user

If you have opencode running as root with config files in `/root/`, this guide explains how to migrate everything to a proper non-root `assistant` user. The assistant becomes a first-class user with its own home, service, health monitoring, and cross-machine sync.

## What gets migrated

| From (root) | To (assistant) |
|-------------|----------------|
| `/root/AGENTS.md` | `/home/assistant/.agents/AGENTS.md` → symlinked to `~/AGENTS.md` |
| `/root/.opencode/` | `/home/assistant/.agents/` (agents, commands, modes, plugins, skills, tools, themes) |
| `/root/.config/opencode/opencode.jsonc` | `/home/assistant/.config/opencode/opencode.jsonc` |
| `/root/.local/share/opencode/` | `/home/assistant/.local/share/opencode/` (sessions, auth, database) |
| `/root/.local/state/opencode/` | `/home/assistant/.local/state/opencode/` (runtime state) |
| `/root/.cache/opencode/` | `/home/assistant/.cache/opencode/` (cache) |
| `/root/docs/` or machine profiles | `/home/assistant/setup/<hostname>/` |
| `/root/maintenance/reports/` | `/home/assistant/maintenance/reports/` |
| System-level opencode service | User-level service under `assistant` |
| System-level health/restart timers | User-level timers under `assistant` |

The migration preserves all session data (conversations, auth, settings). Nothing is lost.

### How skills and configs are merged

The `.opencode/` directory contains subdirectories for agents, commands, modes, plugins,
skills, tools, and themes. During migration, the agent reviews all of these and makes
intelligent decisions — it does NOT blindly copy or overwrite.

- **If a subdirectory already exists** in the destination, the script prompts before overwriting.
- **The agent prepares merge notes** explaining which customizations were preserved and why.
- **The user reviews and approves** before the script runs.

This avoids the scenario where a stock skill is overwritten by a customized one, or
vice versa. The agent understands the contents and asks for human judgment.

## How migration works

Migration is a two-stage process:

1. **The assistant agent audits** the existing root-based setup, merges the AGENTS.md,
   finds all session files, and customizes `migrate.sh.template` into `migrate.sh`

2. **You (the user) review** the customized script, stop opencode, and run `sudo bash migrate.sh`

The agent cannot stop opencode (it runs inside it). The agent cannot run the migration
script (it needs to run as root while opencode is stopped). The agent prepares everything;
you execute.

## Agent's workflow (what the assistant does)

### 1. Audit the existing setup

The agent reads:
- `/root/AGENTS.md` — existing identity and rules
- `/root/.config/opencode/opencode.jsonc` — provider config
- `/root/.opencode/skills/` — any skill files
- `/root/.local/share/opencode/` — session database (conversations)
- `/root/.local/state/opencode/` — runtime state
- `/root/.cache/opencode/` — cache
- `/root/docs/`, `/root/maintenance/` — any documentation or reports

It also checks whether opencode runs as a system-level service or as a one-off process.

### 2. Merge AGENTS.md

The agent creates a merged AGENTS.md that preserves the user's accumulated conventions
while adding essential safety rules:

- **Preserved from old AGENTS.md:** hostname-specific context, user conventions, AUR policy, pronouns, any custom rules
- **Added (from this repo's AGENTS.md):** NEVER restart opencode, NEVER open ports, package installation convention, maintenance report rules
- **Updated:** paths (`/root/maintenance/` → `~/maintenance/`), user identity (`root` → `assistant`), sudo instructions

The merged file goes to `/home/assistant/.agents/AGENTS.md`. The old one is kept intact
(for reference) and never overwritten.

### 3. Find extra session files

The agent checks for any opencode files not in the standard locations and adds them
to the `EXTRA_FILES` array in the migration script.

### 4. Customize migrate.sh

The agent takes `migrate.sh.template` from this repo and fills in:
- `__FILL_HOSTNAME__` → the machine's hostname
- `EXTRA_FILES=()` → any additional files found during audit
- `__AGENT_AGENTS_MERGE_NOTES__` → a summary of what was merged/updated

### 5. Present to you

The agent shows you the customized `migrate.sh` and the merged `AGENTS.md` for review.

## Your workflow (what you do)

### 1. Review

Read the customized `migrate.sh` and the merged `AGENTS.md`. Confirm that:
- Your conventions and rules are preserved in AGENTS.md
- File paths in the script are correct
- No important files were missed

### 2. Stop opencode

If running as a service:
```bash
systemctl stop opencode.service
```

If running as a one-off process:
```bash
kill $(pgrep -f opencode)
```

### 3. Run the migration

```bash
sudo bash /path/to/migrate.sh
```

The script creates the assistant user (if needed), copies all files, installs service
units and timers, sets up Syncthing and the SSH gateway, and enables everything.

### 4. Verify

```bash
ps aux | grep opencode | grep -v grep
journalctl --user --machine=assistant@ -u opencode.service -n 10
```

### 5. Clean up root (the script prompts)

The script asks whether to remove old opencode files from `/root/`. Answer `y` or `n`.
Backup copies remain in `/home/assistant/` regardless.

## After migration

- Your AGENTS.md has been merged — review it and iterate
- Syncthing is running — add remote device IDs to pair with other assistant instances
- SSH keys have been generated and will sync to other machines
- The assistant now runs as a proper non-root user with health monitoring and nightly restarts

## Firewall configuration

After migration, the script prompts you to configure opencode's network access.
Choose based on your security needs:

| Mode | Who can connect | nftables rule | When to use |
|------|----------------|--------------|-------------|
| **1 — localhost** | Only this machine | `tcp dport 4096 accept` on loopback | SSH tunnel is your entry point |
| **2 — local network** | Devices on your LAN | `ip saddr <subnet> tcp dport 4096 accept` | Connect from other machines at home |
| **3 — open** | Anyone on the internet | `tcp dport 4096 accept` | Cloud deployments, remote access without VPN |

The script asks for your LAN subnet if you choose option 2. The rule is added to
nftables immediately. Add it to `/etc/nftables.conf` for persistence across reboots.

If you skip during migration, you can always add the rule later:
```bash
nft add rule inet filter input ip saddr <subnet> tcp dport 4096 accept
```
