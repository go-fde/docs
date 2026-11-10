# go-fde

Pure-Go **full-disk encryption** — a unified dispatcher that opens an encrypted
block device whatever its on-disk format, with no cgo and no root.

A single **`Device` contract** (`ReadAt`/`WriteAt`/`Size`/`Close`) hides *how*
the bytes are protected behind a uniform read/write interface. The
[`fde`](components/fde.md) dispatcher reads the first sector of a device,
recognises the format, and hands back the matching backend — so a caller can
unlock a Linux [LUKS](components/luks.md) container, an Apple
[APFS FileVault 2](components/apfs.md) volume, a Windows
[BitLocker](components/bitlocker.md) volume (read-only), a
[VeraCrypt or TrueCrypt](components/veracrypt.md) volume (read-only, opened by
name), or a [plaintext](components/clear.md) disk through exactly the same
calls.

Each backend reads and writes real containers byte-for-byte: LUKS output is
interoperable with `cryptsetup`, and the APFS Apple-shape format matches
`diskutil apfs encryptVolume`. Every module is pure Go (`CGO_ENABLED=0`) and
layers cleanly on top of any read-write-closable block
device — a raw file, or a [go-diskimages](https://github.com/go-diskimages)
QCOW2 virtual disk.

## Components

| Module | Role | What it does |
|--------|------|--------------|
| [`fde`](components/fde.md) | dispatcher | One `Device` interface over all backends; `Detect`/`Auto` identify the format from the on-disk header (`LUKS\xba\xbe`, `NXSB`, or plaintext). |
| [`luks`](components/luks.md) | backend | Read/write LUKS1 and LUKS2 (big-endian on-disk header); PBKDF2 / Argon2i / Argon2id, AES-XTS and AES-CBC-ESSIV; `cryptsetup`-interoperable. |
| [`apfs`](components/apfs.md) | backend | Read/write APFS FileVault 2 (little-endian); AES-XTS volume keys, AES-KW key wrapping, PBKDF2 / Argon2id; Apple-shape parity with `diskutil`. |
| [`bitlocker`](components/bitlocker.md) | backend | Read-only BitLocker (Windows 7+, To Go): password, recovery password, `.BEK` startup key, clear key; AES-CBC ± Elephant and AES-XTS; equal to cryptsetup's `bitlkOpen` on Windows-made volumes. |
| [`veracrypt`](components/veracrypt.md) | backend | Read-only VeraCrypt and TrueCrypt: AES, Serpent, Twofish and their five cascades, XTS; PBKDF2 (SHA-512, SHA-256, BLAKE2s, RIPEMD-160) and Argon2id; PIM, keyfiles, hidden volumes; 55 volumes from cryptsetup's corpus open and match. |
| [`clear`](components/clear.md) | backend | Plaintext passthrough — forwards all I/O unencrypted so the dispatcher treats unencrypted devices uniformly. |

## Format detection

`fde.Detect(path)` / `fde.DetectFrom(rw)` read the first sector of the device
and decide:

- bytes 0–5 equal `"LUKS\xba\xbe"` → **LUKS**
- bytes 32–35 equal `"NXSB"` → **APFS**
- bytes 3–10 equal `"-FVE-FS-"`, or `"MSWIN4.1"` with BitLocker's identifier
  at byte 424 (To Go) → **BITLOCKER**
- otherwise → **CLEAR**

Pass `fde.Auto` to any `Open` / `OpenFrom` call to auto-detect, or name the
format explicitly with `fde.LUKS`, `fde.APFS`, `fde.BITLOCKER`,
`fde.VERACRYPT`, or `fde.CLEAR`. A VeraCrypt header cannot be told apart from
random bytes, so `Auto` never chooses it: VeraCrypt is always named.
