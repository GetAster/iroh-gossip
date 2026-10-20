# Aster patch ledger: iroh-gossip

This repository is Aster's fork of [n0-computer/iroh-gossip](https://github.com/n0-computer/iroh-gossip). Aster carries the patches listed below on top of upstream releases. Upstream maintainers are welcome to incorporate any of them, in whole or in part, at their discretion.

## Branch and tag model

- `main` is the front door: upstream's README with an Aster preamble, plus this ledger. Nothing is built from `main`.
- `upgrade/<version>` is the patched integration line for one upstream release wave: the upstream release plus the patches in this ledger. Consumers pin it by commit.
- Topic branches carry one change each and are named for what the change is, for example `feat/read-into` or `security/plumtree-bounds`. Each is based on the upstream release tag and is cherry-picked with `git cherry-pick -x` onto the upgrade branch, so every upgrade-branch commit names its source.
- Release tags mark the commits that are released and pinned. Published tags are never moved or rewritten. Historical tags in this repository use several schemes (listed below); new tags use `aster-<repo>-v<version>`, where `<version>` is the published version of the repository's main package (for example `aster-salvo-v1.0.4`, `aster-fory-v1.7.6-aster.1`).
- `main` is refreshed by merging upstream `main` into it. It is not force-pushed.

Tag schemes used in this repository:

- `aster-iroh-gossip-v<version>`, for example `aster-iroh-gossip-v0.99.0`, `aster-iroh-gossip-v0.100.0`, `aster-iroh-gossip-v0.101.1`.
- `aster-iroh-gossip-v<version>-p<n>`, a later commit at the same crate version: `aster-iroh-gossip-v0.101.0-p1`, `-v0.101.0-p2`, and `-v0.101.1-p1` (a patch-block re-pin with no source change).

## Current release

| Field | Value |
|---|---|
| Upstream base | `v0.101.0` |
| Integration branch | `upgrade/iroh-gossip-v0.101` |
| Tag | `aster-iroh-gossip-v0.101.1-p1` |
| Pinned commit | `c0e4a1f2a05ed721122f9289eb27ad8330a9182f` |
| Crate version | 0.101.1 (workspace spec `iroh-gossip = "0.101.1"`) |
| Source repository | `https://forge.emrul.dev/Aster/iroh-gossip.git` |

The fork's own `[patch.crates-io]` (row 17) pins iroh `a4a9a76b` (1.3.2) and noq `e75f3a55` (1.3.2), the stack Aster links it against. The crate code is 0.101.1 (`aed725d1`, row 29); `-p1` changes only the patch block and lock.

Bootstrap pin line for source-tree builds (consumers install a versioned Aster SDK release instead and must not copy this):

```toml
iroh-gossip = { git = "https://forge.emrul.dev/Aster/iroh-gossip.git", rev = "c0e4a1f2a05ed721122f9289eb27ad8330a9182f" }
```

## Patches

Row numbers are stable patch ids shared across Aster's iroh-family forks (iroh, iroh-blobs, iroh-docs, iroh-gossip, iroh-util, noq), so a commit message that says "row 29" resolves here.

| Id | Topic branch | What | Why / what in Aster depends on it | Commits (topic / upgrade) | Status | Conflict or porting notes |
|---|---|---|---|---|---|---|
| 17 | none recorded (original commit `14a76d5`) | `[patch.crates-io]` block pointing the iroh/noq crates at the Aster forks. File: `Cargo.toml`. | Keeps gossip builds on the Aster fork stack and avoids mixing crates.io iroh-base/noq with forked iroh. | `14a76d5` (original); rc.1 copy `511613f`; re-pinned since, latest in `c0e4a1f2` (0.101.1-p1) | carried | Low conflict risk. Pin explicit fork revs or tags, and re-check comments and direct dependency versions after each upstream release. In the 1.0 port it hit a trailing-EOF conflict in `Cargo.toml`. |
| 29 | `fix/upstream-lifecycle-port` (based on `de100e9`, the 0.101.0-p2 pin, not on the upstream tag) | Ports upstream's unreleased connection and peer lifecycle fixes (upstream `main`, 2026-10-09..14; latest release then v0.101.0). **#161** (fixes #172 and #174): prunes `peer_topics` on disconnect, keeps a peer while another topic still uses it, clears HyParView peer data, answers a `Join` from a peer that is already a neighbour (restart with the same id), and raises the neighbour-request timeout from 500 ms to 3 s. **#178** (fixes #145, #101, #155): a failed dial or a closed connection drops the net actor's peer state, so the next `Join` dials again; unused connections close 5 s after no topic uses them; a write error drops only its stream; a repeated `Gossip::shutdown` returns `Ok`. **#189**: a flags byte after the topic in the stream header; only peers that set it get the close timer, and 0.90–0.101 nodes ignore the trailing byte. **#183**: a cancelled task at runtime shutdown stops the actor without a panic. **#191**: maintenance every 5 s refills below half the active view and rejoins through the last 8 contacts when alone; refill at once after a failed request dial; first shuffle about 1 s after a join; frames from non-neighbours no longer change peer sets; re-exports `Ttl` and `Round`. Files: `Cargo.toml`, `src/net.rs`, `src/net/util.rs`, `src/proto.rs`, `src/proto/{hyparview,plumtree,sim,state,topic}.rs`, `tests/e2e.rs`, `tests/paper.rs`. | The 2026-10-15 Aster restart probe (Aster repository, `docs/_internal/investigations/2026-10-15-restart-reconnect.md`, cause 4): gossip never re-dialled a peer after one failed dial, for every topic including Docs live sync. #178 fixes that. Recovery after a peer restart is still bound by iroh's stale connection (causes 1–2, handled in the iroh and noq forks): the dead connection stays `Active` until the 30 s QUIC idle timeout. | upstream `b89fa98` (#183), `fcd81ae` (#161), `f02f79e` (#178), `cd73d93` (#189), `dfeec14` (#191), cherry-picked `-x` as `b0bbb05`, `e372f13`, `26792b9`, `4099a1b`, `2ce6e82` / the same commits on `upgrade/iroh-gossip-v0.101`; released as 0.101.1 at `aed725d1` | carried | Clean apart from `Cargo.lock` on #183 (upstream's syn/yoke-derive bump; the fork's lock kept). Source equals upstream `main` (`6cbbafa`) except `src/bin/sim.rs` (upstream `f128b98`, clippy-only, not carried). Behaviour changes upstream wants as semver-major: unused connections close after 5 s, and shuffles start about 1 s after a join. Downstream effect: a node whose first gossip dial was refused now joins the swarm once admitted, so HyParView `ForwardJoin` meshes it with the other members (seen in Aster `enrollment_replica_trust`). Upstream known bugs still open: a former instance's `Disconnect` removes its same-id replacement (`known_bug_former_instance_leaving_preserves_replacement`), and #47 (a slow peer blocks the actor). Tests: lib 88, e2e 26, paper 25, sim 4, doc 1; `--no-default-features` lib 54; clippy `--all-features --all-targets -D warnings` and fmt clean. **Drop when a released iroh-gossip containing upstream #161, #172 (fixed by #161), #178, #183, #189 and #191 is pinned.** |

Bookkeeping and documentation commits (not needed at runtime):

| Commits | Files | Purpose | Status |
|---|---|---|---|
| `d7d1358` (original) / `0bbff98` (rc.1) | `.gitignore` | Ignores local `docs/aster/` internal review docs. | Recorded in the rc.1 stack; current status not recorded. Optional. |
| `fdecec8` (rc.1) | `Cargo.lock` | Resolves the Aster iroh/noq fork crates. Regenerated each wave. | carried in spirit (regenerated). |
| `161cec0` | `README.md` | Aster fork overview and contribution table. | Replaced by the `main` README preamble under the current model; if reused, refresh PR statuses. |

### Upstreamed or dropped

No iroh-gossip patch has been dropped. Row 29 is itself a backport of upstream work that is merged on upstream `main` but unreleased; it becomes `upstreamed` once a release containing it is pinned.

### Closed security-review branches

These topic branches came from an April 2026 security review. The fixes from that review that held up were merged upstream, so these branches are not carried on the upgrade branch. They are kept for reference.

| Branch |
|---|
| `security/message-size-consistency` |
| `security/panic-on-task-failure` |
| `security/per-peer-connection-limit` |
| `security/plumtree-bounds` |
| `security/rwlock-poisoning` |

## Porting to a new upstream release

Order across the iroh family: noq, then iroh, then iroh-blobs, then iroh-docs and this repository. This fork's `[patch.crates-io]` names iroh and noq by revision, so re-pin it after those move (dependency order noq → iroh → iroh-blobs → iroh-docs / iroh-gossip).

Before touching branches, read the upstream changelog and range from the previous base (the rc.1 port reviewed `v0.99.0..v0.100.0`). If the target release contains upstream #161, #178, #183, #189 and #191, drop row 29.

```bash
git fetch --all --tags --prune
git push origin <new-upstream-tag>      # mirror the upstream tag to the forge
git switch -c upgrade/iroh-gossip-<new-version> <new-upstream-tag>
# row 17: re-point [patch.crates-io] at the new iroh/noq fork revs, regenerate Cargo.lock
# row 29: cherry-pick -x only the upstream fixes the release does not yet contain
git tag -a aster-iroh-gossip-v<published version> -m "Aster iroh-gossip v<published version>"
git push origin upgrade/iroh-gossip-<new-version> aster-iroh-gossip-v<published version>
```

Prefer `cherry-pick -x`. Do not rewrite a published `upgrade/*` branch or tag unless you are deliberately replacing a failed attempt.

Validate in this repository:

```bash
cargo fmt --check
cargo test --locked --all-features      # lib, e2e, paper, sim, doc
cargo clippy --all-features --all-targets -- -D warnings
cargo clippy --no-default-features --all-targets -- -D warnings
```

Whenever iroh-gossip moves, re-pin the iroh-docs fork's own `[patch.crates-io]` to it and run that fork's full suite (`cargo test --all-features`, 118 tests including `tests/sync.rs`): iroh-docs exercises the whole stack.

Gotcha when re-pinning this fork's patch block: `cargo update -p iroh ...` kept crates.io iroh 1.0.2 and reported the patches unused; `cargo update -p iroh --precise 1.3.2` moved it.

Aster validation after updating pins, from the Aster repository root:

```bash
cargo fmt --check
cargo clippy --manifest-path bindings/python/rust/Cargo.toml -- -D warnings
cargo test -p aster_transport_core
cargo test -p aster --all-features
uv run maturin develop -m bindings/python/rust/Cargo.toml
uv run pytest tests/python/ -v --timeout=30
```

Update the fork manifest (canonical `docs/aster-fork-manifest.toml` in the Aster iroh fork; synced copy `iroh-fork-manifest.toml` in the Aster repository) and this ledger's Current release section. Local path-override consumers (for example portal-sync) build whatever is checked out, so switch the local checkout to the released tag after publishing.

Before the current branch model, fork `main` mirrored upstream `main` and was reset to it on each port (`git reset --hard upstream/main`, `git push origin main --force-with-lease`). Under the current model `main` is the front door and is not reset.

## History

### 1.3.0 wave (2026-10-03 onward), current

No upstream iroh-gossip release forced a re-port; base stays `v0.101.0`.

- Same-day follow-up after noq 1.3.1: the fork's own patch block was re-pinned to iroh `1974e958` and noq `867a3a9a`. Lock and pin changes only; the publisher skipped the unchanged version as identical. Validation: iroh-gossip 25 tests, clippy clean.
- **0.101.1** (`aed725d1`, tag `aster-iroh-gossip-v0.101.1`): row 29, the port of upstream's unreleased lifecycle fixes, from the 2026-10-15 restart investigation.
- **2026-10-18, 0.101.1-p1** (`c0e4a1f2`, tag `aster-iroh-gossip-v0.101.1-p1`): re-pins the fork's own patch block to the stack Aster links it against (review finding TM-7, found by `fork_compat.py --check`): iroh `1974e958` → `a4a9a76b` (1.3.2), noq `867a3a9a` → `e75f3a55` (1.3.2). No source change; the crate stays 0.101.1 and the publisher skips it as identical. Validation: `cargo test --locked --all-features` lib 88, e2e 26, paper 25, sim 4, doc 1 (144 in all); clippy `--all-features --all-targets` and `--no-default-features` `-D warnings`; `cargo fmt --check` clean.

### 1.0.1 wave (2026-07-02)

| Upstream base | Tag | Pinned commit | Branch |
|---|---|---|---|
| `v0.101.0` | `aster-iroh-gossip-v0.101.0-p2` | `de100e9aa59386e39a32d6744bd902935c16fcf6` | `upgrade/iroh-gossip-v0.101` |

- No upstream release; pin unchanged from the 1.0 wave.
- **2026-08-13 sweep (`-p1` tags):** the fork's own `[patch.crates-io]`, which had named the GitHub mirror at the 1.0.0-wave iroh (`3c329f0c`) and noq (`ddb8829d`), was moved to Forgejo at this wave's revisions, in dependency order noq → iroh → iroh-blobs → iroh-docs / iroh-gossip. No `src` change, nothing republished.
- Validation: per-fork fmt/clippy/tests, then downstream `aster_transport_core` (242) and `aster --all-features` (94), PyO3 clippy clean, `maturin develop` and 1117 Python tests.
- Local checkout at the end of the cycle: `aster-iroh-gossip-v0.101.0-p1`.

### 1.0 wave (2026-06-21)

| Upstream base | Tag | Pinned commit | Branch |
|---|---|---|---|
| `v0.101.0` | `aster-iroh-gossip-v0.101.0-p2` | `de100e9aa59386e39a32d6744bd902935c16fcf6` | `upgrade/iroh-gossip-v0.101` |

- Patch stack carried forward unchanged from rc.1; cherry-picked clean except a trailing-EOF patch-block conflict in `Cargo.toml`. Tag == pinned commit == validated code.
- Validation: per-fork fmt/clippy/tests, then `aster_transport_core` (229) and `aster` (68) Rust tests and `maturin develop` + 1117 Python tests.
- Local checkout at the end of the cycle: `aster-iroh-gossip-v0.101.0-p1`.

### rc.1 port (2026-06-07)

- Before the port Aster pinned `d7d13582ba` (tag `aster-iroh-gossip-v0.99.0`), upstream base `v0.99.0` (`65b35eb`), branch `upgrade/iroh-gossip-v0.99`; stack range `v0.99.0..aster-iroh-gossip-v0.99.0`. Target `v0.100.0` (`37dcb89`).
- Ported and published: fork `main` mirrored upstream `main` (`37dcb89`); branch `upgrade/iroh-gossip-v0.100` and tag `aster-iroh-gossip-v0.100.0` pushed to `aster-rpc/iroh-gossip`, based on `v0.100.0`.

| Commit | Purpose |
|---:|---|
| `511613f` | Cherry-pick of `14a76d5`: Aster iroh/noq `[patch.crates-io]` entries, updated to explicit rc1 revs. |
| `0bbff98` | Cherry-pick of `d7d1358`: ignore local `docs/aster/` internal notes. |
| `fdecec8` | `Cargo.lock` resolves the Aster iroh/noq fork crates. |

- Validation: `cargo fmt --check`, `cargo test --all-features`.
- The rc.1 command sequence was `git cherry-pick -x 14a76d5` onto `upgrade/iroh-gossip-v0.100` from `v0.100.0` (after reviewing `v0.99.0..v0.100.0`), then tag `aster-iroh-gossip-v0.100.0`. The Aster root patch block then pointed at `https://github.com/aster-rpc/iroh-gossip`; it now points at the Forgejo repository.
- Local checkout at the end of the cycle: `aster-iroh-gossip-v0.100.0`.
