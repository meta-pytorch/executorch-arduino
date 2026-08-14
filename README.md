# ExecuTorch for Arduino

[![CI](https://github.com/meta-pytorch/executorch-arduino/actions/workflows/ci.yml/badge.svg)](https://github.com/meta-pytorch/executorch-arduino/actions/workflows/ci.yml)

Run PyTorch models on Arduino using [ExecuTorch](https://github.com/pytorch/executorch).

```
PyTorch model ──► torch.export ──► .pte ──► model.h (C array)
                                              │
                                    Arduino sketch (.ino)
                                      #include <ExecuTorch.h>
                                      #include "model.h"
                                              │
                                    compile ──► upload ──► runs on the board
```

## Supported boards

| Board | MCU | Status |
|---|---|---|
| Arduino UNO Q | STM32U585 (Cortex-M33) | Supported — CI-compiled and hardware-verified |

Nothing else, and the reason is worth stating plainly: ExecuTorch requires C++17, and
every other official Arduino ARM core (`mbed_*`, `renesas_*`, `samd`) currently bundles
`arm-none-eabi-gcc 7.2.1` from 2017, which rejects valid C++17 that ExecuTorch relies on.
The UNO Q's `arduino:zephyr` core uses a Zephyr SDK toolchain (GCC 12.2.0) instead, which
is why it works. This is a constraint of the board packages, not of this library.

## Install

From the Arduino IDE: **Tools → Manage Libraries…**, search for `ExecuTorch`.

Or manually:

```bash
git clone https://github.com/meta-pytorch/executorch-arduino.git \
  ~/Arduino/libraries/ExecuTorch
```

Library Manager pulls in `Arduino_RouterBridge` automatically — the UNO Q core needs it for
`Serial` and stops the build with an explicit `#error` without it.

## Set the link mode first

**Tools → Board → Arduino UNO Q**, then **Tools → Link mode → Static**.

Without it, every example fails to build:

```
Sketch too big; text section exceeds available space in board
Compilation error: text section exceeds available space in board
```

The board defaults to **Dynamic**, and this is the first thing everyone hits. Two things
make it easy to miss:

- The setting is **per-sketch**. Opening another example puts it back to Dynamic.
- Library examples live in a read-only folder, so the IDE may not remember the setting at
  all. If it keeps reverting, **File → Save As** into your own sketchbook first.

From the command line, put it in the FQBN:

```bash
arduino-cli compile --fqbn arduino:zephyr:unoq:link_mode=static examples/AddModel
```

Dynamic builds the sketch as a Zephyr loadable extension through a relocatable link
(`-r`). `--gc-sections` is passed either way, but it can only work in a final link, where
the linker has an entry point to trace reachability from; under `-r` nothing can be proven
unreachable, so every operator in the library is kept. A loadable extension is also loaded
into RAM, so the retained code costs RAM as well as flash. For AddModel on core 0.90.0
that is 507,876 bytes versus 787,508.

This is not something the library can fix — shipping only the operators actually
registered (15 sources instead of 172) moves the Dynamic build by 852 bytes, and the rest
is the runtime, FlatBuffers and CMSIS-NN.

## Use

```cpp
#include <ExecuTorch.h>
#include "model.h"        // generated from your .pte

using executorch::extension::BufferDataLoader;
using executorch::runtime::Program;

void setup() {
  Serial.begin(115200);
  executorch::runtime::runtime_init();

  auto loader = BufferDataLoader(model_pte, sizeof(model_pte));
  auto program = Program::load(&loader);
  Serial.println(program.ok() ? "loaded" : "failed");
}
```

Start from `examples/HelloExecuTorch`, then `AddModel` for a full inference pass, then
`KeywordSpotting` for a quantized DS-CNN using CMSIS-NN kernels.

Each one needs Link mode set to Static, as above — it does not carry over between sketches.

ExecuTorch's own diagnostics reach your sketch through a weak hook, so the library never
has to depend on `Serial`. Implement it or lose every runtime error message:

```cpp
extern "C" void et_arduino_log(const char* msg) {
  Serial.print("ET| ");
  Serial.println(msg);
}
```

## Bringing your own model

```bash
python extras/tools/export_model.py --help      # PyTorch model  -> .pte
python extras/tools/pte_to_header.py --help     # .pte           -> model.h
```

The exporter and the runtime must come from the same ExecuTorch commit. A `.pte` built
against a different one loads, resolves every operator, and then fails inside
`Method::execute` with `InvalidProgram (0x23)` — nothing in that error says why. The
commit this library was generated from is in `executorch_pin.txt`.

## This repository is generated

Everything under `src/` and `examples/` is build output from
[`examples/arduino/build_arduino_library.sh`](https://github.com/pytorch/executorch/blob/main/examples/arduino/build_arduino_library.sh)
in the ExecuTorch tree. It lives in a separate repository because the Arduino Library
Manager requires `library.properties` at the repository root.

Do not send patches against `src/` — they will be overwritten by the next sync. Fix
things upstream in `pytorch/executorch` under `examples/arduino/`, then run the
**Sync from ExecuTorch** workflow here.

- `executorch_pin.txt` — the ExecuTorch commit this tree was generated from
- `extras/PROVENANCE.txt` — that commit, the CMSIS-NN revision, the operator set, and the
  kernel count
- `extras/THIRD_PARTY_LICENSES/` — licenses for the vendored CMSIS-NN, FlatBuffers, and
  flatcc sources
- `extras/tools/` — the generator and the model-conversion scripts

## Size

Measured on an Arduino UNO Q, board core 0.90.0, at
`arduino:zephyr:unoq:link_mode=static`, against 786,432 bytes of flash and 262,144 bytes
of RAM:

| Example | Flash | RAM | On hardware |
|---|---|---|---|
| HelloExecuTorch | 472,952 (60%) | 3,052 (1%) | `Model loaded OK!`, 1 method |
| AddModel | 507,876 (64%) | 12,268 (4%) | `[1,2,3] + 1 = [2.00, 3.00, 4.00]` |
| KeywordSpotting | 563,672 (71%) | 47,084 (17%) | detects `yes`, logit 8.95 |

Core 0.55.2 reported a 131,072-byte RAM ceiling and 0.90.0 reports 262,144, so figures
from before that change are not comparable.

`KeywordSpotting` hands `MemoryManager` a 40 KB arena, which is bounded on both sides:
28 KB fails `load_method` with `MemoryAllocationFailed` (0x21), and enlarging it far enough
overruns the stack and heap Zephyr reserves before the sketch gets any — a build that can
still run and print the right answer, which is what makes it dangerous rather than safe.
Anything above 40 KB is untested on core 0.90.0.

Registering every portable operator instead of the default set costs about 1.58 MB of
text, which is roughly twice this board's flash. The operator set is chosen at generation
time; `extras/PROVENANCE.txt` records which one produced this build.

## License

BSD-3-Clause, matching ExecuTorch. See [LICENSE](LICENSE), and
`extras/THIRD_PARTY_LICENSES/` for the vendored dependencies.
