# Observer Mode Post-Mortem — Why We Stopped Fighting Firedancer's Replay

**Branch preserved as:** `replay-bypass-attempts`
**Tag:** `v0.2.0-replay-bypass`
**Period:** 2026-04-10 through 2026-04-13 (~4 days)

## The premise that was wrong

"We can make Firedancer's replay tile work for a non-voting observer by
bypassing the tower consensus requirements."

## Why it was wrong

Firedancer's replay tile is architected **top-to-bottom** around voting
validators. The `fd_sched` block pool, `fd_banks` lineage tracking,
`fd_accdb` fork machinery, and replay-to-tower signalling all assume a
steady stream of **root-advance messages** that free resources as
consensus progresses. An unstaked observer produces zero root-advance
messages, so every resource that gets allocated during replay stays
allocated forever. The system always hits its first cap (block_pool
saturation, accdb fork depth, reasm pool exhaustion, take your pick)
and deadlocks.

Every "observer-mode bypass" we added unblocked the next crash but
left the underlying growth unchecked. The list:

1. `FD_REASM_OBSERVER_BYPASS_EQVOC` — unblocks the reasm `out` queue
   (tower never confirms slots). **Correct.**
2. Reasm anchor hijack — rewrites snapshot block_id to match real
   cluster blocks. **Correct.**
3. Reasm re-hijack with 3 triggers (orphan/dead-parent/stuck-subtree) —
   jump forward when the current chain stalls. **Correct but only a
   coping mechanism**; doesn't solve the underlying deadlock.
4. Multi-map snap_root removal — snap_root migrates frontier→ancestry
   after children link. **Correct.**
5. `FD_SCHED_OBSERVER_BYPASS_POH` (mblk + tick paths) — skip PoH hash
   verification against snapshot's stale baseline. **Correct but
   papering over the real issue**: the virtual anchor bank can't
   produce a valid PoH chain without replaying every intervening slot.
6. `verify_ticks_final` TOO_FEW_TICKS bypass — same issue, same paper.
7. `FD_REPLAY_OBSERVER_BYPASS_INVALID_TXN` — accept non-committable
   txns that fail against stale accdb state. **Correct but papering**:
   transactions fail because accdb is months behind reality.
8. `cache_size_gib` 2→4 — mainnet needs more than 2 GiB vinyl cache.
   **Correct core plumbing.** Keep.
9. `max_live_slots` 32→128 — more block_pool runway. **Kicks the can**
   — at cluster cadence (2.5 slots/sec) we still hit the wall in ~50
   seconds.
10. Auto-root-advance from `replay_block_finalize` — `fd_sched_root_notify`
    requires `ref_q_empty` which is never true mid-replay. Crashed.
11. Deferred auto-root-advance from `after_credit` with
    `fd_sched_is_drained` gate — first advance worked, second
    segfaulted at `0x10` because `bank_idx` got recycled after pool
    pruning and stored pending records became stale.
12. `rt_sigprocmask` in accdb seccomp — stock filter was missing it,
    every tile that touched io_uring crashed with SIGSYS. **Correct
    core plumbing.** Keep.
13. accdb `max_depth` 32→512 — xid lineage grew past 32 once replay
    actually processed blocks. **Papering**: in steady state we'd
    eventually hit 512 too.

## The hot loop that ended the debugging run

After all 13 bypasses the process did survive for minutes at a time,
but the terminal failure mode was the `after_credit` eviction loop:

```
WARNING src/discof/replay/fd_replay_tile.c:2286 [after_credit]:
  banks are full and partially executed frontier banks are being evicted
```

firing at **~1.75 million iterations per second** because:
- `consensus_root` never advanced (no tower votes)
- Eviction marked banks dead but prune couldn't drain them (refcnts held)
- `fd_banks_is_full` stayed true forever
- Every `after_credit` cycle re-logged the same warning

In 40 minutes: **3+ billion log lines**, **11 GB of log spam**, one full
CPU core burning on log writes alone, and zero forward progress. The
replay tile was technically "alive" — not crashed — but doing useful
work was impossible.

## The insight we missed on day 1

The Parivaan MEV engine already has a **three-path detection model**:

| Path | Latency | Depends on replay? |
|---|---|---|
| Heuristic CP math (prefilter) | 1-5 µs | **No** — direct from shred tile |
| BPF shadow execution (CLMM/DLMM/Phoenix) | 70-300 µs | No — reads accdb at a pinned xid |
| Replay ground-truth TXN_EXECUTED | 2-10 ms | **Yes** — only this one |

Paths 1 and 2 consume shreds directly in the MEV tile's `after_frag`
callback. Path 3's only contribution is an after-the-fact correction
that runs ~1000× slower than path 1. For MEV arb detection at µs
latency, **path 3 is immaterial**: the hot path has already parsed the
same TX from shreds and updated `pool_store` before replay even sees
the slot start.

We spent 4 days trying to make path 3 work in observer mode. It was
never worth the effort.

## What we should have done

**Day 1 answer:** disable the replay tile entirely in observer mode.

Concretely:
1. Skip creation of `replay`, `execrp:*`, `execle:*`, `tower` tiles when
   `observer_mode=true` in config
2. Keep `shred`, `reasm`, `repair`, `gossip`, `mev`, `accdb`, `sign`,
   `snap*`, `sock`, `net`, `store`, `gui` tiles
3. Pin accdb xid at the snapshot slot — never update it
4. MEV tile reads pool vault balances from accdb at the pinned xid
5. MEV tile updates `pool_store` from shred-parsed TXs (already does this
   via `pool_feed`)
6. For CPI coverage, wire Layer 3 full-TX BPF simulation in the MEV tile
   for unknown-program TXs (the Phase 2 BPF shadow infrastructure
   already exists — just needs to be invoked for the uncovered case)

Result: **no replay**, therefore **no block_pool**, no PoH verification,
no tick verification, no root advancement, no tower, no accdb lineage
growth, no eviction loop, no "banks are full" warnings, no failure class
we were fighting.

## What we kept from the 4 days

- **Reasm eqvoc bypass** — still needed so the reasm→shred-tile path
  doesn't require consensus, even without replay
- **accdb `cache_size_gib` 2→4** — needed for accdb to hold mainnet
- **accdb seccomp `rt_sigprocmask`** — needed for accdb tile to run
- **Core understanding of Firedancer tile boundaries** — expensive but
  earned

## What we archived (branch `replay-bypass-attempts`)

19 commits of observer-mode replay bypass code. Reference implementation
for anyone who wants to revisit the problem. Not on the critical path
anymore.

## The lesson

**If the framework assumes a property (here: tower-driven resource
release), either honor the assumption or stop using the framework's
component that depends on it.** Halfway doesn't work. We tried halfway
13 times, at increasing cost, before admitting the whole premise was
wrong.

Corollary: **when the fix list grows past 3-4 items for a single
feature, stop and re-examine the premise.** We should have had this
conversation on day 2.
