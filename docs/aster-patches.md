# Aster patch ledger: iroh-gossip

This repository is Aster's fork of [n0-computer/iroh-gossip](https://github.com/n0-computer/iroh-gossip). Aster carries the patches listed below on top of upstream releases. Upstream maintainers are welcome to incorporate any of them, in whole or in part, at their discretion.

## Branch and tag model

- `main` is upstream `main` plus one commit: the Aster README preamble and this ledger. Nothing is built from `main`. It is refreshed with `git merge upstream/main` and is never reset or force-pushed. Ledger edits are commits on `main`, never on `upgrade/*`.
- `upgrade/<repo>-v<upstream version>` (for example `upgrade/iroh-v1.4.0`, `upgrade/iroh-blobs-v0.103.1`) is the patched integration line: the upstream release plus the patches in this ledger. A new upstream base always gets a new branch, cut from the upstream release tag; Aster-only fixes and re-pins append to the current branch. Per-minor names (`upgrade/iroh-blobs-v0.103`, `upgrade/salvo-v1.0`, …) are historical. Consumers pin commits on it. Published `upgrade/*` branches and tags are never rewritten; a failed attempt is superseded by new commits and a new version.
- Topic branches carry one change each, are named for the change (for example `feat/read-into`, `security/plumtree-bounds`), are based on the upstream release tag (or, for a patch that builds on another, on that prerequisite patch's commit, which its row names), and are cherry-picked with `git cherry-pick -x` onto the upgrade branch, so every upgrade-branch commit names its source. A topic branch a row names is not rewritten; re-cutting it onto a new upstream base takes a versioned name, `<topic>-v<upstream version>`.
- Version: the upstream version if Aster's registry has never served that version or a higher one in the series; otherwise the next patch above Aster's highest published version. A re-pin without a code change keeps the version and takes the tag suffix `-p<n>`.
- Tags: `aster-<repo>-v<published version>` (re-pins `aster-<repo>-v<published version>-p<n>`), created only after the Aster consumer trial passes against those exact commits, because tags cannot be moved.
- The fork manifest is `iroh-fork-manifest.toml` in the Aster repository (the only copy).
- Aster does not submit patches upstream; upstream maintainers are welcome to take any patch here.

Historical and current tag schemes in this repository:

- `aster-iroh-gossip-v<version>`, for example `aster-iroh-gossip-v0.99.0`, `aster-iroh-gossip-v0.100.0`, `aster-iroh-gossip-v0.101.1`.
- `aster-iroh-gossip-v<version>-p<n>`, a later commit at the same crate version: `aster-iroh-gossip-v0.101.0-p1`, `-v0.101.0-p2`, `-v0.101.1-p1`, `-v0.101.1-p2`, `-v0.101.1-p3` and `-v0.101.1-p4` (patch-block re-pins with no source change).

## Current release

| Field | Value |
|---|---|
| Upstream base | `v0.101.0` |
| Integration branch | `upgrade/iroh-gossip-v0.101` (a historical per-minor name; the next upstream base gets `upgrade/iroh-gossip-v<upstream version>`) |
| Tag | `aster-iroh-gossip-v0.101.1-p6` |
| Pinned commit | `19b4bfbf0fd12c5a44e3d2ddc222b25617faa38d` |
| Crate version | 0.101.1 (workspace spec `iroh-gossip = "0.101.1"`) |
| Source repository | `https://forge.emrul.dev/Aster/iroh-gossip.git` |

The fork's own `[patch.crates-io]` (row 17) pins iroh `0739c259` (1.4.2001) and noq `a3c70b7a` (1.4.2), the stack Aster links it against. The crate code is 0.101.1 (`aed725d1`, row 29); `-p1` to `-p6` change only the patch block and lock.

Bootstrap pin line for source-tree builds (consumers install a versioned Aster SDK release instead and must not copy this):

```toml
iroh-gossip = { git = "https://forge.emrul.dev/Aster/iroh-gossip.git", rev = "19b4bfbf0fd12c5a44e3d2ddc222b25617faa38d" }
```

## Patches

Row numbers are stable patch ids shared across Aster's iroh-family forks (iroh, iroh-blobs, iroh-docs, iroh-gossip, iroh-util, noq), so a commit message that says "row 29" resolves here.

| Id | Topic branch | What | Why / what in Aster depends on it | Commits (topic / upgrade) | Status | Conflict or porting notes |
|---|---|---|---|---|---|---|
| 17 | none recorded (original commit `14a76d5`) | `[patch.crates-io]` block pointing the iroh/noq crates at the Aster forks. File: `Cargo.toml`. | Keeps gossip builds on the Aster fork stack and avoids mixing crates.io iroh-base/noq with forked iroh. | `14a76d5` (original); rc.1 copy `511613f`; re-pinned since, latest in `0de88183` (0.101.1-p4) | carried | Low conflict risk. Pin explicit fork revs or tags, and re-check comments and direct dependency versions after each upstream release. In the 1.0 port it hit a trailing-EOF conflict in `Cargo.toml`. |
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

Order across the iroh family: noq, then iroh, then the satellites in the Aster publisher's order: iroh-util, this repository, iroh-blobs, iroh-docs. This fork's `[patch.crates-io]` names iroh and noq by revision, so re-pin it after those move; iroh-docs pins this fork, so it comes after.

Before touching branches, read the upstream changelog and range from the previous base (the rc.1 port reviewed `v0.99.0..v0.100.0`). If the target release contains upstream #161, #178, #183, #189 and #191, drop row 29.

```bash
git fetch --all --tags --prune
git push origin <upstream-tag>          # mirror the upstream tag (e.g. v0.102.0) to the forge
git switch -c upgrade/iroh-gossip-v<upstream version> <upstream-tag>
# row 17: re-point [patch.crates-io] at the new iroh/noq fork revs, regenerate Cargo.lock, commit
# row 29: cherry-pick -x only the upstream fixes the release does not yet contain
# version bump, only if the version rule requires it (Aster's registry has already
# served <upstream version> or higher in the series): set the crate to the next
# patch above Aster's highest published, then
git commit -am "chore: release <published version>"
git push origin upgrade/iroh-gossip-v<upstream version>
# do not tag yet: tag after the consumer trial (below)
```

Prefer `cherry-pick -x`. Published `upgrade/*` branches and tags are never rewritten; a failed attempt is superseded by new commits and a new version.

An Aster-only fix on the current base is a topic branch, cherry-picked `-x` onto the current upgrade branch, followed by a version-bump commit to the next patch above Aster's highest published version (as 0.101.1 was cut for row 29). A re-pin without a code change (row 17 only) is a commit on the current upgrade branch with no version bump, tagged `aster-iroh-gossip-v<current version>-p<n>` after the consumer trial (as `aster-iroh-gossip-v0.101.1-p1` was).

Validate in this repository:

```bash
cargo fmt --check
cargo test --locked --all-features      # lib, e2e, paper, sim, doc
cargo clippy --all-features --all-targets -- -D warnings
cargo clippy --no-default-features --all-targets -- -D warnings
```

Whenever iroh-gossip moves, re-pin the iroh-docs fork's own `[patch.crates-io]` to it and run that fork's full suite (`cargo test --all-features`, 118 tests including `tests/sync.rs`): iroh-docs exercises the whole stack.

Gotcha when re-pinning this fork's patch block: `cargo update -p iroh ...` kept crates.io iroh 1.0.2 and reported the patches unused; `cargo update -p iroh --precise 1.3.2` moved it. When the lock holds two iroh entries (crates.io and the fork), name the old one: `cargo update -p iroh@<old> --precise <new>`. Update only the crates you name (`cargo update -p`), and revert any unrelated lock churn.

Aster validation after updating pins, from the Aster repository root:

```bash
cargo fmt --check
cargo clippy --manifest-path bindings/python/rust/Cargo.toml -- -D warnings
cargo test -p aster_transport_core
cargo test -p aster --all-features
uv run maturin develop -m bindings/python/rust/Cargo.toml
uv run pytest tests/python/ -v --timeout=30
```

Tag only after the Aster consumer trial passes against these exact commits (tags cannot be moved):

```bash
git tag -a aster-iroh-gossip-v<published version> -m "Aster iroh-gossip v<published version>" <release commit>
git push origin aster-iroh-gossip-v<published version>
# re-pin only: git tag -a aster-iroh-gossip-v<current version>-p<n> -m "..." <re-pin commit>
```

Update `iroh-fork-manifest.toml` in the Aster repository (the only copy), and this ledger's Current release section on `main` (merge upstream `main` first). Local path-override consumers (for example portal-sync) build whatever is checked out, so switch the local checkout to the released tag after publishing.

Before the current branch model, fork `main` mirrored upstream `main` and was reset to it on each port (`git reset --hard upstream/main`, `git push origin main --force-with-lease`). Under the current model `main` is upstream `main` plus one commit (README preamble and this ledger), refreshed with `git merge upstream/main`, and is never reset or force-pushed.

## History

### iroh 1.4.2001 re-pin (2026-10-25), current

| Upstream base | Tag | Pinned commit | Branch |
|---|---|---|---|
| `v0.101.0` | `aster-iroh-gossip-v0.101.1-p6` | `19b4bfbf0fd12c5a44e3d2ddc222b25617faa38d` | `upgrade/iroh-gossip-v0.101` |
| `v0.101.0` | `aster-iroh-gossip-v0.101.1-p5` (pinned the superseded iroh 1.4.1, never published) | `aebdf0231670b803c3af65ce6c67d3439a02a275` | `upgrade/iroh-gossip-v0.101` |

- **`aster-iroh-gossip-v0.101.1-p6`** (`19b4bfbf`): re-pin only, to iroh `0739c259` (`aster-iroh-v1.4.2001`), the same iroh source as `aster-iroh-gossip-v0.101.1-p5` published under the fork version rule; the iroh 1.4.1 commit `aster-iroh-gossip-v0.101.1-p5` pinned was tagged but never published. Suite re-run at the new commit, all passed.
- **`aster-iroh-gossip-v0.101.1-p5`** (`aebdf023`): re-pin only, to iroh `68b684ea` (`aster-iroh-v1.4.1`: the port onto upstream 1.4.1 and row 44, the net report no longer waiting 3 s on an IPv6 path that drops UDP). Only Cargo.toml's `[patch.crates-io]` and Cargo.lock change; the crate is not republished. Validation: `cargo test --locked --all-features`, lib 88, integration and doc tests 26 + 25 + 4 + 1, all passed.

### noq 1.4.1 and 1.4.2 re-pins (2026-10-22)

| Upstream base | Tag | Pinned commit | Branch |
|---|---|---|---|
| `v0.101.0` | `aster-iroh-gossip-v0.101.1-p4` | `0de881834dc91e748bad8028c9f8bd9fd28c26b8` | `upgrade/iroh-gossip-v0.101` |
| `v0.101.0` | `aster-iroh-gossip-v0.101.1-p3` | `a45ab096bb196fe0f418e945a9d2d754be06eaab` | `upgrade/iroh-gossip-v0.101` |

- **0.101.1-p3** (`a45ab096`): re-pin only (row 17), to iroh `41ec78b1` (1.4.0-p1) and noq `13dcb5b7` (1.4.1).
- **0.101.1-p4** (`0de88183`): re-pin only (row 17), to iroh `91a92aa0` (1.4.0-p2) and noq `a3c70b7a` (1.4.2, incremental BBRv3 packet history); the lock follows with no other moves. No source change; the crate stays 0.101.1 and is not republished. Validation on p4: `cargo test --locked --all-features` lib 88, e2e 26, paper 25, sim 4, doc 1 (144); clippy `--all-features --all-targets` and `--no-default-features` `-D warnings`, fmt clean.

### 1.4.0 wave (2026-10-21)

| Upstream base | Tag | Pinned commit | Branch |
|---|---|---|---|
| `v0.101.0` | `aster-iroh-gossip-v0.101.1-p2` | `5fde770e167767eadfd208193cb2bf667096e6a7` | `upgrade/iroh-gossip-v0.101` |

- No upstream iroh-gossip release after `v0.101.0`, so no port; row 29 stays a backport.
- **2026-10-21, 0.101.1-p2** (`5fde770e`, tag `aster-iroh-gossip-v0.101.1-p2`): re-pin only (row 17). iroh, iroh-base and iroh-relay `a4a9a76b` → `261ad623` (1.4.0, `upgrade/iroh-v1.4.0`); noq, noq-udp and noq-proto `e75f3a55` → `535bd884` (1.4.0, `upgrade/noq-v1.4.0`); the lock follows (rustls 0.23.46, lru 0.18.5). The commit touches only Cargo.toml's `[patch.crates-io]` and Cargo.lock. No source change; the crate stays 0.101.1 and is not republished. Validation: `cargo test --locked --all-features` lib 88, e2e 26, paper 25, sim 4, doc 1 (144 in all); clippy `--all-features --all-targets` and `--no-default-features` `-D warnings`; fmt clean.

### 1.3.0 wave (2026-10-03 onward)

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
