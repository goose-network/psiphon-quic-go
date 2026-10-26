# psiphon-quic-go

Fork of [`github.com/Psiphon-Labs/quic-go`](https://github.com/Psiphon-Labs/quic-go)
at `v0.0.0-20250527153145-79fe45fb83b1`, with one change: the `http3`
package is ported from the qpack **v0.4.0** decoder API
(`NewDecoder(emitFunc)` + `DecodeFull([]byte)`) to the **v0.6.0** API
(`NewDecoder()` + `Decode([]byte) DecodeFunc` iterator).

The module path stays `github.com/Psiphon-Labs/quic-go`, so consumers swap
upstream for this fork with a single versioned `replace` directive:

```
replace github.com/Psiphon-Labs/quic-go => github.com/goose-network/psiphon-quic-go@v0.0.0-<commit>
```

## Why

Goose builds two QUIC stacks into one binary:

- **Psiphon** (via `psiphon-tunnel-core`), which imports
  `Psiphon-Labs/quic-go` — whose `http3` package is hardwired to qpack
  v0.4.0's decoder API;
- **sing-box** (`-tags with_quic`: hysteria, hysteria2, TUIC), whose
  `sagernet/quic-go` `http3` needs qpack v0.6.0.

Go's minimal-version selection picks a single qpack version for the whole
build. Pinning qpack at v0.4.0 (the only way upstream Psiphon quic-go
compiles) breaks the sing-box QUIC protocols; selecting v0.6.0 breaks
upstream Psiphon quic-go. Porting the five qpack decoder call sites here —
all simple read-a-full-header-block-then-decode patterns, now funneled
through one `decodeFull` helper — lets both stacks share qpack v0.6.0.

The upstream vendored build (`psiphon-tunnel-core`'s `vendor/`) is
unaffected: it carries its own qpack copy and never resolves this module
from the module graph.

## Scope

- `http3/headers.go` — new `decodeFull` helper (iterate `DecodeFunc` until
  `io.EOF`);
- `http3/conn.go`, `http3/client.go`, `http3/http_stream.go`,
  `http3/server.go` — `NewDecoder()` without the emit func, `decodeFull(...)`
  instead of `DecodeFull(...)`;
- `go.mod` — explicit `require github.com/quic-go/qpack v0.6.0` (upstream
  declares none; it relies on vendoring), plus a full require list so the
  module builds standalone.

Everything else is byte-identical to the upstream import. Tests are dropped:
they pin the old qpack API and are never executed in consumers' builds.

Upgrading to a newer upstream revision: re-import, re-apply the patch above
(`git log -- http3/` shows exactly what changed), and `go mod tidy`.
