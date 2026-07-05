# BOOTSTRAP — Domovoy Setup From Scratch

Set up the `domovoy` AI agent on a fresh Linux machine. Creates a dedicated
system user (UID 588), clones the bootstrap and skill repos to `~/Public/`,
configures Syncthing for fleet sync (with domain-standard ports), installs
opencode as a user-level service with health monitoring and cross-machine SSH
access, then hands off to DOMOVOY_SETUP.md for the AI agent layer.

## 1. Create the domovoy user

```bash
useradd -r -m -d /home/domovoy -s /bin/bash -u 588 domovoy
echo "domovoy ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/domovoy
loginctl enable-linger domovoy
usermod -aG systemd-journal domovoy
```

Lingering keeps the user's systemd session alive without login — required for
timers and Syncthing. The `systemd-journal` group lets Domovoy read its own
user services' logs with `journalctl --user`.

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

## 2. Set up SSH identity + clone repos

### Generate SSH key

First, create a fleet-standard SSH key pair. Every Domovoy uses the same key name
(`id_domovoy`) so fleet-wide tools like the SSH key gateway work without custom
config per machine.

```bash
sudo -u domovoy mkdir -p /home/domovoy/.ssh /home/domovoy/setup/$(hostname)
sudo -u domovoy ssh-keygen -t ed25519 -N "" \
    -C "domovoy@$(hostname)" -f /home/domovoy/.ssh/id_domovoy
```

Create an SSH config for GitHub (the key name is non-standard, so SSH needs an
explicit `IdentityFile` directive):

```bash
cat > /home/domovoy/.ssh/config <<'EOF'
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_domovoy
EOF
chown -R domovoy:domovoy /home/domovoy/.ssh
chmod 600 /home/domovoy/.ssh/id_domovoy
chmod 644 /home/domovoy/.ssh/id_domovoy.pub /home/domovoy/.ssh/config
```

Each Domovoy derives its unique commit email from the SSH key fingerprint (first 8
base64 characters of the SHA256 hash). This email is a stable, anonymous identity
that carries across machines without leaking topology information:

```bash
FP=$(ssh-keygen -lf /home/domovoy/.ssh/id_domovoy.pub -E sha256 | awk '{print $2}')
EMAIL="${FP:7:8}@domovoy"
echo "$EMAIL"
```

Copy the public key to the machine profile so it syncs to the fleet via Syncthing:

```bash
sudo -u domovoy cp /home/domovoy/.ssh/id_domovoy.pub \
    /home/domovoy/setup/$(hostname)/ssh.pub
```

> **Add the key above to GitHub.** Go to **Settings → SSH and GPG keys** and
> add the key at `~/.ssh/id_domovoy.pub` as both an **Authentication Key** and
> a **Signing Key** (two separate entries, same key). After adding, verify:
> ```bash
> ssh -T git@github.com
> ```
> Expected output: `Hi <user>! You've successfully authenticated...`

### Clone repos

Now clone the repositories via SSH so Domovoy can push changes back:

```bash
sudo -u domovoy git clone git@github.com:alexindigo/domovoy-bootstrap.git \
    /home/domovoy/Public/domovoy-bootstrap
sudo -u domovoy git clone git@github.com:alexindigo/domovoy-skills.git \
    /home/domovoy/Public/domovoy-skills
```

Both repos follow a store/fridge model:
```
  github.com/.../domovoy-bootstrap = store (canonical, public identity + templates)
  ~/Public/domovoy-bootstrap/       = farmer's market (local clone for dev + push)
  ~/.agents/                         = fridge (runtime, Syncthing-synced across fleet)

  github.com/.../domovoy-skills    = store (canonical, public skill library)
  ~/Public/domovoy-skills/          = farmer's market (local clone for dev + push)
  ~/.agents/skills/                  = fridge (runtime, Syncthing-synced across fleet)
```

`~/.agents/` is NOT a git clone — it arrives via Syncthing from the fleet
(see step 3) or is built manually from the `~/Public/domovoy-bootstrap/`
templates for a standalone machine. Nothing clones directly into `.agents/`.

After clone, configure per-repo git identity (author name, commit email, and
SSH signing) per the `git-repo-identity` skill:

```bash
sudo -u domovoy git -C /home/domovoy/Public/domovoy-bootstrap config user.name "domovoy"
sudo -u domovoy git -C /home/domovoy/Public/domovoy-bootstrap config user.email "$EMAIL"
sudo -u domovoy git -C /home/domovoy/Public/domovoy-bootstrap config gpg.format ssh
sudo -u domovoy git -C /home/domovoy/Public/domovoy-bootstrap config user.signingkey /home/domovoy/.ssh/id_domovoy.pub
sudo -u domovoy git -C /home/domovoy/Public/domovoy-bootstrap config commit.gpgsign true
sudo -u domovoy git -C /home/domovoy/Public/domovoy-bootstrap config tag.gpgsign true
echo "$EMAIL $(cat /home/domovoy/.ssh/id_domovoy.pub)" \
    > /home/domovoy/Public/domovoy-bootstrap/.git/allowed_signers
sudo -u domovoy git -C /home/domovoy/Public/domovoy-bootstrap config gpg.ssh.allowedSignersFile .git/allowed_signers

sudo -u domovoy git -C /home/domovoy/Public/domovoy-skills config user.name "domovoy"
sudo -u domovoy git -C /home/domovoy/Public/domovoy-skills config user.email "$EMAIL"
sudo -u domovoy git -C /home/domovoy/Public/domovoy-skills config gpg.format ssh
sudo -u domovoy git -C /home/domovoy/Public/domovoy-skills config user.signingkey /home/domovoy/.ssh/id_domovoy.pub
sudo -u domovoy git -C /home/domovoy/Public/domovoy-skills config commit.gpgsign true
sudo -u domovoy git -C /home/domovoy/Public/domovoy-skills config tag.gpgsign true
echo "$EMAIL $(cat /home/domovoy/.ssh/id_domovoy.pub)" \
    > /home/domovoy/Public/domovoy-skills/.git/allowed_signers
sudo -u domovoy git -C /home/domovoy/Public/domovoy-skills config gpg.ssh.allowedSignersFile .git/allowed_signers
```

Verify the first commit signs correctly:
```bash
# If the first commit comes out unsigned (known git 2.54 quirk),
# amend and re-sign: git commit --amend -S --no-edit
```

## 3. Set up Syncthing

Syncthing is how the fleet shares identity, skills, and machine profiles. It
runs as a domovoy user-level service. If joining an existing fleet, configure
it BEFORE installing systemd services — the fleet's templates and skills arrive
through Syncthing.

### Create essential directories

These directories are needed before Syncthing starts — shared folders, service
units, and machine profiles need their paths to exist:

```bash
sudo -u domovoy mkdir -p /home/domovoy/{.agents,.config/{systemd/user,opencode},.local/bin,setup/$(hostname),maintenance/{reports,tasks},models}
```

### Install and configure

```bash
sudo pacman -S syncthing   # Arch. Other distros: use your system's package manager.
sudo -u domovoy syncthing generate --home=/home/domovoy/.config/syncthing
```

### Create the service unit

The bootstrap repo ships a `syncthing.service` template (same pattern as the other
service templates in step 5):

```bash
sudo -u domovoy cp /home/domovoy/Public/domovoy-bootstrap/templates/syncthing.service \
    /home/domovoy/.config/systemd/user/syncthing.service
```

### Configure fleet-standard ports

(won't conflict with other syncthing instances on the same machine):

- Listen: `tcp://0.0.0.0:22013`
- Discovery: `22133`
- GUI: disabled (headless)

Edit `/home/domovoy/.config/syncthing/config.xml`:
```xml
<options>
    <listenAddress>tcp://0.0.0.0:22013</listenAddress>
    <localAnnouncePort>22133</localAnnouncePort>
</options>
<gui enabled="false">...</gui>
```

Add shared folders in `config.xml`. Fleet folders should ignore permissions since
UID/GID differ across machines (`ignorePerms="true"`):

```xml
<folder id="domovoy-agents" label="Domovoy Skills &amp; Identity" path="/home/domovoy/.agents" type="sendreceive" ignorePerms="true" .../>
<folder id="domovoy-setup" label="Domovoy Machine Profiles" path="/home/domovoy/setup" type="sendreceive" ignorePerms="true" .../>
```

### Pair with existing machines

Add remote device IDs to `config.xml` on each machine. Syncthing handles discovery,
NAT traversal, and relay automatically.

### Enable Syncthing

```bash
sudo -u domovoy XDG_RUNTIME_DIR=/run/user/588 systemctl --user enable --now syncthing.service
```

Wait for the shared folders to reach "Up to Date" with the fleet before continuing.

## 4. Set up AGENTS.md symlink

On a fleet machine, `~/.agents/AGENTS.md` arrives via Syncthing. On a standalone
machine, copy it from the bootstrap repo:

```bash
# Fleet: no action — Syncthing brings .agents/AGENTS.md
# Standalone:
sudo -u domovoy cp /home/domovoy/Public/domovoy-bootstrap/AGENTS.md /home/domovoy/.agents/AGENTS.md
```

Then create the symlink opencode reads:
```bash
sudo -u domovoy ln -sf .agents/AGENTS.md /home/domovoy/AGENTS.md
```

## 5. Install systemd services

Service templates live in the bootstrap repo at `~/Public/domovoy-bootstrap/templates/`.

### opencode service

```bash
sudo -u domovoy cp /home/domovoy/Public/domovoy-bootstrap/templates/opencode.service \
    /home/domovoy/.config/systemd/user/opencode.service
```

The template:
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

### Health check watchdog

```bash
sudo -u domovoy cp /home/domovoy/Public/domovoy-bootstrap/templates/opencode-health-check \
    /home/domovoy/.local/bin/opencode-health-check
sudo -u domovoy chmod +x /home/domovoy/.local/bin/opencode-health-check
sudo -u domovoy cp /home/domovoy/Public/domovoy-bootstrap/templates/opencode-health.service \
    /home/domovoy/.config/systemd/user/opencode-health.service
sudo -u domovoy cp /home/domovoy/Public/domovoy-bootstrap/templates/opencode-health.timer \
    /home/domovoy/.config/systemd/user/opencode-health.timer
```

The health check curls `localhost:4096/health` every 30 seconds. If
unresponsive for 30 seconds, it restarts the service.

### Nightly restart timer

```bash
sudo -u domovoy cp /home/domovoy/Public/domovoy-bootstrap/templates/opencode-restart.service \
    /home/domovoy/.config/systemd/user/opencode-restart.service
sudo -u domovoy cp /home/domovoy/Public/domovoy-bootstrap/templates/opencode-restart.timer \
    /home/domovoy/.config/systemd/user/opencode-restart.timer
```

`try-restart` only restarts if running. Timer fires daily at 04:00. Prevents
long-running memory leaks from accumulating.

### SSH key gateway

When Syncthing syncs another machine's `ssh.pub`, this gateway adds it to
`authorized_keys`:

```bash
sudo -u domovoy cp /home/domovoy/Public/domovoy-bootstrap/templates/rebuild-authorized-keys \
    /home/domovoy/.local/bin/rebuild-authorized-keys
sudo -u domovoy chmod +x /home/domovoy/.local/bin/rebuild-authorized-keys
sudo -u domovoy cp /home/domovoy/Public/domovoy-bootstrap/templates/domovoy-ssh-gateway.service \
    /home/domovoy/.config/systemd/user/domovoy-ssh-gateway.service
sudo -u domovoy cp /home/domovoy/Public/domovoy-bootstrap/templates/domovoy-ssh-gateway.path \
    /home/domovoy/.config/systemd/user/domovoy-ssh-gateway.path
```

The path watcher monitors `~/setup/` and rebuilds `~/.ssh/authorized_keys`
whenever a new `ssh.pub` arrives. This makes every domovoy instance
SSH-reachable from every other.

> **Cross-user service management:** `systemctl --user` commands invoked
> from root (or any account that isn't domovoy itself) require the inline
> `XDG_RUNTIME_DIR` pattern shown in the code blocks below. See the
> `domovoy-scripts` skill for the full conventions — trigger scripts,
> rollback, error handling, and ownership rules.

### Enable everything

```bash
sudo -u domovoy XDG_RUNTIME_DIR=/run/user/588 systemctl --user daemon-reload
sudo -u domovoy XDG_RUNTIME_DIR=/run/user/588 systemctl --user enable --now \
    opencode.service \
    opencode-health.timer \
    opencode-restart.timer \
    syncthing.service \
    domovoy-ssh-gateway.path
```

## 6. Migration (if replacing a prior agent)

If this machine previously ran opencode under another user (root, a prior agent
account like `assistant`, or a human user), see **MIGRATION.md** in this repo
for the full procedure. The migration preserves sessions, auth, and identity
while switching to `domovoy`. Skip this step for a fresh machine.

## 7. Switch to domovoy

Bootstrap is complete. Reconnect your opencode client to `domovoy`'s opencode
service (port 4096).

---

**BOOTSTRAP ENDS HERE.** The operating system and user-level services are ready.
Continue with **DOMOVOY_SETUP.md** for the AI agent layer: machine profiles,
local model pool, and opencode provider configuration.

---

## Verification checklist

- [ ] `systemctl --user status opencode.service` shows `active (running)`
- [ ] `systemctl --user list-timers` shows health, restart timers armed
- [ ] `curl http://localhost:4096/health` returns 200
- [ ] `sudo whoami` works (passwordless sudo)
- [ ] `loginctl show-user domovoy | grep Linger` shows `yes`
- [ ] `systemctl --user status syncthing.service` is `active`
- [ ] Syncthing folders "Up to Date" (if fleet)
- [ ] `systemctl --user status domovoy-ssh-gateway.path` shows `active (waiting)`
- [ ] opencode reads AGENTS.md (skills load correctly)

## What's next

- `DOMOVOY_SETUP.md` — machine profiles, model pool, opencode config
- `MIGRATION.md` — if moving from a prior agent user
- `syncthing-setup` skill — NAS/central-hub versioning configuration
- `git-repo-identity` skill — per-repo git identity with SSH commit signing
