# dz-udp

Receives **Solana shreds from DoubleZero Edge multicast**, reassembles them (Reed-Solomon FEC + entry
decoding) and delivers the transactions **as their original wire bytes** to local consumers — with an optional
raw-shred forward to your validator / RPC node's TVU port.

```
DoubleZero multicast (doublezero1:7733)
   leader / root / retransmit groups ──► ONE socket, ONE accumulator
                                            ├─► unix socket   : [len u32][slot u64][tx bytes] frames
                                            ├─► --node        : raw shreds → 127.0.0.1:8001 (node TVU), first copy only
                                            └─► --listen      : Yellowstone gRPC Subscribe (transactions)
```

## Features
- Joins every `edge-solana-*` multicast group routed on the interface (or the ones given with `--group`) on **one**
  socket; the groups are pipeline stages (leader → root → retransmit), merged into **one accumulator** so a shred
  missing on one group is filled from another, without waiting for FEC or a timeout.
- Transactions are emitted as soon as their entry is complete, **as the original bytes** (legacy, v0 and v1
  transactions), framed `[len u32 LE][slot u64 LE][bytes]` on a unix socket.
- `--node`: forwards each shred (first copy of each `(slot, index, type)` only) to a UDP address — e.g. your node's
  TVU — so the node replays from the fastest source too. `--node-first` forwards before decoding.
- **Rejoin watchdog**: no packet for `--rejoin-after-ms` → leave + re-join every group on a re-resolved interface
  address (a tunnel flap otherwise leaves the feed silently dead).
- Waits for the tunnel at startup instead of exiting.
- Stale gate: batches from slots already left behind are dropped, not delivered.
- Status line every 10 s: decoded tx/s, FEC recoveries, per-group first-arrival and coverage, decode / write /
  packet→socket latency histograms.
- Busy-poll (`SO_BUSY_POLL`) receive on a pinned thread (`--cpu`).

## Requirements
- Linux, Rust (stable).
- A **DoubleZero Edge subscription** for Solana shreds and the DoubleZero client connected, e.g.
  `doublezero connect multicast --subscribe-feed solana-shreds-full` — see the
  [DoubleZero docs](https://docs.doublezero.xyz/solana/edge/subscribe/).

## Build & run
```bash
cargo build --release
./target/release/dz-udp --cpu 30 --socket /tmp/dz-udp.sock \
    --node 127.0.0.1:8001 --node-first --listen 127.0.0.1:9004
```

| Flag | Default | |
|---|---|---|
| `--interface` | `doublezero1` | interface carrying the multicast |
| `--group ip:port` | (routing table) | groups to join, repeatable; all must share one port |
| `--default-port` | `7733` | port for discovered groups |
| `--rejoin-after-ms` | `2000` | silence before re-joining the groups |
| `--stuck-timeout-ms` | `50` | how long an incomplete batch waits before being decoded from what it has |
| `--cpu` | — | pin the receive+decode thread |
| `--node` / `--node-first` | — | raw shred forward (see above) |
| `--busy-poll-us` | `200` | `SO_BUSY_POLL` on the receive socket (0 = off) |
| `--listen` | — | Yellowstone gRPC address; absent = no gRPC, no full decode |
| `--socket` | — | unix socket path for the raw transaction feed |

### Reading the unix socket
Each frame: `u32` LE payload length, `u64` LE slot, then the transaction bytes exactly as on the wire
(deserialize with any Solana transaction decoder that supports your versions).

## Related
- [shreds-udp](https://github.com/nad019/shreds-udp) — same pipeline for a **unicast UDP** shred provider.

## Credits & third-party licenses
This project is MIT-licensed (see `LICENSE`). It includes or depends on third-party software; their licenses are
respected and apply to their code:

| Component | Use | License |
|---|---|---|
| [`shredstream`](https://github.com/shredstream/shredstream-sdk-rust) 2.0.0 by ShredStream.com | **vendored and modified** in `vendor/shredstream` (changes listed in `vendor/shredstream/MODIFICATIONS.md`; original `LICENSE` kept) | MIT |
| [`yellowstone-grpc-proto`](https://github.com/rpcpool/yellowstone-grpc) (Triton One) | gRPC types | Apache-2.0 |
| Solana / Anza crates (`solana-transaction`, `solana-message`, `solana-signature`, `solana-entry`, …) | transaction types | Apache-2.0 |
| `tokio`, `tonic`, `prost`, `clap`, `socket2`, `libc`, `anyhow` | runtime / gRPC / CLI / sockets | MIT or MIT/Apache-2.0 |

DoubleZero is a service of its respective owners; this tool only consumes a feed you are subscribed to and does
not redistribute it.
