# BOOTSTRAP — Assistant Setup From Scratch

Set up the `assistant` AI agent on a fresh Linux machine. Creates a dedicated non-root user, clones this repo, and configures opencode as a user-level service with health monitoring, cross-machine SSH access, and Syncthing sync.

## 1. Create the assistant user

```bash
useradd -m -s /bin/bash assistant
echo "assistant ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/assistant
loginctl enable-linger assistant
```

Lingering keeps the user's systemd session alive without login — required for timers and Syncthing.

## 2. Clone this repo

```bash
sudo -u assistant git clone https://github.com/alexindigo/assistant-bootstrap.git /home/assistant/.agents
```

The repo provides: AGENTS.md (identity), skill definitions, and templates.

## 3. Create directory structure

```bash
sudo -u assistant mkdir -p /home/assistant/{.config/opencode,.local/bin,setup/$(hostname),maintenance/{reports,tasks},models}
```

## 4. Set up AGENTS.md symlink

```bash
sudo -u assistant ln -sf .agents/AGENTS.md /home/assistant/AGENTS.md
```

opencode reads `AGENTS.md` from its working directory. The symlink points to the cloned repo copy.

## 5. Configure opencode service

Create `/home/assistant/.config/systemd/user/opencode.service` from the template:

```bash
sudo -u assistant cp /home/assistant/.agents/templates/opencode.service \
    /home/assistant/.config/systemd/user/opencode.service
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
- `WorkingDirectory=%h` — resolves to `/home/assistant`, where AGENTS.md lives

## 6. Set up health check watchdog

Copy templates:

```bash
sudo -u assistant cp /home/assistant/.agents/templates/opencode-health-check \
    /home/assistant/.local/bin/opencode-health-check
sudo -u assistant chmod +x /home/assistant/.local/bin/opencode-health-check
sudo -u assistant cp /home/assistant/.agents/templates/opencode-health.service \
    /home/assistant/.config/systemd/user/opencode-health.service
sudo -u assistant cp /home/assistant/.agents/templates/opencode-health.timer \
    /home/assistant/.config/systemd/user/opencode-health.timer
```

The health check curls `localhost:4096/health` every 30 seconds. If unresponsive for 30 seconds, it restarts the service.

## 7. Set up nightly restart timer

```bash
sudo -u assistant cp /home/assistant/.agents/templates/opencode-restart.service \
    /home/assistant/.config/systemd/user/opencode-restart.service
sudo -u assistant cp /home/assistant/.agents/templates/opencode-restart.timer \
    /home/assistant/.config/systemd/user/opencode-restart.timer
```

`try-restart` only restarts if running. Timer fires daily at 04:00. Prevents long-running memory leaks from accumulating.

## 8. Configure opencode provider

Edit `/home/assistant/.config/opencode/opencode.jsonc` to set your AI provider:

```jsonc
{
    "model": {
        "provider": "anthropic",
        "name": "claude-sonnet-4-20250514"
    }
}
```

For local models, see the full guide in the assistant-bootstrap repo.

## 9. Generate SSH key + set up Syncthing

The assistant needs cross-machine access. Generate an SSH key and set up Syncthing
to share skills, machine profiles, and SSH public keys between instances.

### Generate SSH key

```bash
sudo -u assistant ssh-keygen -t ed25519 -N "" \
    -C "assistant@$(hostname)" -f /home/assistant/.ssh/id_ed25519
sudo -u assistant cp /home/assistant/.ssh/id_ed25519.pub \
    /home/assistant/setup/$(hostname)/ssh.pub
```

The public key syncs to other machines via Syncthing. The SSH gateway (below) adds it
to `authorized_keys` so every assistant can reach every other.

### Install Syncthing

```bash
sudo pacman -S syncthing   # Arch. Other distros: use your system's package manager.
sudo -u assistant syncthing generate --home=/home/assistant/.config/syncthing
```

Configure custom ports (won't conflict with other syncthing instances on the same machine):

- Listen: `tcp://0.0.0.0:22013`
- Discovery: `22133`
- GUI: disabled (headless)

Edit `/home/assistant/.config/syncthing/config.xml`:
```xml
<options>
    <listenAddress>tcp://0.0.0.0:22013</listenAddress>
    <localAnnouncePort>22133</localAnnouncePort>
</options>
<gui enabled="false">...</gui>
```

Add shared folders in `config.xml`:
```xml
<folder id="assistant-agents" label="Assistant Skills" path="/home/assistant/.agents" type="sendreceive" .../>
<folder id="assistant-setup" label="Assistant Profiles" path="/home/assistant/setup" type="sendreceive" .../>
```

### Pair with existing machines

Add remote device IDs to `config.xml` on each machine. Syncthing handles discovery,
NAT traversal, and relay automatically.

### Enable Syncthing

```bash
sudo -u assistant systemctl --user enable --now syncthing.service
```

## 10. Set up SSH key gateway

When Syncthing syncs another machine's `ssh.pub`, this gateways adds it to `authorized_keys`:

```bash
sudo -u assistant cp /home/assistant/.agents/templates/rebuild-authorized-keys \
    /home/assistant/.local/bin/rebuild-authorized-keys
sudo -u assistant chmod +x /home/assistant/.local/bin/rebuild-authorized-keys
sudo -u assistant cp /home/assistant/.agents/templates/assistant-ssh-gateway.service \
    /home/assistant/.config/systemd/user/assistant-ssh-gateway.service
sudo -u assistant cp /home/assistant/.agents/templates/assistant-ssh-gateway.path \
    /home/assistant/.config/systemd/user/assistant-ssh-gateway.path
```

The path watcher monitors `~/setup/` and rebuilds `~/.ssh/authorized_keys` whenever
a new `ssh.pub` arrives. This makes every assistant instance SSH-reachable from every other.


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
sudo -u assistant systemctl --user daemon-reload
sudo -u assistant systemctl --user enable --now \
    opencode.service \
    opencode-health.timer \
    opencode-restart.timer \
    syncthing.service \
    assistant-ssh-gateway.path
```

## 12. Create machine profile

```bash
sudo -u assistant mkdir -p /home/assistant/setup/$(hostname)
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
- [ ] `loginctl show-user assistant | grep Linger` shows `yes`
- [ ] `systemctl --user status syncthing.service` is `active`
- [ ] `ls /home/assistant/setup/$(hostname)/ssh.pub` exists
- [ ] `systemctl --user status assistant-ssh-gateway.path` shows `active (waiting)`

## What's next

- Add remote device IDs to Syncthing config to pair machines
- Create `SYSTEM_SETUP.md` — the rebuild blueprint for this machine
- See `ASSISTANT_SETUP.md` in the repo for local AI model setup
