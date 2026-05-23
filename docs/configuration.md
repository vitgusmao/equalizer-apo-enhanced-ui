# Configuration & Interfaces

This document covers Equalizer APO's configuration system: file format, filter directives, registry values, and command-line interfaces. The canonical user reference remains [`Wiki/Configuration reference.txt`](../Wiki/Configuration%20reference.txt); this document summarizes and indexes it.

## Configuration Files

### Format Overview

Configuration files are line-based; each command follows `Command: Parameters`:

- Lines that do not match `Command: Parameters` are silently ignored.
- Comment lines starting with `#` are supported.
- Unknown commands are silently ignored.
- Configuration files are automatically reloaded when modified (file-watcher driven).

Example:

```
Device: High Definition Audio Device Speakers; Benchmark
Preamp: -6 dB
Include: example.txt
Filter  1: ON  PK  Fc  50 Hz  Gain  -3.0 dB  Q 10.00

Channel: L
Preamp: -5 dB
Include: demo.txt

Channel: 2 C
Filter  1: ON  HP  Fc  30 Hz
```

### Default Configuration Files

Located in `Setup/config/`:

| File | Purpose |
|------|---------|
| `config.txt` | Main file loaded by Equalizer APO; ships with a default preamp and graphic EQ |
| `example.txt` | Simple bass-boost example (two peaking filters at 20 Hz and 45 Hz) |
| `demo.txt` | Demonstrates each supported BiQuad type (PK, Modal, LP, HP, LPQ, HPQ, LS, HS, LS 6dB, HS 12dB, NO, AP) |
| `iir_lowpass.txt` | IIR filter via `Eval`-computed coefficients |
| `multichannel.txt` | Per-channel filtering with separate preamps and includes |
| `selective_delay.txt` | `Copy`, `Channel`, and `Delay` for frequency-selective delay |

### Configuration Loading

- The main configuration file is `config.txt` in the config directory.
- The directory path is stored in registry value `HKEY_LOCAL_MACHINE\SOFTWARE\EqualizerAPO\ConfigPath`.
- Files are monitored for changes; modifications trigger automatic reload.
- `Include` paths are relative to the file that contains the `Include` directive.
- Included files in the config path or its subdirectories are also watched.

## Filter Directives

Two categories: **filtering commands** (affect audio) and **control commands** (control execution flow).

### Filtering Commands

#### Preamp

Sets preamplification in decibels. Multiple `Preamp` lines on the same channel sum.

```
Preamp: -6.5 dB
```

#### BiQuad Filters (Parametric EQ)

Standard biquadratic filters; the two parameter styles are interchangeable:

```
Filter <n>: ON <Type> Fc <Frequency> Hz Gain <Gain> dB Q <Q>
Filter <n>: ON <Type> Fc <Frequency> Hz Gain <Gain> dB BW Oct <Bandwidth>
```

| Type | Description | Uses Fc | Uses Gain | Uses Q/BW |
|------|-------------|:-------:|:---------:|:---------:|
| PK, PEQ, Modal | Peaking / parametric EQ | ✓ | ✓ | ✓ |
| LP, LPQ | Low-pass | ✓ |  | optional |
| HP, HPQ | High-pass | ✓ |  | optional |
| BP | Band-pass | ✓ |  | optional |
| LS, LSC | Low-shelf | ✓ | ✓ | optional |
| HS, HSC | High-shelf | ✓ | ✓ | optional |
| LS 6dB, LS 12dB | Low-shelf, corner-frequency shortcut | ✓ | ✓ |  |
| HS 6dB, HS 12dB | High-shelf, corner-frequency shortcut | ✓ | ✓ |  |
| NO | Notch | ✓ |  | optional |
| AP | All-pass | ✓ |  | ✓ |

`LS 6dB`/`HS 12dB`-style shortcuts use the corner frequency; `LSC`/`HSC` use the center frequency with dB/oct slope. Implementation: `filters/BiQuadFilterFactory.cpp`.

#### IIR Filters (Custom Coefficients)

```
Filter <n>: ON IIR Order <m> Coefficients <b0> <b1> ... <bm> <a0> <a1> ... <am>
```

Requires `2*(order+1)` coefficients. Coefficients depend on sample rate; combine with `If` blocks or inline expressions to vary by rate. Implementation: `filters/IIRFilterFactory.cpp`.

#### Delay

```
Delay: <t> ms
Delay: <n> samples
```

Milliseconds are sample-rate-independent. Implementation: `filters/DelayFilterFactory.cpp`.

#### Copy

Replaces or adds audio on a target channel from a linear combination of sources:

```
Copy: <Target>=<Factor>*<Source>+...
Copy: L=L+0.5*R
Copy: L=R+-6dB*C
Copy: LFE=L L=0.0 R=0.0 C=0.0 RL=0.0 RR=0.0
```

Factors may be linear or specified in dB. The target channel may also appear on the right-hand side. Constants need a decimal point to disambiguate from channel indices. Implementation: `filters/CopyFilterFactory.cpp`.

#### GraphicEQ

Linear interpolation in logarithmic frequency space:

```
GraphicEQ: 25 6; 40 4.5; 63 3; 100 1.5; 160 0; 250 0; 400 0; 630 0; 1000 0; 1600 0; 2500 0; 4000 0; 6300 1.5; 10000 3; 16000 3
```

Implementation: `filters/GraphicEQFilterFactory.cpp`.

#### Convolution

```
Convolution: <File>
```

Reads an impulse response file (WAV/FLAC/OGG) relative to the configuration file's directory. The sample rate must match the device. Multi-channel files are assigned round-robin. Files in the config path are watched for change. Implementation: `filters/ConvolutionFilterFactory.cpp`.

### Control Commands

#### Include

```
Include: <File>
```

Loads another configuration file. Implementation: `filters/IncludeFilterFactory.cpp`.

#### Device

```
Device: <Pattern 1>; <Pattern 2>; ...
```

Each pattern is space-separated keywords that must ALL appear in `"<Device_name> <Connection_name> <GUID>"`. Multiple patterns separated by `;` (any one matches). Special pattern `all` always matches. The Benchmark utility uses device name "Benchmark" and connection "File output". Implementation: `filters/DeviceFilterFactory.cpp`.

#### Channel

```
Channel: <Channel position 1> <Channel position 2> ...
```

Channel identifiers per layout:

| Layout | L | R | C | LFE | RL | RR | RC | SL | SR |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Mono |  |  | 1 |  |  |  |  |  |  |
| Stereo | 1 | 2 |  |  |  |  |  |  |  |
| Quadraphonic | 1 | 2 |  |  | 3 | 4 |  |  |  |
| Surround | 1 | 2 | 3 |  |  |  | 4 |  |  |
| 5.1 (Type A) | 1 | 2 | 3 | 4 |  |  |  | 5 | 6 |
| 5.1 (Type B) | 1 | 2 | 3 | 4 | 5 | 6 |  |  |  |
| 7.1 | 1 | 2 | 3 | 4 | 5 | 6 |  | 7 | 8 |

Channels may be selected by identifier or numeric index. The special identifier `all` selects every channel.

**Bass-redirect note:** many audio systems apply bass redirection after Equalizer APO processes the signal. For effective low-frequency filtering, apply filters to all channels, not just LFE. Implementation: `filters/ChannelFilterFactory.cpp`.

#### Stage

```
Stage: <stage 1> <stage 2>
```

- `pre-mix` — per-application filtering before mixing (higher CPU; required for upmixing).
- `post-mix` — after mixing; default; preferred for room correction.
- `capture` — single stage for input devices.

Implementation: `filters/StageFilterFactory.cpp`.

### Expression Commands

#### Constants

Runtime variables available in expressions:

| Name | Description |
|------|-------------|
| `e`, `pi` | Mathematical constants |
| `inputChannelCount` | Number of input channels to the current APO stage |
| `outputChannelCount` | Number of output channels from the current APO stage |
| `sampleRate` | Sample rate in Hz |
| `deviceName` | Audio device name |
| `connectionName` | Connection name |
| `deviceGuid` | Device endpoint GUID |
| `stage` | Current APO stage (`pre-mix`, `post-mix`, or `capture`) |

#### If / ElseIf / Else / EndIf

```
If: <expression>
    <commands>
ElseIf: <expression>
    <commands>
Else:
    <commands>
EndIf:
```

Expressions are evaluated as boolean. `If` blocks may nest but cannot conditionally execute `Device` statements. Implementation: `filters/IfFilterFactory.cpp`.

#### Eval and Inline Expressions

```
Eval: <expression>
<Command>: ... `<expression>` ...
```

**Operators:** arithmetic (`+ - * / ^`), assignment (`=`, `+=`, `-=`, `*=`, `/=`), logical (`and`, `or`, `xor`), comparison (`== != < > <= >=`), bitwise (`&`, `|`, `<<`, `>>`), string concatenation (`+` with a string operand), type cast (`(float)`, `(int)`), ternary (`cond ? a : b`), array literal (`{1,2}`), array access (`a[0]`).

**Functions:** math (`abs`, `sin`, `cos`, `tan`, `sinh`, `cosh`, `tanh`, `ln`, `log`, `log10`, `exp`, `sqrt`, `min`, `max`, `sum`), string (`strlen`, `tolower`, `toupper`, `str2dbl`), array (`sizeof`), regex (`regexSearch`, `regexReplace`), registry (`readRegString`, `readRegDWORD` — monitor registry values and trigger reload on change).

Example:

```
Eval: linGain = 0.5
Filter: ON PK Fc 1000 Hz Gain `20*log10(linGain)` dB Q 10.0
```

Implementation: `filters/ExpressionFilterFactory.cpp`, `parser/`.

## Registry Values

Equalizer APO stores its configuration under `HKEY_LOCAL_MACHINE\SOFTWARE\EqualizerAPO`.

| Value | Type | Purpose |
|-------|------|---------|
| `ConfigPath` | REG_SZ | Path to the configuration directory (e.g., `C:\Program Files\EqualizerAPO\config`) |
| `EnableTrace` | REG_SZ | `true` enables trace logging to `EqualizerAPO.log`; `false` disables |
| `DeviceTestPipeName` | REG_SZ | Temporary pipe name used by Device Selector's test dialog |
| `Child APOs` | subkey | Saved GUIDs of original driver APOs so they can be chained and restored on uninstall |

Per-device installation state is tracked in `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\MMDevices\Audio` via the standard Windows MMDevice keys.

## CLI Interfaces

### Benchmark Utility

The Benchmark utility runs the FilterEngine offline against a generated sweep or an input file. Useful for performance measurement and frequency-response validation.

**Invocation:** `Benchmark [options]`

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `-h`, `--help` | | | Show usage information |
| `-v`, `--verbose` | switch | off | Print trace/error messages to console instead of the logfile |
| `-i`, `--input <file>` | string |  | Read sound from a file instead of generating a sweep |
| `-o`, `--output <file>` | string | `%TEMP%\testout.wav` | Output file |
| `-f`, `--from <freq>` | float | 1.0 | Sweep start frequency (Hz) |
| `-t`, `--to <freq>` | float | 20000.0 | Sweep end frequency (Hz) |
| `-l`, `--length <s>` | float | 200.0 | Sweep duration (s) |
| `-r`, `--rate <rate>` | integer | 44100 | Sample rate of generated audio (Hz) |
| `-c`, `--channels <count>` | integer | 2 | Channel count of generated audio |
| `--batchsize <n>` | integer | 65536 | Frames per processing batch |
| `--devicename <name>` | string | `Benchmark` | Device name for `Device:` matching |
| `--connectionname <name>` | string | `File output` | Connection name for `Device:` matching |
| `--guid <guid>` | string | (empty) | Device GUID for `Device:` matching |
| `--nopause` | switch | off | Skip the trailing key-press prompt |

Implementation: `Benchmark/Benchmark.cpp`.

## Logging

Log file: `C:\Windows\ServiceProfiles\LocalService\AppData\Local\Temp\EqualizerAPO.log`

- Created only when an error occurs; under normal operation the file does not exist.
- Append mode; entries include a thread ID, timestamp, and caller address.

To enable detailed initialization/configuration tracing:

1. Open `regedit.exe`.
2. Navigate to `HKEY_LOCAL_MACHINE\SOFTWARE\EqualizerAPO`.
3. Set `EnableTrace` to `true`.
4. Reproduce the issue (play or record audio).
5. Set `EnableTrace` back to `false` after capture to prevent unbounded log growth.

Implementation: `helpers/LogHelper.cpp`.

## See Also

- [`Wiki/Configuration reference.txt`](../Wiki/Configuration%20reference.txt) — canonical reference.
- [`Wiki/Documentation.txt`](../Wiki/Documentation.txt) — user documentation and troubleshooting.
- `docs/architecture.md` — how the FilterEngine consumes these directives.
- `docs/features.md` — capability-level descriptions of each filter type.
