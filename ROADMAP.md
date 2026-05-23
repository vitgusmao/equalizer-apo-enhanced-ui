# Roadmap

Planned work for Equalizer APO. Items are ordered by priority; status reflects the current state, not a guarantee of delivery.

## 1. Cover the entire codebase with unit tests

**Status:** Not started.

The project currently has **no automated test suite**. All verification is manual via the `Benchmark/` console utility and post-install smoke testing. Adding unit tests is the highest-priority engineering investment because it unlocks safe refactoring, regression protection during platform updates (Qt, FFTW, libsndfile, new Windows versions), and confidence when adding new filter types.

### Scope

Cover every non-trivial production unit in the repository:

- **Core processing** — `FilterEngine`, `FilterConfiguration`, the triple-buffered reload mechanism, the crossfade transition (`doTransition`).
- **Filter implementations** — every concrete `IFilter` in `filters/` (BiQuad, IIR, GraphicEQ, Convolution, Delay, Copy, Channel, Preamp, VSTPlugin, Expression, Device, If, Stage, Include, LoudnessCorrection). Each filter needs:
  - Coefficient/impulse-response correctness (compare to a known-good reference for representative inputs).
  - Channel-handling correctness (`getAllChannels`, `getInPlace`, `getSelectChannels`).
  - Sample-rate / frame-count edge cases.
- **Filter factories** — directive parsing for every `IFilterFactory`. Cover malformed input, edge cases (`Filter: ON …` variants, `BW Oct` vs `Q`, dB-suffixed factors in `Copy`).
- **Configuration parser** — `parser/` muParserX extensions (`LogicalOperators`, `StringOperators`, `RegistryFunctions`, `RegexFunctions`).
- **APO installation logic** — `AbstractAPOInfo`, `DeviceAPOInfo`, `VoicemeeterAPOInfo` install-mode selection, child-APO chaining, registry round-trip (test against a mocked registry layer in `helpers/RegistryHelper`).
- **Helpers** — `StringHelper` (UTF-8/UTF-16 conversion, path manipulation), `ChannelHelper` (layout/index mapping), `MemoryHelper` (SIMD alignment guarantees), `LogHelper` (trace-vs-error filtering), `PrecisionTimer`.
- **Editor** — non-UI logic in `Editor/` (analysis pipeline in `AnalysisThread`, filter-table model, configuration serialization). Keep UI smoke tests separate from unit tests.

### Out of scope (initial pass)

- `libHybridConv-0.1.1/` — vendored third-party code; test only the EqualizerAPO wrapper (`libHybridConv_eapo.cpp`).
- `helpers/aeffectx.h` — third-party VST 2.x header.
- COM-DLL entry points (`EqualizerAPO/EqualizerAPO.cpp`, `ClassFactory.cpp`, `DllMain.cpp`) — integration-test via Benchmark instead; unit-testing COM activation paths is high cost / low value.
- NSIS scripts under `Setup/`.

### Approach

1. **Pick a framework.** Candidates: GoogleTest (most common for MSVC C++), Catch2 (header-only, ergonomic), doctest (minimal compile-time cost). Decision criteria: MSVC v143 compatibility across x86/x64/ARM64, CMake/MSBuild integration, ability to run from the existing `build.bat`.
2. **Carve out a `tests/` project** in `EqualizerAPO.sln` that links against the Common library and selected filter sources. Mirror the source layout: `tests/filters/`, `tests/parser/`, `tests/helpers/`, etc.
3. **Establish reference fixtures** for filter correctness — small WAV files plus golden output traces; use libsndfile for I/O so tests share the production audio path.
4. **Mock the boundary layers** that touch Windows (`RegistryHelper`, `ServiceHelper`, MMDevice queries) behind thin interfaces so the bulk of the logic is testable without a real audio device.
5. **Wire into `build.bat`** so a release build runs the test binary and fails on any regression. Match the existing per-architecture matrix (Win32 / x64 / ARM64).
6. **Track coverage** with OpenCppCoverage (free, MSVC-native) and publish a coverage delta on every release.

### Non-goals

- 100% line coverage. Target meaningful branches and the audio-correctness contract, not synthetic line numbers.
- Replacing the Benchmark utility. Benchmark remains the primary tool for performance regression and end-to-end signal-chain validation.

### Acceptance criteria

- Every directory listed under "Scope" has a corresponding `tests/<dir>/` with at least one test file.
- `build.bat` exits non-zero on any test failure for any architecture.
- A new filter cannot be added without a corresponding test (enforce via code review checklist or a CI lint).
