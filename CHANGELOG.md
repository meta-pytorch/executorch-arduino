# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - 2026-08-20

### Added

- `ETModel`, a wrapper over the runtime for sketches that just want to run a
  model. `begin()`, `setInput()`, `run()`, `output()`, `argmax()`. Loading a
  method by hand meant querying method metadata, counting planned buffers,
  allocating a span array, sizing each buffer, and assembling a
  `HierarchicalAllocator` and `MemoryManager` before any inference — around
  sixty lines, none of them a decision the sketch author makes. `AddModel`
  drops from 163 lines to 77 and `KeywordSpotting` from 204 to 91.
  `program()` and `method()` return the underlying objects for anything the
  wrapper does not cover, and `HelloExecuTorch` stays on the raw API as a
  worked reference.
- `model.error()` returns a readable reason rather than a status code, and the
  runtime's own diagnostics reach `Serial` through a weak `et_arduino_log`
  hook the examples implement. An undersized arena now reports the shortfall
  in bytes instead of surfacing as `0x21`.
- A nightly non-gating Dynamic-mode compile that logs sizes. Trimming vendored
  operators from 172 sources to 15 recovered only 852 bytes, so the library
  cannot fit; the job exists to detect any upstream change.

### Changed

- The platform layer is written against the Arduino API rather than Zephyr's.
  The previous backend called `k_uptime_ticks` and `k_malloc` for seven
  functions that all have Arduino equivalents; `arduino_pal.cpp` uses
  `micros()`, `malloc()` and `free()`. Only one backend ships, so link order
  cannot select between two definitions of the same `et_pal_*` symbols.
- Timestamps report real units. The Zephyr backend returned a tick ratio of
  `{1, 1}` with a comment stating it did not know the conversion; `micros()`
  is microseconds, so the ratio is `{1000, 1}`.
- CI pins the board core and `Arduino_RouterBridge@0.4.3` rather than tracking
  latest, so a failure indicates a genuine incompatibility.
- Linting runs as `--library-manager update` rather than `submit`. Submit mode
  checks that a name is absent from the index, so it only worked before the
  library was accepted.
- Memory figures were measured against core 0.55.2, which reports a
  131,072-byte RAM ceiling, while 0.90.0 reports double that. All three
  examples re-measured on 0.90.0.
- `KeywordSpotting` uses a 40 KB arena. 28 KB worked on core 0.55.2 and fails
  on 0.90.0 by 180 bytes.

### Fixed

- Link mode guidance. Examples fail to build under the board's default Dynamic
  setting with `Sketch too big; text section exceeds available space`. The
  README had described this as a silent failure, so a user searching the actual
  message found nothing. The guidance now appears before first use and explains
  why it is easy to miss: the setting is per-sketch, and library examples open
  from a read-only folder.
- Arduino cores that define `abs` as a macro in `Arduino.h` broke c10's
  templates and `<complex>`. `ExecuTorch.h` now undefines `abs`, `min`, `max`
  and `round` before the runtime headers.
- Cores older than GCC 9 fail with one line naming the toolchain instead of
  pages of template errors.

### Known limitations

- Only the Arduino UNO Q is supported. ExecuTorch requires C++17 and the other
  official Arm cores ship `arm-none-eabi-gcc 7.2.1` from 2017, which cannot
  compile it; the UNO Q's Zephyr core provides GCC 12.2.0. The platform layer
  no longer assumes Zephyr, so `architectures` opens up when another core
  updates its toolchain.
- Static link mode remains mandatory.
- No latency measurements yet.

## [0.1.0] - 2026-08-12

First release, targeting the Arduino UNO Q (STM32U585, Cortex-M33).

### Added
- ExecuTorch runtime, portable kernels, and the Cortex-M backend's CMSIS-NN int8 ops,
  vendored as an Arduino 1.5-format library
- `ExecuTorch.h` umbrella header
- Examples: `HelloExecuTorch` (runtime init and model load), `AddModel` (a full
  inference pass with portable ops), `KeywordSpotting` (quantized DS-CNN over real MFCC
  features, using CMSIS-NN kernels)
- `extras/tools/` — the library generator plus the model export and conversion scripts
- `executorch_pin.txt` and `extras/PROVENANCE.txt`, recording the ExecuTorch commit,
  CMSIS-NN revision, operator set, and kernel count behind a given build
- CI that compiles every example for `arduino:zephyr:unoq:link_mode=static` and checks
  the library against Arduino Library Manager submission rules
- A **Sync from ExecuTorch** workflow that regenerates the library from an upstream
  commit, re-exports the example models in the same run, and opens a pull request

### Changed
- The library now lives at the repository root (`library.properties`, `src/`,
  `examples/`) rather than under `arduino_lib/`. The Arduino Library Manager only reads
  the root `library.properties`, so the previous layout described a library with no
  sources and could not be published or installed.
- Header renamed from `ExecuTorchArduino.h` to `ExecuTorch.h`, matching upstream
- `library.properties` `url` now points at this repository instead of
  `pytorch/executorch`

### Removed
- The checked-in `arduino_lib/` copy, which duplicated the entire vendored tree
- Root-level copies of the generator and Python tooling, which now travel in
  `extras/tools/` where the Arduino library spec excludes them from compilation

### Known limitations
- The UNO Q is the only supported board. ExecuTorch requires C++17, and every other
  official Arduino ARM core (`mbed_*`, `renesas_*`, `samd`) currently bundles
  `arm-none-eabi-gcc 7.2.1` from 2017, which rejects valid C++17 that ExecuTorch uses.
  The UNO Q's `arduino:zephyr` core ships a Zephyr SDK toolchain (GCC 12.2.0).
- `link_mode=static` is required. The board's Dynamic default builds the sketch as a
  Zephyr loadable extension, which a library this size will not start as.
