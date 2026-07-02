# Domovoy Bootstrap

Bootstrap a [Domovoy](https://github.com/alexindigo/domovoy-bootstrap) — a personal
household daemon / AI sysadmin agent — on any Linux machine. Creates a dedicated system
user `domovoy` (UID 588), configures opencode as a user-level service with health
monitoring, and sets up cross-machine sync via Syncthing with an SSH key gateway so
Domovoys can reach each other.

- **BOOTSTRAP.md** — fresh setup from scratch on a new machine
- **MIGRATION.md** — migrate an existing opencode setup to the domovoy user (from root, a prior agent user like `assistant`, or a regular login)
- **migrate.sh.template** — migration script template (the domovoy agent fills in placeholders, then the user runs it)
- **AGENTS.md** — reference identity and safety rules (merge into your own, never blindly replace)
- **templates/** — ready-to-use systemd units and scripts
