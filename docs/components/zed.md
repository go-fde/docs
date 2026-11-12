# zed — Prim'X ZED! archives (read-only)

`github.com/go-fde/zed` (v0.3.0) reads Prim'X ZED! encrypted archives
(`.zed`) in pure Go, for users who hold a password. It lists the entries of an
archive and reads their files. It is not a block-device backend: it is a file
archive reader, so it has no `Device` and the [`fde`](fde.md) dispatcher does
not open it.

```go
a, err := zed.OpenFile("archive.zed", password)
if err != nil { // zed.ErrPassword, ErrUnsupported, ErrLimit, ErrFormat
    log.Fatal(err)
}
defer a.Close()
for _, e := range a.Entries() { // name, path, directory, size, times
    if e.Dir {
        continue
    }
    data, err := a.ReadFile(&e) // or a.Open(&e) for an io.ReadCloser
    // ...
}
```

`Open(r io.ReaderAt, size int64, password string, opts ...Option)` takes any
reader, and `WithLimits` replaces the default limits.

## What it reads

- Password users: PKCS#12 key derivation (RFC 7292 appendix B) with SHA-1 or
  SHA-256 (`pkcs12_hashfunc` 21 and 22), or, for ZED! Free 2026.3 containers,
  `pkcs12_hashfunc` 43, which is PBKDF2-HMAC-SHA-512 with the password in
  UTF-8: the password check (PBA) is the first 8 bytes of the derivation, and
  the key and IV derivation (PBE) is one derivation whose first
  `encryption_strength` bytes are the key and the next 16 bytes the IV. PBA
  and PBE unwrap the archive key.
- AES-128, AES-192 and AES-256 in the two end-of-stream modes of the format,
  `STREAM` (mode 103) and `CTS` (mode 104, CBC-CS3), for file names and for
  the 512-byte chunks of file streams.
- zlib-compressed file contents, directories, and the three times of each
  entry.

It was checked against every test vector of the 2017 format document, and
against two real containers, one made by ZED! Free 2024.4 and one by ZED!
Free 2026.3. Both are extracted byte for byte: a folder holding a PNG
screen capture (11 508 bytes) comes out identical to the original from each,
through the whole file-stream path (stream name, 512-byte chunks, CTS, zlib).

## What it refuses or leaves out

- **RSA users** (certificate or token): not supported. An archive with a
  password user still opens with that password; an archive with none returns
  `ErrUnsupported`.
- **Writing**: not implemented.
- The optional streams (the background picture, the seal) and the on-the-fly
  folder variant: ignored.

## The format

| Layer | What it holds |
|-------|---------------|
| MS-CFB container | The file system of the archive (read by [go-filesystems/cfb](https://github.com/go-filesystems/cfb): v3 and v4 sectors, FAT, DIFAT, mini stream). |
| Metadata stream `\x055haaaaqaIekzeecnWj31zxh0Nc` | An MS-OLEPS property set with two `VT_BLOB` values, `_ctlfile` and `_catalog`, named by its dictionary. |
| `_ctlfile` | Obfuscated with a static AES key. TLV items: global properties (mode, key size, IV) and the access list. |
| `_catalog` | TLV items, one `fileprops` per file or directory; names are encrypted. |
| One stream per file | zlib data in 512-byte chunks, each AES-CBC with an IV derived from the chunk number and the file's IV (see the next section). The stream name is the shuffled `file_id` in hex. |

## Where the 2017 document differs from ZED! Free 2024.4 and 2026.3

The format is documented in Sylvain Beucler's 2017 note. One container made by
ZED! Free 2024.4 and one by 2026.3 showed these differences (a row is for both
unless it names a version):

| 2017 document | ZED! Free 2024.4 and 2026.3 |
|---------------|------------------|
| `pkcs12_hashfunc` is 21 (SHA-1) or 22 (SHA-256) | **2026.3 writes 43**: PBKDF2-HMAC-SHA-512 with the password in UTF-8, not PKCS#12 with UTF-16BE. PBA is its first 8 bytes; PBE is one derivation of `encryption_strength + 16` bytes, the key then the 16-byte IV. Found by trying hash families and password encodings against `pba_checksum`. The IV position for 24- and 32-byte keys (right after the key) is inferred, not seen on a real container. |
| Created with `{00000188-1000-3CA8-8868-36F59DEFD14D}` (Zed! Free 1.0.188) | 2024.4: `{00000018-A040-3CA8-8868-36F59DEFD14D}`; 2026.3: `{00000006-B030-3CA8-8868-36F59DEFD14D}` |
| A file's name and chunks use the global `files_iv` ("the same key and IV") | The IV of a file is `files_iv` XOR the file's raw `file_id`; the chunk IV is then `AES-ECB(key, LE16(chunk) XOR that IV)`. Applied to archives whose version bytes are `0200` (zed v0.2.0); `0100` archives keep the global IV, which rests on the 2017 vectors, not on a real `0100` file. Found on the real container: a wrong CBC IV garbles exactly the first block of each name. |
| `_ctlfile` is property index 3, `_catalog` index 4 | The dictionary names `_ctlfile` = 4, `_catalog` = 5, and a third blob `_seal` = 6. Names are resolved through the dictionary; 3 and 4 are only a fallback. |
| The version bytes after the delimiter are `0100` for the free version | `0200` (the TLV layout reads the same). |
| Mode 103 or 104, key 16 or 32 bytes | Mode 104, 16 bytes. |
| `pba_iter` typically 200000, `pbe_iter` 100000 | 2024.4: both 200000, with SHA-256. 2026.3: `pba_iter` 200000, `pbe_iter` 100000, with hash 43. |
| Items listed in the TLV tables | More items exist in `_ctlfile` and in the catalog header; unknown items are skipped. |
| Optional streams "not discussed" | A seal stream and a storage with a PNG background picture; ignored. |

Two CTS test vectors of the document say "160 times" but decrypt to one
alphabet; the tests check what the bytes hold.

## Limits

The archive is untrusted input, so what it asks for is bounded and refused
before the work is done (`DefaultLimits()`):

- key derivation work, counted in PKCS#12 SHA-256 rounds: 10 000 000 for one
  user and 40 000 000 for all users together (ZED! Free writes 100 000 and
  200 000 iterations). One PBKDF2-HMAC-SHA-512 iteration (hash 43) counts as 5
  rounds, measured at 225.3 ns against 46.4 ns for a PKCS#12 SHA-256 round on
  one core (ratio 4.86); each scheme is counted as it derives (PKCS#12: the
  check, then key and IV in two passes, three with SHA-1; PBKDF2: one pass).
  At the defaults a hostile header costs at most about 1.9 s of derivation on
  one core, whichever hash it names, and is refused before anything is
  derived. When `MaxTotalIterations` is left zero it is four times
  `MaxIterations`;
- `pba_checksum` between 1 byte and one hash output, salts and the wrapped key
  at most 256 bytes;
- 64 users, 100 000 entries nested at most 256 deep, paths of at most 4096
  bytes and 64 MiB of paths in all (`MaxTotalPathBytes`: paths are stored in
  full, so 100 000 entries at the per-path cap would hold 400 MB), 64 MiB of
  metadata, 4 GiB per decompressed file;
- a file is read up to its declared size; a zlib bomb that inflates past it is
  refused with `ErrFormat`;
- names containing `/`, `\`, `:` or NUL, and names that are exactly `.` or
  `..`, are refused (`note.txt` is fine), on every OS, as are names ending in
  a dot or a space and the Windows device names by the rules of Go's Windows
  `filepath.IsLocal`: CON, PRN, AUX, NUL, COM1-9 and LPT1-9 (superscript
  digits ¹ ² ³ included), CONIN$ and CONOUT$, with or without an extension;
  two entries with one path are refused too.

The CFB layer bounds every chain by the sectors the file can hold and refuses
cycles. The parsers are fuzzed in CI, with their corpora committed.

## Tests and status

Coverage is 100 % of statements, `-race` is clean, and CI runs amd64, arm64,
s390x, fuzzing and `govulncheck`. The CFB reader is its own module,
[go-filesystems/cfb](https://github.com/go-filesystems/cfb) (v0.1.0, moved
unchanged from zed's `internal/cfb`, which v0.2.0 still contains; the release
after v0.2.0 imports the module). Interoperability with real containers beyond
the two above (extracted byte for byte, a folder and a PNG each, with the
per-file IV of `0200` archives) is not verified for: several files, ciphertext
under 16 bytes, nested folders, several users, SHA-1, mode 103, 24- and 32-byte
keys (for hash 43 the IV placement after the key is inferred), other
`pkcs12_hashfunc` values than 21, 22 and 43, and any `0100` archive.
Benchmark timings against a reference extractor were not taken: none could be
run.

## Credit and references

The format knowledge and the test vectors come from Sylvain Beucler's
".zed archive file format" (2017), <https://www.beuc.net/zed/>, whose licence
permits copying with or without modification when the notice is kept; the
repository carries a verbatim copy with its notice. John the Ripper's
`zed2john.py` confirmed the version bytes. The other references are MS-CFB,
MS-OLEPS, NIST SP 800-38A addendum (CBC-CS3), RFC 7292 appendix B and
RFC 1950.
