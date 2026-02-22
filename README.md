# zfstic

ZFS snapshot management inspired by [restic](https://restic.net/)'s CLI design.

**Why?** [sanoid](https://github.com/jimsalterjrs/sanoid) couples *creation policy* to *retention policy*: if you want monthly snapshots kept on a remote backup, you must also keep monthly snapshots locally. zfstic breaks that coupling — take snapshots at whatever frequency you like, then apply completely independent retention policies locally and remotely.

---

## Requirements

- bash 4+
- `awk` with `strftime` support (gawk, mawk, or any modern BSD awk)
- `zfs` in PATH

## Installation

```bash
# Latest release (recommended — pinned, reproducible)
curl -fsSL https://github.com/laermannjan/zfstic/releases/latest/download/zfstic \
    -o /usr/local/bin/zfstic && chmod +x /usr/local/bin/zfstic
```

Or from source:

```bash
git clone https://github.com/laermannjan/zfstic
sudo install -m 755 zfstic/zfstic /usr/local/bin/zfstic
```

---

## Commands

### `zfstic snapshot`

Create a new ZFS snapshot with a consistent naming convention.

```
zfstic snapshot [options] <dataset>

  --label <name>   Append a label  →  @zfstic-2026-02-22T10:00:00-<label>
  -r, --recursive  Atomic recursive snapshot (zfs snapshot -r)
  -n, --dry-run    Print what would be created without doing it
```

Snapshots are named `@zfstic-<UTC-timestamp>[-label]`. The consistent prefix lets
`zfstic forget --prefix zfstic-` scope retention to only zfstic-managed snapshots,
leaving any sanoid or manual snapshots untouched.

```bash
zfstic snapshot tank/data
# → tank/data@zfstic-2026-02-22T10:00:00

zfstic snapshot --label hourly -r tank
# → tank@zfstic-2026-02-22T10:00:00-hourly  (and all children, atomically)
```

### `zfstic forget`

Apply a retention policy to a dataset's snapshots. Prints what will be kept and
removed; use `--dry-run` to preview without destroying anything.

```
zfstic forget [options] <dataset>

  --keep-last    n    Keep the n most recent snapshots unconditionally
  --keep-hourly  n    Keep the most recent snapshot per hour, up to n hours
  --keep-daily   n    Keep the most recent snapshot per day, up to n days
  --keep-weekly  n    Keep the most recent snapshot per ISO week, up to n weeks
  --keep-monthly n    Keep the most recent snapshot per month, up to n months
  --keep-yearly  n    Keep the most recent snapshot per year, up to n years
  -r, --recursive     Apply to dataset and all child datasets independently
  -n, --dry-run       Show plan without destroying anything
  --prefix <pfx>      Only consider snapshots whose name starts with <pfx>
```

At least one `--keep-*` flag is required. A snapshot satisfying multiple policies
is kept once and shows all matching reasons — identical behaviour to restic.

**The most recently created snapshot is always kept regardless of policy.** This
is a hard safety rule that also ensures syncoid always has a common base snapshot
for incremental replication (see [Using with syncoid](#using-with-syncoid) below).

```bash
# Preview: keep 24 hourly + 7 daily on tank/data
zfstic forget --keep-hourly 24 --keep-daily 7 --dry-run tank/data

# Apply recursively — each child dataset is evaluated independently
zfstic forget --keep-hourly 24 --keep-daily 7 -r tank

# Only manage zfstic-created snapshots; leave sanoid/manual ones alone
zfstic forget --keep-daily 7 --prefix zfstic- tank/data
```

Example output:

```
tank/data
────────────────────────────────────────────────────────────
Keeping 3 of 5 snapshot(s)

  To keep:
    2026-02-22 10:00:00  zfstic-2026-02-22T10:00:00-hourly  last, hourly
    2026-02-22 09:00:00  zfstic-2026-02-22T09:00:00-hourly  hourly
    2026-02-22 08:00:00  zfstic-2026-02-22T08:00:00-hourly  hourly

  To remove:
    2026-02-22 07:00:00  zfstic-2026-02-22T07:00:00-hourly
    2026-02-22 06:00:00  zfstic-2026-02-22T06:00:00-hourly
```

---

## Using with syncoid

zfstic handles retention only. Use [syncoid](https://github.com/jimsalterjrs/sanoid)
for replication — it already does incremental ZFS send/receive well (GUID-based
common-snapshot detection, SSH transport, resume support).

The key workflow that solves the sanoid limitation:

```bash
#!/usr/bin/env bash
# Run hourly via cron or systemd timer

# 1. Create snapshot
zfstic snapshot --label hourly tank/data

# 2. Replicate BEFORE forgetting locally
syncoid tank/data backup@remote:tank/backup

# 3. Prune local — aggressive short-term retention
zfstic forget tank/data --keep-hourly 24 --keep-daily 7 --prefix zfstic-

# 4. Prune remote — independent long-term retention
ssh backup@remote zfstic forget tank/backup \
    --keep-daily 30 --keep-monthly 12 --keep-yearly 5 --prefix zfstic-
```

Local and remote retention policies are now completely independent.

### The common-snapshot constraint

ZFS incremental replication is chain-dependent: syncoid finds the incremental base
by matching snapshot GUIDs on both sides. If local and remote are pruned to disjoint
snapshot sets, syncoid must fall back to a full send.

zfstic's implicit keep-latest rule prevents this in normal operation: the newest
local snapshot always survives `forget`, and since syncoid ran before the local
forget, that snapshot exists on the remote with the same GUID.

**One caveat:** syncoid must run at least as often as your local retention window.
If you keep 24 hourly snapshots locally but only run syncoid weekly, the common
snapshot will eventually age out of the local window and syncoid will need to
do a full send. Design your schedule so that syncoid frequency ≥ local retention
frequency.

---

## Recursive snapshots: atomic vs per-dataset

`zfstic snapshot -r tank` issues a single `zfs snapshot -r` command, which ZFS
executes atomically — all datasets in the tree get an identical timestamp. This is
equivalent to sanoid's `recursive = zfs` and is the correct default for consistency
(e.g. a database spanning multiple child datasets).

`zfstic forget -r tank` applies retention *independently per dataset*, so different
subtrees can have different effective retention even if they share snapshot names.

---

## License

[MIT](LICENSE)
