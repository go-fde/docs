# bitlocker — BitLocker (read-only)

`github.com/go-fde/bitlocker` reads BitLocker (BDE) volumes in pure Go. It
handles volumes made by Windows 7 and later, BitLocker To Go included, and
returns the decrypted volume as an `io.ReaderAt`. It does not write:
`WriteAt` returns `ErrReadOnly`.

## What it reads

| | |
|---|---|
| Volumes | Windows 7 and later, BitLocker To Go |
| Methods | AES-CBC 128/256 with and without the Elephant diffuser, AES-XTS 128/256 |
| Sectors | 512 bytes and 4 KiB |
| Protectors | password, recovery password, startup key (`.BEK` file), clear key |

**Refused** with `ErrUnsupported` (their metadata still reads with
`ReadInfo`):

- Windows Vista metadata;
- encrypt-on-write ("used space only") volumes;
- volumes in the middle of an encryption or decryption.

cryptsetup does not activate these volumes either. TPM and smart-card
protectors need hardware.

## Key chain

```text
password ──UTF-16LE──SHA-256──SHA-256──┐
recovery password ──8×(group/11)──SHA-256──┤  2^20 × SHA-256 with the
                                       └─ protector's salt (fixed count)
                                                │
.BEK startup key / clear key ───────────────────┤
                                                ▼
                                AES-CCM ─▶ volume master key (VMK)
                                                │ AES-CCM
                                                ├─▶ metadata validation: SHA-256 of the block
                                                ▼
                                full-volume encryption key (FVEK)
                                                │ per sector
                                                ▼
                     AES-CBC (IV = AES(offset)) [+ Elephant] or AES-XTS (tweak = offset / sector)
```

## Usage

```go
import "github.com/go-fde/bitlocker"

dev, err := bitlocker.Open("volume.img", []byte("password"))
if err != nil {
    log.Fatal(err)
}
defer dev.Close()

boot := make([]byte, 512)
_, err = dev.ReadAt(boot, 0) // the volume's NTFS / exFAT / FAT boot sector
```

The secret is interpreted the way cryptsetup interprets it:

- a password, for password protectors;
- a recovery password (`"123456-…"`, 48 digits), for recovery protectors,
  when it has that shape;
- the contents of a `.BEK` file, for startup-key protectors.

A clear-key protector opens with any secret, `nil` included.

- `OpenFrom(r, size, secret, limits)` reads from any `io.ReaderAt` that has
  a `Close` method.
- `ReadInfo(r, size)` reads the metadata without a secret: GUID, method,
  protectors and description, the same fields `cryptsetup bitlkDump` prints.
- Through the dispatcher, use `fde.Open(fde.Auto, …)` or
  `fde.OpenBitLocker(…)`.

## Offsets

`ReadAt` offsets are those of the decrypted volume, which has the size of
the device. The volume is the one `cryptsetup bitlkOpen` maps:

- the first sectors come from their encrypted copy;
- the three 64 KiB metadata areas, and the location of that copy, read as
  zeros.

## Interoperability

The tests open the 21 Windows-made volumes in cryptsetup's
`tests/bitlk-images.tar.xz`, with every secret listed in its `images.conf`.
For each volume they compare:

- the SHA-256 of the whole decrypted device with cryptsetup's;
- the file system UUID with blkid's;
- every protector GUID with the reference.

The archive is GPL-2.0+, so CI fetches it pinned by commit and SHA-256
instead of committing it.

## Untrusted images

- Every region the image declares must lie inside the device, be
  sector-aligned and not overlap another one.
- A metadata copy must pass its CRC-32. The metadata is then authenticated
  with the VMK: the AES-CCM validation record must hold the SHA-256 of the
  block.
- The key derivation is 2²⁰ rounds of SHA-256 whatever the image says.
  `Limits.MaxKeyDerivations` (16 by default) bounds how many protectors may
  run it; past that the open fails with `ErrLimit`.
- The parsers are fuzzed.

## References

- cryptsetup `lib/bitlk/bitlk.c`: metadata, key chain and the device-mapper
  layout this package reproduces.
- Linux `drivers/md/dm-crypt.c`: `eboiv` and the Elephant diffuser.
- [libbde format documentation](https://github.com/libyal/libbde/blob/main/documentation/BitLocker%20Drive%20Encryption%20(BDE)%20format.asciidoc)
  and [dislocker](https://github.com/Aorimn/dislocker).
