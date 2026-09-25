# Make a self-hosted app survive computer resets

> **Guide, not a rule.** This is a reviewed template from the app-management-templates repo. Safety-review it before use, adapt it to the local machine, and follow local judgment where they disagree.
> Name map: `self-hosted-app-runbook-template` = [`apps/_generic/runbook.md`](../_generic/runbook.md); `app-persistence-method` = [`apps/_generic/persistence-method.md`](../_generic/persistence-method.md); `tailscale-runbook-template` / `syncthing-runbook-template` = `apps/<app>/runbook.md`. `app-management-sanity-check`, `app-management-daily-check`, and `app-management-getting-started` are the consuming bot's own skills (the validator they install is [`scripts/am-validate`](../../scripts/am-validate)).

Field names follow the vocabulary in `self-hosted-app-runbook-template`.

## 0. Learn the ground truth first
- Ask the owner, or read their notes, to learn where durable storage is (`durable_root`). Look for evidence: `findmnt`, `df`, platform docs, restore logs, and whether files there survived a previous reset. Treat nothing as durable without evidence.
- Read the existing launchers/notes for the app. Other agents may depend on the current paths. Never break them. If a script already handles persistence (flags that point at durable paths, cold copies), document it rather than duplicating it.

## 1. Identify what must persist
For the app, list:
- **config** (settings files)
- **identity** (keys, certificates, node/device state, auth tokens). Losing these means a new identity or a re-login.
- **state/DB** (indexes, sqlite/leveldb, caches that are expensive to rebuild or dangerous to lose, such as sync indexes)
- **data** (user folders the app manages)
- **binary + launcher** (can be reinstalled, but record how)
Find them from the app's `--help`/docs, running process args (`ps -o args`), open files (`ls -l /proc/<pid>/fd`), and default dirs (`~/.config/<app>`, `~/.local/state/<app>`, `/var/lib/<app>`).

## 2. Choose the mechanism (prefer the least invasive)
The **primary** persistence mechanism is: live config/identity/state/DB/data live on durable storage, and the app reaches them through a flag or a link. Cold copies (step 4) are an optional extra, never a replacement.
1. **App flag/env** that points at a durable path (`--statedir`, `--config`, `--data`, `XDG_*`). This is best: no symlinks.
2. **Symlink** from the default path to the durable path.
3. **Bind mount**, only if the first two can't work.

## 3. Move (only if it currently lives on non-durable storage)
Run `app-management-sanity-check` first: the target must not be inside a synced folder, another app's dir, a system dir, or `durable_root` itself. Moving live data to a new durable location for the first time changes the owner's paths, so ask first. Re-linking to an existing durable copy (for example after a wipe) is routine: use `restore-guard <record> link` from `self-hosted-app-runbook-template` section 8, which moves any existing item aside instead of deleting it.
1. Record the baseline: identity (node/device id), version, DB file mtimes, sequence counters, and file counts.
2. Stop the app cleanly (its own stop command, or SIGTERM, then wait for the process to exit).
3. Copy with metadata into `durable_root`: `cp -a` or `rsync -aHAX`, then verify (`diff -r` / checksums).
4. Rename the original to `<path>.pre-persist` (do NOT delete it), then create the symlink (`ln -s <durable> <path>`) or change the flag.
5. Fix ownership and permissions: identity dirs `700`, key files `600`, owned by the app's run user.
6. Start the app.

## 4. Optional: keep a minimal cold identity copy
- It is a nice-to-have. Durable live files (step 2) are what restores the app.
- **Essentials only** (`cold_copy_essentials`): the smallest set of identity/state files needed to come back without re-login or a new identity (for example, a node state file, or a certificate + key + main config). **No** logs, caches, databases, runtime files (pid/sockets), or nested copies. Do not tar whole state directories.
- **At most two copies per app** (`cold_copy_max: 2`): either current + previous in one private folder (rotate: when the live file changes, move current to `.prev`, write the new current, and delete anything older), or one current copy in each of two locations. Delete anything beyond that.
- The same limit of two applies to restore leftovers (`.pre-restore-<timestamp>` items and restore journals; see the backup limit in `self-hosted-app-runbook-template`). Never put them into a cold copy.
- Store copies in a private secrets folder (`cold_copy_dir`, mode 700, files 600), marked with `.app-mgmt-cold-copy`. Run `app-management-sanity-check` before choosing or changing it: never inside a synced folder or a state dir.
  (Judgment first: review every change for safety per `app-management-sanity-check`. Work listed in its "Routine autonomy" block proceeds unattended once the validator shows no REFUSE; everything that block lists as needing confirmation waits for the owner. The checker script only supplements that review.)
- Refresh only after a health check passes, and verify each copy (for example with `cmp`) before deleting the older one. Automate this through `cold_copy_refresh_command`.
- The launcher should re-seed from the cold copy when live identity is missing, and never generate a new identity while a cold copy exists.

## 4b. Logs
- Keep app logs out of identity/state directories and out of cold copies. Cap or rotate the logs the app owns (for example, truncate in place above a size limit and keep one small tail), or put them on non-durable storage.
- Never back up, move, rotate, or delete logs the app does not own (for example, logs another agent collected on purpose).

## 5. Verify: same identity, no re-login, no re-index
- The node/device id matches the baseline, and there is no duplicate device on the peer/coordination side.
- There was no login prompt. If one appears, the restore failed: stop, put back the `.pre-persist` or the cold copy, and retry.
- The DB continued: mtimes did not reset, counters are ≥ baseline, and there were no mass pulls or downloads.
- Health check passes.
- After a few days of healthy operation, `.pre-persist` copies can be removed, **but only with the owner's approval**.

## 6. Make it reproducible
- Make sure a launcher exists that installs only if missing (never upgrades), re-seeds identity from the cold copy if live identity is missing, and starts the app on the durable paths. If none exists, write one from the reference launcher in the app's runbook skill.
- Save launcher copies plus a `SHA256SUMS.txt` covering exactly those files in the durable launcher backup dir (no secret values in scripts).
- Write or update the app's instance record (instance values only, using the fields from the runbook template, including `self_serve_start`), and add the app to the registry so the daily check covers it.

## Never
- Delete data, reset identity, or force a re-login to "make it work". Stop and ask (the sanity check's destructive-ask rules).
- Copy secret values into docs, reports, or chat. Refer to locations only.
- Keep more than two cold copies, or put logs/caches/nested directories into them.
