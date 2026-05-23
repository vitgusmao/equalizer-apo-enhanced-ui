# Development Workflow

This document covers code style, the procedures for extending the project, and the release process. For building and prerequisites, see `docs/getting-started.md`.

## Build Pipeline

The complete build pipeline (Visual Studio + qmake + NSIS) is documented in `docs/getting-started.md` under "For Developers". This section covers only development-specific extras.

### Code Formatting (`reformat.bat`)

`reformat.bat` applies uncrustify with `uncrustify.cfg` to every `.cpp`/`.h` file in the tree, skipping:

- Build directories (`\build-*`)
- Resource headers (`\resource.h`)
- Vendored code: `libHybridConv-0.1.1\`, `helpers\aeffectx.h`, `helpers\UncaughtExceptions.h`, `VoicemeeterClient\VoicemeeterRemote.h`

Run this before every commit to keep diffs minimal.

## Code Style

Style is enforced by **uncrustify** with `uncrustify.cfg` at the repository root. Highlights (see the config file for the complete set):

- **Indentation:** 4-column tabs (`indent_columns=4`, `indent_with_tabs=2`).
- **Continuation indent:** 4 columns.
- **Operator spacing:** add space around arithmetic, assignment, and comparison operators (`sp_arith=add`, `sp_assign=add`, `sp_compare=add`).
- **Pointer/reference:** `T *p` style — no space before `*`/`&`, space after (`sp_before_ptr_star=remove`, `sp_after_ptr_star=add`).
- **Parentheses/brackets:** no space inside (`sp_inside_paren=remove`, `sp_inside_square=remove`).
- **Commas:** space after, not before (`sp_after_comma=add`).
- **Max blank lines:** 2 consecutive (`nl_max=2`); blank lines eaten at the top/bottom of braced blocks.
- **One-liner functions:** allowed (`nl_func_leave_one_liners=true`).

Refer to `uncrustify.cfg` for any setting not listed.

### License Header

All `.cpp` and `.h` files carry the GPL v2 header defined in `EqualizerAPO.licenseheader`:

```cpp
/*
    This file is part of Equalizer APO, a system-wide equalizer.
    Copyright (C) %CurrentYear%  Jonas Thedering

    This program is free software; you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation; either version 2 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License along
    with this program; if not, write to the Free Software Foundation, Inc.,
    51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.
*/
```

The QtCreator license-header plugin substitutes `%CurrentYear%` and inserts/updates this block automatically.

## Adding a New Filter Type

Filters follow a factory pattern; the core (DLL) and Editor sides are independent.

### Core Filter (DLL)

1. **Filter class** — implement `IFilter` (`IFilter.h`):
   - `initialize(float sampleRate, unsigned maxFrameCount, std::vector<std::wstring> channelNames)` — returns the actual channel names (may differ if the filter changes channel count).
   - `process(float **output, float **input, unsigned frameCount)`.
   - Override channel-handling methods if needed: `getAllChannels()`, `getInPlace()`, `getSelectChannels()`.

2. **Factory class** — implement `IFilterFactory` (`IFilterFactory.h`):
   - `createFilter(const std::wstring &configPath, std::wstring &command, std::wstring &parameters)` — parses your directive and returns an `IFilter`.
   - Lifecycle hooks: `startOfConfiguration()`, `startOfFile()`, `endOfFile()`, `endOfConfiguration()`.

3. **Register** the factory in `FilterEngine.cpp` (search for the `factories.push_back(...)` block). Existing factories live alongside: `DeviceFilterFactory`, `IfFilterFactory`, `ExpressionFilterFactory`, `IncludeFilterFactory`, `StageFilterFactory`, `ChannelFilterFactory`, `IIRFilterFactory`, `BiQuadFilterFactory`, `PreampFilterFactory`, `DelayFilterFactory`, `CopyFilterFactory`, `ConvolutionFilterFactory`, `GraphicEQFilterFactory`, `VSTPluginFilterFactory`, `LoudnessCorrectionFilterFactory`.

### Editor GUI (Qt)

1. Implement `IFilterGUI` in a new `Editor/guis/YourFilterGUI.h/cpp`.
2. Add a corresponding `IFilterGUIFactory` subclass (e.g., `YourFilterGUIFactory.h/cpp`).
3. Add a Qt Designer layout `Editor/guis/YourFilterGUI.ui` if the dialog is non-trivial; the Editor build runs `uic` automatically.
4. Register the GUI factory in the Editor's filter list (see the existing factories for the pattern).

Reference example: BiQuad spans `filters/BiQuadFilter.cpp`, `filters/BiQuadFilterFactory.cpp`, `Editor/guis/BiQuadFilterGUI.cpp`, `Editor/guis/BiQuadFilterGUI.ui`, and the factory.

## Translations

The Editor and Device Selector use Qt's translation framework. Translation sources live in `Editor/translations/` (e.g., `Editor_de.ts`) and `DeviceSelector/translations/` (e.g., `DeviceSelector_de.ts`).

Workflow:

1. **Extract strings:** `lupdate Editor.pro` (and equivalent for DeviceSelector) updates the `.ts` files with new translatable strings.
2. **Translate:** edit the `.ts` files (Qt Linguist works well).
3. **Compile:** `lrelease Editor.pro` generates the binary `.qm` files consumed at runtime.

Translation targets are declared via `TRANSLATIONS` in `Editor.pro`.

## Release Process

Per `Release checklist.txt`:

1. Update the version in `version.h`.
2. Run `build.bat` to produce the installers.
3. Commit trunk with message `Version x`.
4. Export the trunk to an external folder and create `EqualizerAPO-src-x.zip` from the folder contents.
5. Upload `EqualizerAPO32-x.exe`, `EqualizerAPO64-x.exe`, `EqualizerAPOARM64-x.exe`, and `EqualizerAPO-src-x.zip` to a new release folder.
6. Update `readme.txt` using the SVN log.
7. Mark `EqualizerAPO64-x.exe` as the default download.
8. Update the "Downloads" tool URL to `https://sourceforge.net/projects/equalizerapo/files/x/`.
9. Create a tag in `tags/x` with message `Tag release x`.

(Step 5 in the checklist file predates ARM64; include the ARM64 installer in modern releases.)

## Testing

The **Benchmark** project is the primary correctness/performance harness:

- Runs the FilterEngine offline against a generated sweep or an audio file.
- Reports processing time, CPU load, latency, peak output level, and clipped-sample count.
- Useful for tuning new filters and reproducing regressions.

There is **no automated unit-test suite**. All verification is currently manual via Benchmark, the Configuration Editor's analysis panel, and end-user smoke testing after install. Adding tests would be a worthwhile contribution.

## Project Structure

```
equalizer-apo-enhanced-ui/
├── build.bat                  # Full release build
├── reformat.bat               # Apply uncrustify formatting
├── uncrustify.cfg             # Code style configuration
├── EqualizerAPO.licenseheader # License header template
├── Release checklist.txt      # Release procedure
├── version.h                  # Current version
├── EqualizerAPO.sln           # Visual Studio solution
├── IFilter.h / IFilterFactory.h  # Core filter interfaces
├── FilterEngine.{h,cpp}       # Signal processing coordinator
├── FilterConfiguration.{h,cpp}# Active filter chain
├── AbstractAPOInfo.h / DeviceAPOInfo.* / VoicemeeterAPOInfo.*  # APO install state
├── filters/                   # Filter implementations + factories
│   └── loudnessCorrection/
├── Editor/                    # Qt configuration editor
│   ├── Editor.pro
│   ├── guis/                  # Per-filter GUI dialogs
│   ├── helpers/
│   ├── icons/                 # Light + dark-mode icons
│   ├── translations/          # .ts/.qm files
│   ├── widgets/
│   └── sounds/
├── DeviceSelector/            # Qt device installation tool
├── EqualizerAPO/              # Core APO COM object (DLL)
├── Benchmark/                 # Performance/correctness console tool
├── VoicemeeterClient/         # Voicemeeter IPC broker
├── helpers/                   # Utility libraries (registry, log, VST, …)
├── parser/                    # muParserX extensions
├── libHybridConv-0.1.1/       # Vendored convolution library
├── Setup/                     # NSIS installer scripts
│   ├── lib32/  lib64/  libARM64/  # Shipped DLLs per architecture
│   └── config/                # Default configuration files
└── Wiki/                      # Original user/developer documentation (.txt + images)
```

## Additional Resources

- `Wiki/Developer.txt` — original developer notes on APO COM registration, registry layout, and system integration. Predates the Qt 6 / ARM64 / DeviceSelector changes; cross-reference against current code.
- `Wiki/Configuration reference.txt` — canonical configuration format reference.
- `License.txt` — GNU General Public License v2.
