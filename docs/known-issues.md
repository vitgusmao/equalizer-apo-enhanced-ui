# Known Issues & Limitations

This document catalogs compatibility limitations, workarounds, stability considerations, and deprecated features. A search for `TODO`/`FIXME`/`XXX`/`HACK` markers in the main source tree (excluding vendored libraries and build directories) returns no matches, so the items below come from `Wiki/Documentation.txt` (the Troubleshooting section is canonical) and from commit messages.

## Compatibility Limitations

### ARM64 VST Plugin Support
**Limitation:** On ARM64, VST plugin support is limited because native ARM64 VST plugins are rare in the ecosystem. The APO itself has full functionality on ARM64.

**Workaround:** Use native ARM64 VST plugins where available, or fall back to the built-in filter types (peaking, shelf, high-pass, low-pass, notch, etc.).

**Source:** commit `53d885f`.

### Hardware-Accelerated OpenAL Bypass
**Limitation:** Applications using hardware-accelerated OpenAL access the audio device directly, bypassing APOs entirely.

**Workarounds:**

- Switch the application to a different output backend if supported (e.g., DirectSound).
- Force OpenAL to fall back to software by replacing `OpenAL32.dll` with a non-accelerated build (e.g., from <http://kcat.strangesoft.net/openal.html>).
- Rename or move the vendor-specific OpenAL library in `C:\Windows\System32` or `C:\Windows\SysWOW64` (typically `*_oal.dll`, e.g., `ct_oal.dll`). **Warning:** This is an unsupported driver modification.

There is no way to enable APO support for hardware-accelerated OpenAL itself.

**Source:** `Wiki/Documentation.txt` — "Hardware-accelerated OpenAL".

### Audio Enhancements Must Be Enabled
**Limitation:** If Windows audio enhancements are disabled for a device, the APO will not be loaded.

**Workaround:**

1. Open **Control Panel → Sound**, double-click your device.
2. **Enhancements tab:** ensure "Disable all enhancements" is **unchecked**.
3. **Advanced tab** (if the Enhancements tab is absent): ensure "Enable audio enhancements" is **checked**.

**Source:** `Wiki/Documentation.txt` — "Control Panel".

### Sound Card Driver Conflicts
**Limitation:** Some sound card drivers disable their own features when they detect a third-party APO at the same processing stage.

**Workaround:** In Device Selector, install Equalizer APO in only one stage (pre-mix *or* post-mix) by unchecking one "Install APO" checkbox. The original driver APO remains active in the other stage, recovering its features.

**Source:** `Wiki/Documentation.txt` — "Configurator" troubleshooting section.

### Voicemeeter Integration Requires VoicemeeterClient
**Limitation:** Voicemeeter integration requires `VoicemeeterClient.exe` to be running.

**Workaround:** The installer creates a startup link; ensure it is enabled. Manually launch the client from the install directory if it has been closed or has crashed.

**Source:** `VoicemeeterClient/` and `VoicemeeterAPOInfo.h`.

### Bluetooth Device Installation Mode (Windows 11)
**Limitation:** On Windows 11, combined Bluetooth devices (e.g., headphones + hands-free) may not work with the default EFX (post-mix-only) installation mode.

**Resolution:** Device Selector auto-detects this and falls back to SFX/MFX. Older versions of the Configurator required manual selection.

## Original APO Stability

**Issue:** Preserving the original driver APO can cause audio instabilities (crackling, stuttering, or full audio failure).

**Workaround:**

1. Open Device Selector.
2. Select your device.
3. Enable **Troubleshooting** options.
4. Uncheck both **"Use original APO"** checkboxes (pre-mix and post-mix).
5. Apply changes (audio service restart or reboot).

**Trade-off:** You lose driver-supplied enhancements. To compromise, uncheck only one box to retain partial driver functionality.

**Note:** The Configurator was replaced by Device Selector in commit `37aeceb`; the equivalent troubleshooting options live in the new tool.

**Source:** `Wiki/Documentation.txt` — "Configurator" troubleshooting section.

## UI / Framework Notes

### Dark Mode
Supported in the Configuration Editor and Device Selector since version 1.4 (commit `a57f063`). A partial-dark-mode display issue in the Editor was fixed in version 1.3.2 (commit `c584689`).

### Qt 6 Scrollbar Rendering
**Minor issue:** Scrollbars may not initially render correctly in some dialogs despite a correct dialog size. The Device Selector hides and re-enables scrollbars after layout as a workaround.

**Impact:** UI-only; no audio processing impact. Reference: `DeviceSelector/DeviceSelector.cpp`.

## Deprecated / Legacy

### Library DLL Renames (Backwards Compatible)
In commit `53d885f`, the following library DLLs were renamed:

- `libfftw3f-3.dll` → `fftw3f.dll`
- `libsndfile-1.dll` → `sndfile.dll`

Existing configurations continue to work; the rename is internal and does not affect filter directives.

### Configurator Replaced by Device Selector
The original Win32 Configurator has been removed in favor of the Qt-based Device Selector (commit `37aeceb`). All Configurator options — including the troubleshooting dialog for installation mode and original-APO handling — were ported.

User documentation referring to "Configurator" should be read as "Device Selector" for version 1.4 and later. The wiki pages predate the rename and are gradually being updated.

## Source-Code Status

A repository-wide grep for `TODO`, `FIXME`, `XXX`, and `HACK` (excluding `libHybridConv-0.1.1/` and `build-*` directories) returned no matches — there are no in-source tracking markers. Any new limitations should therefore be raised on the SourceForge issue tracker rather than inferred from source comments.

### VST Plugin Architecture Matching
**Requirement:** VST plugins must match the host architecture (x86, x64, or ARM64). Most users will need x64 plugins.

**Default location:** the `VSTPlugins` subdirectory inside the EqualizerAPO install folder; the Editor's VST plugin dialog can also open plugins from any directory.

## Recent Fixes (for reference)

- **1.4** — ARM64 support; Qt 6.7.2; FFTW 3.3.10; libsndfile 1.2.2; DeviceSelector replaces Configurator.
- **1.3.2** — Fixed partial dark mode display in Configuration Editor (`c584689`).
- **1.3** — DeviceSelector introduced; automatic SFX/MFX fallback for combined Bluetooth devices.
- **Earlier** — Voicemeeter stuttering resolved via reimplementation based on official examples; concurrent-init crash in GraphicEQ/Convolution filters fixed; VST removal crash fixed.
