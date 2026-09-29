# my-bb

The **true** status of Backblaze Computer Backup on a Mac, and control of it
from the command line.

Backblaze's window can say *Paused* while the backend is running, *0
remaining* while the backend counts hundreds, and its menu bar helper can die
without a word. The backend's own state files and logs, under
`/Library/Backblaze.bzpkg/bzdata`, tell the truth every time. my-bb reads
them, shows each fact with the raw value and the file it came from, and
changes things only through Backblaze's own command line tool, `bzcli`. It
never writes into Backblaze's data directory itself.

```
last backup   2026-09-29 03:27:05 (1h46 ago)               [bzstat_lastbackupcompleted]
remaining     176,402 files / 1.1 TB                       [bzstat_remainingbackup 05:11]
                  123,638   684.6 GB  /Volumes/stuff
                   48,712   366.1 GB  /Volumes/backup
                      233    45.4 GB  /Volumes/2
                    3,819   910.8 MB  /
initial       complete 2026-09-28 17:58:28                 [bzstat_endfirstbackupmillis]
pass          running 1h15: transmitting                   [bztransmit pid 52900]
paused        no                                           [no pauseinfo.xml]
drives        4: / /Volumes/2 /Volumes/backup /Volumes/stuff [bzvolumes, seen 04:34]
unreadable    none                                         [bzlist_skipped_files 04:27]
inherit       done 27.09, 4.9 GB of pieces left in bzinherit/ (14 .ibs) [bz_ibs_progress]
access        yes_none                                     [bzstat_bzserv_perm]
account       license billing_active, not frozen           [status.json, bzdc_synchostinfo]
uploads       20260929 (UTC): 0 ok, 0 failed               [bzstat_upload_success]
client        10.0.3.1076, bzserv up 1d9h, heartbeat 0h01 ago [ps, bzserv_heartbeat]
(a sample of 12 remaining file names: -V)
```

A condition that needs attention is printed to **stderr**, with its raw value,
and one summary line ends the status:

```
last backup   2026-09-26 16:30:40 (2d3h ago)   STALE: last backup 2d3h ago >26h   [bzstat_lastbackupcompleted]
!!! 1 condition(s) hold: stale [exit 1]
```

## Usage

```
my-bb [-S] [-Q] [-V|-VV] [-D|-DD] [--config <FILE>] [<VERB> [<ARG>]]
```

| verb | what it does |
|---|---|
| *(none)* | the status |
| `start` | start a backup now; this also ends a pause (confirmed: the pause file is gone) |
| `pause` | pause the backup (confirmed: Backblaze's pause file appears) |
| `exclude <DIR>` | exclude a folder, through `bzcli`; confirmed in `bzinfo.xml` |
| `no-exclude <DIR>` | remove that exclusion, matched case-insensitively against the stored spelling |
| `exclusions` | list the excluded folders |
| `inherit` | guide an *Inherit Backup State*, then watch it live |
| `inherit cancel` | forget my-bb's own inherit state; Backblaze's inherit goes on |
| `check` | the problem check with mail, as the LaunchAgent runs it |
| `diag [<BLOCK>]` | the building blocks one by one; a bare `diag` lists them |

Modifiers: `-S` prints the `bzcli` / `launchctl` command a verb would run and
runs nothing; `-Q` prints only conditions and errors; `-V` / `-VV` add detail
(file names, the event log, the exclusion count); `-D` / `-DD` add
diagnostics.

Admin: `--install` / `--uninstall` (the LaunchAgent that runs `check`),
`--create-config [<FILE>]`, `--test-mail`, `--version`, `--run-tests
[<FILTER>]`.

Exit status: **0** healthy, **1** a condition holds, **2** my-bb could not
tell (an error, printed as `ERROR <code>`).

## What the status knows, and from where

- **Remaining** is shown per drive: Backblaze keeps exact counts only per
  drive. The file names it writes (`bzlist_filesremaining.txt`) are a short
  sample, and `-V` labels them as one ("sample of 13 of 187,936").
- **Unreadable** files: the count and reason come from the newest pass's
  `bztransmit` log line (`skipping_DDD: <REASON> - N files`); the names from
  `bzlist_skipped_files.txt` when it lists them, otherwise the first ten the
  log names, labelled "first 10 of 850". A log line older than the current
  skipped-files report belongs to a pass that is over and is not counted.
- **Paused**: `pauseinfo.xml` exists only while paused, with the reason and
  the end of the pause.
- **Inherit**: `bz_ibs_progress.xml` shows the progress, but it is not trusted
  on its own. It has been seen to stop at "600/1000" after a complete success
  when the helper feeding it died. The `bztransmit` log decides the result,
  and with no inherit process running, a progress file that has not changed
  for `INHERIT_STALL_AFTER_IN_MINUTES` is reported as stalled, not watched.
  Downloaded pieces left in `bzinherit/` after an inherit are reported as a
  fact; removing them is up to you.
- Backblaze's logs are in **UTC**; my-bb shows local time.

## Conditions and mail

`CONDITIONS` in the config lists the conditions that count. One list decides
everything: flagging in the status, exit status 1, and the mail. A condition
left out of it is still shown as a fact, just never flagged.

| condition | holds when |
|---|---|
| `stale` | the last completed backup is older than `LAST_BACKUP_MAX_AGE_IN_HOURS` |
| `service` | bzserv's heartbeat is older than `SERVICE_HEARTBEAT_MAX_AGE_IN_MINUTES` |
| `paused` | paused for longer than `PAUSE_MAX_DURATION_IN_HOURS` |
| `helper` | the menu bar helper (bzbmenu) is not running |
| `unreadable` | the newest pass skipped files it could not read |
| `drives` | a drive has not been seen for `DRIVE_MAX_ABSENCE_IN_DAYS` (Backblaze deletes a drive's backup after 30 days away) |
| `freeze` | the account is in Safety Freeze |
| `license` | the license status is not `billing_active` |
| `access` | bzserv lacks Full Disk Access |
| `upload` | uploads failed today |
| `initial` | the first backup is not complete after `INITIAL_BACKUP_MAX_DELAY_IN_DAYS` |
| `check` | the installed LaunchAgent has not run `check` for twice its interval |

With `MAIL_TO` set, `check` sends **one** mail when a condition starts to hold
and **one** `RESOLVED` mail when it clears. The mail goes out through

```
"$NOTIFY_CMD" --mail "$MAIL_TO" <subject> <body>
```

and counts as sent only when that command exits 0: a failed send is retried by
the next check, never dropped. `--test-mail` sends one mail through exactly
that path.

`--install` writes a LaunchAgent (`LAUNCHAGENT_LABEL`) that runs `my-bb check`
every `CHECK_INTERVAL_IN_MINUTES`, loads it and confirms launchd knows it.

## Install

Put `my-bb` in your `PATH` (it is a `#!/bin/dash` script, macOS only). It runs
as your user: everything under Backblaze's data directory is world-readable,
and `bzcli` needs no root either.

my-bb needs a config and says where to write one when none is found:

```
my-bb --create-config > ~/.my-bb.conf
```

Every key is commented there. The search order, first found wins:
`$MY_BB_CONFIG`, `--config <FILE>`, `/LINKS/default/my-bb.conf` (or
`/LINKS/default/my-bb`), `~/.my-bb.conf`, `/etc/my-bb.conf`,
`/usr/local/etc/my-bb.conf`.

`my-bb --run-tests` runs the built-in suite against generated Backblaze data
trees and stubs: no Backblaze, no mail, no launchd.
