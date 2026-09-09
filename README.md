# zip-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

The ZIP container, over [flate-nv](../flate-nv), read the way a ZIP is
meant to be read: parse the end-of-central-directory record out of the
tail a caller supplies, parse the central directory out of the range
that points at, then ask for the byte range one entry needs. Local
headers, ZIP64, stored and deflated entries, a writer that hands back
bytes as it goes, and `is_safe_name` — the check an extractor has to
make and half of them do not.

It is for the program that has a `.zip` and wants one thing out of it,
or wants to build one: a package manager unpacking a bundle, a build
tool producing a release artifact, a document reader opening a `.docx`
or an `.odt` (both are ZIPs), a firmware updater reading one file out
of an image on flash.

```
novo pkg add zip-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use zipread
use zipdir

fn extract_one(archive: Bytes, name: Str) -> Result<Bytes, ZipError>
    let a = zipread.open(archive)!
    zipread.read_entry(a, archive, name)
```

For an archive already in memory. For one on disk, `open` unrolls into
four calls that each read a small range — a 64 KiB tail, the
directory, a 30-byte local header, the entry's data — so extracting
one file from a 2 GiB archive reads about as many bytes as the file
occupies.

## The layer, and why

`core` — no effects at all.

For this package that is not a constraint being worked around, it is
the format's own shape. A ZIP is random-access by construction: the
directory at the end says where everything is, so a caller that can
seek should read three small ranges and nothing else. The sans-IO
design and the format agree, which is why the read side is arithmetic
over ranges rather than a state machine:

```novo
pub struct ZipRange
    start: Int
    length: Int
```

`ZipRange` is the requested action `docs/publishing.md` § Design
describes — the core says "give me bytes 4,982,113 through
4,982,143", the host performs it, and the core never learns what a
file is.

The one function that meets a stream stays inside the budget by
**binding** its cost rather than spending one:

```novo
pub fn read_forwards<S: Read[e]>(src: S) -> Result<[(ZipEntry, Bytes)], ZipError> [e]
```

`S: Read[e]` binds the effect parameter of the standard library's
`Read` trait and the clause uses it, so the row means *whatever the
impl behind `S` supplies*.

## The load-bearing interface

```novo
pub fn find_end(tail: Bytes, tail_offset: Int) -> Result<ZipEndOfDirectory, ZipError>
pub fn parse_directory(data: Bytes, end: ZipEndOfDirectory) -> Result<ZipArchive, ZipError>
pub fn entry_named(a: ZipArchive, name: Str) -> ?ZipEntry
pub fn data_range(e: ZipEntry, local: ZipLocalHeader) -> ZipRange
pub fn decode_entry(e: ZipEntry, data: Bytes) -> Result<Bytes, ZipError>
```

Five calls, in that order, and every other read-side function is a
convenience over them. Two things in it are worth knowing before you
use them:

**The central directory is the authority, not the local header.** A
ZIP stores every entry's metadata twice, and the two routinely
disagree — a streaming writer that did not know a file's size writes
zeroes in the local header and the truth in the directory. `ZipEntry`
is what the directory said. `ZipLocalHeader` is a separate value for
the caller who wants to compare, and `check_local` is the comparison,
which accounts for the flag bit that makes the zeroes legal.

**`data_range` needs the local header, and that is the format's
fault.** The directory records where the local HEADER starts; the
header's length depends on its own name and extra field; so the first
byte of an entry's data cannot be computed from the directory alone.
The interface pays that once, here, rather than hiding it behind a
guess.

`read_forwards` exists for a ZIP arriving over a socket, and is
**documented as the weaker path**: forwards reading trusts the local
headers, and an entry under flag bit 3 has no size until its data
descriptor has been passed, so a streaming reader has to scan for the
next signature. A caller who can seek should not use it.

## The reference implementation, and what is specification

PKWARE's APPNOTE.TXT is the specification; Info-ZIP's `unzip` is the
reference implementation and the oracle for the archives that are
legal but strange. The distinction, because it decides what a test may
assert:

**Specification, and binding**

- The four signatures (`PK\x03\x04`, `PK\x01\x02`, `PK\x05\x06`,
  `PK\x06\x06`), the field layouts, and the little-endian encoding
  throughout.
- The end record's position: within the last 65,557 bytes, because
  its comment field is 16 bits. That bound is what makes a ZIP
  openable without reading it.
- Method 0 (stored) and method 8 (raw DEFLATE, no container). The
  CRC32 over the *uncompressed* bytes.
- ZIP64: the 0xFFFFFFFF and 0xFFFF sentinels, the locator, and the
  ZIP64 end record behind it.
- MS-DOS packed times: a 1980 epoch, two-second resolution, and 2107
  as the last representable year.

**Convention, which this package follows and a test may not treat as
correctness**

- That an entry is a directory because its name ends with `/`. There
  is no directory type in the format.
- Unix permissions in the high 16 bits of the external attributes
  word. Universal among Unix writers, in no specification.
- That the LAST entry with a given name wins. A ZIP may hold
  duplicates; every extractor picks the last, and nothing says so.
- Where a writer splits blocks and how well it compresses — that is
  flate-nv's, and flate-nv's README says the same thing about it.

**Deliberately not ported:** encryption. ZipCrypto is broken and
AES-encrypted ZIPs are a vendor extension; an encrypted entry is
reported as `EncryptedEntry` with its name and refused, because a
package that handed back ciphertext as if it were the file would be
worse than one that says no. Also not ported: multi-disk archives,
methods other than 0 and 8 (reported as `UnsupportedMethod` with the
raw code, so a caller meets bzip2 as the number 12 rather than as a
bare refusal), and anything to do with a filesystem.

## The name check is not optional

```novo
pub fn is_safe_name(name: Str) -> Bool
```

A ZIP entry name is attacker-controlled text. `../../etc/passwd` is a
legal entry name, and an extractor that joins it onto an output
directory writes outside that directory — "zip slip", found in
extractors in every language including the standard libraries of
several.

`is_safe_name` is true only when the name is relative, has no `..`
component, no drive letter, no leading `/`, and no NUL or backslash.
**A caller extracting to disk must call it.** It is a predicate rather
than a sanitiser because there is no single right repair — dropping
the entry, stripping components, and rewriting the name are all
reasonable and the choice is the caller's — but this package will not
pretend the question does not exist.

## Status

Every function is `todo()`. `novo test` runs the API suite, and every
assertion in it reaches `not implemented: zip-nv.<fn>` — which is the
expected result until the bodies land, and is what makes the suite a
description of the interface rather than of nothing.
`novo test --isolate` is the readable form: one verdict per test,
naming the function it stopped at.

| module | public types | functions | implemented |
| --- | --- | --- | --- |
| `zipentry` | 2 | 11 | no |
| `zipdir` | 2 | 11 | no |
| `zipread` | 2 | 13 | no |
| `zipwrite` | 1 | 10 | no |
| `ziperror` | 1 | 3 | no |
| **total** | **8** | **48** | **no** |
