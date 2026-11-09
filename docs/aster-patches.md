# Aster patch ledger: iroh-gossip

This repository is Aster's fork of [n0-computer/iroh-gossip](https://github.com/n0-computer/iroh-gossip). Aster carries the patches listed below on top of upstream releases. Upstream maintainers are welcome to incorporate any of them, in whole or in part, at their discretion.

## Branch and tag model

- `main` is upstream `main` plus one commit: the Aster README preamble and this ledger. Nothing is built from `main`. It is refreshed with `git merge upstream/main` and is never reset or force-pushed. Ledger edits are commits on `main`, never on `upgrade/*`.
- `upgrade/<repo>-v<upstream version>` (for example `upgrade/iroh-v1.4.0`, `upgrade/iroh-blobs-v0.103.1`) is the patched integration line: the upstream release plus the patches in this ledger. A new upstream base always gets a new branch, cut from the upstream release tag; Aster-only fixes and re-pins append to the current branch. Per-minor names (`upgrade/iroh-blobs-v0.103`, `upgrade/salvo-v1.0`, …) are historical. Consumers pin commits on it. Published `upgrade/*` branches and tags are never rewritten; a failed attempt is superseded by new commits and a new version.
- Topic branches carry one change each, are named for the change (for example `feat/read-into`, `security/plumtree-bounds`), are based on the upstream release tag (or, for a patch that builds on another, on that prerequisite patch's commit, which its row names), and are cherry-picked with `git cherry-pick -x` onto the upgrade branch, so every upgrade-branch commit names its source. A topic branch a row names is not rewritten; re-cutting it onto a new upstream base takes a versioned name, `<topic>-v<upstream version>`.
- Version (option E, from 2026-10-25): a release with Aster patches publishes as `major.minor.P` with `P = 1000 * (upstream_patch + 1) + n`, where `n` (1 to 999) counts Aster patch releases on that upstream version (upstream 0.101.0 plus the first is `0.101.1001`), so it never collides with a crates.io release. Canonical rule: `_internal/docs/aster-rpc/261003_fork-pin-upgrade-runbook.md`; `publish-native-stack.py check` enforces it. Versions already published (0.101.1 and earlier) are grandfathered. A re-pin without a code change keeps the version and takes the tag suffix `-p<n>`.
- Tags: `aster-<repo>-v<published version>` (re-pins `aster-<repo>-v<published version>-p<n>`), created only after the Aster consumer trial passes against those exact commits, because tags cannot be moved.
- The fork manifest is `iroh-fork-manifest.toml` in the Aster repository (the only copy).
- Aster does not submit patches upstream; upstream maintainers are welcome to take any patch here.

Historical and current tag schemes in this repository:

- `aster-iroh-gossip-v<version>`, for example `aster-iroh-gossip-v0.99.0`, `aster-iroh-gossip-v0.100.0`, `aster-iroh-gossip-v0.101.1`, `aster-iroh-gossip-v0.101.1002`, `aster-iroh-gossip-v0.102.1001`.
- `aster-iroh-gossip-v<version>-p<n>`, a later commit at the same crate version: `aster-iroh-gossip-v0.101.0-p1`, `-v0.101.0-p2`, `-v0.101.1-p1`, `-v0.101.1-p2`, `-v0.101.1-p3` and `-v0.101.1-p4` (patch-block re-pins with no source change).

## Current release

| Field | Value |
|---|---|
| Upstream base | `v0.102.0` (`a16f016c`) |
| Integration branch | `upgrade/iroh-gossip-v0.102.0` |
| Tag | `aster-iroh-gossip-v0.102.1001` |
| Pinned commit | `39a2ed2c8b5e99d2fb49f887e9589804cd441c82` |
| Crate version | 0.102.1001 (workspace spec `iroh-gossip = "0.102.1001"`) |
| Source repository | `https://forge.emrul.dev/Aster/iroh-gossip.git` |

The fork's own `[patch.crates-io]` (row 17) pins iroh `9ba1447d` (1.4.2002) and noq `379f73677c` (1.4.1001), the stack Aster links it against, unchanged from 0.101.1003. Upstream v0.102.0 contains every upstream fix rows 29 and 58 backported and the fixes rows 55 and 56 made locally, so the stack is now row 17, row 83 (re-cut), rows 84 and 85 (new) and the `.gitignore` line; see the 2026-11-09 History entry.

Bootstrap pin line for source-tree builds (consumers install a versioned Aster SDK release instead and must not copy this):

```toml
iroh-gossip = { git = "https://forge.emrul.dev/Aster/iroh-gossip.git", rev = "39a2ed2c8b5e99d2fb49f887e9589804cd441c82" }
```

## Patches

Row numbers are stable patch ids shared across Aster's iroh-family forks (iroh, iroh-blobs, iroh-docs, iroh-gossip, iroh-util, noq), so a commit message that says "row 29" resolves here.

| Id | Topic branch | What | Why / what in Aster depends on it | Commits (topic / upgrade) | Status | Conflict or porting notes |
|---|---|---|---|---|---|---|
| 17 | none recorded (original commit `14a76d5`) | `[patch.crates-io]` block pointing the iroh/noq crates at the Aster forks. File: `Cargo.toml`. | Keeps gossip builds on the Aster fork stack and avoids mixing crates.io iroh-base/noq with forked iroh. | `14a76d5` (original); rc.1 copy `511613f`; re-pinned since; on `upgrade/iroh-gossip-v0.102.0` as `ceb31220` | carried | Low conflict risk. Pin explicit fork revs or tags, and re-check comments and direct dependency versions after each upstream release. In the 1.0 port it hit a trailing-EOF conflict in `Cargo.toml`. |
| 83 | `perf/idle-topic-timers-v0.102.0` (tip `24bc7032`, based on `v0.102.0`; the 0.101 commit was `4e280690`) | Idle topics set no periodic timers. Plumtree arms `EvictCache` only while either message cache holds entries, for the earliest expiry and no sooner than `cache_evict_interval` (it re-armed every second unconditionally). HyParView runs `DoMaintenance` every 5 s while the active view is short or a recovery (`RefillLimit::Recovery`) runs, and every 60 s (`STABLE_MAINTENANCE_INTERVAL`) otherwise; a view that turns short re-arms the short cadence, and a timer generation (`DoMaintenance(u64)`) keeps a superseded timer from running or re-arming. Join retry timers (#217) and shuffle timers are unchanged. Files: `src/proto/plumtree.rs`, `src/proto/hyparview.rs`, `src/proto/state.rs`, `src/proto/util.rs`. | portal-sync §V: the gossip actor was 35-47% of idle tokio samples, two timers per topic and about 200 topics per daemon. | `24bc7032` / `7d2e8dfc` (0.101: `4e280690`) | re-cut | Re-cut on upstream #196 (`f2b1f29c`: refill limits and backoff for a short view) and #217 (`5cf0350f`: join retry). New condition in the re-cut: a running recovery also keeps the 5 s cadence, because only a maintenance pass ends a recovery. Losing an active peer still refills at once (`remove_active`), so the 60 s backstop only matters for a view that stays short of refills. Tests: `plumtree::test::eviction_timer_runs_only_while_caches_hold_entries`, `hyparview::tests::maintenance_backs_off_while_the_view_is_healthy`, `hyparview::tests::maintenance_runs_soon_to_end_a_recovery`. It changes the event order of every simulation, so seeded paper and sim outcomes move (row 84). |
| 84 | `test/paper-heal-window-v0.102.0` (tip `4751c93a`, based on `v0.102.0`) | `tests/paper.rs` `hyparview_heals_without_waiting_for_a_shuffle`: the healing window is the measured 20 max-latency steps plus one `neighbor_request_timeout` in max-latency steps (3 s / 100 ms = 30), both read from the `NetworkConfig` the swarm is built with, and the test asserts the window stays below `shuffle_interval` (60 s). File: `tests/paper.rs`. | The sim drops messages to a removed peer, so a neighbor request to a failed peer ends only by its timeout. A survivor left with one neighbor whose live passive peers refused it (full active views; it asks with low priority) and whose next requests went to failed peers stays in recovery until that timeout, then heals with no shuffle or maintenance pass, after the old 20-step (2 s) window. Over 300 seeds at 50 percent failure with the old window: upstream v0.102.0 fails 1/300 (seed 75: peer 190, active view `[135]` after losing 189, 176 and 68, 10 live passive peers at the failure, recovery with requests outstanding to 112, 109 and 144, whole at 28 steps); the row-83 tree fails 2/300 (seed 1: peer 94, active `[95]` after losing 69 and 116, 14 live passive peers at the failure, 18 refusals, requests outstanding to failed peers 112 and 134, whole at 34 steps; seed 59: peer 180, active `[162]`, whole at 22). Healing is otherwise unchanged by row 83 (50 percent: mean 4.48 vs 4.47 steps, p50 4 on both). With the new window both trees pass all 300 seeds (max 34 and 28). | `4751c93a` / `aa2e7e18` | carried | Replaces the previous port's seed skip (`19b68bcf`/`f8ea553e`, never published), whose explanation, a survivor "whose passive peers all failed", was wrong: every stranded survivor above had live passive peers. Applies to upstream v0.102.0 unchanged. |
| 85 | none recorded (upgrade-only) | `handle_dial_failed` in `src/net.rs`: `!…is_some_and(has writer)` becomes `…is_none_or(no writer)`. No behaviour change. | Upstream v0.102.0 fails `clippy::nonminimal_bool` on the Rust 1.96 toolchain Aster builds with, so the fork's `clippy -D warnings` gate needs it. | `69afcf4c` | carried | Drop once upstream's code passes clippy on Aster's toolchain. |

Bookkeeping and documentation commits (not needed at runtime):

| Commits | Files | Purpose | Status |
|---|---|---|---|
| `d7d1358` (original) / `0bbff98` (rc.1) / `93ab58e4` (0.101) / `a6a162c3` (0.102, `cherry-pick -x` of `93ab58e4`) | `.gitignore` | Ignores local `docs/aster/` internal review docs. | carried: the owner's checkout keeps untracked review notes there. |
| `fdecec8` (rc.1) | `Cargo.lock` | Resolves the Aster iroh/noq fork crates. Regenerated each wave. | carried in spirit (regenerated). |
| `161cec0` | `README.md` | Aster fork overview and contribution table. | Replaced by the `main` README preamble under the current model; if reused, refresh PR statuses. |

### Upstreamed or dropped

Absorbed by upstream v0.102.0 (`a16f016c`), dropped in the 2026-11-09 port (0.102.1001). The rows as they stood in 0.101.1003:

| Id | What | Upstream | Notes |
|---|---|---|---|
| 29 | Backport of upstream's connection and peer lifecycle fixes (#161, #178, #183, #189, #191). | `fcd81ae8` (#161), `f02f79e8` (#178), `b89fa981` (#183), `cd73d931` (#189), `dfeec141` (#191) | upstreamed: released in v0.102.0. Commits on 0.101: `b0bbb05`, `e372f13`, `26792b9`, `4099a1b`, `2ce6e82`. |
| 55 | The actor's per-peer send and the recv loop's `in_event_tx` send `try_send` + drop, with `send_queue_full_drops` / `in_event_full_drops` counters; per-peer queue cap 1024. | `dee7a261` (#185) | absorbed in substance: byte-capped per-peer send queues, drops counted by `msgs_dropped` and `peers_send_queue_full`; the actor no longer awaits a per-peer send, so the recv-loop drop is not re-added. Our counter names are gone; nothing in Aster reads them. Fixes upstream #47. Commits on 0.101: `f4f43823`, `7f8bbbca`, `a528b006`. |
| 56 | Time-boxes `conn.open_uni()` (5 s), with `open_uni_stalls`. | `dee7a261` (#185) | absorbed in effect: upstream opens a stream with a 15 s `OPEN_TIMEOUT` and closes the connection on expiry. The `open_uni_stalls` counter is gone; nothing in Aster reads it. Commit on 0.101: `6eb304c0`. |
| 58 | Upstream #195 (per-peer close flag) and #187's send side (oversized message dropped, stream kept). | `5592b477` (#195), `6bb9fe07` (#187) | upstreamed: released in v0.102.0, including #187's API-level refusal. Commits on 0.101: `7b30e0f7`, `602bbb71`. |

Also in v0.102.0 and relevant to consumers: #198 (`c5f44db7`) marks `Event`, `Command` and `Metrics` `#[non_exhaustive]` (iroh-docs 0.101.1034 and Aster gained catch-all arms), and #206 (`c156ecbf`) keeps a connection while streams are open.

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

Before touching branches, read the upstream changelog and range from the previous base (the rc.1 port reviewed `v0.99.0..v0.100.0`; the 0.102 port `v0.101.0..v0.102.0`). Row 83 changes simulation event order, so re-run `PAPER_SEEDS=300 cargo test --features test-utils --test paper` on the port; give each worktree its own `CARGO_TARGET_DIR`, as two worktrees of this crate sharing one target reuse a stale test binary.

```bash
git fetch --all --tags --prune
git push origin <upstream-tag>          # mirror the upstream tag (e.g. v0.102.0) to the forge
git switch -c upgrade/iroh-gossip-v<upstream version> <upstream-tag>
# row 17: re-point [patch.crates-io] at the new iroh/noq fork revs, regenerate Cargo.lock, commit
# rows 83, 84, 85 and the .gitignore line: cherry-pick -x from their topic branches (re-cut a row whose topic no longer applies)
# version bump, only if the version rule requires it (Aster's registry has already
# served <upstream version> or higher in the series): set the crate to the next option E version (`1000 * (upstream_patch + 1) + n`), then
git commit -am "chore: release <published version>"
git push origin upgrade/iroh-gossip-v<upstream version>
# do not tag yet: tag after the consumer trial (below)
```

Prefer `cherry-pick -x`. Published `upgrade/*` branches and tags are never rewritten; a failed attempt is superseded by new commits and a new version.

An Aster-only fix on the current base is a topic branch, cherry-picked `-x` onto the current upgrade branch, followed by a version-bump commit to the next option E version (`1000 * (upstream_patch + 1) + n`) (as 0.101.1 was cut for row 29). A re-pin without a code change (row 17 only) is a commit on the current upgrade branch with no version bump, tagged `aster-iroh-gossip-v<current version>-p<n>` after the consumer trial (as `aster-iroh-gossip-v0.101.1-p1` was).

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

### iroh-family upstream wave (2026-11-09), current

| Upstream base | Tag | Pinned commit | Branch |
|---|---|---|---|
| `v0.102.0` | `aster-iroh-gossip-v0.102.1001` | `39a2ed2c8b5e99d2fb49f887e9589804cd441c82` | `upgrade/iroh-gossip-v0.102.0` |

- **0.102.1001** (`39a2ed2c`): port onto upstream v0.102.0. Stack: `ceb31220` row 17 (patch block, iroh 1.4.2002 / noq 1.4.1001), `7d2e8dfc` row 83 (re-cut, from `perf/idle-topic-timers-v0.102.0` `24bc7032`), `aa2e7e18` row 84 (from `test/paper-heal-window-v0.102.0` `4751c93a`), `69afcf4c` row 85 (lint), `a6a162c3` `.gitignore` (from `93ab58e4`), `39a2ed2c` version. Rows 29, 55, 56 and 58 dropped as absorbed (see "Upstreamed or dropped"). Option E: upstream patch 0, n = 1. iroh-docs re-pins to it in `aster-iroh-docs-v0.101.1034`.
- Validation at the pinned commit, 3 runs: `cargo test --locked --all-features` lib 208 (3 ignored), e2e 27, paper 26, sim 4, doc 1, all passed each run; `--no-default-features --lib` 105; clippy `--all-features --all-targets` and `--no-default-features --all-targets` `-D warnings` clean; fmt clean on the files the patches touch. `PAPER_SEEDS=300` for the heal test passes on the port and on upstream v0.102.0 with row 84.
- A first port of this wave carried a seed skip in the heal test instead of row 84; it was reviewed, replaced, and never pushed.

### Idle timers (2026-11-08)

- **`aster-iroh-gossip-v0.101.1003`** (`4e280690`): row 83 (idle topics set no periodic timers). Lib and integration suites pass (one early full-suite run had a single lib failure that did not reproduce in 3 full runs, 5 lib runs, 6 parallel lib runs and 15 net-test loops). iroh-docs re-pins to it in `aster-iroh-docs-v0.101.1033`.

### i-stall-hardening wave (2026-11-03)

| Upstream base | Tag | Pinned commit | Branch |
|---|---|---|---|
| `v0.101.0` | `aster-iroh-gossip-v0.101.1002` | `602bbb719ece3c2314faf732639a29a582fdf9ed` | `upgrade/iroh-gossip-v0.101` |

- **2026-11-03, 0.101.1002** (`602bbb71`, tag `aster-iroh-gossip-v0.101.1002`): rows 55 and 56 (`f4f43823`, `6eb304c0`, topic `fix/i-stall-hardening`), then the version bump (`d2e14fb3`, option E, `n = 2`: the grandfathered 0.101.1 takes the `n = 1` slot, so 0.101.1001 is not used), then the de-async follow-up (`7f8bbbca`, row 55), the cap raise and slow-peer test rewrite (`a528b006`, row 55) and the upstream backport (`7b30e0f7` = upstream `5592b477` / #195; `602bbb71` = upstream `6bb9fe07` / #187's send side, row 58). The tag points at the last of these; nothing at `d2e14fb3` was tagged or published. The patch block is unchanged from 0.101.1-p7. Validation at the pinned commit: `cargo test --locked --all-features` lib 93, e2e 26 (also 20 consecutive runs with zero instrumentation, all green), paper 25, sim 4, doc 1; clippy `--all-features --all-targets -D warnings` and fmt clean. Aster consumer trial: fork_compat suites green at the pinned commit; Aster local suites green on the re-pinned tree (`aster_transport_core`, `aster`, build.sh, Python 1620).

### iroh 1.4.2002 and noq 1.4.1001 re-pin (2026-10-28)

| Upstream base | Tag | Pinned commit | Branch |
|---|---|---|---|
| `v0.101.0` | `aster-iroh-gossip-v0.101.1-p7` | `c3dea9a36f14a8f9a15e1de76aa2d9746d8ca660` | `upgrade/iroh-gossip-v0.101` |

- **`aster-iroh-gossip-v0.101.1-p7`** (`c3dea9a3`): re-pin only, to iroh `9ba1447d` (Aster iroh 1.4.2002) and noq `379f73677c` (Aster noq 1.4.1001). Only Cargo.toml's `[patch.crates-io]` and Cargo.lock change (iroh, iroh-base, iroh-dns, iroh-relay to 1.4.2002; noq, noq-proto, noq-udp to 1.4.1001); the crate is not republished. Validation: `cargo test --locked --all-features`, 144 passed, 0 failed; clippy `-D warnings` clean with `--all-features` and with `--no-default-features`. The earlier commit `3fb846f1` (iroh `34ad5b11`, noq 1.4.3) was superseded by `c3dea9a3` before tagging, because noq's version moved from 1.4.3 to 1.4.1001 under the fork version rule (option E); nothing at it was tagged or published.

### iroh 1.4.2001 re-pin (2026-10-25)

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
