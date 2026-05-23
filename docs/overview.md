# Equalizer APO Overview

## What is Equalizer APO?

Equalizer APO is a free, open-source system-wide audio processor for Windows that acts as an Audio Processing Object (APO) driver module. It provides parametric equalization, graphic EQ, convolution, audio effects, and VST plugin support directly at the Windows audio processing level — applying audio enhancement to all applications without modification, whether you're listening to music, watching videos, or using communication software.

## Key Features

- **System-wide audio processing** — automatically processes audio for all applications on selected audio devices.
- **Parametric equalization** — peaking, shelving, high-pass, low-pass, band-pass, notch, and all-pass biquad filters with customizable frequency, gain, and Q/bandwidth.
- **Graphic equalizer** — linear interpolation in logarithmic frequency space with arbitrary band layouts.
- **Convolution filtering** — impulse-response-based processing for room correction and speaker modeling.
- **VST plugin support** — load third-party VST plugins directly into the processing chain.
- **Configuration Editor** — Qt-based graphical editor with real-time frequency response visualization and analysis.
- **Device Selector** — modern, DPI-aware tool for selecting audio devices and verifying APO installation.
- **Dark mode** — automatic dark/light theme tracking the Windows Settings app.
- **Multi-channel support** — per-channel and per-stage filtering for stereo, surround, and beyond.
- **Expression language** — conditional logic, mathematical expressions, regex, and registry integration in configuration files.
- **Voicemeeter integration** — extends processing to Voicemeeter virtual audio routing.

## Who Is It For?

- **Audiophiles** — fine-tune system audio frequency response without external hardware.
- **Gamers** — apply room correction or spatial enhancement to any game.
- **Audio engineers** — combine with Room EQ Wizard for measured room correction.
- **Home theater enthusiasts** — apply speaker and room calibration filters system-wide.
- **Technical users** — full control via text-based configuration files and an expression language.
- **Voicemeeter users** — add effects processing on virtual audio routes.

## Current Version

**1.4** — see `version.h` (`MAJOR 1`, `MINOR 4`, `REVISION 0`).

## Supported Platforms

- **Windows x86 (32-bit)** — full feature support.
- **Windows x64 (64-bit)** — full feature support; recommended for modern systems.
- **Windows ARM64** — full feature support; VST plugin support is limited because native ARM64 VST plugins are rare (added in commit `53d885f`).

The Qt 6.7.2 binaries shipped with the installer include modifications for Windows 7 compatibility (commit `a57f063`).

## Quick Links

- `docs/architecture.md` — system decomposition, data flow, key abstractions, glossary.
- `docs/features.md` — what users can do, grouped by capability.
- `docs/getting-started.md` — install instructions for end users and build instructions for developers.
- `docs/configuration.md` — configuration file format, filter directives, registry values, CLI flags.
- `docs/known-issues.md` — compatibility limitations, workarounds, and recent fixes.
- `docs/development.md` — code style, adding a new filter, translations, release process.
- `docs/dependencies.md` — runtime and build dependencies; tech stack.
