# apfs — APFS FileVault 2

`github.com/go-fde/apfs` is pure-Go **read/write** support for APFS FileVault 2
full-disk encryption. The on-disk format is little-endian. It reads both Apple's
exact on-disk shape (what `diskutil apfs encryptVolume` produces) and a legacy
self-consistent shape produced by this package's own `Format` helpers.

## Key hierarchy

APFS FileVault 2 protects volumes with AES-XTS block-level encryption:

```text
passphrase
    │  PBKDF2-SHA256
    ▼
  KEK (Key Encryption Key)  ─── AES-KW (RFC 3394) ──▶  wrapped KEK (Container Key Bag)
  KEK                       ─── AES-KW (RFC 3394) ──▶  wrapped VEK (Container Key Bag)

  VEK (Volume Encryption Key)
      │  AES-XTS (per 512-byte sector)
      ▼
  plaintext blocks
```

Key bag entries live at the location named by the NX container superblock's
`nx_keylocker` field (offset 1296). Two on-disk shapes are supported, dispatched
on the `nx_flags & NX_CRYPTO_SW` bit through the same `apfs.Open` /
`apfs.OpenFrom` entry points:

- **Apple shape** (`NX_CRYPTO_SW` set): the keybag is encrypted at rest with
  AES-XTS-128 keyed on `containerUUID || containerUUID`; decrypted bytes follow
  Apple's `apfs_obj_phys` + `apfs_kb_locker` layout. The container keybag's
  tag=2 entry carries an ASN.1 VEKBLOB; the volume keybag's tag=3 entry carries
  an ASN.1 KEKBLOB.
- **Legacy self-consistent shape** (`NX_CRYPTO_SW` clear): the keybag is
  plaintext on disk; tag=3 encodes KDF parameters (PBKDF2-SHA256 or Argon2id)
  plus the wrapped KEK inline, tag=2 holds the raw AES-KW(KEK, VEK) ciphertext.
  Used by this package's `Format` / `FormatArgon2id` helpers and the test suite.

## Supported features

| Feature | Status |
|---------|--------|
| APFS NX container superblock parsing | ✅ (nx_keylocker at +1296, nx_flags) |
| Apple-shape container keybag parsing (NX_CRYPTO_SW path) | ✅ |
| Apple-shape volume keybag parsing | ✅ |
| Legacy self-consistent keybag parsing | ✅ (auto-dispatched on `nx_flags & 0x4`) |
| Key derivation: PBKDF2-SHA256 | ✅ |
| Key derivation: Argon2id | ✅ (package-defined locker layout) |
| AES Key Wrap / Unwrap (RFC 3394) | ✅ |
| Cipher: AES-128-XTS (`vek_size` = 32) | ✅ |
| Cipher: AES-256-XTS (`vek_size` = 64) | ✅ |
| At-rest keybag XTS encryption / decryption | ✅ |
| ASN.1 VEKBLOB / KEKBLOB build + parse | ✅ |
| Keybag block builder | ✅ (`PackKeybagBlock` — Apple's exact obj_phys + apfs_kb_locker shape) |
| Personal Recovery Key (PRK) | ✅ (passphrase-style locker with a well-known UUID) |
| Institutional Recovery Key (IRK) | ✅ (RSA-OAEP wrapping of the KEK) |
| T2 / Secure Enclave mediated keys | ❌ (hardware access required) |
| Encrypted metadata (APFS snapshots, etc.) | ❌ (payload encryption only) |

## Apple-shape encrypted-container support

Beyond the legacy format, this package exposes the byte-level primitives needed
to build and parse keybags in **Apple's exact on-disk shape** — the layout
`diskutil apfs encryptVolume` produces. The recipe was reverse-engineered
byte-for-byte against two independently-encrypted Apple reference DMGs and is
locked in by parity tests:

- `TestVEKBlob_HMACAgainstAppleReference` — re-computes the HMAC over Apple's
  reference VEKBLOB and asserts byte-equality with Apple's stored value.
- `TestKeybagChain_PassphraseUnlocksVEK` — builds a complete Apple-shape
  container + volume keybag pair, then walks the structure end-to-end with only
  the passphrase + the two UUIDs + the two paddrs, recovering the VEK
  byte-for-byte — the same chain Apple's `apfs.kext` walks on mount.

The recipe (validated against Apple's reference):

- **At-rest XTS**: AES-XTS-128, key = `uuid || uuid` (container UUID for the
  container keybag, volume UUID for the volume keybag), 512-byte XTS sectors,
  tweak = `paddr × 8 + sector_index`.
- **Keybag block layout**: 32-byte `obj_phys` (type `0x6b657973` "syek", sealed
  Fletcher-64 cksum) + 16-byte `apfs_kb_locker` (version=2, nkeys, nbytes
  including the 16-byte header AND the trailing 16-byte alignment pad) +
  16-byte-aligned entries.
- **VEKBLOB / KEKBLOB**: ASN.1 DER with HMAC-SHA256 keyed by
  `SHA-256(\x01\x16\x20\x17\x15\x05 || salt)` over the `[3]` inner envelope. The
  `[2]` field is an 8-byte OCTET STRING (Apple's opaque `info_t`), not a
  minimal-length INTEGER. The inner `[3]` contains the AES-KW(KEK, VEK) RFC-3394
  ciphertext (40 bytes for a 32-byte VEK).

## Usage

### Open an encrypted APFS volume image

```go
import "github.com/go-fde/apfs"

dev, err := apfs.Open("/dev/disk2s2", []byte("my passphrase"))
if err != nil {
    log.Fatal(err)
}
defer dev.Close()

// Read decrypted blocks. Offsets are absolute byte offsets in the device,
// aligned to 512-byte sector boundaries.
buf := make([]byte, 4096)
_, err = dev.ReadAt(buf, 0)

// Write encrypted blocks.
_, err = dev.WriteAt(buf, 0)
```

### Create a new APFS FDE container

`Format` and `FormatOn` write a minimal APFS NX superblock and key bag to an
existing file or block device, then return an open `*Device` ready for payload
I/O. Container parameters:

| Parameter | Value |
|-----------|-------|
| Block size | 4096 bytes |
| NX superblock | block 0 |
| Key bag | block 1 (referenced via `nx_keylocker` at NX SB offset 1296) |
| Payload start | block 2 (byte offset 8192) |
| VEK / KEK size | 32 bytes each (AES-256) |
| Key derivation | PBKDF2-SHA256, 1000 iterations, 16-byte salt |
| Key wrapping | AES Key Wrap (RFC 3394) |
| Cipher | AES-256-XTS, 512-byte sectors |

This is the **legacy self-consistent format** (`nx_flags & NX_CRYPTO_SW` clear,
plaintext keybag). To produce the **Apple-shape** format byte-for-byte, use
`go-filesystems/apfs.FormatContainerEncrypted` (or `…GPT`), which sits on top of
this package.

```go
// The file must exist before calling Format.
f, _ := os.Create("disk.apfs")
f.Close()

dev, err := apfs.Format("disk.apfs", []byte("passphrase"))
if err != nil { log.Fatal(err) }
defer dev.Close()

// WriteAt offset is absolute from the start of the container.
// Payload starts at block 2 = byte offset 2 × 4096 = 8192.
dev.WriteAt(myData, 2*4096)
```

`FormatOn` accepts the same `blockRW` interface as `OpenFrom`, enabling
container creation inside a QCOW2 virtual disk.

### Detect an APFS container

```go
ok, err := apfs.Detect("/path/to/disk.img")
if ok {
    fmt.Println("this is an APFS container")
}
```

### Layer on top of another block device (e.g. QCOW2)

`OpenFrom` accepts any value satisfying:

```go
interface {
    io.ReaderAt
    WriteAt([]byte, int64) (int, error)
    io.Closer
}
```

```go
import (
    apfsfde     "github.com/go-fde/apfs"
    image_qcow2 "github.com/go-diskimages/qcow2"
)

qdev, err := image_qcow2.OpenDevice("disk.qcow2")
if err != nil { log.Fatal(err) }

// apfsDev.Close() also closes qdev.
apfsDev, err := apfsfde.OpenFrom(qdev, []byte("passphrase"))
if err != nil {
    qdev.Close()
    log.Fatal(err)
}
defer apfsDev.Close()
```

## Block addressing

`ReadAt`/`WriteAt` use **absolute** byte offsets from the start of the
underlying device (block 0 of the APFS container). Reads and writes must be
aligned to 512-byte sector boundaries with lengths that are multiples of 512
bytes — constraints imposed by AES-XTS, which operates on fixed 512-byte
sectors. The XTS tweak for a sector is `byteOffset / 512`, matching Apple's
kernel `com.apple.filesystems.apfs` kext.

## Recovery keys

- **Personal Recovery Key (PRK)** — a regular passphrase locker tagged with the
  well-known UUID `EBC6C064-0000-11AA-AA11-00306543ECAC`. Add one with
  `AddRecoveryKey(rw, existing, prk)`; unlock by passing it as the passphrase.
- **Institutional Recovery Key (IRK)** — an RSA keypair held by an MDM
  administrator. The KEK is wrapped with RSA-OAEP-SHA256 under the public key
  (`AddInstitutionalKey`); unlock later with the private key via
  `OpenWithPrivateKey`.

## Argon2id format

Apple does not document an on-disk layout for Argon2id-protected lockers. This
package defines a self-consistent layout (kdfType, time/memory/parallelism
costs, salt, wrapped KEK) used by `FormatArgon2id` / `FormatArgon2idOn` and
`AddArgon2idPassphrase`.

## On-disk format references

- *Apple File System Reference* (developer.apple.com)
- *"Infiltrate the Vault: Security Analysis and Decryption of Lion Full Disk
  Encryption"* — Ligh, Walters et al. (PasswordsCon 2012)
- `apfs.h` and `IOKit/storage/APFS/APFSConstants.h` from the Darwin xnu tree
- RFC 3394 — AES Key Wrap Algorithm

!!! note
    `hdiutil create -encryption AES-256` is **DMG-envelope** (UDIF) encryption,
    not APFS FDE — a different layer entirely.
