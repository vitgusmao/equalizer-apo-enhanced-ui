# Architecture & Design Concepts

Equalizer APO is a system-wide audio processing engine for Windows that integrates into the Windows audio pipeline as an Audio Processing Object (APO). It provides flexible, configuration-driven audio filtering with support for parametric equalization, graphic equalization, convolution, VST plugins, and custom expressions.

The architecture is decomposed into six primary modules plus supporting infrastructure:

1. **EqualizerAPO DLL** — the core audio processing object
2. **Editor** — Qt-based graphical configuration interface
3. **DeviceSelector** — Qt-based device management and APO installation tool (replaced the legacy Configurator in commit `37aeceb`)
4. **Benchmark** — console application for performance testing
5. **VoicemeeterClient** — integration with Voicemeeter virtual audio mixer
6. **Setup** — NSIS-based installer generation

Supporting subsystems include filter implementations (`filters/`), expression parser extensions (`parser/`), utility libraries (`helpers/`), and a vendored hybrid convolution library (`libHybridConv-0.1.1/`).

## System Decomposition

### EqualizerAPO DLL (Core Audio Processing)

**Location:** `EqualizerAPO/`

Implements the APO COM object loaded by the Windows Audio Service. `EqualizerAPO.h/cpp` implements `IAudioProcessingObject`, `IAudioProcessingObjectConfiguration`, `IAudioProcessingObjectRT`, and `IAudioSystemEffects` and delegates audio processing to `FilterEngine`. `ClassFactory.h/cpp` provides the COM class factory; `DllMain.cpp` is the DLL entry point.

The Windows Audio Service instantiates EqualizerAPO as a COM object registered under the APO GUID; the constructor (`EqualizerAPO/EqualizerAPO.cpp:41`) initializes a `FilterEngine` that drives the actual filtering. Audio buffers arrive via `APOProcess()` and are routed through `FilterEngine`. Child APO references are preserved so the audio driver's original APO chain continues to function.

### FilterEngine and FilterConfiguration

**Location:** repository root — `FilterEngine.h/cpp`, `FilterConfiguration.h/cpp`

`FilterEngine` is the signal processing coordinator. It loads configuration files, instantiates filters via the factory pattern, and orchestrates buffer flow:

- `initialize()` — sets sample rate, channel counts, and maximum frame size.
- `loadConfig()` — parses configuration files and creates filters.
- `watchRegistryKey()` — enables hot-reload on configuration file changes.
- `process()` — main processing loop (overloaded for mono and multi-channel paths).

`FilterConfiguration` holds a snapshot of the active filter chain. It exposes `read()` (input acquisition), `process()` (serial filter chain execution), `doTransition()` (smooth crossfade between configurations), and `write()` (output emission).

`FilterEngine` keeps three `FilterConfiguration` objects (`currentConfig`, `nextConfig`, `previousConfig`) to support lock-free reload during real-time processing. A background thread watches for configuration changes and prepares the next config while the audio thread keeps using the current one.

### Filter Abstractions

**Location:** repository root — `IFilter.h`, `IFilterFactory.h`

**`IFilter`** is the virtual base for all audio filters:

- `initialize(sampleRate, maxFrameCount, channelNames)` returns the actual channel names (may differ when a filter changes the channel count).
- `process(output, input, frameCount)` performs in-place or out-of-place audio processing.
- Pragma `AVRT_VTABLES_BEGIN/END` ensures real-time-safe vtable access.

**`IFilterFactory`** creates filters from configuration commands:

- `createFilter(configPath, command, parameters)` parses a directive and returns filter instances.
- Lifecycle hooks: `startOfConfiguration()`, `startOfFile()`, `endOfFile()`, `endOfConfiguration()`.

### Filter Implementations

**Location:** `filters/`

Concrete filter types, each with an `IFilter` implementation and an `IFilterFactory`:

- **BiQuadFilter** — second-order IIR filters (peaking, shelf, high-pass, low-pass, band-pass, notch, all-pass).
- **IIRFilter** — higher-order cascaded IIR filters with user-supplied coefficients.
- **ConvolutionFilter** — impulse-response convolution (room correction, reverberation) backed by `libHybridConv`.
- **GraphicEQFilter** — fixed- or variable-band graphic equalizer; internally generates an impulse response and convolves.
- **PreampFilter** — gain adjustment.
- **DelayFilter** — sample-accurate delay (milliseconds or samples).
- **CopyFilter** — multi-channel routing with linear or dB factors.
- **ChannelFilter** — channel selection (mono/stereo downmix, surround remix).
- **VSTPluginFilter** — third-party VST 2.x plugin hosting.
- **ExpressionFilter** — expression-based parameter computation via muParserX.
- **DeviceFilter** — conditional sections matched against device name, connection name, or GUID.
- **IfFilter** — conditional execution based on configuration expressions.
- **StageFilter** — pre-mix vs. post-mix branching (LFX vs. GFX).
- **IncludeFilter** — file inclusion (`filters/IncludeFilterFactory.cpp`).
- **LoudnessCorrectionFilter** — volume-aware loudness curve compensation (`filters/loudnessCorrection/`).

### APO Installation and Device Management

**Location:** repository root — `AbstractAPOInfo.h`, `DeviceAPOInfo.h/cpp`, `VoicemeeterAPOInfo.h/cpp`

`AbstractAPOInfo` is the base class for managing APO installation state on a device, exposing `install()`, `uninstall()`, `reinstall()`, and various property queries.

`DeviceAPOInfo` is the concrete implementation for audio devices. It manages registry entries under `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\MMDevices\Audio`, tracks two APO positions (pre-mix LFX and post-mix GFX) via the `InstallMode` enum (`INSTALL_LFX_GFX`, `INSTALL_SFX_MFX`, `INSTALL_SFX_EFX`), and preserves the original driver APOs by storing their GUIDs in `HKEY_LOCAL_MACHINE\SOFTWARE\EqualizerAPO\Child APOs` and chaining them into the processing pipeline.

`VoicemeeterAPOInfo` extends `AbstractAPOInfo` for the Voicemeeter virtual audio interface. It uses the Voicemeeter Remote API (from `VoicemeeterClient/VoicemeeterRemote.h`) and IPC pipes to coordinate effects with a running Voicemeeter instance.

### Editor (Configuration GUI)

**Location:** `Editor/`

Interactive configuration builder using Qt 6.7.2. Users design filter chains and save them as text configuration files.

- `MainWindow.h/cpp` — main application window.
- `FilterTable.h/cpp` — drag-and-drop filter chain editor.
- `guis/` — visual widgets per filter type (`BiQuadFilterGUI`, `ConvolutionFilterGUI`, `VSTPluginFilterGUI`, …).
- `AnalysisThread.h/cpp` — real-time frequency response analysis, feeding `AnalysisPlotView`.
- `MainWindow.ui` — Qt Designer layout.

The Editor reads filter metadata via `IFilterGUIFactory` and writes text-format configuration files into the `config/` directory; running EqualizerAPO instances hot-reload the changes.

### DeviceSelector (APO Installation Tool)

**Location:** `DeviceSelector/`

Qt-based replacement for the legacy Configurator (introduced in commit `37aeceb`). Features:

- DPI-aware rendering.
- A test dialog (`DeviceTestDialog`) backed by `DeviceTestThread` that plays/records test audio to confirm the APO is active.
- `ReceiveThread` listens for installation status from the running APO DLL.
- Automatic install-mode troubleshooting (tries different modes and reports which works).
- Hot-restart of the Windows Audio Service to avoid mandatory reboot.
- Voicemeeter support (creates a `VoicemeeterAPOInfo` if Voicemeeter is detected).

### Benchmark (Performance Testing)

**Location:** `Benchmark/`

Console utility for measuring filter-chain performance without installing to an audio device. Loads a configuration, reads from a file (or generates a sweep), runs it through `FilterEngine`, writes the output, and reports CPU usage and processing time. See `docs/configuration.md` for the CLI flags.

### VoicemeeterClient (Voicemeeter Integration)

**Location:** `VoicemeeterClient/`

Utility process that brokers between EqualizerAPO and Voicemeeter via Voicemeeter's IPC mechanism, allowing EqualizerAPO to inject itself as an audio effect inside the Voicemeeter virtual mixer. The COM interface lives in `VoicemeeterClient/VoicemeeterRemote.h`.

### Setup (Installer)

**Location:** `Setup/`

NSIS scripts and resources used to build installers. `Setup/Setup.nsi` is the common script; `Setup32.nsi`, `Setup64.nsi`, and `SetupARM64.nsi` are per-architecture wrappers. `build.bat` at the repository root invokes NSIS to package the EqualizerAPO DLL, Editor, DeviceSelector, and supporting files.

### Supporting Infrastructure

#### Filters Directory (`filters/`)
All concrete filter and factory classes — see "Filter Implementations" above.

#### Parser Extensions (`parser/`)
muParserX extensions for the configuration expression language:

- `LogicalOperators.h/cpp` — boolean logic.
- `StringOperators.h/cpp` — string operations.
- `RegistryFunctions.h/cpp` — Windows registry queries (with reload-on-change behavior).
- `RegexFunctions.h/cpp` — regex matching used in `If` and inline expressions.

#### Helpers Library (`helpers/`)
Utility modules including `RegistryHelper`, `StringHelper`, `LogHelper` (writes to `EqualizerAPO.log`), `MemoryHelper` (SIMD-aligned allocation), `ChannelHelper`, `ServiceHelper` (Windows Audio Service control), `VSTPluginInstance`/`VSTPluginLibrary` (VST 2.x wrapping), `PrecisionTimer`, and the third-party VST header `aeffectx.h`.

#### Hybrid Convolution Library (`libHybridConv-0.1.1/`)
Vendored third-party library by Christian Borss (2009). Implements partition-based convolution with FFT acceleration; used by `GraphicEQFilter` and `ConvolutionFilter` for low-latency impulse-response processing. `libHybridConv_eapo.cpp` provides the EqualizerAPO-specific wrapper.

## Data Flow: Audio Processing Pipeline

### Input
1. Windows Audio Service sends audio samples to `EqualizerAPO::APOProcess()` (`EqualizerAPO/EqualizerAPO.cpp:59`).
2. EqualizerAPO retrieves the current `FilterConfiguration` from `FilterEngine`.
3. `FilterConfiguration::read()` copies the input into working buffers (`FilterConfiguration.cpp:46`).

### Filter chain
4. `FilterConfiguration::process()` runs the filter chain serially (`FilterConfiguration.cpp:48`).
5. Buffers are swapped between filters to avoid redundant copies; FilterEngine specifies channel routing.
6. Output from filter *n* becomes input to filter *n+1*.

### Output
7. `FilterConfiguration::write()` copies results to the output buffer (`FilterConfiguration.cpp:51`).
8. EqualizerAPO returns the processed audio to the Windows Audio Service.
9. If a child APO is present, its output becomes EqualizerAPO's input on the next frame.

### Configuration reload
- Background thread detects file changes (`FilterEngine.cpp:49: watchRegistryKey()`).
- A new configuration is parsed into `nextConfig` without interrupting audio.
- At a frame boundary, `FilterEngine` swaps `currentConfig` and `nextConfig`.
- `doTransition()` performs a short crossfade to prevent clicks.
- The old config (`previousConfig`) is retained briefly for safe deallocation.

## Key Patterns

### Filter Factory Pattern
Every filter type ships two classes:

- **Filter class** (e.g., `BiQuadFilter`) — processes audio samples.
- **Factory class** (e.g., `BiQuadFilterFactory`) — parses configuration text and creates filter instances.

Example flow:

```
Config line:  Filter 1: ON PK Fc 100 Hz Gain 10 dB Q 0.5
→ BiQuadFilterFactory::createFilter() parses the parameters
→ Creates BiQuadFilter(PEQ, 100 Hz, Q=0.5, gain=10 dB)
→ FilterEngine appends it to the chain and initializes it with the sample rate
```

### Real-Time Safety
- `AVRT_VTABLES_BEGIN/END` macros ensure deterministic vtable access.
- Lock-free configuration swap (triple-buffer pattern across `currentConfig`/`nextConfig`/`previousConfig`).
- A synchronized semaphore prevents config loading during processing.
- No allocations on the hot `process()` path.

### APO Chaining
Windows can stack multiple APOs. EqualizerAPO preserves the original driver APO:

- The original APO GUID is saved under `HKEY_LOCAL_MACHINE\SOFTWARE\EqualizerAPO\Child APOs`.
- On initialization, EqualizerAPO instantiates the child APO COM object (`EqualizerAPO/EqualizerAPO.cpp:78`).
- EqualizerAPO's output is forwarded to the child APO; the child's output is returned to Windows.
- Driver-supplied effects (e.g., spatial audio, bass enhancement) coexist with EqualizerAPO.

## Glossary

- **APO (Audio Processing Object)** — COM-based audio effect module loaded by the Windows audio driver's real-time thread.
- **GFX (Global Effect)** — APO applied after all streams are mixed together (post-mix); one per output device.
- **LFX (Local Effect)** — APO applied before mixing (pre-mix); one per device, per direction.
- **SFX / MFX / EFX** — modern Windows 8.1+ naming for stream-effect, master-effect, and ecosystem-effect APO slots.
- **MMDevice** — Windows multimedia device abstraction; represents a physical or virtual audio endpoint.
- **FxProperties** — registry key under `MMDevices\Audio\[Render|Capture]\<device GUID>\FxProperties`, holding the APO GUIDs for a device.
- **Processing Mode** — Windows 8.1+ concept defining audio intent (default, raw, speech, movie, music); each mode may have its own APO chain.
- **Parametric EQ** — biquad-based equalizer with adjustable center frequency, gain, and Q/bandwidth.
- **Graphic EQ** — fixed- or variable-band equalizer; internally realized via an impulse response.
- **Hybrid Convolution** — efficient FIR filtering combining FFT in the frequency domain with overlap-add in the time domain.
- **Configuration File** — text-based filter specification, typically `C:\Program Files\EqualizerAPO\config\config.txt`. See `docs/configuration.md`.

## Architecture Diagrams

A processing-stages diagram is available in the Wiki: `Wiki/Stages.svg` (PNG and ODG variants are alongside).

## Multi-Architecture Support

As of commit `53d885f`, EqualizerAPO supports three architectures with separate `.vcxproj` configurations:

- **x86** — 32-bit legacy support.
- **x64** — 64-bit primary target.
- **ARM64** — Windows on ARM (VST plugin support limited; see `docs/known-issues.md`).

`build.bat` compiles all three. See `docs/getting-started.md` for the build workflow.

## Configuration

Configuration files are text, hot-reloaded at runtime. See `docs/configuration.md` for the full directive set, registry values, and expression-language reference.

## Historical Context

- The legacy **Configurator** was replaced by the Qt-based **DeviceSelector** in commit `37aeceb`.
- **Voicemeeter** support was added via `VoicemeeterClient` and `VoicemeeterAPOInfo`.
- **VST plugin** hosting is implemented through `helpers/aeffectx.h` (VST 2.x).
- **ARM64** support was added in commit `53d885f`, which also updated FFTW to 3.3.10 and libsndfile to 1.2.2.
- **Qt 6.7.2** replaced Qt 5 in commit `a57f063`, bringing dark mode and DPI scaling improvements.
