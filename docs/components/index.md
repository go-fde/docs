# Components

`go-fde` is a set of dependency-free Go modules (standard library only,
`CGO_ENABLED=0`) layered around one small block-device contract. The
dispatcher selects a backend; each backend reads and writes its real on-disk
format.

| Module | Import path | Role | What it does |
|--------|-------------|------|--------------|
| [`fde`](fde.md) | `github.com/go-fde/fde` | dispatcher | One `Device` interface plus `Open`/`OpenFrom`/`Create`/`CreateFrom` over all backends. `Detect`/`Auto` identify the format from the on-disk header. |
| [`luks`](luks.md) | `github.com/go-fde/luks` | backend | LUKS1 and LUKS2 read/write; PBKDF2 / Argon2i / Argon2id; AES-XTS and AES-CBC-ESSIV; interoperable with `cryptsetup`. |
| [`apfs`](apfs.md) | `github.com/go-fde/apfs` | backend | APFS FileVault 2 read/write; AES-XTS, AES Key Wrap (RFC 3394), PBKDF2 / Argon2id; PRK/IRK recovery; Apple-shape parity with `diskutil`. |
| [`clear`](clear.md) | `github.com/go-fde/clear` | backend | Plaintext passthrough satisfying the same interface — no encryption, so the dispatcher can handle unencrypted devices uniformly. |

## Shared shape

Every backend exposes a `Device` whose `ReadAt`/`WriteAt`/`Size`/`Close`
methods the dispatcher returns uniformly. The offset semantics differ per
backend (LUKS hides its header and counts from payload sector 0; APFS and CLEAR
use absolute offsets from the device start) — see each component page.

All backends also provide an `OpenFrom` / `CreateFrom` variant that accepts any
value satisfying:

```go
interface {
    io.ReaderAt
    WriteAt([]byte, int64) (int, error)
    io.Closer
}
```

so a container can live inside a [go-diskimages](https://github.com/go-diskimages)
QCOW2 virtual disk (or any other RW backend), not just a plain file.
