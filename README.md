# app-management-templates

Reviewed app templates and helper scripts for the **App Management** bot: a bot that installs, keeps durable, health-checks, restores, and carefully upgrades self-hosted apps on machines whose storage can be reset.

> ## ⚠️ Templates are guides, not rules
> Every consuming bot must:
> 1. **Safety-review each file before use** (run its own full sanity-check review on every runbook and script: data loss, identity/login loss, secret exposure, effects on other apps and bots, downtime, disk use).
> 2. **Pin a commit.** Fetch files from `https://raw.githubusercontent.com/bstone108/app-management-templates/<commit>/<path>`, never from a moving branch, and record the commit it used.
> 3. **Verify script checksums** against the `sha256` values in `index.json` at that same commit.
> 4. **Review the diff before adopting any update.** Never adopt repo changes silently or automatically.
> 5. **Adapt to the local machine** and follow local judgment where a template disagrees with it.
>
> Nothing here contains installation-specific values, secrets, hostnames, or device IDs. Never add any.

## Layout
```
README.md                         this file
index.json                        apps (id, name, template path, scripts used, summary) and scripts (path, purpose, sha256)
apps/_generic/runbook.md          generic self-hosted app runbook: config field vocabulary, registry schema and examples,
                                  install / persistence / start / health / update / restore / repair procedures
apps/_generic/persistence-method.md   how to make an app survive computer resets
apps/tailscale/runbook.md         Tailscale runbook (owner lifecycle, node identity, Taildrop)
apps/syncthing/runbook.md         Syncthing runbook (install policy, pairing lifecycle, durable DB/identity)
scripts/am-validate               read-only registry/record sanity validator
scripts/restore-guard             restore safeguards (move aside, rollback, commit, bounded prune)
scripts/tailscale-launcher        Tailscale launcher (install only if missing, never upgrades)
scripts/taildrop-fetch            Taildrop fetch into a marked shared inbox
scripts/taildrop-cleanup          guarded Taildrop retention cleanup
scripts/syncthing-launcher        Syncthing start launcher (no install, no upgrade)
scripts/syncthing-snapshot        Syncthing cold-copy helper (essentials only, max two)
```

## How a consuming bot uses this repo
1. Look up the app in `index.json`. No entry → follow the bot's own written spec.
2. Fetch the template and its scripts at a pinned commit; verify each script's sha256 against `index.json`.
3. Review everything, adapt it to the local machine, fill values from the app's instance record, and record the commit used.
4. When the repo moves, report that newer versions exist and review the diff; adopt only after review.
5. If the repo is unreachable, fall back to the bot's own written spec.

Every script is a `bash` skeleton that reads the app's instance record, refuses unsafe targets, and never prints secrets. Test any script in a throwaway folder before installing it.
