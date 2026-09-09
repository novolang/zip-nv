# Changelog

All notable changes to zip-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-09

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `zipdir` — the end-of-central-directory record and the directory it
  points at, which is what a ZIP is read backwards from. `find_end`
  takes the TAIL of a file plus the offset it came from, because the
  record is within the last 65,557 bytes and nowhere else — which is
  what lets a caller open a 2 GiB archive by reading 64 KiB.
  ZIP64 is resolved here, not exposed as an option.
- `zipread` — the load-bearing interface, and it is arithmetic over
  ranges rather than a state machine. `ZipRange` is the requested
  action the core hands back and the host performs; `data_range` needs
  the local header, because a ZIP's directory records where the
  header starts and the header's length depends on its own name. That
  asymmetry is the format's, and the interface pays it once rather
  than guessing. `read_forwards<S: Read[e]>` is the streaming path for
  a ZIP arriving over a socket, charged whatever the caller's stream
  costs (SPEC § 5.6), and documented as the weaker one.
- `zipentry` — one entry as the CENTRAL DIRECTORY describes it, which
  is the authority; `ZipLocalHeader` is a separate value for the
  caller who wants to compare the two copies, and `check_local` is
  the comparison. `is_safe_name` is the zip-slip check an extractor
  has to make, as a predicate rather than a sanitiser, because there
  is no single right repair.
- `zipwrite` — entries in, archive bytes out. The one stateful thing
  in this package, because a ZIP is written forwards and its directory
  needs offsets only known once everything before them is written. It
  accumulates the directory and nothing else, so writing a 4 GiB
  archive holds a few kilobytes of state.
- `ziperror` — twelve reasons, with `entry_of` for the ones about a
  file and `offset_of` for the ones about the container, and
  `is_entry_local` for a caller recovering what it can. `ZipFlateFailed`
  carries flate-nv's own `FlateError` through rather than flattening
  it, so `flateerror.offset_of` still finds the bad byte.

### Depends on

- `flate-nv` `^0.0.1`, for method 8 — a raw DEFLATE stream, no zlib or
  gzip wrapper, because the ZIP's own structures carry the CRC and the
  lengths — and for the CRC32 that is a ZIP entry's only integrity
  check. Both packages are `core`, so `dep-layer` holds.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented: zip-nv.<fn>`.
  Run it with `--isolate` for one verdict per test naming the function
  it stopped at.
- Encryption is reported and not supported. ZipCrypto is broken and
  AES-encrypted ZIPs are a vendor extension; an encrypted entry is
  `EncryptedEntry` with its name.
- Methods other than 0 and 8 are `UnsupportedMethod` with the raw
  code, so a caller meets bzip2 as the number 12 rather than as a bare
  refusal.
- Multi-disk archives are out of scope.
