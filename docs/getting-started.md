# Getting Started

This guide covers both end-user installation and developer build instructions for Equalizer APO 1.4.

## For End Users

### Downloading

Equalizer APO is distributed as a free Windows installer from SourceForge. Three architectures are provided:

- **EqualizerAPO32-1.4.exe** — for 32-bit Windows.
- **EqualizerAPO64-1.4.exe** — for 64-bit Windows (recommended).
- **EqualizerAPOARM64-1.4.exe** — for Windows on ARM (added in commit `53d885f`).

Project page: <https://sourceforge.net/projects/equalizerapo/files/>.

To determine your Windows architecture, open **Start → Control Panel → System** and look for "System type".

### Installing

1. **Run the installer.** Execute the downloaded setup program and follow the on-screen instructions. The default install path is `C:\Program Files\EqualizerAPO` (or `C:\Program Files (x86)\EqualizerAPO` for the 32-bit build).
2. **Choose audio devices.** The Device Selector runs automatically during install. Pick the audio device(s) where you want the APO installed. To find your default output, open **Start → Control Panel → Sound**.
3. **Reboot.** Allow the system to restart so the Windows Audio Service reloads with the new APO.
4. **Verify.** After reboot, the APO should be active. Re-launch Device Selector from your installation directory (e.g., `C:\Program Files\EqualizerAPO\DeviceSelector.exe`) and use its built-in test dialog to confirm the APO is processing audio.

### Managing Devices Later

Run **DeviceSelector.exe** any time to:

- Add or remove devices the APO is installed on.
- Toggle troubleshooting options if you encounter audio instabilities.
- Restart the audio service to apply changes without a reboot.

### Configuring Audio Filters

Equalizer APO uses plain-text configuration files:

1. Navigate to `C:\Program Files\EqualizerAPO\config\` (adjust path for your install).
2. Open `config.txt` in a text editor or in the Configuration Editor (Editor.exe).
3. By default, `config.txt` includes `example.txt`, which applies a mild low-frequency boost and a slight volume reduction. Edit the `Preamp` value while audio is playing — changes apply instantly.
4. For full filter syntax and configuration directives, see `docs/configuration.md`.

For measurement-driven room correction, **Room EQ Wizard** (a free third-party tool) can produce filter exports compatible with Equalizer APO.

## For Developers

### Prerequisites

Required to build Equalizer APO from source:

1. **Visual Studio 2019 or 2022** — Community edition is fine. The newest toolset (`v143`) is preferred.
2. **libsndfile 1.2.2** — install both 32-bit and 64-bit versions to `C:\Program Files (x86)\libsndfile` and `C:\Program Files\libsndfile`.
3. **FFTW 3.3.10** — extract prebuilt 32-bit and 64-bit archives to `C:\Program Files (x86)\fftw3` and `C:\Program Files\fftw3`; generate import libraries with Visual Studio's `lib` tool.
4. **muParserX 3.0.1** — must be exactly 3.0.1 (the semicolon operator was removed in 3.0.2). Extract to `C:\Program Files`.
5. **TCLAP** — header-only template library; extract to `C:\Program Files`. Used by the Benchmark utility.
6. **Qt 6.7.2** — install 32-bit, 64-bit, and ARM64 MSVC builds under `C:\Qt`. Qt 6.7.2 includes modifications for Windows 7 compatibility (commit `a57f063`).
7. **NSIS** — for installer creation. Install the plugins NSISpcre, AccessControl, and nsArray.

> **Note on `Wiki/Developer.txt`:** the wiki lists older versions (Qt 5, no version pins for FFTW/libsndfile). The versions above reflect the current `build.bat` and the most recent updates (commits `a57f063` and `53d885f`).

### Building

Run `build.bat` from the repository root. The script:

1. Locates the latest Visual Studio with `vswhere`, then calls `VsDevCmd.bat` (`build.bat:1-2`).
2. Builds the C++ projects via `msbuild EqualizerAPO.sln /p:Configuration=release /p:platform=<plat> /t:rebuild /m` for `win32`, `x64`, and `ARM64` (`build.bat:3-8`).
3. For each architecture, creates a Qt build directory (e.g., `build-Editor-Desktop_Qt_6_7_2_MSVC2022_64bit`), calls `vcvarsall.bat`, runs qmake against `Editor/Editor.pro`, and builds with `jom` (`build.bat:10-35`).
4. Calls NSIS to produce `Setup32.nsi`, `Setup64.nsi`, and `SetupARM64.nsi` (`build.bat:37-49`).

The artifacts are three installer executables in `Setup/`:

- `Setup/EqualizerAPO32-1.4.exe`
- `Setup/EqualizerAPO64-1.4.exe`
- `Setup/EqualizerAPOARM64-1.4.exe`

For a quick smoke test without packaging, you can run msbuild manually for a single platform.

### Code Formatting

Before committing, run `reformat.bat`. It iterates all `.cpp` and `.h` files and applies:

```
uncrustify -c uncrustify.cfg -l CPP --replace --no-backup <file>
```

The script skips build directories, resource headers, and vendored third-party code: `libHybridConv-0.1.1\`, `helpers\aeffectx.h`, `helpers\UncaughtExceptions.h`, and `VoicemeeterClient\VoicemeeterRemote.h` (`reformat.bat:4`).

### Further Reading

- `docs/architecture.md` — module decomposition and audio pipeline details.
- `docs/configuration.md` — configuration file format and registry values.
- `docs/development.md` — code style, adding a new filter, translation workflow, release process.
- `docs/dependencies.md` — full dependency catalog.
- `Wiki/Developer.txt` — original (partially dated) developer notes covering APO COM registration and system integration.
