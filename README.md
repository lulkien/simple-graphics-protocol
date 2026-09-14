# simple-graphics-protocol

The wire contract between the
[**simple-graphics-controller**](https://github.com/lulkien/simple-graphics-controller)
daemon (`@sgc`) and its clients: one Unix-socket protocol with an abstract-socket
address (`@sgc`), `rmp-serde` framing, and file descriptors passed over
`SCM_RIGHTS`.

It defines the messages both sides speak, and nothing else — no daemon, no client
session. Apps use a client library:

| crate | for |
| --- | --- |
| [libsgc-rs](https://github.com/lulkien/libsgc-rs) | Rust |
| [libsgc-c](https://github.com/lulkien/libsgc-c) | C and C++ (`<libsgc.h>`, `<sgc.hpp>`) |

## What is on the wire

- **`Resource`** — what the daemon can hand out: `Fbdev`, `Drm { card }`, and
  `Input(InputResource)` where `InputResource` is `Mouse(u8)`, `Keyboard(u8)` or
  `Touch(u8)` (the index is the device of that class).
- **Client → server**: `Acquire { resource }`, `Release { resource }`, `Ack`.
- **Server → client**: `Advertise { available_resources }` (sent on connect and
  again whenever the list changes — the whole list, not a delta),
  `Grant { resource }` with one fd, `Deny { reason }`, `Revoke { resource }`.

The session rules a client has to know — the display comes first, a device that
is unplugged is suspended rather than revoked, and a `Grant` for a resource you
already hold means the device came back and the fd replaces yours — are written
up in [docs/PROTOCOL.md](docs/PROTOCOL.md), which is the reader-facing version of
this crate's types.

## Use it

```console
cargo add simple-graphics-protocol
```

It is a plain library: `ClientRequest`/`ServerMessage` (de)serialisation plus the
fd-passing frame helpers, with the socket address and the framing constants the
libraries and the daemon agree on.

## License

Unlicense — public domain, see [LICENSE](LICENSE).
