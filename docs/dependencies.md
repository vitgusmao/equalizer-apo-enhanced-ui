# Dependencies & Tech Stack

Equalizer APO has a modular dependency surface: a small set of runtime libraries shipped with the installer, a larger set of build-time tools and SDKs, and one vendored library.

## Runtime Dependencies (shipped with the installer)

These DLLs are bundled in `Setup/lib32/`, `Setup/lib64/`, and `Setup/libARM64/` and are required on end-user machines.

### Qt 6.7.2 — Configuration Editor & Device Selector
- **Modules:** Core, Gui, Widgets, Svg (with iconengines, imageformats, platforms, styles plugins).
- **Used for:** the Editor and DeviceSelector GUIs (32-bit, 64-bit, and ARM64).
- **Shipped builds:** custom-modified for Windows 7 compatibility (`a57f063`).
- **Files:** `Setup/lib32/Qt6*.dll`, `Setup/lib64/Qt6*.dll`, `Setup/libARM64/Qt6*.dll`; plugin DLLs under `Setup/lib*/qt/`.
- **Cited in:** `Editor/Editor.pro`, `DeviceSelector/DeviceSelector.vcxproj`.

### FFTW 3.3.10 — Fast Fourier Transform
- **DLL:** `fftw3f.dll` (single-precision float build).
- **Used for:** FFT inside `GraphicEQFilter`, `ConvolutionFilter`, and `libHybridConv`; analysis in the Editor.
- **Note:** renamed from `libfftw3f-3.dll` in commit `53d885f`; older configurations still work.
- **Cited in:** `Common.vcxproj`, `Benchmark/Benchmark.vcxproj`, `Editor/Editor.pro`.

### libsndfile 1.2.2 — Audio File I/O
- **DLL:** `sndfile.dll`.
- **Used for:** reading/writing audio (WAV/FLAC/OGG) in convolution impulse-response loading, Benchmark I/O, and the Editor.
- **Note:** renamed from `libsndfile-1.dll` in commit `53d885f`.
- **Cited in:** `Common.vcxproj`, `Editor/Editor.pro`.

### Microsoft Visual C++ Runtime
- **DLLs:** `vcruntime140.dll`, `vcruntime140_1.dll` (where present), `msvcp140.dll`, `msvcp140_1.dll`, `msvcp140_2.dll`.
- **Used for:** C++ standard library + CRT support for the v143 (Visual Studio 2022) toolset.

## Build / Development Dependencies (not shipped)

### Visual Studio 2019 or 2022
- **Used for:** building the C++ projects (`EqualizerAPO.sln`, `Common.vcxproj`, `EqualizerAPO/EqualizerAPO.vcxproj`, `Benchmark/Benchmark.vcxproj`, `DeviceSelector/DeviceSelector.vcxproj`, `VoicemeeterClient/VoicemeeterClient.vcxproj`).
- **Toolset:** v143 preferred. Each `.vcxproj` declares Win32, x64, and ARM64 platforms.

### Qt 6.7.2 (development install)
- **Used for:** `qmake` build of `Editor.pro` and DeviceSelector. `build.bat` expects `qmake.exe` under `C:\Qt\6.7.2\msvc2022\bin\`, `C:\Qt\6.7.2\msvc2022_64\bin\`, and `C:\Qt\6.7.2\msvc2019_arm64\bin\`.
- **Build tool:** `jom` (parallel make, ships with Qt Creator) — `C:\Qt\Tools\QtCreator\bin\jom\jom.exe`.

### FFTW 3.3.10 (development install)
- **Install paths:** `C:\Program Files (x86)\fftw3` (32-bit) and `C:\Program Files\fftw3` (64-bit).
- **Import libraries:** must be generated with Visual Studio's `lib` tool (per FFTW Windows install guide).

### libsndfile 1.2.2 (development install)
- **Install paths:** `C:\Program Files (x86)\libsndfile` and `C:\Program Files\libsndfile`.

### muParserX 3.0.1
- **Used for:** expression parsing in `filters/ExpressionFilterFactory.cpp` and the `parser/` extensions.
- **Version pin:** must be exactly 3.0.1 — the semicolon operator was removed in 3.0.2.
- **Install path:** under `C:\Program Files`. Static library; not shipped.

### TCLAP
- **Used for:** Benchmark command-line argument parsing (`Benchmark/Benchmark.vcxproj`).
- **Type:** header-only template library; compiled into the Benchmark binary.

### NSIS (Nullsoft Scriptable Install System)
- **Used for:** building the installers via `Setup/Setup.nsi`, `Setup/Setup32.nsi`, `Setup/Setup64.nsi`, `Setup/SetupARM64.nsi`.
- **Required plugins:** NSISpcre (regex), AccessControl (registry/file permissions), nsArray (array data structure).

### uncrustify
- **Used for:** enforcing C++ code style via `uncrustify.cfg`. Invoked by `reformat.bat`.

## Vendored Dependencies (in-tree, no install)

### libHybridConv 0.1.1 — Hybrid Convolution
- **Location:** `libHybridConv-0.1.1/`.
- **Used for:** partition-based, FFT-accelerated convolution for `GraphicEQFilter` and `ConvolutionFilter`.
- **Author:** Christian Borss (2009); GPL v2.
- **Internal dependency:** uses FFTW for the FFT step.
- **Equalizer APO wrapper:** `libHybridConv_eapo.cpp`/`.h`.

## VST SDK

### `helpers/aeffectx.h` — VST 2.x Plugin Header
- **Used for:** hosting VST 2.x plugins in `VSTPluginFilter`, `VSTPluginInstance`, and `VSTPluginLibrary`.
- **SDK:** VST 2.x (not VST 3.x).
- **Not shipped** — header-only; static compile-time integration.
- **Excluded from reformatting** (`reformat.bat`) as third-party code.

## Target Platforms

- **Windows 7 and later** — core APO.
- **Windows 10 / 11** — full feature support including dark mode.
- **Architectures:** Win32 (x86), x64, ARM64.

## C++ Standard

- **Standard:** C++17 (`stdcpp17` in the `.vcxproj` files).
- **Compiler:** MSVC v143 (Visual Studio 2022).

## See Also

- `docs/getting-started.md` — install paths and step-by-step setup for each dependency.
- `docs/architecture.md` — how the runtime libraries are integrated.
- `Wiki/Developer.txt` — original developer notes (lists older versions; superseded by recent commits for FFTW, libsndfile, and Qt).
