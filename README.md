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

That's it. `snapshot` is essentially `zfs snapshot` with a `zfstic-` prefix and
a local timestamp - just enough structure for `forget` to work with. Snapshots
come out looking like `tank/documents@zfstic-2026-02-23T09:00:00CET`.

Before a risky operation you take an ad-hoc snapshot with a label:

```bash
zfstic snapshot --label pre-migration tank/documents
```

Today is Monday Feb 23. You ran that at 09:43, between two cron shots, before
kicking off a migration. There's also one from three days ago, Feb 21 at 14:43,
before an app upgrade:

```bash
zfstic snapshot --label app-upgrade tank/documents
```

Now run `forget` with a policy:

```bash
zfstic forget \
    --keep-last 5 --keep-daily 3 --keep-weekly 2 \
    --dry-run \
    tank/documents
```

```

  tank/documents:
    keep    zfstic-2026-02-23T11:00:00CET  [last]
    keep    zfstic-2026-02-23T10:00:00CET  [last]
    keep    zfstic-2026-02-23T09:43:00CET-pre-migration  [last]
    keep    zfstic-2026-02-23T09:00:00CET  [last]
    keep    zfstic-2026-02-23T08:00:00CET  [last]
    remove  zfstic-2026-02-23T07:00:00CET
    ...
    keep    zfstic-2026-02-23T00:00:00CET  [daily, weekly]
    remove  zfstic-2026-02-22T23:00:00CET
    ...
    keep    zfstic-2026-02-22T00:00:00CET  [daily]
    remove  zfstic-2026-02-21T23:00:00CET
    ...
    remove  zfstic-2026-02-21T14:43:00CET-app-upgrade
    ...
    keep    zfstic-2026-02-21T00:00:00CET  [daily]
    remove  zfstic-2026-02-20T23:00:00CET
    ...
    remove  zfstic-2026-02-17T00:00:00CET
    keep    zfstic-2026-02-16T00:00:00CET  [weekly]
    --- 9 keep, 185 remove --- (dry run)

```

`--keep-last 5` grabbed the five most recent snapshots. The `pre-migration` snap
squeaked in at third place.

Three days back, `app-upgrade` at 14:43 on Feb 21 didn't make it. Not in the last
5, and not the oldest snapshot of its day - the midnight cron shot is. So it's gone.

This is a deliberate design choice. For each time period, zfstic keeps the *oldest*
snapshot in it, not the newest. The reason: if it kept the newest, `--keep-daily 1
--keep-weekly 1 --keep-monthly 1 --keep-yearly 1` would all select the same
snapshot - the most recent one - and you'd end up with one snapshot no matter how
many policies you stack. By anchoring to the oldest, each policy reaches a
genuinely different point in time. This is where zfstic differs from restic.

`--keep-daily 3` means: look at the last 3 calendar days (Feb 21, 22, 23) and keep
the oldest snapshot in each. `--keep-weekly 2` does the same for the last 2 ISO
weeks. These policies are independent - they don't stack or extend each other. A
day or week with no snapshots doesn't consume a slot, so if you snapshot weekly,
`--keep-daily 7` gives you at most 7 snapshots, but they might span 7 weeks.

Today is a Monday. Feb 23 00:00 is the first snapshot of the current ISO week, so
both `--keep-daily 3` and `--keep-weekly 2` claim it. One claim is enough - it
survives once. Feb 16 is also a Monday, and the oldest snapshot of the previous
ISO week, so `--keep-weekly 2` keeps it as the anchor for that period. Everything
between Feb 17 and Feb 22 that no policy wants is gone.

Before running `forget` for real, replicate first:

```bash
syncoid tank/documents backup@nas:tank/documents
zfstic forget --keep-last 5 --keep-daily 3 --keep-weekly 2 tank/documents
```

ZFS sends are chain-dependent. syncoid finds its incremental base by matching
snapshot GUIDs between source and destination. Prune first and you might destroy
exactly the snapshot syncoid needs. zfstic guarantees the most recent snapshot
always survives `forget` - shown as `[latest]` if no other policy claims it. Since
you replicated first, that snapshot exists on both sides and syncoid can always do
an incremental send. The order in which you prune local vs remote doesn't matter -
what matters is that replication comes before either.

Once local pruning is done, apply a completely different policy on the remote:

```bash
ssh backup@nas zfstic forget \
    --keep-daily 30 --keep-monthly 12 --keep-yearly 5 \
    tank/documents
```

Local and remote policies are completely independent. That is the whole point.

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

Creates `<dataset>@zfstic-<timestamp><tz>[-label]`. The timestamp uses local time
with the system timezone abbreviation. `-r` issues a single `zfs snapshot -r`,
executed atomically by ZFS - every dataset in the tree gets an identical timestamp.

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

At least one `--keep-*` flag is required. A snapshot kept by multiple policies
shows all matching reasons. `-r` applies retention independently per dataset.

The most recently created snapshot is always kept even if no policy claims it
(shown as `[latest]`). This ensures syncoid always has a common snapshot for
incremental sends.

---

## License

[MIT](LICENSE)
