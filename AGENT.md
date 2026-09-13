# AGENT.md — simple-graphics-protocol

## Purpose
Standalone wire-protocol crate shared between the sgc daemon and its clients (msgpack framing, SCM_RIGHTS fd passing, resource/event types). Consumed by libsgc-rs and libsgc-c as a git dependency.

## Architecture & Scope
- **Library-only crate** — lib.rs exports protocol types, serialization, and framing
- **No async, no tokio** — pure synchronous framing over Unix sockets; protocol work is driven by the consumer
- **Wire format is build-agnostic** — same code path regardless of cfg(feature = ...) backends
- **Protocol-owned resource semantics** — Resource and InputResource enums carry all data; no external state

## Rust Best Practices (per rust-skills)
- [`own-borrow-over-clone`] — Prefer `&T` borrowing over `.clone()`; see `serialize()` which takes `&T`
- [`own-cow-conditional`] — Use `Cow<'a, T>` for conditional ownership where needed
- [`err-result-over-panic`] — Return `Result<T, E>` instead of panicking for recoverable errors (e.g. `deserialize`, `parse_frame_header`)
- [`err-from-impl`] — Implement `From<E>` for error conversions to enable `?` operator (see `ProtocolError`)
- [`err-lowercase-msg`] — Error messages start lowercase, no trailing punctuation (enforced by `thiserror`)
- [`mem-with-capacity`] — Use `Vec::with_capacity()` when payload size is known (see `serialize_framed`)
- [`perf-iter-over-index`] — Prefer iterators over manual indexing (see wire_dump example iteration)
- [`num-nonzero`] — Use `NonZero*` types to forbid zero and unlock niche optimization (not yet in protocol, but principle applies to card indexes)
- [`api-from-not-into`] — Implement `From<T>`, not `Into<U>` (protocol crate implements `FromStr` for Resource parsing via Serde)
- [`doc-all-public`] — Document all public items with `///` doc comments (all enum variants, function args, error variants)
- [`doc-errors-section`] — Include `# Errors` section documenting error conditions
- [`doc-example`] — Keep runnable examples in docs/tests; `wire_dump.rs` is the canonical wire dump example
- [`lint-missing-docs`] — Warn on missing documentation for public items
- [`lint-clippy-nursery-selected`] — Enable high-value `clippy::nursery` lints selectively
- [`api-sealed-trait`] — Use sealed traits to prevent external implementations while allowing use (future-proof: consider `Resource` sealing)
- [`anti-stringly-typed`] — Don't use strings where enums or newtypes would provide type safety (Resource already uses enum, good)

## Key Code Conventions
- All public types derive `Debug, Clone, Serialize, Deserialize, PartialEq, Eq, Hash`
- Error type `ProtocolError` uses `thiserror` with `#[from]` for clean `?` propagation
- Framing constants: `FRAME_HEADER_LEN = 4`, `MAX_FRAME_PAYLOAD = 1 MiB`
- MessagePack uses **named** encoding (no integer tags); unit variants wire as bare strings
- All string/field names on wire are arbitrary-length msgpack strings; code must be shape-tolerant
- `#[derive(Error)]` with `#[error("...")]` for all fallible operations

## Common Pitfalls to Avoid
- ❌ Do NOT use `.unwrap()` in production paths — use `?`, `expect()` only for invariants indicating bugs
- ❌ Do NOT compare floats with `==` — not applicable here but principle holds for any future numeric comparisons
- ❌ Do NOT accept `&Vec<T>` when `&[T]` works — protocol types use enums directly, not Vec-wrapped
- ❌ Do NOT hold locks across await points — this crate is synchronous; no async