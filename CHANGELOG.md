# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.1] — unreleased

### Fixed
- **Link mode guidance.** Every example fails to build in the board's default Dynamic
  link mode with `Sketch too big; text section exceeds available space`. The README
  previously described this as a silent runtime failure ("prints nothing at all"), so
  anyone searching for the error they actually saw found nothing. It is now a prominent
  section before first use, with the real error text, the IDE steps, and the two reasons
  it is easy to miss: the setting is per-sketch, and library examples live in a read-only
  folder where the IDE may not persist it.
- **Memory figures.** The published tables were measured on board core 0.55.2, which
  reported a 131,072-byte RAM ceiling. Core 0.90.0 reports 262,144, so those numbers did
  not correspond to anything a current user would see. Re-measured all three examples on
  0.90.0, with the hardware-verified output alongside each.

### Changed
- CI pins `arduino:zephyr@0.90.0` and `Arduino_RouterBridge@0.4.3` instead of resolving to
  latest, so a red build means a real incompatibility rather than an upstream release
  landing unannounced — which is exactly what happened when the core moved 0.55.2 → 0.90.0.
- CI lints with `--library-manager update` rather than `submit`. Submit mode asserts the
  name is *not* already in the index, so it could only pass before the library was first
  accepted.

### Added
- A nightly, non-gating Dynamic-mode compile that records sizes. The published library
  cannot be made to fit in Dynamic — trimming the vendored operators to only those
  registered, 15 sources instead of 172, moved it 852 bytes — so this exists to notice if
  that ever changes upstream instead of rediscovering it from a bug report.

## [0.1.0] — unreleased

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
