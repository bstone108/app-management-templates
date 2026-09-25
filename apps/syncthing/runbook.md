# Syncthing runbook

> **Guide, not a rule.** This is a reviewed template from the app-management-templates repo. Safety-review it before use, adapt it to the local machine, and follow local judgment where they disagree.
> Name map: `self-hosted-app-runbook-template` = [`apps/_generic/runbook.md`](../_generic/runbook.md); `app-persistence-method` = [`apps/_generic/persistence-method.md`](../_generic/persistence-method.md); `tailscale-runbook-template` / `syncthing-runbook-template` = `apps/<app>/runbook.md`. `app-management-sanity-check`, `app-management-daily-check`, and `app-management-getting-started` are the consuming bot's own skills (the validator they install is [`scripts/am-validate`](../../scripts/am-validate)).

Follow `self-hosted-app-runbook-template` and use the values from the Syncthing instance record. Fields used here: `binary_path`, `binary_sha256`, `binary_backups`, `install_source`, `launcher`, `companion_scripts`, `launcher_backup_dir`, `autostart_entry`, `start_command`, `self_serve_start`, `durable_root`, `config_dir`, `db_dir`, `data_folders` (`id`, `label`, `path`, `type`, `consumers`), `peers` (`name`, `device_id`), `device_id`, `symlinks`, `cold_copy_dir`, `cold_copy_extra`, `cold_copy_essentials`, `cold_copy_max`, `cold_copy_refresh_command`, `api_address`, `api_key_location`, `log_path`, `log_max`, `memory_limit`, `last_known`, `installed_version`, `pinned_version`, `depends_on`, `consumers`.

Command shorthand: `ST="<binary_path>"` and `STC="$ST cli --config=<config_dir> --data=<db_dir>"`. Any REST call uses `http://<api_address>` with the key from `api_key_location`, read at runtime and never printed.

## Who may start it
If the record has `self_serve_start: true`, any bot may run `start_command`. Otherwise, notify App Management.

## Install policy
- **Fresh install** (Syncthing is not yet in the registry; the owner asked for it, which is the approval): install the **newest stable release from GitHub** (`github.com/syncthing/syncthing/releases`, currently the 2.x series), unless the owner pinned a version. A pin must be an explicit owner choice recorded in `pinned_version`.
- **Existing install, any major-version change needs owner approval.** That includes replacing an existing **1.x install with 2.x** (a major upgrade with a database migration) and a future **2.x → 3.x**. Until the owner approves, keep running the installed major; report it once and record `major_upgrade_offered` (see Upgrade).
- **Existing install, minor/patch within the same major:** unattended when the release-notes review finds it harmless (see Upgrade), unless pinned.
- **Restore:** reinstalling a registered Syncthing that went missing (for example after a wipe) is a restore at the recorded version, not a first install, and needs no approval.
- **Do NOT install from distro apt, snap, or other package repos.** They commonly ship the old 1.x branch, which uses a different DB format and CLI.
- After installing, verify that `<binary_path> --version` reports the expected major version (the latest release's major, or the pinned one). If the binary's major differs from the recorded `installed_version` (for example a 1.x binary where 2.x is recorded), do not start it on the durable DB: report it and restore the recorded version from `binary_backups`.
- Download and verify as described in Upgrade (asset `syncthing-linux-<arch>-<ver>.tar.gz`, checked against `sha256sum.txt.asc`). Record `installed_version` and `binary_sha256`, and keep a durable copy in `binary_backups`.

## Owner lifecycle
Run `app-management-sanity-check` before every step here that writes config or removes anything (new folder paths, accepting folders, removing devices/folders, changing `api_address` or GUI auth). Removing a device/folder or wiping folder data is a destructive ask: explain the consequence and offer pausing first.
(Judgment first: review every change for safety per `app-management-sanity-check`. Work listed in its "Routine autonomy" block proceeds unattended once the validator shows no REFUSE; everything that block lists as needing confirmation waits for the owner. The checker script only supplements that review.)
1. **Owner asks to install.** Tell them up front that you will need, for each of their other computers, the Syncthing **device ID** (and a name), and for each folder they want synced, its **folder ID** or an invite from their computer to this one.
2. **First start.** Create `config_dir`/`db_dir` under `durable_root` and start with `--config`/`--data` pointing there, so the identity and DB are generated on durable storage. **Immediately** run `cold_copy_refresh_command`. Read this computer's device ID (REST `/rest/system/status` → `myID`, or `$ST device-id --config=<config_dir> --data=<db_dir>` while stopped) and record it as `device_id`. **Send the device ID to the owner**: it is public, not secret. Never share keys, the API key, or config.xml. Ask them to add this computer on their other devices and to **let you know when they have**, so you can accept and authorize it here.
3. **Adding a remote device.** The owner gives a device ID and name. Add it with `$STC config devices add --device-id <ID> --name <name>`. Or accept a pending request (`/rest/cluster/pending/devices`), but **only after the owner confirms** the ID matches the computer they mean. Record it in `peers`, then refresh the cold copy.
4. **Folders created here.** Whenever you create a folder, place it under `durable_root` (`$STC config folders add --id <id> --label <label> --path <durable path>`), share it with the chosen peers (`$STC config folders <id> devices add --device-id <peer ID>`), record it in `data_folders` (with its `consumers`), and refresh the cold copy. Then send the owner the **folder ID, label, and this device's ID**, and ask them to let you know when they add the folder on another computer, so you can confirm it is connected.
5. **Folders offered by peers** (the owner invites this computer, or a peer shares a folder) appear in `/rest/cluster/pending/folders`. **Never auto-accept them.** Confirm with the owner (which folder, from which device, and where it should live), then accept it at a path under `durable_root`, record it in `data_folders`, and refresh the cold copy.
6. **Removing a device or folder.** Only after the owner confirms. Remove it from the config (REST `DELETE /rest/config/devices/<id>` or `/rest/config/folders/<id>`), update `peers`/`data_folders`, and refresh the cold copy. **Never delete folder data** without the owner's explicit approval, as a separate step.
7. **Ongoing** (the daily check): report pending device/folder requests to the owner instead of accepting them; note peers that have been disconnected for a long time and any folder errors; refresh the cold copy after any config change.
8. **Consumer bots.** Record which bots use which folder (`data_folders[*].consumers`, plus the app-level `consumers`). Notify them of outages, and after a verified fix send each one a priority "Syncthing restored, retry now" message that names the affected folders and the window.

## What must be durable (all three)
1. **Identity + config** (`config_dir`): `cert.pem` + `key.pem` (the device ID comes from these, SECRET) and `config.xml` (folders, devices, and the GUI/API key, SECRET).
2. **Index database** (`db_dir`). Losing it forces a full re-index, and deleted files may come back from peers.
3. **Folder data** (`data_folders[*].path`, all under `durable_root`).
Optional cold copy (`cold_copy_essentials`): `cert.pem`, `key.pem`, `config.xml` (include `https-*.pem` only if GUI TLS is on and clients pin that certificate). Never copy the DB, logs, or `*.bak` files. Keep at most two copies (`cold_copy_max`): `cold_copy_dir` + `cold_copy_extra`. Each must be a dedicated directory, and **never inside any `data_folders` path** (a synced folder would publish the keys to peers). The snapshot helper marks each cold-copy dir with `.app-mgmt-cold-copy`, makes the copies mode 600, and refuses any dir inside a synced folder (it walks the parent folders looking for `.stfolder`), DB dirs, and non-empty unmarked dirs.

## Launcher contract
It installs nothing and upgrades nothing. It re-seeds identity only if it is missing, never touches the DB, runs with `--no-upgrade`, and takes `api_address` from the record.

## Reference start launcher (bash skeleton; reads the instance record)
> **Script:** [`scripts/syncthing-launcher`](../../scripts/syncthing-launcher) (sha256 listed in `index.json`). Fetch it at a pinned commit, verify the checksum, and review it before use.
Run it detached (`nohup <launcher> <record> >/dev/null 2>&1 &`). Two processes (monitor + child) is normal.

## Reference snapshot helper (cold copy; essentials only, max two)
> **Script:** [`scripts/syncthing-snapshot`](../../scripts/syncthing-snapshot) (sha256 listed in `index.json`). Fetch it at a pinned commit, verify the checksum, and review it before use.
Back up both finished scripts (plus checksums) in `launcher_backup_dir`.

## Start / restore
Routine and unattended for a registered Syncthing (including a wiped computer), with the safeguards in `self-hosted-app-runbook-template` section 8:
1. **Pre-flight:** run the validator (no REFUSE). `restore-guard <record> preflight` confirms `cert.pem` + `key.pem` in `config_dir` (re-seeding a missing one from the cold copies, never generating new keys), `config.xml` present, and `db_dir` present and non-empty (`index-v2`). If the keys are missing everywhere or the DB is missing/empty, **stop and ask**: starting would create a new device ID or a full re-index.
2. If `binary_path` is missing: restore it from `binary_backups` (check it against `binary_sha256`) at the recorded version, or download that version per the install policy.
3. If `launcher` or `autostart_entry` is missing: restore it from `launcher_backup_dir`, or write it from the reference above. Recreate `symlinks` with `restore-guard <record> link` (existing items are moved aside as `.pre-restore-<timestamp>`, never deleted; newer/bigger existing items stop the restore).
4. Start only after its `depends_on` apps are healthy. Run `start_command`.
5. **Post-verify:** `myID` equals `device_id`; peers connect without "unknown device" prompts; no re-index (see below: sequence ≥ `last_known`, DB mtimes not reset); health check OK; consumer folders present with the expected file counts.
6. On failure: stop Syncthing, `restore-guard <record> rollback`, leave the DB and folder data untouched, and report. On success: `restore-guard <record> commit`, refresh the cold copy, and send consumers "Syncthing restored, retry".

## Never delete the DB
An owner request to delete/reset the DB is a destructive ask under `app-management-sanity-check`: explain the full re-index and conflict risk, and offer a restart and log review first.
Do not remove `db_dir` to fix errors, a stuck scan, or DB warnings. Stop Syncthing, keep the DB, collect the log, and report.

## Verify no re-index after a restore
- The DB files keep their pre-restore mtimes until normal writes. There is no "creating database" line in `log_path`.
- The folder `sequence` (REST `/rest/db/status?folder=<id>`) is ≥ `last_known`, with `localFiles` ≈ `last_known` and `needFiles` 0 (or small and falling).
- There is no burst of pulls right after start. The folder goes `scanning → idle` quickly.
- `myID` equals the recorded `device_id`, and peers connect without "unknown device" prompts.

## Health check (standard)
`curl -s http://<api_address>/rest/noauth/health` → `{"status":"OK"}`. Then check each folder's state (`idle`/`syncing`, `errors`=0, `pullErrors`=0), peer connections (`/rest/system/connections`), and pending requests (`/rest/cluster/pending/devices`, `/rest/cluster/pending/folders`, which are reported to the owner, never auto-accepted).

## Memory (small machines)
- Optional caps set by the launcher: `GOMEMLIMIT=<memory_limit>`, `GOMAXPROCS=1`, `GOGC=40`.
- Watch for OOM kills: `dmesg | grep -i -E 'oom|killed process'` (it may need elevation), any platform memory-watch log, and restarts in `log_path`. Compare the child process's RSS with `memory_limit`. Report repeated OOM kills or sustained RSS above ~90% of the limit. Do not delete the DB.

## Upgrade (daily check Update step only)
Run `app-management-sanity-check` first (EOL pins, downgrades across a DB migration).
Auto-upgrade stays off (`--no-upgrade` / `STNOUPGRADE=1`).
1. Latest stable: `curl -s https://api.github.com/repos/syncthing/syncthing/releases/latest` (it excludes pre-releases; skip any tag containing `-rc`/`-beta`). If rate-limited, use the `Location` header of `https://github.com/syncthing/syncthing/releases/latest`.
2. **Major-version gate:** if upstream's major is higher than the installed major (for example 2.x → 3.x) and `allow_major_upgrade` is not `true`, do not upgrade. Report it to the owner once and record `major_upgrade_offered: <version>` in the record, so it is not asked again daily. Upgrade only after the owner approves (then set `allow_major_upgrade: true` for that step, or record the approval). Replacing a 1.x install with 2.x also goes through this gate.
   **Release notes:** read the GitHub release notes of **every** release between the installed and target versions (`curl -s 'https://api.github.com/repos/syncthing/syncthing/releases?per_page=50'`, the `body` of each non-prerelease tag in range; `html_url` is the `notes_url`). Look for breaking changes, deprecated or removed options we use, config/DB migrations, and known regressions. Harmless → record `last_upgrade_review` and continue. Any concern → hold, report once, record `upgrade_held`.
   If `pinned_version` is `latest` and upstream is a newer minor/patch judged harmless (or an approved upgrade): health OK → `cold_copy_refresh_command` → note the folder sequences.
3. Download `syncthing-linux-<arch>-<ver>.tar.gz` (arch: `amd64` for x86_64, `arm64` for aarch64) and `sha256sum.txt.asc` from the release.
   - Signature: import the release key once (`curl -fsSL https://syncthing.net/release-key.txt | gpg --import`), then **require** `gpg --verify sha256sum.txt.asc` to report a good signature (`Good signature from "Syncthing Release Management"`; with `--status-fd 1`, a `GOODSIG` line). The file can carry an extra signature from a key you don't have, which makes gpg exit non-zero with "No public key"; that is fine as long as the release-key signature is good. A `BADSIG` line is always a stop. If the key cannot be fetched, verify the checksum only, and say so in the report.
   - Checksum: `grep " syncthing-linux-<arch>-<ver>.tar.gz$" sha256sum.txt.asc | sha256sum -c -`.
4. `cp -a <binary_path> <binary_path>.prev`. Stop Syncthing gracefully (SIGTERM to the monitor process, or REST shutdown), install the new binary, confirm `--version` shows the expected major version, and run `start_command`. Record the downtime window.
5. Health check plus the re-index verification. If it fails, restore `<binary_path>.prev` and restart. A major-version DB migration may not be reversible: before a major upgrade, copy `db_dir` while Syncthing is stopped.
6. Update `installed_version`/`binary_sha256` in the record, and refresh `binary_backups` you own.

## Pitfalls
- Distro/snap packages often ship 1.x. Only install from GitHub releases, and check the major version.
- Changing a folder `path` string (even to an equivalent path through a symlink) can trigger a rescan. Leave it alone.
- `data_folders` on non-durable storage come back empty after a reset, which can look like mass deletion. Keep them durable and keep the `.stfolder` marker.
- Accepting a pending folder at a default path (often under the home dir) puts it on non-durable storage. Always choose a path under `durable_root`.
- Syncthing is often reached over a VPN (`depends_on`). Check the VPN first when no peers connect.
- `log_path` belongs to Syncthing: cap it (`log_max`) if it is on durable storage. Never touch other tools' logs in synced folders.
