# opensrv-mysql 0.7.0, patched: no reads from freed memory, no OK for a malformed command

This is opensrv-mysql **0.7.0** at upstream commit `53a741f395389807ca83f58af4f73291a60bcd98`
(the commit the published crate's `.cargo_vcs_info.json` names, one before the `v0.7.0` tag,
which adds only a changelog entry; `mysql/src/` and `mysql/examples/` are byte-identical to the
`.crate` on crates.io, sha256
`4148ab944991b0a33be74d2636a815268974578812a9e4cf7dc785325e858154`), with the changes below,
published as **`netget-opensrv-mysql`** so it can sit on crates.io alongside upstream. The
library name is still `opensrv_mysql`, and **the public API is upstream's, unchanged**, so code
written against upstream compiles as it is; a dependent writes
`opensrv-mysql = { package = "netget-opensrv-mysql", version = "0.7.0" }`. Only this crate
(`mysql/`) is published from the fork; the rest of the workspace is upstream's. Every changed
site carries a `NetGet patch (NETGET.md)` comment.

[NetGet](https://github.com/netget-net/netget)'s MySQL server is built on it.

## Why

| | upstream 0.7.0 | the peer sends |
|---|---|---|
| `packet_reader.rs:132-141` | `next_async` builds the packet it returns from a raw-pointer slice of `self.bytes`, then runs `self.bytes = rest.to_vec()` before returning it whenever the read held more than one packet. The returned packet points into freed memory. | any two commands in one read. Measured: two pipelined `COM_STMT_SEND_LONG_DATA` chunks each drew a stray OK (the freed bytes no longer parsed), and every reply after was one behind. `mysql_async`'s long data, and a statement close followed by the next command, both pipeline this way. |
| `lib.rs:730` | every packet `commands::parse` rejects is answered OK | a `COM_STMT_EXECUTE` too short to name its statement: the client is told its write succeeded. A short `COM_STMT_SEND_LONG_DATA` or `COM_STMT_CLOSE`, which have no reply, get one, and the session is out of step. |
| `commands.rs` | `COM_STMT_RESET` is not parsed; it reaches the OK above and the long data stays | a reset between long data and an execute: the execute uses the data the client discarded instead of the value it sent. |

**Upstream has fixed the first on `main`** — databendlabs/opensrv#67, "fix: Use After Free in
PacketReader", merged 19 September 2024 — but has published no release since 0.7.0 (February
2024), so every opensrv-mysql on crates.io still has it. The other two are not fixed upstream.

The sync `PacketReader::next` (`:44-61`) has the same raw-pointer slice but is sound: it
neither replaces nor resizes the buffer before returning, and the next call, which does, needs
the `&mut self` the returned packet borrows. It is unchanged (and unused).

## The changes

- `mysql/src/packet_reader.rs`: `next_async` keeps the unparsed bytes as
  `bytes[start..start + remaining]` and steps `start` past a returned packet instead of
  replacing the buffer; the next call drains it. The `AsyncRead` impl (which hands buffered
  bytes to TLS) hands over exactly those bytes, not the zeroed tail of the buffer after them.
- `mysql/src/lib.rs`, the parse-failure arm: an empty packet or a `COM_STMT_EXECUTE` is
  answered ERR **1835 `ER_MALFORMED_PACKET`**, `Malformed communication packet.` (mysqld's
  text); a `COM_STMT_SEND_LONG_DATA` or `COM_STMT_CLOSE` gets no reply, like a well-formed one.
  Any other command the crate does not implement keeps upstream's OK, which is what its comment
  says the OK was for.
- `mysql/src/commands.rs` / `mysql/src/lib.rs`: `COM_STMT_RESET` is parsed; it discards the
  statement's long data and is answered OK.
- `mysql/Cargo.toml`: the package is renamed `netget-opensrv-mysql`, with
  `[lib] name = "opensrv_mysql"` and `repository` pointing at this fork.
- `mysql/LICENSE`: a copy of the workspace's Apache-2.0 licence, so the published crate
  carries it (upstream's `.crate` ships none).

## What it does not change

- **`ParamParser`'s iterator still panics** on five malformed parameter blocks (`params.rs:81`,
  `:86`, `:90-91`, `:103`, `:127`). Its fields are `pub(crate)`, so NetGet never iterates it:
  it copies each execute from beneath the crate and decodes it with a parser of its own.
- **Long data is still appended without limit** (`lib.rs:663-678`). NetGet withholds chunks
  past 64 MiB a statement / 128 MiB a connection beneath the crate, before it sees them, and
  answers the next execute 1105.

Both would need API upstream does not have; keeping the API upstream's keeps this a drop-in
replacement.

## Tests

In NetGet, `tests/vendored_opensrv_patch_test.rs` fails if Cargo.lock stops resolving
`netget-opensrv-mysql` at this version from crates.io or upstream `opensrv-mysql` enters the
graph, and runs the bare crate over an in-memory pipe with every shape in the table above.
`tests/server/mysql/` covers the server built on it.

## Upgrading

Branch from the commit that published the new upstream release, re-apply every change above
(`grep -rn 'NetGet patch' mysql/src` on this branch lists every site), rename the package
again, update the version here and in NetGet's test, and run NetGet's test and MySQL server
suite. If upstream releases the reader fix, answers malformed commands and parses
`COM_STMT_RESET`, NetGet can go back to upstream opensrv-mysql and this fork can be retired.
