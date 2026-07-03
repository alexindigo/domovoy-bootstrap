# BOOTSTRAP — Domovoy Setup From Scratch

Set up the `domovoy` AI agent on a fresh Linux machine. Creates a dedicated system user (UID < 1000), clones this repo, and configures opencode as a user-level service with health monitoring, cross-machine SSH access, and Syncthing sync.

## 1. Create the domovoy user

```bash
useradd -r -m -d /home/domovoy -s /bin/bash -u 588 domovoy
echo "domovoy ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/domovoy
loginctl enable-linger domovoy
```

Lingering keeps the user's systemd session alive without login — required for timers and Syncthing.

### Shell choice

Domovoy's login shell is `/bin/bash` if this machine will **collaborate with
other Domovoys** (receives SSH/rsync migrations from peer machines). A **solo**
machine (single domovoy, never a migration target) can use `nologin` for tighter
security — local operation is unaffected since opencode execs `bash` directly
and systemd services bypass the login shell. Reversible with `usermod -s`.

```bash
# Collaborative (peer fleet, receives SSH):
useradd -r -m -d /home/domovoy -s /bin/bash -u 588 domovoy

# Solo machine (local only, no SSH inbound):
useradd -r -m -d /home/domovoy -s /usr/bin/nologin -u 588 domovoy
```

## 2. Clone this repo

```bash
sudo -u domovoy git clone https://github.com/alexindigo/domovoy-bootstrap.git /home/domovoy/.agents
```

The repo provides: AGENTS.md (identity), skill definitions, and templates.

Also clone the skill library into Domovoy's public workspace:

```bash
sudo -u domovoy git clone https://github.com/alexindigo/domovoy-skills.git /home/domovoy/Public/domovoy-skills
```

Domovoy keeps its own clones of both repos in `~/Public/` for reading, contributing,
and pushing changes. A separate clone of bootstrap goes to `.agents/` for runtime
identity. The store/fridge model:
```
  ~/Public/domovoy-skills/   = farmer's market (local clone of the store)
  ~/.agents/skills/           = fridge (runtime, Syncthing-synced)
  github.com/.../domovoy-skills = store (canonical, public)
```

After clone, set up Domovoy's git identity and SSH signing per the
`git-repo-identity` skill. Each Domovoy generates its own SSH key pair and uses
the key's fingerprint as a unique commit email (`<8-hex-chars>@domovoy`).
The public key is added to GitHub as an Authentication + Signing key, and a copy
goes to `setup/<hostname>/ssh.pub` for cross-machine SSH fleet access.

## 3. Create directory structure

```bash
sudo -u domovoy mkdir -p /home/domovoy/{Public,.config/opencode,.local/bin,setup/$(hostname),maintenance/{reports,tasks},models}
```

Populate the machine's environment file for fleet-specific service URLs:

```bash
sudo -u domovoy tee /home/domovoy/setup/$(hostname)/ENVIRONMENT.md <<'EOF' >/dev/null
# Environment — $(hostname)

## Network
| Setting | Value |
|---------|-------|
| LAN subnet | `<lan-subnet>` |
| DNS suffix | `.home` |

## Services
| Service | URL |
|---------|-----|
| SearXNG | `<searxng-url>` |

## Syncthing
| Setting | Value |
|---------|-------|
| Hub device ID | `<hub-device-id>` |
| Hub device name | `<hub-name>` |
EOF
```

Customize the values (SearXNG URL, LAN subnet, Syncthing hub ID) for this
household. This file syncs across the fleet via Syncthing and is read by skills
that need infrastructure URLs. It is never committed to a public repo.

## 4. Set up AGENTS.md symlink

```bash
sudo -u domovoy ln -sf .agents/AGENTS.md /home/domovoy/AGENTS.md
```

opencode reads `AGENTS.md` from its working directory. The symlink points to the cloned repo copy.

## 5. Configure opencode service

Create `/home/domovoy/.config/systemd/user/opencode.service` from the template:

```bash
sudo -u domovoy cp /home/domovoy/.agents/templates/opencode.service \
    /home/domovoy/.config/systemd/user/opencode.service
```

Or create it manually:

```ini
[Unit]
Description=opencode headless server
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/opencode serve --hostname 0.0.0.0 --port 4096
Restart=on-failure
RestartSec=5
MemoryMax=16G
WorkingDirectory=%h

[Install]
WantedBy=default.target
```

- `MemoryMax=16G` — systemd kills and restarts if memory leaks exceed 16 GB
- `WorkingDirectory=%h` — resolves to `/home/domovoy`, where AGENTS.md lives

## 6. Set up health check watchdog

Copy templates:

```bash
sudo -u domovoy cp /home/domovoy/.agents/templates/opencode-health-check \
    /home/domovoy/.local/bin/opencode-health-check
sudo -u domovoy chmod +x /home/domovoy/.local/bin/opencode-health-check
sudo -u domovoy cp /home/domovoy/.agents/templates/opencode-health.service \
    /home/domovoy/.config/systemd/user/opencode-health.service
sudo -u domovoy cp /home/domovoy/.agents/templates/opencode-health.timer \
    /home/domovoy/.config/systemd/user/opencode-health.timer
```

The health check curls `localhost:4096/health` every 30 seconds. If unresponsive for 30 seconds, it restarts the service.

## 7. Set up nightly restart timer

```bash
sudo -u domovoy cp /home/domovoy/.agents/templates/opencode-restart.service \
    /home/domovoy/.config/systemd/user/opencode-restart.service
sudo -u domovoy cp /home/domovoy/.agents/templates/opencode-restart.timer \
    /home/domovoy/.config/systemd/user/opencode-restart.timer
```

`try-restart` only restarts if running. Timer fires daily at 04:00. Prevents long-running memory leaks from accumulating.

## 8. Configure opencode provider

Edit `/home/domovoy/.config/opencode/opencode.jsonc` to set your AI provider:

```jsonc
{
    "model": {
        "provider": "anthropic",
        "name": "claude-sonnet-4-20250514"
    }
}
```

For local models, see the full guide in the domovoy-bootstrap repo.

## 9. Generate SSH key + set up Syncthing

The domovoy needs cross-machine access. Generate an SSH key and set up Syncthing
to share skills, machine profiles, and SSH public keys between instances.

### Generate SSH key

```bash
sudo -u domovoy ssh-keygen -t ed25519 -N "" \
    -C "domovoy@$(hostname)" -f /home/domovoy/.ssh/id_ed25519
sudo -u domovoy cp /home/domovoy/.ssh/id_ed25519.pub \
    /home/domovoy/setup/$(hostname)/ssh.pub
```

The public key syncs to other machines via Syncthing. The SSH gateway (below) adds it
to `authorized_keys` so every domovoy can reach every other.

### Install Syncthing

```bash
sudo pacman -S syncthing   # Arch. Other distros: use your system's package manager.
sudo -u domovoy syncthing generate --home=/home/domovoy/.config/syncthing
```

Configure custom ports (won't conflict with other syncthing instances on the same machine):

- Listen: `tcp://0.0.0.0:<sync-port>`
- Discovery: `22133`
- GUI: disabled (headless)

Edit `/home/domovoy/.config/syncthing/config.xml`:
```xml
<options>
    <listenAddress>tcp://0.0.0.0:<sync-port></listenAddress>
    <localAnnouncePort>22133</localAnnouncePort>
</options>
<gui enabled="false">...</gui>
```

Add shared folders in `config.xml`:
```xml
<folder id="domovoy-agents" label="Domovoy Skills &amp; Identity" path="/home/domovoy/.agents" type="sendreceive" .../>
<folder id="domovoy-setup" label="Domovoy Machine Profiles" path="/home/domovoy/setup" type="sendreceive" .../>
```

### Pair with existing machines

Add remote device IDs to `config.xml` on each machine. Syncthing handles discovery,
NAT traversal, and relay automatically.

### Enable Syncthing

```bash
sudo -u domovoy systemctl --user enable --now syncthing.service
```

## 10. Set up SSH key gateway

When Syncthing syncs another machine's `ssh.pub`, this gateways adds it to `authorized_keys`:

```bash
sudo -u domovoy cp /home/domovoy/.agents/templates/rebuild-authorized-keys \
    /home/domovoy/.local/bin/rebuild-authorized-keys
sudo -u domovoy chmod +x /home/domovoy/.local/bin/rebuild-authorized-keys
sudo -u domovoy cp /home/domovoy/.agents/templates/domovoy-ssh-gateway.service \
    /home/domovoy/.config/systemd/user/domovoy-ssh-gateway.service
sudo -u domovoy cp /home/domovoy/.agents/templates/domovoy-ssh-gateway.path \
    /home/domovoy/.config/systemd/user/domovoy-ssh-gateway.path
```

The path watcher monitors `~/setup/` and rebuilds `~/.ssh/authorized_keys` whenever
a new `ssh.pub` arrives. This makes every domovoy instance SSH-reachable from every other.


## 10a. Configure network access (nftables)

opencode listens on port 4096. Decide who should reach it:

```bash
# Option 1: localhost only (most secure)
# no rule needed — service binds to 0.0.0.0, nftables allows loopback by default

# Option 2: local network (enter your subnet)
nft add rule inet filter input ip saddr <subnet> tcp dport 4096 accept
# Add this to /etc/nftables.conf for persistence

# Option 3: open to all networks
nft add rule inet filter input tcp dport 4096 accept
```

The migration script asks this interactively. For a fresh install, add the rule manually.
## 11. Enable everything

```bash
sudo -u domovoy systemctl --user daemon-reload
sudo -u domovoy systemctl --user enable --now \
    opencode.service \
    opencode-health.timer \
    opencode-restart.timer \
    syncthing.service \
    domovoy-ssh-gateway.path
```

## 12. Create machine profile

```bash
sudo -u domovoy mkdir -p /home/domovoy/setup/$(hostname)
```

Create `setup/<hostname>/SYSTEM_INFO.md` with hardware specs, OS details, and storage layout.
Load the `system-info` skill for guidance on what to include.

---

## Verification checklist

- [ ] `systemctl --user status opencode.service` shows `active (running)`
- [ ] `systemctl --user list-timers` shows health, restart timers
- [ ] `curl http://localhost:4096/health` returns 200
- [ ] opencode reads AGENTS.md (skills load correctly)
- [ ] `sudo whoami` works (passwordless sudo)
- [ ] `loginctl show-user domovoy | grep Linger` shows `yes`
- [ ] `systemctl --user status syncthing.service` is `active`
- [ ] `ls /home/domovoy/setup/$(hostname)/ssh.pub` exists
- [ ] `systemctl --user status domovoy-ssh-gateway.path` shows `active (waiting)`

## What's next

The domovoy now exists with SSH keys on GitHub and both repos cloned to
`~/Public/`. Continue with **DOMOVOY_SETUP.md** (in this repo) to build the
AI agent layer: model pool architecture, llama.cpp setup, and opencode
provider configuration.

If you're migrating from an existing setup (root, a prior agent user like
`assistant`, or a regular login), see **MIGRATION.md** in this repo.

For maintenance going forward:
- Create `setup/<hostname>/SYSTEM_INFO.md` and `SYSTEM_SETUP.md` per the
  `system-info` and `system-documentation` skills
- Add remote device IDs to Syncthing config to pair machines
- Load the `syncthing-setup` skill for NAS/central-hub configuration
- See `ASSISTANT_SETUP.md` in the repo for local AI model setup
