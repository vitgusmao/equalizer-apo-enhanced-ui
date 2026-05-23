# Features

Equalizer APO is a system-wide audio equalizer for Windows that provides professional-grade audio processing with a user-friendly configuration editor. This document describes the capabilities available to users. For *how* features are implemented, see `docs/architecture.md`.

## Core Audio Processing

### System-Wide Equalization
Equalizer APO operates as a Windows Audio Processing Object (APO), processing audio system-wide for any application sending audio to selected devices. Unlike application-specific equalizers, Equalizer APO works transparently across all audio sources — music players, games, browsers, communication apps — without modifying them.

### Per-Device Configuration
Every audio device (speakers, headphones, USB DAC, Voicemeeter virtual devices, microphones) can have independent filter configurations. This is ideal for comparing equipment, tailoring settings per listening environment, or running asymmetric playback/recording chains.

### Real-Time Preview and Analysis
The Configuration Editor's analysis panel provides real-time frequency response visualization:

- Combined frequency response of all applied filters.
- Peak gain, latency, and processing time for the current configuration.
- Live audio analysis from any device.
- Instant visual feedback while adjusting filters to avoid clipping and maintain balance.

## Filter Types

### Parametric Filters (Biquad)
Standard biquad filter types with adjustable frequency, gain, and Q/bandwidth:

- **Peaking (PK/PEQ/Modal)** — boost or attenuate a specific frequency range.
- **High-pass (HP/HPQ)** — remove low-frequency rumble.
- **Low-pass (LP/LPQ)** — attenuate high frequencies.
- **Band-pass (BP)** — isolate a specific frequency range.
- **Notch (NO)** — remove a narrow frequency band.
- **All-pass (AP)** — adjust phase without affecting amplitude.
- **Low-shelf / High-shelf (LS, LSC, HS, HSC)** — boost or attenuate below/above a frequency.
- **6 dB / 12 dB shelf variants (LS 6dB, HS 12dB, …)** — corner-frequency-based shelf shortcuts.

Implementations live in `filters/BiQuadFilter.cpp` and `filters/BiQuadFilterFactory.cpp`; the Editor counterpart is `Editor/guis/BiQuadFilterGUI.cpp`.

### Graphic Equalizer
Multi-band graphic EQ with linear interpolation in logarithmic frequency space. Supports 15-band, 31-band, and arbitrary variable-band layouts. Implementation: `filters/GraphicEQFilter.cpp`; Editor: `Editor/guis/GraphicEQFilterGUI.cpp`.

### Generic IIR Filter
Custom infinite-impulse-response filter with user-supplied coefficients (`Filter: ON IIR Order m Coefficients …`), enabling arbitrary filter designs beyond standard types. Implementation: `filters/IIRFilterFactory.cpp`.

### Convolution Filter
Impulse-response convolution backed by the vendored `libHybridConv` library. Supports multi-channel WAV/FLAC/OGG impulse responses. Common applications:

- Room acoustic correction.
- Reverberation.
- Head-related transfer functions (HRTF) for spatial audio.
- Speaker/headphone simulation.

Implementation: `filters/ConvolutionFilter.cpp`; Editor: `Editor/guis/ConvolutionFilterGUI.cpp`.

### Loudness Correction Filter
Automatically adjusts shelving filter parameters based on system volume, compensating for human ear sensitivity at low listening levels. Includes:

- Calibration mode with test sound for reference-level setup.
- Configurable reference level, offset, and attenuation factor.

Implementation: `filters/loudnessCorrection/LoudnessCorrectionFilter.cpp`; Editor: `Editor/guis/LoudnessCorrectionFilterGUI.cpp`.

### VST Plugin Filter
Load and control third-party VST 2.x plugins inline:

- Native x86, x64, and ARM64 plugin support (must match host architecture).
- Plugin parameter automation via configuration expressions.
- State persistence including plugin chunk data.
- Multiple plugin instances across channels.

Implementation: `filters/VSTPluginFilter.cpp`; Editor: `Editor/guis/VSTPluginFilterGUI.cpp`. See `docs/known-issues.md` for ARM64 plugin availability caveats.

## Audio Routing and Channel Control

### Channel Selection
Apply filters selectively to specific channels (L, R, C, LFE, RL, RR, RC, SL, SR), all channels, or custom combinations. Implementation: `filters/ChannelFilterFactory.cpp`; Editor: `Editor/guis/ChannelFilterGUI.cpp`.

### Channel Copy and Mixing
Duplicate and mix audio between channels with linear or dB-scaled gains. Useful for mono-to-stereo upmixing, custom downmixing, LFE routing for measurement, and crossfeed. Implementation: `filters/CopyFilter.cpp`; Editor: `Editor/guis/CopyFilterGUI.cpp`.

### Delay
Sample-accurate delay in milliseconds or samples; ms is sample-rate-independent. Useful for phase alignment across channels or devices. Implementation: `filters/DelayFilter.cpp`; Editor: `Editor/guis/DelayFilterGUI.cpp`.

### Preamp
Adjust overall output level to prevent clipping from positive gains. Cumulative across multiple sections; isolated per channel/device. Implementation: `filters/PreampFilter.cpp`; Editor: `Editor/guis/PreampFilterGUI.cpp`.

### Device Filter
Enable sections only for specific devices (matched against device name, connection name, or GUID). Implementation: `filters/DeviceFilterFactory.cpp`; Editor: `Editor/guis/DeviceFilterGUIDialog.cpp`.

### Stage Selection
Choose where in the audio pipeline filters are applied:

- **pre-mix** — per-application processing before mixing; enables upmixing.
- **post-mix** — applied once after mixing; more CPU-efficient; preferred for room correction.
- **capture** — applied to input devices.

Implementation: `filters/StageFilterFactory.cpp`.

## Configuration Editor

### Visual Filter Editing
Qt-based editor with tabbed multi-config management, visual dialogs for every filter type, FFTW-backed real-time frequency response plotting, and drag-and-drop filter ordering. Entry point: `Editor/MainWindow.cpp`; analysis: `Editor/AnalysisThread.cpp` and `Editor/AnalysisPlotView.cpp`.

### Configuration Workflow
- Open, edit, and save text configuration files via the GUI.
- Switch between visual and raw-text editing modes.
- Instant mode: changes apply immediately without restarting any application.
- Import/export configurations for sharing and backup.

### Advanced Configuration
- **Include** command for modular configurations.
- **If / ElseIf / Else / EndIf** based on device properties, channel counts, sample rates, and registry values.
- **Expression Language** with arithmetic, logical operators, trig functions, regex matching, and registry queries.
- **Inline expressions** to embed dynamic values in filter parameters.

See `docs/configuration.md` for the full directive set.

## Device Selector and Installation

### Device Management
The Qt-based DeviceSelector (replacing the legacy Configurator since commit `37aeceb`) provides:

- Modern, DPI-aware interface.
- Per-device install / uninstall for speakers, microphones, USB DACs, and headphones.
- Per-device enable/disable without uninstall.
- Batch selection for configuring multiple devices at once.

Implementation: `DeviceSelector/DeviceSelector.cpp`.

### Installation Testing
- Test dialog (`DeviceSelector/DeviceTestDialog.cpp`) plays/records audio to verify the APO is functioning correctly.
- Automatic troubleshooting: tries different installation modes and reports which works.
- Works around combined Bluetooth device issues on Windows 11 (auto-falls back to SFX/MFX).
- Original-APO preservation option to keep vendor audio features.

### Service Management
Restart the Windows Audio Service to apply changes without a system reboot. Implementation: `helpers/ServiceHelper.cpp`.

## Voicemeeter Integration

- Automatic detection and support for Voicemeeter virtual devices.
- Separate APO installation per Voicemeeter output (e.g., A1–A5, B1–B3 in Voicemeeter Potato).
- Real-time client synchronization via `VoicemeeterClient.exe`.
- Sample-rate persistence for virtual devices.
- Reimplementation based on official Voicemeeter examples eliminated earlier stuttering issues.

Implementation: `VoicemeeterAPOInfo.cpp`, `VoicemeeterClient/`.

## Performance Testing

### Benchmark Application
Command-line tool for measuring filter-chain performance:

- Generates logarithmic sine sweeps or processes existing audio files.
- Reports CPU usage, latency, and processing time.
- Supports arbitrary sample rates, channel counts, and batch sizes.
- Configurable device name / connection name / GUID for config matching.

See `docs/configuration.md` for the full flag list. Implementation: `Benchmark/Benchmark.cpp`.

## Platform Support

### Architecture Coverage
- **x86 (32-bit)** — full feature support including VST plugins.
- **x64 (64-bit)** — full feature support; recommended for modern systems.
- **ARM64** — equivalent functionality; VST plugin support limited to native ARM64 plugins.

### Operating System Compatibility
- Windows 10 and later — full feature support.
- Windows 7 and 8.1 — supported via Qt 6.7.2 with Windows 7 compatibility modifications.

### Dark Mode
- Automatic dark/light synchronization with Windows Settings.
- Consistent across Configuration Editor and DeviceSelector.
- Adaptive icon set in `Editor/icons/dark-mode/`.

## Multilingual Support

Editor and DeviceSelector ship with English (source) and German translations. Translation files live in `Editor/translations/` and `DeviceSelector/translations/` (Qt `.ts`/`.qm` format).

## Advanced Use Cases

- **Room acoustic correction** — combine impulse-response measurements from Room EQ Wizard with convolution filters.
- **Sample-rate-dependent processing** — use `If` blocks and `Eval` expressions to vary filter coefficients per rate.
- **Volume-dependent behavior** — loudness correction filter or expression-based gain.
- **Multi-device workflows** — distinct profiles per pair of headphones/monitors/rooms/recording chains.
- **System-wide effects** — bass enhancement, presence-peak reduction, headphone crossfeed, studio-monitor simulation with HRTF.
