# go-fde

Pure-Go **full-disk encryption** — a unified dispatcher that opens an encrypted
block device whatever its on-disk format, with no cgo and no root.

A single **`Device` contract** (`ReadAt`/`WriteAt`/`Size`/`Close`) hides *how*
the bytes are protected behind a uniform read/write interface. The
[`fde`](components/fde.md) dispatcher reads the first 36 bytes of a device,
recognises the format, and hands back the matching backend — so a caller can
unlock a Linux [LUKS](components/luks.md) container, an Apple
[APFS FileVault 2](components/apfs.md) volume, or a [plaintext](components/clear.md)
disk through exactly the same calls.

Each backend reads and writes real containers byte-for-byte: LUKS output is
interoperable with `cryptsetup`, and the APFS Apple-shape format matches
`diskutil apfs encryptVolume`. All four modules are standard-library only
(`CGO_ENABLED=0`) and layer cleanly on top of any read-write-closable block
device — a raw file, or a [go-diskimages](https://github.com/go-diskimages)
QCOW2 virtual disk.

## Components

| Module | Role | What it does |
|--------|------|--------------|
| [`fde`](components/fde.md) | dispatcher | One `Device` interface over all backends; `Detect`/`Auto` identify the format from the on-disk header (`LUKS\xba\xbe`, `NXSB`, or plaintext). |
| [`luks`](components/luks.md) | backend | Read/write LUKS1 and LUKS2 (big-endian on-disk header); PBKDF2 / Argon2i / Argon2id, AES-XTS and AES-CBC-ESSIV; `cryptsetup`-interoperable. |
| [`apfs`](components/apfs.md) | backend | Read/write APFS FileVault 2 (little-endian); AES-XTS volume keys, AES-KW key wrapping, PBKDF2 / Argon2id; Apple-shape parity with `diskutil`. |
| [`clear`](components/clear.md) | backend | Plaintext passthrough — forwards all I/O unencrypted so the dispatcher treats unencrypted devices uniformly. |

## Format detection

`fde.Detect(path)` / `fde.DetectFrom(rw)` read the first 36 bytes of the device
and decide:

- bytes 0–5 equal `"LUKS\xba\xbe"` → **LUKS**
- bytes 32–35 equal `"NXSB"` → **APFS**
- otherwise → **CLEAR**

Pass `fde.Auto` to any `Open` / `OpenFrom` / `Create` call to auto-detect, or
name the format explicitly with `fde.LUKS`, `fde.APFS`, or `fde.CLEAR`.
