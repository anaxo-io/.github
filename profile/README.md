# Anaxo

Software for crypto OTC desks, and the open-source Rust libraries it runs on.

**OTC desk automation.** Quoting, hedging and settlement for OTC desks that serve
their clients in Telegram. Watch the demo or request access at [anaxo.io](https://anaxo.io).
The desk itself is not open source.

## Libraries

| Library | What it does |
| --- | --- |
| [depthbook](https://github.com/anaxo-io/depthbook) · [crate](https://crates.io/crates/depthbook) | In-process L2 order book store: fixed-capacity sorted sides, scale-9 integer prices, sequence-gap detection. |
| [outcry](https://github.com/anaxo-io/outcry) · [crate](https://crates.io/crates/outcry) | Shared-memory queue: one writer, many readers, nobody waits. An implementation of the FastQueue design from CppCon 2024. |
| [ws-fanout](https://github.com/anaxo-io/ws-fanout) · [crate](https://crates.io/crates/ws-fanout) | WebSocket fan-out server: publish once, every subscriber gets it. Auth, snapshots, non-blocking backpressure. |

All three run in the desk: venue order books are kept in depthbook, written to an
outcry queue, and served to the traders' console by ws-fanout.

[anaxo.io](https://anaxo.io) · info@anaxo.io
