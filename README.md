# Parivaan on Firedancer

**[Live architecture diagram →](https://sellasiyer.github.io/parivaan-architecture/)**

A Solana arbitrage engine compiled in-tree as a native [Firedancer](https://github.com/firedancer-io/firedancer) tile. Shredstream-only observer — no consensus, no voting, no replay. Built Feb–Jul 2026 and deployed bare-metal in the EWR and AMS regions.

It never turned a profit. The interesting part is why, and how the instrumentation proved it.

---

## What it does

The engine ingests ~14,000 shreds/sec straight off Jito's ShredStream, parses swap transactions out of the wire format before any replay round-trip, maintains live reserves for 3.14M pools across 15 DEX programs, runs a budget-capped SPFA cycle search for 2–6 hop arbitrage routes, and fires a signed transaction over QUIC to the next leaders.

Running as a tile inside the validator rather than talking to one over RPC is the whole design. It was the fourth architecture I built for this problem — the three before it (an RPC/Geyser bot in Rust, a modified Agave validator, and a 16-crate workspace forked from the Carbon framework) each measured slower, and the network hop was the floor every time.

## Serial hot path — 111 µs

Measured single-FEC trace, shred consumed to signed transaction on the wire:

| Stage | Cost |
|---|---|
| `after_frag` + `fd_store` eviction | 5 µs |
| Pool feed parse — wire format → CP swap deltas | 50 µs |
| SPFA cycle scan — adjacency + Tarjan SCC | 10 µs |
| Fire gate — profit floor, cache age, hop count, region | 5 µs |
| Router TX build + ed25519 sign — v0, up to 4 ALTs | 36 µs |
| QUIC stream send — TPU fanout | 5 µs |
| **Total** | **111 µs** |

Only the main thread is serial. The ingest tile, the DFS cycle worker, and the Phase 3b BPF shadow executor all overlap it. BPF runs 70–300 µs with a ~15% commit rate and corrects state *after* the fire rather than gating it — deliberately off the critical path.

## Where it was actually lost

Forensics across the landed signatures, versus competitors hitting the same pool in the same slots:

| | Parivaan | Same-pool winners |
|---|---|---|
| Detection to fire | 36–120 µs | comparable |
| Pool state freshness | shred-time, current | shred-time, current |
| Swap instruction | correct | correct |
| Priority fee paid | 25,000 lamports | 7,000–14,000 lamports |
| Slot arrival | +1 to +2 | +0 |
| QUIC QoS class | **unstaked** | **staked relayer** |

Every landed signature errored `Custom:3005` — DEX slippage. Not because the engine's state was stale; it reads the same shred stream as the leader, so reserves track the active slot continuously. The price had simply already moved.

Competitors landed one to two slots earlier while paying a quarter of the priority fee. Stake-weighted queue position at the leader's QUIC endpoint is worth more than fee bidding at unstaked rates.

A Solana slot is ~400 ms. The entire code path is 111 µs. Optimizing it further would have changed nothing — the constraint sat at the wire submission layer, above anything the engine controls, and closing it means buying staked submission rather than writing faster code.

## Engineering notes

**Minimal topology.** Firedancer's stock build runs 22 tiles. This runs 11 in the minimal observer configuration — `replay`, `repair`, `tower`, `txsend`, `execrp`×8, `ipecho`, `genesi`, `gossip` and `gossvf` all removed behind compile flags. The MEV tile absorbed what they were doing: `fd_store` eviction, cold-path bootstrap, and peer/leader discovery through an in-tile HTTPS worker that drains 4,500+ peers and a 432K-slot leader schedule into the existing consumers with no fire-path changes.

**Knowing when to stop.** Before the observer build there were four days spent trying to make Firedancer's replay tile work for a non-voting node. Thirteen separate bypasses, each unblocking the next crash, ending in an eviction loop that burned a full core writing 11 GB of identical log lines with zero forward progress. Replay's resource model assumes tower-driven root advancement; an unstaked observer never produces it, so nothing is ever freed. The fix was deleting the tile, not patching it. Written up in [`OBSERVER_MODE_LESSONS.md`](docs/OBSERVER_MODE_LESSONS.md) — the rule I took from it is that when the fix list for one feature passes three or four items, the premise is what's wrong.

**SPFA tuning.** Constraining the optimizer probe and capping the relax/queue budgets moved p50 from 3.9 ms to 7.7 µs and p99 from 68 ms to 33 µs — 506× and 2,060×. The engine self-monitors with rdtsc brackets and emits a timing histogram every 256 runs, which is how the regression was visible at all.

## Stack

C (in-tree with Firedancer), Rust (the three preceding architectures), Kafka-style lock-free SPSC rings between threads, QUIC, ed25519, Prometheus + Grafana for telemetry. Bare metal: AMD EPYC, 384 GiB DDR5, bonded 25 GbE with IRQ affinity pinned off the latency-critical cores.

---

Built with [Claude Code](https://claude.com/claude-code) as the primary development environment.

**Sella Iyer** · [LinkedIn](https://linkedin.com/in/sella-iyer) · [GitHub](https://github.com/sellasiyer)
