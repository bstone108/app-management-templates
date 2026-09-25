# Generic self-hosted app runbook

> **Guide, not a rule.** This is a reviewed template from the app-management-templates repo. Safety-review it before use, adapt it to the local machine, and follow local judgment where they disagree.
> Name map: `self-hosted-app-runbook-template` = [`apps/_generic/runbook.md`](../_generic/runbook.md); `app-persistence-method` = [`apps/_generic/persistence-method.md`](../_generic/persistence-method.md); `tailscale-runbook-template` / `syncthing-runbook-template` = `apps/<app>/runbook.md`. `app-management-sanity-check`, `app-management-daily-check`, and `app-management-getting-started` are the consuming bot's own skills (the validator they install is [`scripts/am-validate`](../../scripts/am-validate)).

This runbook has no installation-specific values. Every `field_name` below comes from the app's **instance record** (`<app_dir>/apps/<app>.md` YAML block, with the summary row in `<app_dir>/registry.json`). The records hold values only. Procedures and generic commands live in the skills. If a field is missing, find its value, add it to the record, and then continue.

**Sanity gate:** before saving any record/registry value, before carrying out an owner instruction, and before any step below that writes config or deletes/moves files (sections 2, 4, 7, 8, 8a, 8b), run `app-management-sanity-check`. A REFUSE finding stops that step for this app. A WARN about the step itself (its path, exposure, version, or data) needs the owner's explicit confirmation, recorded in `owner_confirmed_risks`; other WARNs are reported, and routine repair or upgrade continues per "Routine autonomy" in `app-management-sanity-check`. Never silently change an owner's setting: propose the fix.
(Judgment first: review every change for safety per `app-management-sanity-check`. Work listed in its "Routine autonomy" block proceeds unattended once the validator shows no REFUSE; everything that block lists as needing confirmation waits for the owner. The checker script only supplements that review.)

## Config field vocabulary
| Field | Meaning |
|---|---|
| `name` | app id |
| `consumers` | bots/people/apps that use it (notify them on outage/restore) |
| `depends_on` | other managed apps that must be healthy first |
| `self_serve_start` | `true` = any bot may run `start_command` when the app is down. `false` (default) = bots notify App Management instead |
| `durable_root` | storage that survives computer resets/updates |
| `app_dir` | the app-management folder under `durable_root` (registry, records, launchers) |
| `install_method` / `install_source` | apt / static-binary / container / pip …, and the upstream URL/repo |
| `binary_path` / `binary_sha256` / `binary_backups` | where the executable lives, its checksum, durable copies |
| `launcher` / `companion_scripts` / `launcher_fallbacks` | wrapper scripts that install-if-missing, restore, and start the app |
| `launcher_backup_dir` | durable copies of the launchers (with `SHA256SUMS.txt`) |
| `autostart_entry` | optional desktop/service autostart file |
| `start_command` | the one command that starts (and, where supported, restores) the app. **It never upgrades** |
| `run_mode` | how it runs (for example `kernel`/`userspace`, user/root) |
| `config_dir` / `state_dir` / `state_file` / `db_dir` | identity, config, runtime state, database |
| `socket_path` | control socket (pass it to every CLI call) |
| `node_name` | the expected node/host name on its network |
| `api_address` / `api_key_location` | local API/GUI listen address, and where its key is stored (read at runtime, never print it) |
| `data_folders` | user data the app manages: list of `{id, label, path, type, consumers}` (paths under `durable_root`; `consumers` = bots using that folder) |
| `peers` | paired remote devices/nodes: list of `{name, device_id}` (device IDs are public, not secret) |
| `device_id` | this machine's public ID on the app's network (for example, a Syncthing device ID) |
| `taildrop_dir` / `taildrop_group` | shared inbox for received files (under `durable_root`) and the group all bots share for it |
| `taildrop_fetch` | `loop` / `schedule` / `off`: how received files are moved into `taildrop_dir` |
| `taildrop_retention_days` | days before files in `taildrop_dir` are deleted by the daily check, or `off` |
| `taildrop_fetch_command` / `taildrop_cleanup_command` | installed guarded fetch helper and retention cleanup script |
| `operator_user` | the user allowed to operate a root tailscaled without sudo (`tailscale set --operator`), if set |
| `symlinks` | links from conventional paths into `durable_root` |
| `cold_copy_dir` / `cold_copy_extra` | optional private copies of identity/state for re-seeding |
| `cold_copy_essentials` | the exact files a cold copy may contain (identity/state only: no logs, caches, DBs, nested dirs) |
| `cold_copy_max` | maximum retained cold copies (always ≤ 2) |
| `cold_copy_refresh_command` | refreshes cold copies (only after health passes) |
| `restore_command` | restores from cold copies (often the same as `start_command`) |
| `secret_locations` | where secrets live (auth keys, private keys, state). Never put secret values in records, docs, or reports |
| `log_path` / `log_max` | the app's own log and its size cap. Keep it outside state dirs and cold copies |
| `memory_limit` | runtime memory cap, if any |
| `health_command` / `health_expected` | optional override. Default is the standard check in the app's runbook skill |
| `reindex_check` | how to prove state/DB continuity (for example, a sequence counter), if the runbook doesn't define it |
| `last_known` | baseline numbers for regression checks (sequence, file counts, RSS, node id …) |
| `version_command` / `installed_version` | how to read the version, and the value last recorded |
| `pinned_version` | `latest` or a specific version, with the reason |
| `latest_version_source` / `latest_version_seen` | optional override of the runbook's upstream query, and the last value seen |
| `allow_major_upgrade` | optional, default `false`. A new **major** version needs owner approval unless `true` |
| `major_upgrade_offered` | the major version already offered to the owner (so it is not re-asked daily) |
| `last_upgrade_review` | why the last unattended upgrade was judged harmless: `{from, to, notes_url, judged_harmless_because, date}` |
| `upgrade_held` | a minor/patch upgrade held because its release notes raised a concern (reported once, not re-asked daily): `{version, reason, notes_url, date}` |
| `api_auth` | optional: `none` / `password` / `token`, how the GUI/API authenticates (checked by the sanity check when `api_address` is not loopback) |
| `owner_confirmed_risks` | risks the owner explicitly accepted after a sanity-check warning: list of `{risk: <finding id>, date, owner_said}`. REFUSE-level findings can never be listed here |
| `upgrade_command` / `upgrade_also` / `rollback` | optional override of the runbook's upgrade, extra companion packages upgraded in the same step, and rollback notes |
| `runbook_skill` | the app-specific runbook template to follow |

## Registry schema
`registry.json` holds the global fields `durable_root`, `app_dir`, `launcher_backup_dir`, `daily_check_time`, `report_policy` (default `silent-when-healthy`), `validator_path` (the installed sanity-check validator, kept in `launcher_backup_dir`), `restore_guard_path` (the installed restore safeguards script from section 8, kept in `launcher_backup_dir`), and `ignored_services` (services/binaries already proposed to the owner and declined or deferred: `{name, path, decision, date}`; never re-proposed), plus one row per app with: `name`, `config_record`, `runbook_skill`, `install_method`, `start_command`, `self_serve_start`, `depends_on`, `consumers`, `installed_version`, `pinned_version`, `latest_version_seen`, and optionally `major_upgrade_offered`, `upgrade_held`, `last_upgrade_review` (mirrors of the record fields).

Minimal example `registry.json`:
```json
{
  "durable_root": "<durable_root>",
  "app_dir": "<durable_root>/app-management",
  "launcher_backup_dir": "<durable_root>/app-management/launchers",
  "daily_check_time": "04:00 owner-local",
  "report_policy": "silent-when-healthy",
  "validator_path": "<durable_root>/app-management/launchers/am-validate",
  "restore_guard_path": "<durable_root>/app-management/launchers/restore-guard",
  "ignored_services": [{"name": "<service>", "path": "<path>", "decision": "declined", "date": "<YYYY-MM-DD>"}],
  "apps": [
    {"name": "<app>", "config_record": "apps/<app>.md", "runbook_skill": "<app>-runbook-template",
     "install_method": "<apt|static-binary|...>", "start_command": "<launcher path>",
     "self_serve_start": false, "depends_on": [], "consumers": ["<bot name>"],
     "installed_version": "<x.y.z>", "pinned_version": "latest", "latest_version_seen": "<x.y.z>"}
  ]
}
```

Minimal example `apps/<app>.md`:
````markdown
# Instance config: <app>
```yaml
name: <app>
runbook_skill: <app>-runbook-template
consumers: [<bot name>]
depends_on: []
self_serve_start: false
durable_root: <durable_root>
install_method: <apt|static-binary>
binary_path: <path>
launcher: <path to launcher>
launcher_backup_dir: <durable_root>/app-management/launchers
start_command: <path to launcher>
state_dir: <durable path>
cold_copy_dir: <private secrets dir>
cold_copy_essentials: [<file>, <file>]
cold_copy_max: 2
log_path: <path outside state_dir>
log_max: <size, e.g. 5MB>
device_id: <public id, if the app has one>
peers: [{name: <peer name>, device_id: <peer id>}]
data_folders: [{id: <folder id>, label: <label>, path: <durable_root>/<folder>, type: <sendreceive>, consumers: [<bot name>]}]
secret_locations: [<path>]
installed_version: <x.y.z>
pinned_version: latest
allow_major_upgrade: false
last_upgrade_review: {from: <x.y.z>, to: <x.y.w>, notes_url: <url>, judged_harmless_because: "<patch release; bug fixes only; no migrations>", date: <YYYY-MM-DD>}
owner_confirmed_risks: []
```
Known issues: <instance-specific notes only>
````

## 1. Purpose
Say in one line what the app does and who its `consumers` are. Check each app in `depends_on` first.

## 2. Install (pinned or latest), only if missing
Run `app-management-sanity-check` first (install paths, pins, and where the binary/launcher are written).
A first install (an app not yet in the registry) needs the owner's approval. Reinstalling a registered app that went missing is part of a restore (section 8) and is routine.
1. Check whether it is already installed: `version_command`. If it is present, **do not upgrade it here**. Upgrades happen only in the daily check's Update step.
2. If it is missing, install from `install_source` using `install_method`:
   - apt: add the vendor repo/keyring for the detected distro and codename (from `/etc/os-release`, never hardcoded). Install `<pkg>=<pinned_version>` and `apt-mark hold <pkg>` when pinned. Otherwise install the current version.
   - static binary: download the release asset for `pinned_version` (or latest stable, never a pre-release), verify the checksum/signature, and place it at `binary_path`. Or restore it from `binary_backups`.
   - Record `installed_version`.
3. If `launcher` is missing, copy it from `launcher_backup_dir` and verify it (`sha256sum -c SHA256SUMS.txt` inside that dir). If no launcher exists at all, write one from the reference launcher in the app's runbook skill, fill it from the record, and back it up (plus a checksum) in `launcher_backup_dir`.
4. Recreate `autostart_entry` from `launcher_backup_dir` if it is used.

## 3. What lives where
Fill a table from the record: item | path | durable? | secret? | notes. It must cover the binary, launcher, config, identity, state, DB, data folders, logs, and cold copies. Anything important that is not under `durable_root` is a defect. Fix it with the `app-persistence-method` skill.

## 4. Persistence symlinks
Run `app-management-sanity-check` first (link targets must not overlap synced folders, other apps' dirs, or system dirs).
For each entry in `symlinks`: use `restore-guard <record> link <link> <target>` (section 8), which moves anything already there aside instead of replacing it. Never replace a real directory with a link unless the app is stopped and the directory has been moved first (see `app-persistence-method`).

## 5. Start
- If the record has `self_serve_start: true`, any bot may run `start_command`. Otherwise, bots notify App Management and App Management runs it.
- `start_command` must be idempotent (running it while the app is healthy breaks nothing) and must **never upgrade**. It installs only if missing.
- Let it finish. Run long starts in the background and wait. An interrupted start can leave the app down.

## 6. Health check
Use the standard check in the app's runbook skill, or `health_command`/`health_expected` if the record overrides it. Also check that the process exists and the ports/sockets are present.

## 7. Update procedure (daily check Update step only)
Run `app-management-sanity-check` first (EOL pins, downgrades across a DB migration, major upgrades).
1. Get the upstream version from the runbook's source (or `latest_version_source`). Ignore pre-releases. Record `latest_version_seen`.
2. If `pinned_version` is not `latest`, stop here and just report.
2b. **Major-version gate:** if upstream's major version is higher than the installed major and `allow_major_upgrade` is not `true`, do not upgrade. Report it to the owner **once** and record `major_upgrade_offered: <version>` in the record and registry, so it is not re-asked daily. Upgrade only after the owner approves. Minor/patch updates within the same major go on to the release-notes review (2c).
2c. **Release-notes review** (every upgrade, even patch releases): read the release notes/changelog of **every** release between the installed version (exclusive) and the target (inclusive). The upgrade is **harmless** only if: same major version, no breaking changes, no deprecations affecting our config, no required config or database migrations, no removed features we use, and no known serious regressions or other concern. If harmless, record `last_upgrade_review: {from, to, notes_url, judged_harmless_because, date}` and continue unattended. If anything raises a concern, or the notes cannot be found, **hold**: do not upgrade, report it to the owner once, and record `upgrade_held: {version, reason, notes_url, date}` in the record and registry (not re-asked daily; re-evaluate when a newer release appears). Upgrade only after the owner approves.
3. Make sure the app is healthy and `cold_copy_refresh_command` has just run. Note the `last_known` metrics.
4. Upgrade using the runbook's procedure (or `upgrade_command`), including `upgrade_also`. Keep the previous binary/package available for rollback.
5. Run the health check and the verification checklist. If anything fails, roll back, recheck, and report.
6. Update `installed_version`.

## 8. Restore (a missing or broken app that is already in the registry, including on a wiped computer)
This is routine and proceeds **unattended** (see "Routine autonomy" in `app-management-sanity-check`), but only with these safeguards:
1. **Pre-flight.** Run the validator (no REFUSE for this app). Confirm the durable data and identity are present and look valid before starting anything: `state_file` non-empty; the identity essentials (`cold_copy_essentials`, for example cert/key/config) present in `config_dir`; `db_dir` present and non-empty; `data_folders` present. `restore-guard <record> preflight` (below) does the identity/state/DB part.
2. **Reinstall** if missing (section 2) at `installed_version`, or at the latest version only if it passes section 7's rules (same major, release notes reviewed and judged harmless, recorded in `last_upgrade_review`). Recreate the launcher from `launcher_backup_dir` (`sha256sum -c`) or from the runbook's reference launcher.
3. **Links and files: never overwrite or delete existing data.** Create each entry in `symlinks` with `restore-guard <record> link <link> <target>`: anything already at the link path is moved aside with a timestamped `.pre-restore-<timestamp>` suffix. Moved-aside items are **never deleted during a restore**; afterwards (on `commit` or the daily `prune`), only items beyond the two newest per path are pruned, and never one that is still in use (a recorded path or any symlink points at it or into it). If the existing item looks **newer or bigger** than the durable copy, **stop and ask** the owner. An existing folder with no non-empty files (typical right after a wipe) is simply moved aside. A pre-flight also stops if an earlier restore left an uncommitted journal: commit it (if that restore was verified) or roll it back first. In kernel mode (`run_mode: kernel`) the script uses sudo for root-owned state.
4. **Identity: durable storage first, cold copy second.** Re-seed only what is missing, from the cold copies. **Never** generate a new identity, never re-login, and never start with an empty database when a durable one exists. If the only way forward is a new identity, a re-login, or a fresh database, **stop and ask** the owner.
5. **Start one app at a time**, in `depends_on` order. Let each start finish.
6. **Post-verify:** the same identity (device ID / node name / node ID vs `last_known`), no login prompt, no full re-index (sequence counters continue from `last_known`, DB file mtimes are not all reset to now), the health check passes, and the consumers' features work (section 9).
7. **On any failure:** stop the app, run `restore-guard <record> rollback` (it removes only the links/copies this restore made and puts the `.pre-restore` items back), leave all data untouched, and report to the owner.
8. **After success** (post-verify passed): run `restore-guard <record> commit` so a later rollback can never undo this restore, which also prunes restore leftovers to the limit of two (see Pitfalls: backup limit), then run `cold_copy_refresh_command`, send each consumer bot a priority "restored, retry" message, and report the restore briefly (what, why, the downtime window). The two newest `.pre-restore` items per path stay for rollback; older ones are pruned only by `commit`/`prune`, and never while still in use.

Restore safeguards (bash skeleton; reads the instance record). Install it at the registry's `restore_guard_path` (in `launcher_backup_dir`, covered by `SHA256SUMS.txt`); `restore-guard` below means that path:
> **Script:** [`scripts/restore-guard`](../../scripts/restore-guard) (sha256 listed in `index.json`). Fetch it at a pinned commit, verify the checksum, and review it before use.

## 8a. First-time setup (an app not yet in the registry; needs the owner's approval)
A **first install** means only an app that is **not yet in the registry**, and it needs the owner's approval. An app already in the registry whose identity is missing everywhere is not a first install: it is a restore that must stop and ask (a new identity).
Run `app-management-sanity-check` first on every new path, address, and retention value before saving it.
Only when neither durable state nor any cold copy exists. Otherwise, use section 8.
- General: create the durable dirs first (`state_dir`/`config_dir`/`db_dir` under `durable_root`, mode 700), so the app generates its identity directly on durable storage. Record the new identity (node/device id) in `last_known`, and take the first cold copy immediately after the first healthy start.
- **Tailscale:** install the package, then start `tailscaled --statedir=<state_dir> --socket=<socket_path>` (plus the `run_mode` tun flag). The owner authorizes the node in whichever way they prefer: (a) a login URL from `tailscale --socket=<socket_path> up --hostname=<node_name>`, sent to the owner, or (b) an auth key the owner provides through a secure secret request (never in chat), stored at a `secret_locations` path and used with `--auth-key=file:<that path>`. Never put the key in the registry. Once it shows Running/Online, take the first cold copy of `state_file`. Then follow the owner lifecycle in `tailscale-runbook-template` (available functions, Taildrop offer, retention question).
- **Syncthing:** install the newest stable release from GitHub (never distro/snap packages, which often ship 1.x) and verify the major version. Start it with `--config=<config_dir> --data=<db_dir>` on durable paths, so the keys and DB are generated there. Take the first cold copy immediately. Record `device_id` (REST `/rest/system/status` → `myID`) and send it to the owner (it is public). Add peers and folders only as the owner directs, with folder paths under `durable_root`. Follow the owner lifecycle in `syncthing-runbook-template`.

## 8b. Repair and notify
1. Run `app-management-sanity-check` first. Repair using sections 5–8. Never delete state/DB, and never force a re-login.
2. Verify with the health check and the checklist in section 9.
3. **Once the fix is verified**, send every consumer bot that reported the app as broken, and every `consumers` entry affected by the outage, a **priority / wake-up message**: the service is restored, and they can retry their work. Include what was down and roughly for how long (the exact window if known). Leave out secrets.

## 9. Verification checklist
- [ ] `version_command` matches the expected version
- [ ] health check passes
- [ ] identity unchanged (same node/device id as before; no duplicate device, no login prompt)
- [ ] state/DB continued (no re-index, counters ≥ `last_known`, no mass downloads)
- [ ] data folders present with the expected file counts
- [ ] `consumers` can use it (for example, a peer is connected)
- [ ] cold copies refreshed after health passed: essentials only, ≤ `cold_copy_max` (2), older copies pruned
- [ ] affected consumers notified (priority/wake-up) that the service is restored

## 10. Pitfalls
- A login/auth prompt after a restore means the restore failed. Retry from cold copies. It is not a normal login request.
- Never delete a DB/state to "fix" an app. Report instead.
- Never apply an owner's risky instruction, or "fix" their setting, without the sanity check and their explicit confirmation.
- Take cold copies only when the app is healthy, so a broken state never overwrites a good copy.
- Installed software (packages, binaries outside `durable_root`) can vanish on computer updates. Launchers must reinstall it if missing, but never upgrade it.
- **Backup limit: at most two per app, because storage is limited.** Cold copies: essentials only, at most two. Restore leftovers follow the same limit: at most the two newest `<path>.pre-restore-<timestamp>` (and `<path>.failed-restore-<timestamp>`) items per recorded path, and at most two `.committed-<timestamp>` / `.rolled-back-<timestamp>` journals per app. `restore-guard` prunes them only on `commit` (after post-verify passed) or `prune` (daily check), never during a restore or rollback, never an item the committed journal references, never an item that a recorded path or any symlink currently points to (or into), and only exact names next to the app's recorded paths (`symlinks` links, `state_file`, `config_dir` essentials). Restore leftovers and journals never go into cold copies (no nested copies).
- Never archive a directory into itself, and never include logs, caches, or nested copies in a cold copy.
- Never prune an item that a recorded path or any symlink currently points to (or into): the link would dangle and the app would lose its data. `restore-guard` checks this before every removal.
- Cap or rotate the app's own log. Never touch logs the app doesn't own.
