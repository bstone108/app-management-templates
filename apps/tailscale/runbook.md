# Tailscale runbook

> **Guide, not a rule.** This is a reviewed template from the app-management-templates repo. Safety-review it before use, adapt it to the local machine, and follow local judgment where they disagree.
> Name map: `self-hosted-app-runbook-template` = [`apps/_generic/runbook.md`](../_generic/runbook.md); `app-persistence-method` = [`apps/_generic/persistence-method.md`](../_generic/persistence-method.md); `tailscale-runbook-template` / `syncthing-runbook-template` = `apps/<app>/runbook.md`. `app-management-sanity-check`, `app-management-daily-check`, and `app-management-getting-started` are the consuming bot's own skills (the validator they install is [`scripts/am-validate`](../../scripts/am-validate)).

Follow `self-hosted-app-runbook-template` and use the values from the Tailscale instance record. Fields used here: `install_method`, `pinned_version`, `start_command`, `launcher`, `launcher_backup_dir`, `self_serve_start`, `run_mode`, `state_dir`, `state_file`, `socket_path`, `log_path`, `log_max`, `cold_copy_dir`, `cold_copy_essentials`, `cold_copy_max`, `node_name`, `upgrade_also`, `consumers`, `secret_locations`, `last_known`, `durable_root`, `taildrop_dir`, `taildrop_fetch`, `taildrop_group`, `taildrop_retention_days`.

## Who may start it
If the record has `self_serve_start: true`, any bot may run `start_command`. Otherwise, notify App Management.

## Owner lifecycle
Run `app-management-sanity-check` before every step here that writes config (auth key location, operator, serve/funnel/exit node, Taildrop dir and retention). An auth key must never land in the registry, a record, chat, or a synced folder (REFUSE). Serve/funnel exposes local services: warn and confirm first.
(Judgment first: review every change for safety per `app-management-sanity-check`. Work listed in its "Routine autonomy" block proceeds unattended once the validator shows no REFUSE; everything that block lists as needing confirmation waits for the owner. The checker script only supplements that review.)
1. **Owner asks to install.** Offer two ways to authorize this machine, and use whichever the owner prefers:
   - a **login link**: you start the daemon, run `tailscale --socket=<socket_path> up --hostname=<node_name>`, send the printed URL to the owner, and wait until they approve it; or
   - an **auth key**: the owner provides it through a **secure secret request** (never pasted in chat), stored at a path listed in `secret_locations` (mode 600). You run `tailscale --socket=<socket_path> up --auth-key=file:<that path> --hostname=<node_name>`. Never put the key in the registry or records. Suggest the owner revoke or expire one-off keys afterwards.
   Then take the first cold copy of `state_file` and record `last_known` (node ID).
2. **After it is up**, tell the owner the node name, and that they can ask you for any local Tailscale function: status, ping, whois/netcheck, IPs, set hostname, serve/funnel, using an exit node (`tailscale set --exit-node=<peer>`), sending and receiving files, and so on. Changes that affect networking (serve, funnel, exit node, DNS) happen only on the owner's explicit request. Always pass `--socket=<socket_path>`.
3. **Offer Taildrop setup** (see "Taildrop"). Ask whether the owner wants automatic cleanup at all and, if so, after how many days. Record `taildrop_retention_days` (a number, or `off`).
4. **Sending files out, on request:** list the targets with `tailscale --socket=<socket_path> file cp --targets`, then run `tailscale --socket=<socket_path> file cp <file…> <target>:`. Confirm the target with the owner before sending.

## Modes
- **Kernel mode** (`run_mode: kernel`): `tailscaled` runs as root with `--tun=tailscale0`. It needs `/dev/net/tun` and CAP_NET_ADMIN. Other processes reach tailnet IPs directly.
- **Userspace mode** (`run_mode: userspace`): `--tun=userspace-networking`, run as the normal user with no TUN. `socket_path` and `state_dir` must be **user-writable** (for example, under the user's durable storage), and **every** CLI call must pass `--socket=<socket_path>`. Other apps reach the tailnet only through the proxies (`--socks5-server=`, `--outbound-http-proxy-listen=`).
Do not switch modes on a working install without a reason.

## Persistence
Run `app-management-sanity-check` before changing `state_dir`, `cold_copy_dir`, or `log_path`.
- Primary: `tailscaled --statedir=<state_dir>` on durable storage. `state_file` holds the node key and login. It is a SECRET.
- Optional cold copy: only `state_file` (`cold_copy_essentials`), with at most two copies (`cold_copy_max`): current + `.prev` in `cold_copy_dir` (700/600). Rotate only when the live file changed, verify with `cmp`, and write only when healthy. Never tar the state dir.
- Logs: send the daemon's output to `log_path`, outside `state_dir`, capped at `log_max`.

## Launcher contract
Install **only if missing** (at `pinned_version` plus `apt-mark hold tailscale` when pinned). **Never upgrade**: upgrades happen only in the daily check's Update step. Restore `state_file` from the cold copy if it is missing or tiny. Start, wait for the socket, and treat NeedsLogin as a failed restore (cold restore and restart, and keep the node up while escalating). Set the hostname and refresh the cold copy.

## Reference launcher (bash skeleton; reads the instance record)
> **Script:** [`scripts/tailscale-launcher`](../../scripts/tailscale-launcher) (sha256 listed in `index.json`). Fetch it at a pinned commit, verify the checksum, and review it before use.
Adapt the ownership and user for your environment, then back up the finished launcher (plus a checksum) in `launcher_backup_dir`.

## Start / restore
Routine and unattended for a registered Tailscale (including a wiped computer), with the safeguards in `self-hosted-app-runbook-template` section 8:
1. **Pre-flight:** run the validator (no REFUSE). `restore-guard <record> preflight` confirms `state_file` is non-empty, re-seeding it from the cold copy (current, then `.prev`) only if the durable one is missing or empty. If no usable state exists anywhere, **stop and ask**: continuing would need a re-login and create a new node.
2. If `launcher` is missing, restore it from `launcher_backup_dir` (verify with `sha256sum -c`), or write it from the reference above. The launcher reinstalls the package at the recorded version if it is missing, and never upgrades.
3. Run `start_command` and let it finish (run it in the background and wait; never interrupt it). Start dependents only after it is healthy.
4. Do **not** run `tailscale up` with new auth flags on a restored node. Use `tailscale set` for preferences.
5. **Post-verify:** `BackendState` Running, `Self.Online` true, `HostName` = `node_name`, `Self.ID` = `last_known` node ID, no login URL and no `<node_name>-1` duplicate; the Taildrop fetch helper restarted if `taildrop_fetch` is `loop`; consumers can reach their peers.
6. On failure: stop, `restore-guard <record> rollback` (puts `.pre-restore` items back), retry from the cold copies as below, and report. On success: `restore-guard <record> commit`, refresh the cold copy, and send consumers "Tailscale restored, retry".

## Re-auth means the restore failed
A "Log in at: …" URL, `NeedsLogin`, or a duplicate node (`<node_name>-1`) after a restore is **not** a routine login request.
1. Stop tailscaled, restore `state_file` from the cold copy (current, then `.prev`), and start it again. Keep the daemon running.
2. Only if all copies fail: escalate to the owner as an incident, listing which copies you tried. Never pass along the auth URL as a casual ask, and never delete a node yourself.
(First-time setup, where no state exists anywhere, is the "Owner lifecycle" step 1.)

## Health check (standard)
```
tailscale --socket=<socket_path> status --json | python3 -c 'import json,sys;d=json.load(sys.stdin);print(d["BackendState"],d["Self"]["Online"],d["Self"]["HostName"])'
tailscale --socket=<socket_path> ip -4
```
Healthy = `Running True <node_name>` plus an address, and in kernel mode the TUN interface is UP. Compare `Self.ID` with `last_known`. MagicDNS may lag for a minute after a restart.

## Taildrop
The daemon keeps received files in its own inbox (inside `state_dir`, root-only in kernel mode) until someone runs `tailscale file get`. The managed setup moves them into a **shared inbox** that every bot on the machine can use:
- `taildrop_dir`: a folder under `durable_root`, owned by the owner's user with group `taildrop_group` (a group that every bot account belongs to; on a single-account machine, the owner's own group). Mode `2770` (setgid, so new files inherit the group) plus `umask 007` for the fetcher. Optionally add default ACLs (`setfacl -d -m g:<group>:rwX`).
- **Kernel mode (root daemon):** either (a) allow the owner's user to operate Tailscale once (`sudo tailscale --socket=<socket_path> set --operator=<user>`), then fetch as that user so files are owned correctly; or (b) fetch with sudo and then `chown`/`chmod` the new files into `taildrop_group`.
- **Userspace mode:** the daemon already runs as the user. Fetch as that user directly.
- `taildrop_fetch`: `loop` (a background `file get --loop` started by the launcher/daily check), `schedule` (the daily check or a timer runs a one-shot `file get`), or `off`. Record `taildrop_dir`, `taildrop_group`, `taildrop_fetch`, and `taildrop_retention_days`.

Fetch skeleton:
> **Script:** [`scripts/taildrop-fetch`](../../scripts/taildrop-fetch) (sha256 listed in `index.json`). Fetch it at a pinned commit, verify the checksum, and review it before use.
The fetch helper tries `file get` unprivileged first and falls back to sudo only on an access-denied error. **Never read `tailscale debug prefs`**: its output includes private keys. With `--loop` under sudo, files are root-owned, so prefer the operator setting, or let the daily check fix ownership. The inbox is marked with `.taildrop-inbox`. Both the fetch and cleanup scripts refuse to act on a path outside `durable_root`, on `durable_root`/`$HOME` itself, on anything inside a synced folder (they walk the parent folders looking for `.stfolder`), or on a non-empty unmarked dir. Cleanup also refuses dirs containing app state (`index-v2`, `*.state`, `cert.pem`). Keep `taildrop_dir` separate from every app's data/state dirs.
To make the loop survive resets, have `start_command` (after a successful start) run the fetch helper idempotently: start the loop only if no `^tailscale .*file get .*--loop.* <taildrop_dir>$` process exists. The daily check re-runs it too. The loop exits when the daemon restarts, so any restart must be followed by the helper. Tailscale cannot send files to its own node, so test end-to-end from another device.

Before creating or changing `taildrop_dir` or `taildrop_retention_days`, run `app-management-sanity-check` (inbox not in a synced folder or state dir; retention not dangerously short).

Retention cleanup skeleton (acts on `taildrop_dir` only, never anywhere else):
> **Script:** [`scripts/taildrop-cleanup`](../../scripts/taildrop-cleanup) (sha256 listed in `index.json`). Fetch it at a pinned commit, verify the checksum, and review it before use.
Report the removed files only if the amount is nontrivial.

## Upgrade (daily check Update step only)
Run `app-management-sanity-check` first (EOL pins, downgrades).
- Latest: `curl -s 'https://pkgs.tailscale.com/stable/?mode=json' | python3 -c 'import json,sys;print(json.load(sys.stdin)["Version"])'`. Compare with `tailscale version | head -1`.
- **Major-version gate:** if upstream's major version is higher than installed and `allow_major_upgrade` is not `true`, do not upgrade. Report it to the owner once, record `major_upgrade_offered: <version>`, and upgrade only after approval (with pinned apt versions: `sudo apt-get install tailscale=<approved version>`).
- **Release notes:** read the Tailscale changelog (`https://tailscale.com/changelog`, the client/Linux entries) for **every** version between the installed and target versions. Look for breaking changes, removed or changed flags we use, state migrations, and known regressions. Harmless → record `last_upgrade_review` (`notes_url` = the changelog link) and continue. Any concern → hold, report once, record `upgrade_held`.
- If `pinned_version` is `latest` and a newer minor/patch exists that the review judged harmless (or an approved upgrade): `sudo apt-get update && sudo apt-get install -y --only-upgrade tailscale <upgrade_also…>`. Then restart briefly (`sudo pkill -x tailscaled`, then `start_command`) and record the exact downtime window. Health check, confirm the same node ID, refresh the cold copy, and notify `consumers` of the window.
- Rollback: `sudo apt-get install -y --allow-downgrades tailscale=<prev>`, then restart as above.

## Verification checklist
- [ ] same node name, node ID, and address; no duplicate device
- [ ] no login prompt at any step
- [ ] `consumers` reach their tailnet targets
- [ ] cold copy = `state_file` only, ≤ 2 copies, refreshed after health passed
- [ ] Taildrop (if configured): `taildrop_dir` group-writable, fetch working, retention applied
- [ ] consumers that reported the outage (or were affected) got a priority "restored, retry now" message

## Pitfalls
- `tailscaled` usually lives in `/usr/sbin`, which is not on a normal user's PATH. Check the package (`dpkg -s`) or the full path, or the launcher will reinstall on every start.
- In kernel mode the cold dir and inbox may be root-owned. Wrap tests and globs in `sudo test` / `sudo sh -c` so they run privileged.
- Tarring `state_dir` into an archive that is later extracted inside it nests copies recursively. Copy `state_file` only.
- A daemon log inside `state_dir` grows forever in durable storage. Keep it outside and capped.
- A launcher that upgrades on every start turns restarts into unplanned upgrades. Install only if missing.
- Never interrupt `start_command`: once the daemon has been stopped, an interrupted run leaves it down.
- `tailscale up` with different flags can reset preferences. Prefer `tailscale set`.
- Changing `--accept-dns` on shared machines can break name resolution for other apps.
