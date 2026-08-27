# delta-induction

A 10-script Bash system built as a Linux system-administration induction task for **Delta Force, NIT Trichy**. It implements a role-based multi-user "game" on a single Linux box — three roles (**wardens**, **guards**, **bashers**) with different privileges, enforced entirely through native Linux primitives: Unix groups, ACLs, cron/`at`, background daemons, and shell hooks. No application code, no database — just the OS.

## Concept

Three roles are provisioned as real Linux users/groups from a `roster.yaml` manifest:

| Role | Analogy | Powers |
|---|---|---|
| **Wardens** | admins | Run privileged scripts (`verifyHeist`, `trendSetters`, `wipeTimeline`), read all logs |
| **Guards** | moderators | Read-only visibility into tax/audit logs via ACLs |
| **Bashers** | regular users | Sandboxed home directories, penalized for command overuse |

## Scripts

| Script | Purpose |
|---|---|
| `initRoster` | Parses `roster.yaml` with `yq`, creates the three Unix groups, creates a home directory + user per entry, installs each person's `public_key` into `~/.ssh/authorized_keys` |
| `secureVault` | Creates `/opt/Bashrot_vault`, owned `root:wardens`, `chmod 770`, then layers POSIX ACLs so `guards` get read access without joining the `wardens` group |
| `noCapSecurity` | Background daemon that seeds the vault with thousands of decoy symlinks (`TOTAL_SYMLINKS=6767`) pointing away from the real file, and rotates them every 45 minutes — a moving-target defense so brute-force `ls`/`cat` sweeps of the vault can't find the real content |
| `generateLore` | Root-only script that pulls random entries from `slang.txt` and drops "lore" content into the vault's hidden subdirectory, feeding the decoy system above |
| `collectTax` | Walks every basher's home directory, computes usage (`MAX_SIZE` threshold), and appends a permissioned log (`chmod 750` + ACLs for `guards`/`wardens`) — a disk-quota-style audit trail |
| `auditTax` | Read-only report over `collectTax`'s log, gated so only `guards`/`wardens` group members can invoke it |
| `lPenalty` | Hooked into each basher's `.bashrc` via `PROMPT_COMMAND`; counts commands run and, past a threshold (`THRESHOLD=100`), drops the user into a restricted shell (`rbash`) for a cooldown period (`RBASH_DURATION=1800`) |
| `verifyHeist` | Warden-only script (checks group membership via `id -nG`) that "verifies" a basher's attempt to retrieve the real file from the vault, logging results to `/var/log/heist.log` |
| `trendSetters` | Reads `heist.log` and computes a leaderboard with streak multipliers and clutch bonuses, comparing against the previous run to show rank changes |
| `wipeTimeline` | Teardown script: kills the `generateLore` and `noCapSecurity` daemons via their PID files, cleans the vault and basher homes |

## What it demonstrates

- **Access control**: Unix groups + POSIX ACLs (`setfacl`) layered for fine-grained, non-hierarchical permissions (guards can read what bashers can't, without full warden rights)
- **Privilege enforcement in scripts**: every privileged script checks `$EUID` and validates group membership via `id -nG "$SUDO_USER"` before doing anything
- **Background daemons**: PID-file-tracked long-running processes (`noCapSecurity`, `generateLore`) that can be cleanly started and killed
- **Shell hooks**: `PROMPT_COMMAND` used to intercept every command a restricted user runs
- **YAML-driven provisioning**: `yq` used to turn a declarative roster into real system state (users, groups, SSH keys)
- **Scheduled/idempotent execution**: `groupadd -f`, safe re-runs, log rotation patterns

## Usage

All scripts require root:

```bash
sudo ./scripts/initRoster      # provision users/groups from roster.yaml
sudo ./scripts/secureVault     # create the ACL-protected vault
sudo ./scripts/noCapSecurity & # start the decoy daemon
sudo ./scripts/generateLore    # seed decoy content
sudo ./scripts/collectTax      # audit basher disk usage
sudo ./scripts/verifyHeist     # (as a warden) verify a heist attempt
sudo ./scripts/trendSetters    # (as a warden) view the leaderboard
sudo ./scripts/wipeTimeline    # teardown everything
```

## Files

- `roster.yaml` — declarative list of wardens/guards/bashers with usernames and SSH public keys
- `slang.txt` — word list consumed by `generateLore` for decoy content
- `scripts/` — the 10 scripts above
