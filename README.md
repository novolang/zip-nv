# zip-nv

ZIP is an archive format: many files in one, each compressed on its own, with an
index at the end. It is specified by PKWARE's
[APPNOTE.TXT](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT). This
package reads and writes ZIP archives in novo-lang, over
[flate-nv](https://novo-lang.org/packages/flate-nv), and touches no filesystem.

It is for a program that has a `.zip` and wants one thing out of it, or wants to
build one: a package manager unpacking a bundle, a build tool producing a
release artifact, a document reader opening a `.docx` or an `.odt` — both are
ZIPs — or a firmware updater reading one file out of an image on flash.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What ZIP is

A ZIP is read from the end. The last thing in the file is the
*end-of-central-directory record*, which says how many entries there are and
where the *central directory* begins. The central directory is one record per
entry, holding its name, its sizes, its compression method, its CRC32 and the
offset of its *local header*. The local header sits immediately before the
entry's own bytes.

Every entry's metadata is therefore stored twice, and the two routinely
disagree: a writer that did not know a file's size when it wrote the local
header puts zeroes there and the truth in the directory.

Two compression methods matter. Method 0 is *stored*, meaning the bytes are
unchanged. Method 8 is *deflated*, meaning a raw DEFLATE stream with no zlib or
gzip container around it. The integrity check is a CRC32 over the uncompressed
bytes.

**ZIP64** is the extension that lifts the format's 32-bit limits. A size, an
offset or a count at its all-ones value is a sentinel meaning "look in the ZIP64
records".

| Value | Size |
| --- | --- |
| Local header signature | `PK\x03\x04` |
| Central directory signature | `PK\x01\x02` |
| End-of-directory signature | `PK\x05\x06` |
| ZIP64 end-of-directory signature | `PK\x06\x06` |
| Furthest the end record can be from the end of the file | 65557 bytes |

All fields are little-endian. The end record can be that far from the end
because it carries a comment whose length is a 16-bit number, and that bound is
what makes a ZIP openable without reading it.

## Install

```
novo pkg add zip-nv
```

## Example

```novo
use zipread
use zipdir

fn extract_one(archive: Bytes, name: Str) -> Result<Bytes, ZipError>
    let a = zipread.open(archive)!
    zipread.read_entry(a, archive, name)
```

That is for an archive already in memory. For one on disk, `open` unrolls into
four calls that each read a small range — a 64 KiB tail, the directory, a
30-byte local header, the entry's data — so extracting one file from a 2 GiB
archive reads about as many bytes as the file occupies.

Build and test with:

```
novo pkg build
novo test
```

Today `novo test` fails on purpose: every assertion reaches
`not implemented: zip-nv.<fn>`. `novo test --isolate` is the readable form, with
one verdict per test naming the function it stopped at.

## What the package contains

| Module | Contents |
| --- | --- |
| `zipentry` | An entry as a value: the compression method, the constructors, the setters, the Unix mode, the directory test, the encryption test, and `is_safe_name`. |
| `zipdir` | The end record and the central directory: finding one in a tail, parsing the other, looking an entry up by index or by name, and emitting both. |
| `zipread` | Reading: the byte ranges to ask for, the local header and its check, decoding one entry, the two convenience forms, and the MS-DOS time conversions. |
| `zipwrite` | Writing: `add_stored`, `add_deflated`, `add_directory`, `add_raw` and `finish`, each handing back the bytes to append. |
| `ziperror` | `ZipError`, its message, the offset and entry it happened at, and `is_entry_local`. |

Eight public types and 48 public functions. No function in this package opens a
file, reads a clock or performs any input or output.

The read side is arithmetic over ranges rather than a state machine, because
that is the format's own shape: a ZIP is random-access by construction, so a
caller that can seek should read three small ranges and nothing else.

```novo
pub struct ZipRange
    start: Int
    length: Int
```

The package says "give me bytes 4982113 through 4982143", the caller performs
it, and the package never learns what a file is.

The one function that meets a stream charges its caller rather than declaring an
effect of its own:

```novo
pub fn read_forwards<S: Read[e]>(src: S) -> Result<[(ZipEntry, Bytes)], ZipError> [e]
```

A file charges `[io]`; an in-memory buffer charges nothing.

## How to choose an entry point

**The archive is in memory: `zipread.open` then `zipread.read_entry`.**

**The archive is on disk or behind a network range request: use the five calls
directly.**

```novo
pub fn find_end(tail: Bytes, tail_offset: Int) -> Result<ZipEndOfDirectory, ZipError>
pub fn parse_directory(data: Bytes, end: ZipEndOfDirectory) -> Result<ZipArchive, ZipError>
pub fn entry_named(a: ZipArchive, name: Str) -> ?ZipEntry
pub fn data_range(e: ZipEntry, local: ZipLocalHeader) -> ZipRange
pub fn decode_entry(e: ZipEntry, data: Bytes) -> Result<Bytes, ZipError>
```

In that order, with `zipread.tail_range`, `.directory_range` and
`.local_header_range` telling you what to read before each step. Every other
read-side function is a convenience over these.

**The archive arrives over a socket and cannot be seeked:
`zipread.read_forwards`.** It is the weaker path, and deliberately so. Forwards
reading trusts the local headers, and an entry whose flag bit 3 is set has no
size until its data descriptor has been passed, so a streaming reader has to
scan for the next signature. A caller that can seek should not use it.

**Writing: `zipwrite.writer`, then `add_stored`, `add_deflated`,
`add_directory` or `add_raw`, then `finish`.** Each call hands back the bytes to
append.

## The rules a user needs

1. **A caller extracting to disk must call `zipentry.is_safe_name`.** A ZIP
   entry name is attacker-controlled text. `../../etc/passwd` is a legal entry
   name, and an extractor that joins it onto an output directory writes outside
   that directory. That is the vulnerability called "zip slip", and it has been
   found in extractors in every language, including several standard libraries.

   ```novo
   pub fn is_safe_name(name: Str) -> Bool
   ```

   It is true only when the name is relative, has no `..` component, no drive
   letter, no leading `/`, and no NUL or backslash. It is a predicate rather
   than a sanitiser because there is no single right repair: dropping the entry,
   stripping components and rewriting the name are all reasonable, and the
   choice is the caller's.
2. **The central directory is the authority, not the local header.** `ZipEntry`
   is what the directory said. `ZipLocalHeader` is a separate value for a caller
   who wants to compare, and `zipread.check_local` is the comparison, which
   accounts for the flag bit that makes the local zeroes legal.
3. **`data_range` needs the local header.** The directory records where the
   local *header* starts, and the header's length depends on its own name and
   extra field, so the first byte of an entry's data cannot be computed from the
   directory alone. The interface pays that once, here, rather than hiding it
   behind a guess.
4. **An entry is a directory because its name ends with `/`.** There is no
   directory type in the format. `zipentry.is_directory` applies the
   convention.
5. **The last entry with a given name wins.** A ZIP may hold duplicates. Every
   extractor picks the last, and no specification says so.
6. **MS-DOS packed times have a 1980 epoch, two-second resolution, and 2107 as
   the last representable year.** `zipread.dos_time_of` and `.dos_time_parts`
   convert in both directions.
7. **An encrypted entry is refused, not returned as ciphertext.**
   `EncryptedEntry(name)` is the refusal, and `zipentry.is_encrypted` asks in
   advance.
8. **`Zip64Required(field)` means the archive is truncated or mis-written, not
   that it is unsupported.** This package reads ZIP64. The error means a field
   is at its sentinel value and no ZIP64 record supplies the real one.

The errors are `NoEndOfDirectory`, `BadDirectorySignature(offset, saw)`,
`BadLocalSignature(offset, saw)`, `DirectoryOutOfRange(offset, len)`,
`UnsupportedMethod(name, method)`, `EntryCrcMismatch(name, want, got)`,
`EntrySizeMismatch(name, want, got)`, `Zip64Required(field)`,
`BadZip64Locator(offset)`, `EncryptedEntry(name)`,
`ZipFlateFailed(name, cause)` and `ZipSourceFailed(cause)`.
`ZipFlateFailed` carries flate-nv's own error rather than flattening it, so the
offset inside the compressed stream survives.

## What is specification and what is convention

**PKWARE's APPNOTE.TXT, and binding on this package:**

- The four signatures, the field layouts, and the little-endian encoding
  throughout.
- The end record's position: within the last 65557 bytes.
- Method 0 and method 8, and the CRC32 over the *uncompressed* bytes.
- ZIP64: the all-ones sentinels, the locator, and the ZIP64 end record behind
  it.
- MS-DOS packed times, per rule 6.

**Convention, which this package follows and a test may not treat as
correctness:**

- The trailing `/` that makes an entry a directory (rule 4).
- Unix permissions in the high 16 bits of the external attributes word.
  Universal among Unix writers, and in no specification.
- That the last entry with a given name wins (rule 5).
- Where a writer splits blocks and how well it compresses. That is flate-nv's,
  and flate-nv's README says the same thing about it.

## What is not included

- **Encryption.** ZipCrypto is broken and AES-encrypted ZIPs are a vendor
  extension. An encrypted entry is reported as `EncryptedEntry` with its name
  and refused, because a package that handed back ciphertext as if it were the
  file would be worse than one that says no.
- **Multi-disk archives.**
- **Methods other than 0 and 8.** They are reported as `UnsupportedMethod` with
  the raw code, so a caller meets bzip2 as the number 12 rather than as a bare
  refusal.
- **Anything to do with a filesystem.** That is
  [archive-nv](https://novo-lang.org/packages/archive-nv).
- **Name sanitisation.** See rule 1.

## Related packages

- [flate-nv](https://novo-lang.org/packages/flate-nv) is the only dependency.
  Method 8 is a raw DEFLATE stream, and a ZIP entry's integrity check is exactly
  flate-nv's CRC32.
- [tar-nv](https://novo-lang.org/packages/tar-nv) is the other archive format,
  and differs in that it has no index and no per-entry compression.
- [archive-nv](https://novo-lang.org/packages/archive-nv) is the half that
  touches a disk.

## The reference implementation

PKWARE's APPNOTE.TXT is the specification. Info-ZIP's `unzip` is the reference
implementation and the oracle for the archives that are legal but strange.

## Implementation status

| Module | Public types | Functions | Implemented |
| --- | --- | --- | --- |
| `zipentry` | 2 | 11 | no |
| `zipdir` | 2 | 11 | no |
| `zipread` | 2 | 13 | no |
| `zipwrite` | 1 | 10 | no |
| `ziperror` | 1 | 3 | no |
| **Total** | **8** | **48** | **no** |

## Licence

Apache-2.0. See `LICENSE`.
