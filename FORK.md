# Fork notes

Fork-local rules for coding agents. `AGENTS.md` is upstream's and still applies in
full — this file only covers what is different here. Where they disagree, this file
wins, because it describes this checkout.

This file is fork-only and does not exist upstream, so it never conflicts on rebase.
Keep it that way: put fork-specific guidance here, not in `AGENTS.md`.

## What this fork is

Upstream `pingdotgg/t3code` plus PR [#11356](https://github.com/pingdotgg/t3code/pull/11356),
which adds Devin as a first-class ACP provider, plus a handful of local changes.
`git log upstream/main..HEAD` is the authoritative list; today it is 21 commits, of
which 9 are the PR and the rest are ours.

|            |                                            |
| ---------- | ------------------------------------------ |
| Branch     | `devin-11356`                              |
| `upstream` | `pingdotgg/t3code` (read only, never push) |
| `origin`   | `walde-dev/t3code` (the fork, push here)   |

Upstream has not merged any Devin PR; at least eight attempts since July died
unreviewed. Assume this stays a fork indefinitely rather than waiting for it to land.

### What we changed beyond the PR

- **Identity.** Product name `T3 Code (Walde)`, the Icebox stage art and app icon, a
  `walde` badge in the sidebar header.
- **Identity does not follow the version.** Upstream keys the product name and icon
  off the version string, switching to Nightly branding for nightly-shaped versions.
  Our builds are _stamped with upstream nightly versions on purpose_ (see below), so
  both are now unconditional. Do not reintroduce the version check.
- **Local update feed.** `T3CODE_DESKTOP_UPDATE_DIR` / `T3CODE_DESKTOP_UPDATE_URL`
  publish the mac artifact against a local generic provider instead of GitHub.
- **Ad-hoc signing.** Unsigned mac builds now pass `identity: "-"`.

## The fleet

Two machines run this fork, and they have different jobs. Do not blur them.

| Machine     | Role                                                                         |
| ----------- | ---------------------------------------------------------------------------- |
| **MacBook** | Decides. Rebases onto `upstream/main`, rebuilds the desktop app, pushes.     |
| **anvil**   | Follows. Never rebases; resets to the pushed branch and rebuilds the server. |

Only the Mac rebases. That is deliberate: the branch is rewritten history, and if
both machines rebased they would produce different commits for the same work and
diverge permanently. anvil resets to `origin/devin-11356` and rebuilds.

```bash
# MacBook — decides when to move
~/code/sync-t3-devin.sh --rebase --push     # manual rebase + test + publish
~/code/t3code-watch-upstream.sh             # what launchd runs every 2h

# anvil — follows
~/anvil-sync-t3.sh                          # what its systemd timer runs every 2h
~/anvil-sync-t3.sh --check                  # report without rebuilding
```

`.env` is required on every machine and is gitignored, so it never arrives with a
`git pull`. Copy it deliberately. Without it a server builds cleanly and then has no
relay URL or Clerk key, T3 Connect comes up unconfigured, the tunnel never
registers, and every remote client silently loses that environment. This cost an
evening once; the anvil script now refuses to run without it.

## Staying current with upstream

"Nightly" is not a branch. The nightly channel is a tag cut from `main` a few times a
day, so tracking nightly means rebasing onto `upstream/main`.

`t3code-watch-upstream.sh` reads upstream's newest nightly tag, compares it to a pin
file at `~/.t3-updates/.pinned-upstream`, and when they differ it rebases, rebuilds
the app **stamped with that nightly's version**, stages the update feed, installs if
the app is closed, and pushes. It pins only after the feed is valid, so a failed run
retries rather than going quiet.

Expect conflicts to be rare: the PR is 11 new files (4301 lines, which cannot
conflict) and 13 modified files (329 lines total). The real touchpoints are small —
`builtInDrivers.ts` 5 lines, `contracts/model.ts` 10, `contracts/settings.ts` 33.

**`--rebase` is the mode you want.** `--track-pr` re-fetches the PR head and
`reset --hard`s onto it, which discards every commit of your own. The script blocks
this when it detects commits above `upstream/main`, and `FORCE=1` overrides. That
check compares patch-ids and is deliberately conservative: once the branch has been
rebased, the PR's own commits can shift patch-id and get listed alongside yours.

## How updates actually reach the app

The app **detects** updates through electron-updater and **cannot install** them.
Both halves matter:

- Detection works. The feed is served over `http://127.0.0.1:31339` by a launchd
  agent (`com.walde.t3code-update-feed`) over `~/.t3-updates`. It must be HTTP:
  electron-updater's generic provider issues a GET, and a `file://` URL is not
  something it can fetch.
- The channel in the feed filename must match the channel the app asks for. Ours is
  `nightly` (`desktop-settings.json`), so the build writes `nightly-mac.yml`. Pinning
  the channel to `latest` looks tidy and breaks this silently.
- Blockmaps are deliberately **not** staged. They make electron-updater attempt a
  differential download using HTTP range requests, which the static feed server does
  not implement.
- Installation is done by the watcher copying the verified bundle into
  `/Applications`, not by Squirrel.

**Why Squirrel cannot do it.** It checks that the replacement bundle satisfies the
running app's code-signing requirement. An ad-hoc signature's designated requirement
is `cdhash` of that exact binary, so no rebuild can ever match. A signing certificate
exists to give an identity that stays stable across builds — a distribution problem.
This app never leaves the machine that builds it, so a verified copy is the same
operation without the ceremony. Do not spend money on a Developer ID to "fix" this.

Two traps live in that install path, both found the hard way:

- `pgrep -f` treats its pattern as a regex, and the bundle name contains `(Walde)`,
  which becomes a capture group matching nothing.
- Under `set -o pipefail`, `grep -q` exits early, `ps` takes SIGPIPE, and the
  pipeline reports failure on a _successful_ match.

## Toolchain gotchas on this machine

- **Node 24 is required** (`engines: ^24.13.1`). The Mac's default `node` is v25 and
  will fail the engine check. Use `~/.nvm/versions/node/v24.18.1/bin`.
- **`vp` is not global.** It installs to `node_modules/.bin`, but
  `scripts/dev-runner.ts` spawns `vp` from `PATH`. Export
  `$REPO/node_modules/.bin` or dev dies with `spawn vp ENOENT`.
- **pnpm needs throttling.** The default concurrency times out fetching large
  metadata docs (`@clerk/shared`, `@clerk/react`). Use
  `pnpm install --network-concurrency 4 --fetch-timeout 300000 --fetch-retries 8`.
- **`pnpm install` rewrites `pnpm-lock.yaml`.** That churn is expected; the sync
  script discards it before rebasing. Do not commit it as a real change.
- **The mac artifact build needs Rust** for the bundled `t3-resource-monitor`.
  Homebrew's `rustup` is keg-only and ships no `rustup-init`; its shims live at
  `/opt/homebrew/opt/rustup/bin`.
- **systemd user units do not read your shell profile.** anvil's unit carries an
  explicit `Environment=PATH=` covering `~/.local/bin`, `~/.grok/bin` and
  `~/.opencode/bin`, or half the provider CLIs vanish.

## The Devin provider

Driver files live in `apps/server/src/provider/`: `Drivers/DevinDriver.ts`,
`Layers/DevinAdapter.ts`, `Layers/DevinProvider.ts`, `acp/DevinAcpSupport.ts`, and
`textGeneration/DevinTextGeneration.ts`. It is registered in `builtInDrivers.ts`
alongside the six upstream drivers.

It spawns `devin acp` and speaks ACP protocol version 1, matching
`packages/effect-acp`'s `PROTOCOL_VERSION`. Auth comes from the operator's normal
`devin auth login` session at `~/.local/share/devin/credentials.toml`, with browser
OAuth as fallback and `DEVIN_API_KEY` as an optional override. There is no separate
API key to provision. On a headless box, log in with
`devin auth login --force-manual-token-flow`.

Devin runs wherever the _server_ runs, not where the GUI runs. Selecting anvil as the
environment means anvil spawns `devin acp` and uses anvil's credentials.

Tests:

```bash
cd apps/server
vp test run src/provider/acp/DevinAcpSupport.test.ts \
            src/provider/Layers/DevinAdapter.test.ts \
            src/provider/Layers/DevinProvider.test.ts
```

`acp/DevinAcpCliProbe.test.ts` is the real-CLI integration test, gated behind
`T3_DEVIN_ACP_PROBE=1` with a live turn behind `T3_DEVIN_LIVE_TURN=1`. It needs the
Devin CLI installed and logged in, and it **spends the operator's Devin quota**. Run
it when you touch the ACP transport, not on every change.

Per upstream's "hit every surface" rule, provider-shaped changes now need a decision
for seven adapters, not six. Devin is easy to forget because it is not upstream.

## Do not disturb the operator's T3 Code

Upstream rule 2 already says never write to `~/.t3/userdata`. Additions here:

- On the Mac, the packaged app uses `~/.t3/userdata` and port `3773`. A dev run uses
  `~/.t3/dev` and port `13773`; `~/.t3-devfork` is a fully isolated home used when
  even sharing `caches/` and `worktrees/` is unwelcome.
- On anvil the server runs as the systemd user unit `t3code.service` against
  `T3CODE_HOME=/home/walde/.t3`. Restart it with `systemctl --user restart
t3code.service`, never by killing PIDs.
- anvil originally ran upstream's self-updating versioned runtime
  (`~/.t3/runtime/service-launcher.mjs`). We replaced `ExecStart` with the fork's
  built server, which means **anvil no longer auto-updates** — `anvil-sync-t3.sh` is
  what keeps it current. The original unit is kept at
  `~/t3-backups/t3code.service.launcher.bak`.
