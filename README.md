# Assistant Bootstrap

Bootstrap an AI assistant agent on any Linux machine. Creates a dedicated non-root `assistant` user, configures opencode as a user-level service with health monitoring, and sets up cross-machine sync via Syncthing with an SSH key gateway so assistants can reach each other.

- **BOOTSTRAP.md** — fresh setup from scratch on a new machine
- **MIGRATION.md** — migrate an existing root-based opencode to the assistant user
- **migrate.sh.template** — migration script template (the assistant agent fills in placeholders, then the user runs it)
- **AGENTS.md** — reference identity and safety rules (merge into your own, never blindly replace)
- **templates/** — ready-to-use systemd units and scripts
