# zed — Prim'X ZED! archives (read-only)

`github.com/go-fde/zed` (v0.1.0) reads Prim'X ZED! encrypted archives
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
  SHA-256. The password check (PBA) and the key and IV derivation (PBE) unwrap
  the archive key.
- AES-128, AES-192 and AES-256 in the two end-of-stream modes of the format,
  `STREAM` (mode 103) and `CTS` (mode 104, CBC-CS3), for file names and for
  the 512-byte chunks of file streams.
- zlib-compressed file contents, directories, and the three times of each
  entry.

It was checked against every test vector of the 2017 format document, and
against one container made by ZED! Free 2024.4. That container holds no file,
so the file-stream path is verified by the 2017 vectors only.

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
| MS-CFB container | The file system of the archive (an internal reader: v3 and v4 sectors, FAT, DIFAT, mini stream). |
| Metadata stream `\x055haaaaqaIekzeecnWj31zxh0Nc` | An MS-OLEPS property set with two `VT_BLOB` values, `_ctlfile` and `_catalog`, named by its dictionary. |
| `_ctlfile` | Obfuscated with a static AES key. TLV items: global properties (mode, key size, IV) and the access list. |
| `_catalog` | TLV items, one `fileprops` per file or directory; names are encrypted. |
| One stream per file | zlib data in 512-byte chunks, each AES-CBC with an IV derived from the chunk number. The stream name is the shuffled `file_id` in hex. |

## Where the 2017 document differs from ZED! Free 2024.4

The format is documented in Sylvain Beucler's 2017 note. One container made by
ZED! Free 2024.4 showed these differences:

| 2017 document | ZED! Free 2024.4 |
|---------------|------------------|
| `_ctlfile` is property index 3, `_catalog` index 4 | The dictionary names `_ctlfile` = 4, `_catalog` = 5, and a third blob `_seal` = 6. Names are resolved through the dictionary; 3 and 4 are only a fallback. |
| The version bytes after the delimiter are `0100` for the free version | `0200` (the TLV layout reads the same). |
| Created with `{00000188-1000-3CA8-8868-36F59DEFD14D}` | `{00000018-A040-3CA8-8868-36F59DEFD14D}` |
| Mode 103 or 104, key 16 or 32 bytes | Mode 104, 16 bytes. |
| `pba_iter` typically 200000, `pbe_iter` 100000 | Both 200000, with SHA-256. |
| Items listed in the TLV tables | More items exist in `_ctlfile` and in the catalog header; unknown items are skipped. |
| Optional streams "not discussed" | A seal stream and a storage with a PNG background picture; ignored. |

Two CTS test vectors of the document say "160 times" but decrypt to one
alphabet; the tests check what the bytes hold.

## Limits

The archive is untrusted input, so what it asks for is bounded and refused
before the work is done (`DefaultLimits()`):

- iterations: 10 000 000 for one user, 30 000 000 for all users together
  (ZED! Free writes 100 000 and 200 000);
- `pba_checksum` between 1 byte and one hash output, salts and the wrapped key
  at most 256 bytes;
- 64 users, 100 000 entries nested at most 256 deep, paths of at most 4096
  bytes, 64 MiB of metadata, 4 GiB per decompressed file;
- a file is read up to its declared size; a zlib bomb that inflates past it is
  refused with `ErrFormat`;
- names containing `/`, `\`, `:`, NUL, `.`, `..` or a Windows reserved name
  are refused, as are two entries with one path.

The CFB layer bounds every chain by the sectors the file can hold and refuses
cycles. The parsers are fuzzed in CI, with their corpora committed.

## Tests and status

Coverage is 100 % of statements, `-race` is clean, and CI runs amd64, arm64,
s390x, fuzzing and `govulncheck`. The CFB reader lives in the module as
`internal/cfb` for now; it was meant for a `go-office/cfb` repository, which
could not be created. Interoperability with real containers beyond the one
above (files, directories, several users, SHA-1, mode 103) is not verified.
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
