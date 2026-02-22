# zfstic — project notes for Claude

## What this is

A single-file bash script (`zfstic`) that manages ZFS snapshot retention, inspired
by restic's CLI design. It replaces the retention/scheduling role of sanoid without
replacing syncoid (replication stays syncoid's job).

## Language rationale

Bash was chosen deliberately over Go/Python: single script, zero runtime deps,
easy to install anywhere ZFS runs. Requirements are bash 4+ and awk with strftime
(standard on all modern Linux/FreeBSD). The retention algorithm lives in one awk
invocation per dataset — awk handles the date math portably without GNU-specific
tools.

## Architecture

Single file, subcommands via `case`. No config file — everything is CLI flags,
restic-style.

```
main()             dispatch to cmd_forget / cmd_snapshot
cmd_forget()       parse flags, collect datasets, call apply_retention per dataset
cmd_snapshot()     parse flags, call zfs snapshot [-r] dataset@name
apply_retention()  zfs list | sort | awk retention kernel → print plan → zfs destroy
```

## Key design decisions

**Retention algorithm** (`apply_retention`): one `awk` pass per dataset. Snapshots
are sorted newest-first; awk walks once maintaining per-policy counters and
bucket-seen maps (strftime-keyed). A snapshot is removed only if no policy claims
it. The awk output is tab-separated: `action  snap_name  epoch  reasons  display_date`.

**Implicit keep-latest**: the newest snapshot is always kept, even if no policy
would claim it (shown as reason "latest"). Two purposes:
1. Safety: prevents accidentally wiping all snapshots via a misconfigured policy.
2. syncoid compatibility: ensures a common GUID-matched snapshot always exists on
   both local and remote after `forget` runs, so syncoid can do incremental sends.
   This only holds if syncoid runs *before* local forget (document in help text).

**`--prefix` flag**: scopes retention to only matching snapshot names. Allows
zfstic to manage its own snapshots (`--prefix zfstic-`) without touching sanoid
or manual snapshots. Essential for migration scenarios.

**`-r` on snapshot vs forget**:
- `snapshot -r` → `zfs snapshot -r` (single atomic command, ZFS-native)
- `forget -r` → per-dataset independent retention (zfs list each child separately)
These are intentionally different behaviours.

**No `zfstic replicate`**: replication is syncoid's job. It already handles
GUID-based common-snapshot detection and incremental sends. Adding a replicate
command would duplicate syncoid without improving on it.

## Snapshot naming

`zfstic-<UTC-ISO8601>[-label]`  e.g. `zfstic-2026-02-22T10:00:00-hourly`

Timestamps are UTC. Display dates in `forget` output use local timezone (awk
strftime default).

## The common-snapshot problem (important for forget design)

Unlike restic (content-addressed chunks, no chain dependency), ZFS replication is
chain-dependent. syncoid matches snapshots by GUID to find the incremental base.
If local and remote are pruned to disjoint sets (e.g. local keeps last 3 hours,
remote keeps daily at midnight), there may be no common GUID and syncoid falls
back to a full send.

The implicit keep-latest rule + running syncoid before local forget prevents this
under normal circumstances. The caveat: syncoid must run at least as often as the
local retention window. If local keeps 24h of hourly snaps but syncoid only runs
weekly, the common snapshot ages out regardless.

## Testing

No test suite currently. Manual testing on a real ZFS dataset:
1. `zfstic snapshot --label test tank/testpool` to create known snapshots
2. `zfstic forget --keep-hourly 3 --dry-run tank/testpool` to preview
3. Verify output, then run without --dry-run, confirm with `zfs list -t snapshot`

The awk retention kernel can be tested standalone by piping synthetic
`name\tepoch` lines (sorted newest-first) to it with `-v` policy variables.

## Commits and versioning

**Conventional commits** — use the type that best describes the change:

- `feat:` — new user-visible behaviour (new flag, new command, new output format)
- `fix:` — bug fix (wrong retention count, bad flag parsing, incorrect snapshot name)
- `docs:` — help text, README, CLAUDE.md only — no code change
- `refactor:` — restructure internals, no behaviour change
- `chore:` — CI, release workflow, tooling

No scope needed (single file). Keep the subject line under 72 chars, imperative
mood (`add --keep-last flag`, not `added` or `adds`).

Breaking changes: use `feat!:` or `fix!:` and add `BREAKING CHANGE: <description>`
as a footer. A breaking change is anything that changes CLI flags, output format,
or snapshot naming in a way that breaks existing scripts or cron jobs.

**Versioning** — semantic versioning (`MAJOR.MINOR.PATCH`):
- `PATCH` — bug fixes, no new flags
- `MINOR` — new flags / behaviour, backwards-compatible
- `MAJOR` — breaking CLI or output changes

The `VERSION` variable in the script is the source of truth. Bump it and the git
tag together (`git tag vX.Y.Z`). GitHub Actions publishes the script file as a
release asset on tags matching `v*`, so users can install a pinned version:

    curl -fsSL https://github.com/laermannjan/zfstic/releases/latest/download/zfstic \
        -o /usr/local/bin/zfstic && chmod +x /usr/local/bin/zfstic

## What's out of scope

- `zfstic replicate` — use syncoid
- Config file / YAML policy definitions — use a wrapper shell script + cron
- Sanoid-style automatic scheduling — use systemd timers or cron
- Hold management (`zfs hold`) — out of scope for now
- Clone-aware destroy — zfstic uses plain `zfs destroy`, which will fail if a
  snapshot has dependents (clones). This is intentional; the error is surfaced.
