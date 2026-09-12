# Fork notes

Fork-local rules for coding agents. `AGENTS.md` is upstream's and still applies in
full — this file only covers what is different here. Where they disagree, this file
wins, because it describes this checkout.

This file is fork-only and does not exist upstream, so it never conflicts on rebase.
Keep it that way: put fork-specific guidance here, not in `AGENTS.md`.

## What this fork is

Upstream `pingdotgg/t3code` plus PR [#11356](https://github.com/pingdotgg/t3code/pull/11356),
which adds Devin as a first-class ACP provider. Nothing else, yet.

|                |                                            |
| -------------- | ------------------------------------------ |
| Checkout       | `~/code/t3code-devin`                      |
| Working branch | `devin-11356`                              |
| `upstream`     | `pingdotgg/t3code` (read only, never push) |
| `origin`       | `walde-dev/t3code` (the fork, push here)   |

Upstream has not merged any Devin PR; at least eight attempts since July died
unreviewed. Assume this stays a fork indefinitely rather than waiting for it to land.

## Staying current with upstream

"Nightly" is not a branch. The nightly channel is a tag cut from `main` a few times a
day, so tracking nightly means rebasing onto `upstream/main`. Use the script, not raw
git:

```bash
~/code/sync-t3-devin.sh --rebase --push   # the routine command
~/code/sync-t3-devin.sh --rebase          # rebase + test, no push
LIVE=1 ~/code/sync-t3-devin.sh --rebase   # also run the live Devin CLI probe
```

It refuses on a dirty tree, writes a `sync-rollback/<timestamp>` tag before touching
anything, and on conflict aborts, restores the branch, and prints the conflicting
files. Nothing lands half-applied.

**`--rebase` is the mode you want.** `--track-pr` re-fetches the PR head and
`reset --hard`s onto it, which discards every commit of your own. The script blocks
this when it detects commits above `upstream/main`, and `FORCE=1` overrides the block.
Only use it to deliberately abandon local work and re-sync with the PR author.

That check compares patch-ids and is deliberately conservative: once the branch has
been rebased, the PR's own commits can shift patch-id and get listed alongside yours.
It over-reports rather than silently destroying work, so read the list before
reaching for `FORCE=1`.

Expect conflicts to be rare: the PR is 11 new files (4301 lines, which cannot
conflict) and 13 modified files (329 lines total). The real touchpoints are small —
`builtInDrivers.ts` 5 lines, `contracts/model.ts` 10, `contracts/settings.ts` 33.

Pushes to `origin` are always forced, because rebasing rewrites SHAs. The script uses
`--force-with-lease`, which is the safe form. Do not swap it for plain `--force`.

## Toolchain gotchas on this machine

- **Node 24 is required** (`engines: ^24.13.1`). The default `node` here is v25 and
  will fail the engine check. Use `~/.nvm/versions/node/v24.18.1/bin`.
- **`vp` is not global.** It installs to `node_modules/.bin`, but `scripts/dev-runner.ts`
  spawns `vp` from `PATH`. Export `$REPO/node_modules/.bin` or dev dies with
  `spawn vp ENOENT`.
- **pnpm needs throttling.** The default concurrency times out fetching large metadata
  docs (`@clerk/shared`, `@clerk/react`) against this network. Use
  `pnpm install --network-concurrency 4 --fetch-timeout 300000 --fetch-retries 8`.
  A bare `pnpm install` may fail two or three times before it succeeds.
- **`pnpm install` rewrites `pnpm-lock.yaml`.** That churn is expected; the sync script
  discards it before rebasing. Do not commit it as if it were a real change.

## The Devin provider

Driver files live in `apps/server/src/provider/`: `Drivers/DevinDriver.ts`,
`Layers/DevinAdapter.ts`, `Layers/DevinProvider.ts`, `acp/DevinAcpSupport.ts`, and
`textGeneration/DevinTextGeneration.ts`. It is registered in `builtInDrivers.ts`
alongside the six upstream drivers.

It spawns `devin acp` and speaks ACP protocol version 1, matching
`packages/effect-acp`'s `PROTOCOL_VERSION`. Auth comes from the operator's normal
`devin auth login` session at `~/.local/share/devin/credentials.toml`, with browser
OAuth as fallback and `DEVIN_API_KEY` as an optional override. There is no separate
API key to provision.

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

Upstream rule 2 already says never write to `~/.t3/userdata`. Two additions here:

- A **production T3 Code (Nightly) instance is usually running** on port `3773`. Leave
  that process and that port alone. Read `~/.t3/userdata/server-runtime.json` to see
  its PID before assuming a stray process is yours.
- This fork's dev instance uses `~/.t3-devfork` (server `13773`, web `5733`), kept
  separate so `caches/` and `worktrees/` are not shared with the live install either.
  Worktree runs still default to `<worktree>/.t3`, per upstream.
