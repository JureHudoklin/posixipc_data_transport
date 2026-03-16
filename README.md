# PosixIPC Data Transport

A high-performance Python/C++ library for exchanging numpy arrays between processes using POSIX shared memory (`posix_ipc` and `mmap`).

Designed for low-latency data pipelines where copying through queues or sockets is too slow.

## Features

- **Zero-Copy (ish):** Uses shared memory mapped files.
- **Fast:** Much faster than `multiprocessing.Queue` or `sockets` for large arrays.
- **Type Safe:** Automatically handles `dtype`, `shape`, and dimensions (up to 5D).
- **Synchronization:** Uses semaphores to prevent read/write tearing.
- **Robust:** Automatically cleans up stale shared memory segments on writer startup.
- **Simple API:** User-defined named channels via `set_array` / `get_array`.
- **C++ compatible:** Header-only C++ library shares the same memory layout.

## Installation

### From Source
```bash
git clone https://github.com/JureHudoklin/posixipc_data_transport.git
cd posixipc_data_transport
pip install .
```

### From GitHub (Directly)
```bash
pip install git+https://github.com/JureHudoklin/posixipc_data_transport.git
```

## Usage

### Writer Process (Producer)
The writer creates the shared memory segment. **Note:** Only one writer should exist for a given base name.

```python
import numpy as np
import time
from ipc_transport import PosixIPCWriter

writer = PosixIPCWriter("camera_01")

while True:
    img = np.random.randint(0, 255, (480, 640, 3), dtype=np.uint8)
    writer.set_array("image", img)

    depth = np.random.rand(480, 640).astype(np.float32)
    writer.set_array("depth", depth)

    meta = np.array([1, 2, 3, 4], dtype=np.int32)
    writer.set_array("metadata", meta)

    time.sleep(0.033)
```

### Reader Process (Consumer)
The reader connects to existing shared memory.

```python
import time
from ipc_transport import PosixIPCReader

reader = PosixIPCReader("camera_01")

# Optional: block until the channel is available
reader.wait_for_array("image")

while True:
    img = reader.get_array("image")
    if img is not None:
        print(f"Received image: {img.shape}")

    # Optionally retrieve the write timestamp
    depth, ts = reader.get_array("depth", return_timestamp=True)

    time.sleep(0.01)
```

## API Reference

### `PosixIPCWriter(base_name: str)`
- `set_array(name: str, array: np.ndarray, timestamp: float | None = None)` — write an array to the named channel; timestamp defaults to `time.time()`
- `cleanup()` — release all shared memory and semaphores

### `PosixIPCReader(base_name: str)`
- `get_array(name: str, return_timestamp: bool = False) -> np.ndarray | None | tuple` — read latest data from named channel
- `get_shape(name: str) -> tuple | None`
- `get_dtype(name: str) -> np.dtype | None`
- `get_ndim(name: str) -> int | None`
- `wait_for_array(name: str, timeout: float = 5.0) -> bool` — wait until the channel appears
- `close()` — close memory maps

## C++ Support

A header-only C++ library is included that shares the same memory layout.

### Include
Copy `include/posixipc_data_transport.hpp` to your project.

### C++ Writer Example
```cpp
#include "posixipc_data_transport.hpp"
using namespace posixipc_data_transport;

PosixIPCWriter writer("camera_01");

int width = 640, height = 480, ch = 3;
std::vector<uint8_t> image(width * height * ch);

// Write to named channel — timestamp is set automatically
writer.set_array("image", image.data(), image.size(),
                 {(uint32_t)height, (uint32_t)width, (uint32_t)ch},
                 DataType::UINT8);
```

### C++ Reader Example
```cpp
#include "posixipc_data_transport.hpp"
using namespace posixipc_data_transport;

PosixIPCReader reader("camera_01");

std::vector<uint8_t> buffer(640 * 480 * 3);
double timestamp;
if (reader.read("image", buffer.data(), buffer.size(), &timestamp)) {
    // Process data...
}
```

## Requirements
- Python >= 3.9
- `posix_ipc`
- `numpy`
