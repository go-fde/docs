# veracrypt — VeraCrypt and TrueCrypt (read-only)

`github.com/go-fde/veracrypt` reads VeraCrypt and TrueCrypt volumes in pure
Go and returns the decrypted volume as an `io.ReaderAt`. It does not write:
`WriteAt` returns `ErrReadOnly`.

## How a volume opens

A VeraCrypt header looks like random bytes: nothing on disk names the key
derivation or the cipher. Opening a volume therefore works by trial, as in
cryptsetup's tcrypt support:

```text
password ‖ keyfile pool ──each allowed KDF──▶ 192-byte header key
                                                   │
             each allowed cipher chain (XTS, tweak 0, last link first)
                                                   ▼
               448-byte header: "VERA" / "TRUE", header CRC-32, keys CRC-32
                                                   │ match
                                                   ▼
                     master keys ──XTS, tweak = mk_offset/512 + sector──▶ data area
```

A KDF costs about half a second (500,000 PBKDF2 iterations, or Argon2id at
416 MiB). With a wrong password every pair is tried, so the refusal takes
seconds. Naming the KDF and the cipher in `Options` skips the others.

Because the header cannot be recognised, the [`fde`](fde.md) dispatcher's
`Auto` never chooses VeraCrypt: name it with `fde.VERACRYPT`.

## What it reads

| | |
|---|---|
| Ciphers (XTS) | every XTS chain VeraCrypt offers: AES, Camellia, Kuznyechik, Serpent, Twofish, and the cascades AES-Twofish, AES-Twofish-Serpent, Camellia-Kuznyechik, Camellia-Serpent, Kuznyechik-AES, Kuznyechik-Serpent-Camellia, Kuznyechik-Twofish, Serpent-AES, Serpent-Twofish-AES, Twofish-Serpent |
| KDFs | every KDF VeraCrypt and TrueCrypt use: PBKDF2 with SHA-512, SHA-256, BLAKE2s-256, RIPEMD-160, Whirlpool or Streebog-512; Argon2id; TrueCrypt's PBKDF2-SHA-1 with `Legacy`; with `System`, also VeraCrypt's boot rows (SHA-256 and BLAKE2s-256 at 200,000 iterations, RIPEMD-160 at 327,661, PIM multiplier 2048) |
| Options | PIM, keyfiles (up to 1 MiB each), passwords up to 128 bytes |
| Volumes | normal, hidden, backup headers; system encryption (`System`); VeraCrypt and TrueCrypt 5.0 and later |

**Not supported yet.** A volume that uses these fails like a wrong
password, because nothing on disk can tell the two apart:

- TrueCrypt's LRW and CBC modes, and headers older than version 3.

**Refused** with `ErrUnsupported`: a hidden operating system (a system
header that records a hidden size), a system header opened without
`System`, `System` combined with `Hidden` or `Backup`, and sector sizes
other than 512 bytes.

## Usage

```go
import "github.com/go-fde/veracrypt"

dev, err := veracrypt.Open("volume.hc", veracrypt.Options{
    Password: []byte("password"),
    // Keyfiles: [][]byte{kf1, kf2}, PIM: 485, Hidden: true, …
})
if err != nil {
    log.Fatal(err)
}
defer dev.Close()

boot := make([]byte, 512)
_, err = dev.ReadAt(boot, 0) // the filesystem's first sector
```

- `OpenFrom(r, size, opts)` reads from any `io.ReaderAt` that has a
  `Close` method, such as a partition or a virtual disk.
- Through the dispatcher, use `fde.Open(fde.VERACRYPT, path, password)` or
  `fde.OpenVeraCrypt(…)`. It passes only a password.
- `go-filesystems/detect/fdereg` registers it, so
  `detect.OpenContainer(detect.VeraCrypt, r, size, password)` opens the
  filesystem inside.

## Offsets

`ReadAt` offsets are relative to the data area: offset 0 is the volume's
file system. `Size()` is the volume size the header records. For system
encryption, see below.

## System encryption

`Options{System: true}` opens an encrypted boot drive or system partition,
read-only. Give it the whole drive, or an image of it: the header is at
byte 31744 of the drive, outside the encrypted area, so an image of the
partition alone has no header to read. The header's system flag must be set
(`ErrCorrupt` otherwise), as with cryptsetup's `--tcrypt-system`.

`ReadAt(…, 0)` is the start of the **encrypted area**, which is
`Header().MKOffset` on the drive, not the start of the disk. The area is
`Header().VolumeSize` long.

- Encrypted **partition**: offset 0 is the partition's first byte, so the
  file system is at 0.
- **Whole-drive** encryption: the area starts at sector 63 or later, in front
  of the first partition, so a partition starting at sector `S` is at offset
  `S*512 - MKOffset`. The MBR and the boot loader before the area are not
  encrypted and are not part of the device.

The XTS tweak of each sector is its sector number on the drive, as in
cryptsetup's `iv_offset`. A system data area that overlaps the header at
31744, lies outside the device or is off 512-byte sectors is `ErrCorrupt`.

## Interoperability

The tests open the images in cryptsetup's `tests/tcrypt-images.tar.xz`,
which VeraCrypt and TrueCrypt made, with the passwords, PIMs and keyfiles of
`tests/tcrypt-compat-test`. The FAT file system inside must have the UUID
that test checks.

- All 65 XTS volumes open and match (v0.6.0). They cover VeraCrypt and
  TrueCrypt, SHA-512, SHA-256, BLAKE2s, RIPEMD-160, Whirlpool, Streebog-512
  and Argon2id, PIM, keyfiles, a 72-byte password, hidden volumes, every
  cipher alone, and seven of the ten cascades.
- Three of the 65 are system-encryption images (`sys_vc_1_mbr_full`,
  `sys_vc_1_mbr_part` and `sys_vc_1_gpt_part`, up to 66 MiB), opened as
  `--tcrypt-system` does, with the same FAT UUID. The first two use the
  SHA-256 boot row, the GPT one SHA-512 at 500,000 iterations.
- The 28 LRW and CBC volumes (TrueCrypt before 5.0) must be refused, not
  misread.
- Reversing the cascade's decryption order fails all 22 cascade volumes,
  so the order is checked. The same images confirm Serpent's and
  Kuznyechik's byte orders.
- The corpus has no Camellia-Serpent, Kuznyechik-AES or Kuznyechik-Twofish
  volume. Those three are checked only by a round trip through the test
  writer, with cryptsetup's key offsets.

The archive is GPL-2.0+, so CI fetches it pinned by commit and SHA-256
instead of committing it.

## Untrusted images

- A large PIM multiplies PBKDF2's iterations and Argon2id's memory, and the
  attacker chooses it. `Limits` bounds the number of derivations, the PIM
  (2000 by default), the PBKDF2 iterations and the Argon2id memory (1 GiB by
  default). An open that would exceed them is refused with `ErrLimit` before
  any derivation.
- Once decrypted, the header must pass both CRC-32s. Its data area must lie
  inside the device and be sector-aligned; a system data area must not
  overlap the header at 31744.
- `FuzzParse` fuzzes those checks.

## Performance

AES volumes decrypt through
[go-encryptions/xts](https://github.com/go-encryptions/xts)
`DecryptSectors`, which is ahead of OpenSSL on one core. Twofish runs one
block at a time through `golang.org/x/crypto/twofish`. It has not been
measured against a reference implementation yet
([veracrypt#3](https://github.com/go-fde/veracrypt/issues/3)). Serpent
runs one block at a time through
[go-encryptions/serpent](https://github.com/go-encryptions/serpent),
bitsliced pure Go. It is 2.1× (encryption) and 2.5× (decryption) slower
than Botan on one core
([serpent#2](https://github.com/go-encryptions/serpent/issues/2)).
Camellia runs through
[go-encryptions/camellia](https://github.com/go-encryptions/camellia), pure
Go. On one core it is level with Botan's scalar build and 1.12× faster than
Botan's default build. Kuznyechik runs through
[go-encryptions/kuznyechik](https://github.com/go-encryptions/kuznyechik),
pure Go with lookup tables. On one core it is 1.14× (encryption) and 1.16×
(decryption) faster than Botan 3.14.1.

## References

- cryptsetup `lib/tcrypt/tcrypt.c`: the KDF and cipher tables, the key
  offsets, the header layout and the data offsets.
- The VeraCrypt documentation (veracrypt.io): *Volume Format
  Specification*, *Header Key Derivation*, *PIM*, *Encryption Algorithms*,
  *Cascades*.
