# ExecuTorch Arduino Library

[![CI](https://github.com/meta-pytorch/executorch-arduino/actions/workflows/ci.yml/badge.svg)](https://github.com/meta-pytorch/executorch-arduino/actions/workflows/ci.yml)

Run PyTorch models on Arduino microcontrollers using [ExecuTorch](https://github.com/pytorch/executorch).

## Overview

This repository contains the ExecuTorch runtime packaged as an Arduino library.
It enables running PyTorch models exported with ExecuTorch on resource-constrained
microcontrollers via the Arduino IDE or `arduino-cli`.

```
PyTorch Model ──► torch.export ──► .pte file ──► model.h (C array)
                                                      │
                                          Arduino Sketch (.ino)
                                          #include <ExecuTorchArduino.h>
                                          #include "model.h"
                                                      │
                                          arduino-cli compile ──► Upload ──► Runs on board
```

## Supported Boards

| Board | MCU | Status |
|-------|-----|--------|
| Arduino Uno Q | STM32U585 (Cortex-M33) | ✅ Tested |
| Arduino Nano 33 BLE | nRF52840 (Cortex-M4F) | Planned |
| Arduino Giga R1 WiFi | STM32H747 (Cortex-M7) | Planned |
| Arduino Portenta H7 | STM32H747 (Cortex-M7) | Planned |

## Quick Start

### Installation

Install via Arduino Library Manager (coming soon) or manually:

```bash
git clone https://github.com/meta-pytorch/executorch-arduino.git
cp -r executorch-arduino ~/Arduino/libraries/ExecuTorchArduino
```

### Usage

```cpp
#include <ExecuTorchArduino.h>
#include "model.h"

using executorch::runtime::Program;
using executorch::runtime::Result;
using executorch::extension::BufferDataLoader;

void setup() {
  Serial.begin(115200);
  executorch::runtime::runtime_init();

  auto loader = BufferDataLoader(model_pte, sizeof(model_pte));
  Result<Program> program = Program::load(&loader);
  // ... set inputs, execute, read outputs
}

void loop() {
  delay(2000);
}
```

### Exporting a Model

Export a PyTorch model to `.pte` format and convert to a C header:

```bash
# Export model
python -c "
import torch
from executorch.exir import to_edge
from torch.export import export
class Add(torch.nn.Module):
    def forward(self, x): return x + 1.0
et = to_edge(export(Add().eval(), (torch.tensor([1.,2.,3.]),))).to_executorch()
with open('add.pte','wb') as f: f.write(bytes(et.buffer))"

# Convert to C header
python pte_to_header.py -p add.pte -o model.h
```

### Compile and Upload

```bash
arduino-cli compile --fqbn arduino:zephyr:unoq MySketch
arduino-cli upload  --fqbn arduino:zephyr:unoq -p /dev/cu.usbmodem* MySketch
arduino-cli monitor -p /dev/cu.usbmodem* --config baudrate=115200
```

## Examples

- **HelloExecuTorch** — Minimal example loading a model and printing output
- **AddModel** — Simple `x + 1.0` model demonstrating portable ops
- **KeywordSpotting** — DS-CNN keyword detection with CMSIS-NN acceleration

## Documentation

- [ExecuTorch Documentation](https://pytorch.org/executorch/)
- [Arduino CLI Reference](https://arduino.github.io/arduino-cli/)
- [Model Export Guide](https://pytorch.org/executorch/stable/export-overview.html)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to contribute to this project.

## License

This project is licensed under the BSD License - see the [LICENSE](LICENSE) file for details.
