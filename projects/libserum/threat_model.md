# Threat model: libserum

## What this project does and where untrusted input enters

libserum decodes Serum colorization files for pinball ROMs. A host — Visual Pinball, PinMAME front-ends,
libdmdutil, PPUC, or a ZeDMD-driven real machine — loads a colorization package for the ROM being played and
then passes every monochrome frame the ROM renders through the library to get a coloured one. It is a C++
library with a C API (`src/serum-decode.h`). It has no network code and starts no processes.

**The colorization package is the untrusted input.** Packages are made by community authors and downloaded by
players from forums and file-sharing sites, then dropped into `altcolor/<romname>/`. `Serum_Load(altcolorpath,
romname, flags)` reads from that directory:

- `*.cROMc` — the preferred runtime archive: a cereal binary archive of `SerumData`, LZ4-compressed
  (`src/SerumData.cpp`, `src/LZ4Stream.h`, `src/DecompressingIStream.h`, `src/sparse-vector.h`).
- `*.cRZ` — a zip archive holding a raw `*.cROM`, unpacked with miniz, and `*.cROM` itself — the original raw
  format, parsed field by field in `src/serum-decode.cpp`.
- `*.pup.csv` — scene definitions, parsed by `src/SceneGenerator.cpp` (`Serum_Scene_ParseCSV`).
- `scaling.txt` and `skip-cromc.txt` — small authoring sidecars.

Every count, dimension, offset, index, ID and length that comes out of these files is attacker-controlled:
frame and sprite counts, frame width and height, palette and rotation tables, dynamic-zone and shadow layer
indices, sprite detection areas, background IDs, scene frame counts and durations. These values size
allocations at load time and index buffers on every later call to `Serum_Colorize()`, `Serum_Rotate()` and the
`Serum_Scene_*` functions, so a bad value that survives loading becomes an out-of-bounds access during play.

The library also **writes**: when it loads raw `cROM`/`cRZ` data it may generate a `*.cROMc` next to it.
A package must not be able to make it write anywhere else.

The caller is trusted. The frame passed to `Serum_Colorize()` is assumed to have the dimensions the loaded
package declared, and the ROM name and directory path are chosen by the host application.

## Components that matter most / least

- **Most:** all three load paths (`cROMc`, `cRZ`, `cROM`) and the validation of what they produce;
  `src/sparse-vector.h` and the decompression streams; then the per-frame render path in
  `src/serum-decode.cpp` (identification, colorization, sprites, dynamic zones, shadows, rotations, the
  two-layer upscaler) when driven by a hostile but loadable package.
- **Also in scope:** `src/SceneGenerator.cpp` CSV parsing and scene frame generation, `scaling.txt` parsing,
  `cROMc` generation (path handling, what gets written), and the recursive runtime mutex around the
  process-global state (`Serum_Rotate()` versus `Serum_Colorize()` from different threads).
- **Less important:** logging, `src/test.cpp` (the `serum_test` loader, a developer tool), `tests/`.
- **Out of scope:** `third-party/include/` — cereal, lz4, miniz and `FrameUtil.h` are other projects, and a
  bug that lives entirely inside one of them should go there. It is in scope when libserum passes such a
  library a wrong size, trusts a value it returned without checking, or uses its output unvalidated.

## How to exercise it

- `/src/build/` has `libserum.so`, the static library, `serum_test` and `serum_unit_tests` (RelWithDebInfo).
  `/src/build-san/` is the same under AddressSanitizer and UBSan. `ctest --test-dir build` or run
  `serum_unit_tests` directly; it takes an optional substring to filter test names.
- `serum_test <altcolor-path> <romname>` loads a package through the public API — the simplest driver for the
  load paths. Point it at a directory that contains `<romname>/<romname>.cROMc` (or `.cRZ`, `.cROM`).
- **There are no colorization files in the repository or in the image**, and none can be downloaded during
  the scan. Inputs have to be constructed. `AGENTS.md` documents the formats in detail: "Load flow",
  "SparseVector storage and compression" and "cROMc persistence"; `src/SerumData.h` is the serialized model
  and `src/serum.h` the public structs and flags. Loading a hand-built raw `cROM` with cROMc generation on
  (`Serum_SetGenerateCRomC`) is a way to obtain a well-formed `cROMc` to mutate.
- `tests/unit_tests.cpp` builds frames in memory and includes `src/serum-decode.cpp` directly so the static
  internals are reachable; `tests/README.md` explains the layout and why there must be exactly one such
  translation unit. It is the template for a targeted harness.
- `AGENTS.md` sections "Safety invariants" and "How to validate after changes" state what must hold.

## How you rate severity

- **Critical:** memory corruption with plausible control (out-of-bounds write, use-after-free, double free)
  reachable from a colorization package, at load time or on a later frame. Any write outside the package's own
  directory caused by its contents.
- **High:** any other memory-safety violation reachable from a package — out-of-bounds read, uncontrolled
  overflow, integer overflow that leads to an undersized allocation.
- **Medium:** denial of service from a package: crash, uncaught exception, infinite loop, or memory use far
  out of proportion to the file (decompression bombs, a declared count that allocates gigabytes). On a real
  machine the host is often a small single-board computer, so treat this as a real impact rather than a
  nuisance. Data races with an observable wrong result.
- **Low:** wrong colours or wrong frame matching with no memory-safety aspect, issues that need the trusted
  caller to misuse the API, undefined behaviour with no demonstrated effect.

A sanitizer report with a reproducing file is enough; a working exploit is not required. Please say which
format and which field triggers the bug, and attach the smallest file that does.

## Anything to leave alone

- **`is_real_machine()` and every branch it gates.** It is true only on a Raspberry Pi inside a real machine,
  so in this image every one of those branches is the false one and cannot be executed. They are not dead,
  obsolete or inconsistent code. Do not report them as such and do not propose patches that simplify, merge
  or remove them. A memory-safety bug inside such a branch is still welcome, with a note that it could not be
  run here.
- Colorization files carry no signature or checksum of origin and the format has no authentication; do not
  report the absence. Whether a given package is a faithful colorization is not a security question.
- Runtime behaviour and `cROMc` contents must be identical across platforms. Do not propose fixes that make
  them depend on the operating system or architecture.
- The render path is performance-critical on small hardware: prefer validating once at load time over adding
  checks per pixel or per frame.
- Proposed patches should follow `.clang-format`, add a case to `tests/unit_tests.cpp` where possible, and
  update `AGENTS.md` if they change behaviour, a data format or an API signature — the repository requires that.
