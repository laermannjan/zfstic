# zfstic

ZFS snapshot retention manager inspired by [restic](https://restic.net/)'s CLI design.

---

## Why

I snapshot my documents dataset every hour. Locally I keep 24 hours of hourlies
and a few dailies: enough to undo a bad afternoon, or recover a file I made and
deleted the same morning. I also take ad-hoc snapshots before anything uncertain
(reorganizing a project, running a migration) so I can roll back cleanly. Short
retention by design: the fewer snapshots you keep, the sooner ZFS can reclaim
space after you delete something large.

On the backup server I want the opposite: daily snapshots for a month, weeklies
for a year, monthlies as far back as makes sense.

sanoid can't express this. Its retention is tied to how snapshots are created.
Want monthly snapshots on the remote? You need them locally too. You end up with
the full retention ladder on both sides whether you want it or not.

restic doesn't have this problem. You back up as often as you like and prune
independently. The same backup counts as the hourly, the daily, and the monthly.
It only disappears when no policy wants it.

zfstic is that model for ZFS.

---

## Requirements

- bash 4+
- `awk` with `strftime` and `systime` support (gawk, mawk, or any modern BSD awk)
- `zfs` in PATH

---

## Installation

```bash
curl -fsSL https://github.com/laermannjan/zfstic/releases/latest/download/zfstic \
    -o /usr/local/bin/zfstic && chmod +x /usr/local/bin/zfstic
```

---

## Walkthrough

Your cron job runs once an hour:

```
0 * * * * zfstic snapshot tank/documents
```

`snapshot` is `zfs snapshot` with a `zfstic-` prefix and a local timestamp.
Snapshots look like `tank/documents@zfstic-2026-02-23T09:00:00CET`.

Before a risky operation, add a label:

```bash
zfstic snapshot --label pre-migration tank/documents
```

Now apply a policy:

```bash
zfstic forget \
    --keep-last 5 --keep-daily 3 --keep-weekly 2 \
    --dry-run \
    tank/documents
```

```

  tank/documents:
    keep    zfstic-2026-02-23T11:00:00CET                [last]
    keep    zfstic-2026-02-23T10:00:00CET                [last]
    keep    zfstic-2026-02-23T09:43:00CET-pre-migration  [last]
    keep    zfstic-2026-02-23T09:00:00CET                [last]
    keep    zfstic-2026-02-23T08:00:00CET                [last]
    remove  zfstic-2026-02-23T07:00:00CET
    ...
    keep    zfstic-2026-02-23T00:00:00CET                [daily, weekly]
    remove  zfstic-2026-02-22T23:00:00CET
    ...
    keep    zfstic-2026-02-22T00:00:00CET                [daily]
    remove  zfstic-2026-02-21T23:00:00CET
    ...
    remove  zfstic-2026-02-21T13:17:00CET-app-upgrade
    ...
    keep    zfstic-2026-02-21T00:00:00CET                [daily]
    remove  zfstic-2026-02-20T23:00:00CET
    ...
    remove  zfstic-2026-02-17T00:00:00CET
    keep    zfstic-2026-02-16T00:00:00CET                [weekly]
    --- 9 keep, 185 remove --- (dry run)

```

`pre-migration` survived because `--keep-last 5` is unconditional. `app-upgrade`
at 13:17 on 2026-02-21 didn't - there's a midnight snapshot that day, and zfstic
keeps the *oldest* snapshot per period.

That's the core design. Keeping the oldest ensures each policy anchors to a
distinct point in time. If zfstic kept the newest instead, `--keep-daily 1
--keep-weekly 1 --keep-monthly 1` would all converge on the same snapshot.

`--keep-daily 3` covers the last 3 calendar days. `--keep-weekly 2` covers the
last 2 ISO weeks. Policies don't stack or extend each other - days with no
snapshots contribute nothing. Today is Monday, so 2026-02-23 00:00 qualifies for
both and shows `[daily, weekly]`; it survives once. 2026-02-16, the oldest
snapshot of the previous ISO week, is kept by `--keep-weekly 2`. Everything in
between that no policy touches is gone.

---

ZFS snapshots are independent - each is a complete view of the filesystem state
at a point in time. They share blocks (copy-on-write) but are not dependent on
each other. Incremental sends work by birth transaction group: every block is
stamped with the txg in which it was written. A send from point A to snapshot B
walks B's block tree and ships only blocks born after A. O(changed data), not
O(dataset size).

The source only needs a reference to point A - a snapshot or a bookmark (just a
GUID and txg, no data). The destination needs the actual snapshot at A, because
the incremental stream omits blocks already present there and the destination has
to reconstruct B from A plus the delta. Without that base snapshot on the remote,
the stream can't be applied and a full send is required.

By default, syncoid manages this automatically: before each send it creates a
`syncoid_`-prefixed snapshot on both source and destination, and deletes old ones.
These serve as the common anchor for incremental sends. zfstic's default
`--prefix zfstic-` means `forget` won't touch them - they're invisible to
retention policies. This is the simplest setup and the one we recommend.

For lighter source-side footprint, `syncoid --create-bookmark --no-sync-snap`
creates a bookmark on the source instead of a `syncoid_` snapshot - no sync
snapshot is created or managed on the source at all. The destination still needs
its snapshot - and it's always there: `forget` never removes the youngest snapshot,
which is exactly the one the source's bookmark points to. You can also manage
everything manually with `zfs snapshot`, `zfs bookmark`, and `zfs send` if you
prefer full control.

What still matters is coverage: if the remote should hold daily snapshots, local
must have those daily snapshots at replication time. `forget` always keeps the
newest snapshot regardless of policy, so the remote's anchor is preserved
automatically. For the source, replicate before running `forget` to ensure remote
receives all intermediate snapshots before any are pruned.

```bash
syncoid tank/documents backup@nas:tank/documents
zfstic forget --keep-last 5 --keep-daily 7 tank/documents
ssh backup@nas zfstic forget \
    --keep-daily 30 --keep-monthly 12 --keep-yearly 5 \
    tank/documents
```

The policies on each side are independent. That's the whole point. One cron job
takes snapshots, another replicates and prunes both sides. Each cycle is
self-contained.

---

## Commands

### `zfstic snapshot`

```
zfstic snapshot [options] <dataset>

  --label <name>     Append a label: @zfstic-2026-02-23T10:00:00CET-<label>
  --timezone <tz>    Use this timezone instead of local (e.g. UTC, America/New_York)
  -r, --recursive    Atomic recursive snapshot (zfs snapshot -r)
  -h, --help         Show help
```

Creates `<dataset>@zfstic-<timestamp><tz>[-label]`. `-r` issues a single
`zfs snapshot -r`, executed atomically by ZFS.

### `zfstic forget`

```
zfstic forget [options] <dataset>

  --keep-last    n    Keep the n most recent snapshots unconditionally
  --keep-hourly  n    Keep the oldest snapshot per hour for the last n hours
  --keep-daily   n    Keep the oldest snapshot per day for the last n days
  --keep-weekly  n    Keep the oldest snapshot per ISO week for the last n weeks
  --keep-monthly n    Keep the oldest snapshot per month for the last n months
  --keep-yearly  n    Keep the oldest snapshot per year for the last n years
  -r, --recursive     Apply to dataset and all child datasets independently
  -n, --dry-run       Show plan without destroying anything
  --prefix <pfx>      Only consider snapshots whose name starts with <pfx>
                      Default: "zfstic-". Use --prefix "" to manage all snapshots.
  --timezone <tz>     Timezone for computing period boundaries (e.g. UTC, America/New_York)
                      Default: system local time
  -h, --help          Show help
```

At least one `--keep-*` flag is required. Multiple matching policies are all
listed as reasons. `-r` applies retention independently per dataset.

The newest snapshot is always kept regardless of policy (shown as `[latest]`).
Safety net - not a substitute for correct replication policy design.
