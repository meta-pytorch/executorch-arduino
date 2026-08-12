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

The UNO Q also needs `Arduino_RouterBridge` for `Serial`; install it from Library Manager.
Without it the core stops the build with an explicit `#error`.

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

**Compile with `link_mode=static`.** The UNO Q defaults to Dynamic, which builds the
sketch as a Zephyr loadable extension; a library this size will not start that way and
prints nothing at all.

```bash
arduino-cli compile --fqbn arduino:zephyr:unoq:link_mode=static examples/AddModel
```

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

Measured for `arduino:zephyr:unoq:link_mode=static`, against 786,432 bytes of flash and
131,072 bytes of RAM:

| Example | Flash | RAM |
|---|---|---|
| HelloExecuTorch | 472,756 (60%) | 3,060 (2%) |
| AddModel | 507,688 (64%) | 11,252 (8%) |
| KeywordSpotting | 563,480 (71%) | 33,780 (25%) |

Registering every portable operator instead of the default set costs about 1.58 MB of
text, which is roughly twice this board's flash. The operator set is chosen at generation
time; `extras/PROVENANCE.txt` records which one produced this build.

## License

BSD-3-Clause, matching ExecuTorch. See [LICENSE](LICENSE), and
`extras/THIRD_PARTY_LICENSES/` for the vendored dependencies.
