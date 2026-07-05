# MIGRATION — Move opencode to the `domovoy` user

This guide migrates an existing opencode setup to a dedicated **`domovoy` system
user** (UID 588) with its own home, user-level service, health monitoring, and
cross-machine Syncthing sync.

It covers **three source scenarios** — migrate FROM:

| Source | Typical signs | Strategy |
|--------|---------------|----------|
| **`root`** | opencode config in `/root/`, runs as a system service or one-off | **copy** root → domovoy (§A) |
| **a prior agent user** (e.g. `assistant`, or any earlier name) | opencode runs as a dedicated non-domovoy user with its own home | **copy-then-cutover** (§B) — the safest, reversible path |
| **a regular human `user`** | opencode runs under your normal login account | **copy** selected opencode data → domovoy (§C) |

The Domovoy identity (`domovoy`, UID 588) is the same regardless of source. Only
the *source* differs. Read the scenario that matches you.

---

## Core principles (all scenarios)

1. **The agent prepares; the human executes the cutover.** opencode runs *inside*
   itself — it cannot stop/restart its own service. The agent audits, copies
   durable data, and writes a cutover script; **you** run that script from a root
   TTY while opencode is stopped, then reconnect.
2. **Copy, don't move (until verified).** Keep the source intact as a fallback
   until the new `domovoy` is confirmed working. Delete the old identity only
   after verification.
3. **Nothing is lost.** All session data (conversations, auth, database, settings)
   is preserved. AGENTS.md is *merged*, never blindly overwritten.
4. **System user.** `domovoy` is created with `useradd -r ... -u 588` (see
   BOOTSTRAP.md for the shell-choice rationale: `bash` if it will collaborate with
   other Domovoys over SSH, else `nologin`).

### What gets migrated (opencode data — common to all sources)

| Data | Destination |
|------|-------------|
| AGENTS.md / identity | `/home/domovoy/.agents/AGENTS.md` → symlinked to `~/AGENTS.md` (merged) |
| skills, agents, commands, modes, plugins, tools, themes | `/home/domovoy/.agents/` |
| `opencode.jsonc` provider config | `/home/domovoy/.config/opencode/` |
| sessions, auth, database | `/home/domovoy/.local/share/opencode/` |
| runtime state | `/home/domovoy/.local/state/opencode/` |
| cache | `/home/domovoy/.cache/opencode/` |
| machine profiles / docs | `/home/domovoy/setup/<hostname>/` |
| maintenance reports | `/home/domovoy/maintenance/reports/` |
| service + timers | user-level under `domovoy` (health check, nightly restart) |

**Regenerable, do NOT migrate** (discard; they rebuild on first run): `.npm`,
`.cache/*` (non-opencode), compiler/build temp (`/tmp/...`).

---

## §A — Migrate FROM `root`

opencode was set up as root with config under `/root/`.

1. **Audit** `/root/` for the data above; check whether opencode runs as a
   system service or a one-off process.
2. **Merge AGENTS.md** — preserve the operator's conventions (hostname context,
   AUR policy, pronouns, custom rules); add the essential safety rules from this
   repo's AGENTS.md; rewrite paths `/root/...` → `~/...` and identity `root` →
   `domovoy`.
3. **Fill `migrate.sh.template`** (`__FILL_HOSTNAME__`, `EXTRA_FILES`, merge
   notes) → `migrate.sh`. Present to the human.
4. **Human:** stop opencode (`systemctl stop opencode.service` or
   `kill $(pgrep -f opencode)`), then `sudo bash migrate.sh`.
5. The script creates `domovoy`, copies files, installs the user service +
   timers, sets up Syncthing + SSH gateway, and prompts to remove old `/root/`
   files (backups remain under `/home/domovoy/`).

---

## §B — Migrate FROM a prior agent user (e.g. `assistant`)

The machine already runs opencode as a dedicated agent user with its own home
(e.g. `assistant` at `/home/assistant`). This is a **rename+renumber**, and the
safest approach is **copy-then-cutover** — the old user stays a live fallback
until the new one is verified. (This is the exact path Domovoy itself took from
its original `assistant` identity.)

### Phase A — Create domovoy + copy durable data (agent does this; old user keeps running)

- `useradd -r -m -d /home/domovoy -s <shell> -u 588 -U domovoy`; sudoers; linger.
- **Copy** (not move) durable data → `/home/domovoy`, chown 588:
  `.agents/`, `setup/`, `maintenance/`, `docs/`, `.config/systemd/user/`,
  `.local/bin/`, `.local/state/`, bash rc files, the `AGENTS.md` symlink.
- **Fix enabled-unit `.wants` symlinks** — they may be absolute, pointing back at
  the old home; re-point them to `/home/domovoy/...` own unit files.
- **Defer** large, not-in-use dirs (`models/`, `aur/`) to after cutover (move
  them then — instant rename if on the same filesystem).

### Phase A (Syncthing) — device-ID carryover

If the old user ran Syncthing, keep the **same device identity** so the hub/peers
see the same device, not a new one:

- Copy the Syncthing config dir **including `key.pem` + `cert.pem`** to
  `/home/domovoy/.config/syncthing/`, chown 588.
- Rewrite `config.xml`: folder **paths** → `/home/domovoy/...`; optionally folder
  **IDs**/labels (e.g. `assistant-*` → `domovoy-*`).
- **Never run two Syncthing instances with the same device ID at once.** Stop the
  old one, `disable` it (so it can't auto-start on reboot), then start domovoy's.
- Update the hub/NAS to the new folders (block-hashing → no real re-transfer).
- See the `syncthing-setup` skill for the leaf/hub + versioning architecture.

### Phase C — Cutover (HUMAN runs from a root TTY; opencode is stopped)

A script (prepared by the agent) that does the brief-downtime switch:
1. Stop the old user's opencode + timers (+ Syncthing if not already moved).
2. Copy the **live** opencode state fresh → domovoy:
   `.local/share/opencode/` (sessions/auth/db), `.local/state/opencode/`,
   `.config/opencode/`, `.cache/opencode/`, `.local/share/opentui/`.
3. chown domovoy home to 588.
4. Enable + start domovoy's Syncthing + opencode + timers.
5. Reconnect the client — now served by `domovoy`.

> Controlling per-user services from root, and all other cross-user
> service management: see the `domovoy-scripts` skill for the mandatory
> inline `XDG_RUNTIME_DIR` pattern, error handling, rollback, trigger
> script conventions, and ownership rules.

### Phase D — Verify, then remove the old user (agent does this as domovoy)

- Verify: skills load, sessions/auth present, opencode + syncthing active, timers
  armed, port listening, Syncthing "Up to Date" with the hub.
- **Move** deferred `models/`/`aur/`; **reconcile** any files the old user's home
  still holds that are newer than the copies (e.g. maintenance reports written
  during the migration) — copy the newer version over before deleting.
- Clean regenerables + temp.
- Audit for stray files: `find / -xdev -uid <old_uid>` across all mounts.
- Teardown: stop old services, `loginctl disable-linger <old>`,
  `userdel -r <old>`, remove `/etc/sudoers.d/<old>` + linger file.

**UID note:** the old agent user may have had a regular UID (≥1000). `domovoy` is
a **system UID (588)**. Because we *copy* into a fresh `domovoy` home and chown to
588 as we go, there's no in-place renumber; just verify no files remain owned by
the old UID before deleting the old user.

---

## §C — Migrate FROM a regular human `user`

opencode ran under your normal login account. You are NOT removing that account —
you are extracting opencode into its own `domovoy` identity so it stops living in
your personal home.

1. **Create** `domovoy` (§ system user, as above).
2. **Copy** only opencode's data (the table at the top) from `~/.config/opencode`,
   `~/.local/share/opencode`, `~/.local/state/opencode`, `~/.cache/opencode`,
   `~/.agents` (or `~/.opencode`), plus any `setup/`, `maintenance/` you keep,
   into `/home/domovoy`, chown 588. Do NOT copy unrelated personal files.
3. **Merge AGENTS.md** as in §A.
4. **Cutover:** stop opencode in your account; start it as the `domovoy` user
   service; reconnect.
5. **Do NOT delete your human account.** Just remove the opencode bits from it if
   you want a clean split (optional). Your login stays intact.

---

## Firewall configuration (all scenarios)

After migration, configure opencode's network access to taste. **Never expose a
listening port without deciding this deliberately.**

| Mode | Who can connect | nftables rule | When to use |
|------|----------------|--------------|-------------|
| **1 — localhost** | Only this machine | loopback `tcp dport 4096 accept` | SSH tunnel is your entry point |
| **2 — local network** | Devices on your LAN | `ip saddr <subnet> tcp dport 4096 accept` | Connect from home machines |
| **3 — open** | Anyone | `tcp dport 4096 accept` | Cloud/remote (understand the risk) |

Add the chosen rule to `/etc/nftables.conf` for persistence. Add later with:
```bash
nft add rule inet filter input ip saddr <subnet> tcp dport 4096 accept
```

---

## After migration

- Review the merged AGENTS.md and iterate.
- **Name your Domovoy:** on first session it may offer to take a personal name;
  the system identity stays `domovoy`, only the display name changes.
- Syncthing is running — pair remote device IDs; set File Versioning on the
  central hub/NAS (see `syncthing-setup` skill), not on leaves.
- The Domovoy now runs as a system user with health monitoring and nightly
  restarts.
