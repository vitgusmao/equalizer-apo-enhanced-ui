# Windows Dev Setup and UI Refresh — Design

**Date:** 2026-05-23
**Author:** Vitor Loura (with Claude)
**Status:** Draft — awaiting user review

## Goal

Establish an isolated Windows-side development environment for the
Equalizer APO Enhanced UI project, and a collaboration workflow that lets
Claude (running in WSL) make UI/UX changes which the user previews and
refines on Windows via Qt Designer. The first deliverable is a refreshed
visual language for the Configuration Editor, applied incrementally
across its forms.

The native Equalizer APO driver itself is out of scope. Only the
Qt-based **Editor** (`Editor.pro`) and, later, **DeviceSelector**
(`DeviceSelector.vcxproj`) are touched.

## Scope and non-goals

In scope:

- Installing the Windows toolchain (VS 2022 Build Tools, Qt 6.7.3,
  Qt Creator 19.0.2).
- Verifying the Qt Designer preview loop works end-to-end without
  building the Editor binary.
- A documented ping-pong collaboration workflow between WSL-Claude
  and Windows-user.
- Picking the visual direction ("Refined native") and the form order
  for the first iteration.

Out of scope:

- The 30–45 min native-deps install side quest (FFTW, libsndfile,
  muParserX) needed for a full Editor build. Deferred — see "Deferred
  work" below.
- DeviceSelector restyling. It already has dark-mode support
  (commit `a57f063`) and is in better shape than the Editor; revisit
  after the Editor refresh lands.
- Any change to the APO driver, FilterEngine, or audio behavior.
- QML / Qt Quick migration. We stay in Qt Widgets.

## Environment

**Repository location (already cloned):**

```
Windows:  C:\Users\vitor\Projetos\equalizer-apo-enhanced-ui\
WSL:      /mnt/c/Users/vitor/Projetos/equalizer-apo-enhanced-ui/
```

Single source of truth on the Windows filesystem. Claude edits via
`/mnt/c/...` from WSL; Qt Creator and MSVC operate natively on the
Windows side. No syncing, no duplicate clones. OneDrive is disabled on
this machine, so no sync interference.

**Toolchain (all installed fresh, isolated to the project's needs):**

| Component | Version | Path | Purpose |
|-----------|---------|------|---------|
| VS 2022 Build Tools | latest | default | MSVC compiler + Windows SDK |
| Qt | 6.7.3 (MSVC 2022 64-bit kit) | `C:\Qt\6.7.3\msvc2022_64\` | UI framework |
| Qt Creator | 19.0.2 | `C:\Qt\Tools\QtCreator\` | IDE + Designer + jom |

Qt 6.7.3 is the closest available patch release to the project's
documented 6.7.2 (commit `a57f063`). The version mismatch is harmless
for development; the project's `build.bat` is not used by the dev loop.

**What was deliberately skipped:**

- Full Visual Studio IDE (Build Tools alone are enough).
- Qt Design Studio (Qt Quick / QML tool; we stay in Widgets).
- Qt 6.8/6.9/6.10/6.11 (newer than the project was tested against).
- MinGW, WebAssembly, Android, Qt Sources, Qt PDF, Qt WebEngine,
  Qt Insight Tracker, Qt Installer Framework, Qt Creator 20.0 beta.
- NSIS, Voicemeeter SDK (only needed for installer packaging).

## Iteration loop (deferred-build mode)

The Editor's `.pro` file references three native libraries that aren't
yet installed: `fftw3f.lib`, `sndfile.lib`, `muparserx.lib`. The full
build therefore fails at link time. We accept this and iterate via
Qt Designer's standalone form preview, which does **not** require those
libraries.

**What works without the native deps:**

- Opening `Editor/Editor.pro` in Qt Creator (project model loads).
- Editing any `.ui` file in Qt Designer.
- **Form Preview** (`Ctrl+Alt+R` in Designer) — renders the form
  standalone with real Qt styling.
- Editing `Editor.qrc` resources and icons.
- Editing or adding `.qss` stylesheets (loaded into the Designer
  preview via *Designer → Tools → Form Editor → Style*).
- Editing `.cpp` / `.h` files with full Qt Creator code intelligence.

**What does not work yet:**

- Building the Editor executable.
- Running the Editor against real configurations.
- Verifying changes that depend on runtime state.

This is acceptable because the visual direction (Refined native) is
~95% achievable through `.ui` layout edits and QSS, both of which
Designer previews accurately.

## Collaboration workflow

A ping-pong model on a shared feature branch.

**Branch:** `ui-modernization` (created at the first UI commit). Both
Claude and the user commit directly to it. Small, frequent commits —
one discrete visual change per commit.

**Per-change loop:**

1. User asks for a change ("refresh X", "try Y on Z").
2. Claude edits files on the shared Windows filesystem via `/mnt/c`:
   `.ui`, `.qss`, `.qrc`, icons, occasionally `.cpp`/`.h`.
3. Claude reports which files changed and which `.ui` to open.
4. User opens the `.ui` in Qt Designer, accepts Designer's reload
   prompt if the file was already open, presses `Ctrl+Alt+R`.
5. User sends a screenshot + short note, "looks good", or
   "I'll do the final touch".
6. If "final touch": user edits in Designer/Qt Creator, saves,
   commits or tells Claude to pick it back up.

**Handoff rule:** explicit ping-pong. When Claude is editing a file,
the user only *previews* it in Designer, doesn't edit. When the user
takes over for a final touch, they say so; Claude doesn't touch the
file until they hand it back. Prevents conflicting writes on the
shared working tree.

**What Claude owns:**

- Heavier QSS rewrites (selectors, pseudo-states, cascading rules).
- Multi-file changes and refactors that move widgets between layouts.
- New custom widgets and any `CustomStyle.cpp` extensions.
- Establishing and documenting the design tokens (colors, paddings,
  radii, font scale).

**What the user owns (final touches):**

- Minor spacing/padding adjustments that need a real eye on screen.
- Color value nudges where the monitor calibration matters.
- Trying alternative widget arrangements via Designer drag-and-drop.
- Anything where "make it pop more" is faster done than described.

## Visual direction — Refined native

**Chosen direction:** Modern Windows 11-ish polish. Native-feeling
control set, but tightened: subtle 4–6px rounding, refined typography
scale, better spacing, polished focus rings, hover and pressed states
that feel intentional. Dark mode improved (already partially supported
since `a57f063`).

**Why this and not "Distinct brand" or "Pro audio UI":**

- Lowest risk: doesn't fight Qt Widgets' native rendering, just
  refines it.
- Best ROI per hour: ~95% QSS-only, very little custom widget work.
- Respects the project's long-running native-Windows identity. The
  app should feel like a polished version of itself, not a different
  app.

**Design tokens to establish on the first form (deferred to the
implementation plan):**

- Color palette: neutrals (light + dark), accent, success, warning,
  disabled.
- Typography scale: 1–2 sizes for labels, 1 for headers, weights.
- Spacing scale: 4px grid (4 / 8 / 12 / 16 / 24).
- Corner radius: 4px for small controls, 6px for cards/groups.
- Focus ring: 2px accent outline with offset.
- Hover/pressed/disabled states: standardized opacity / fill shifts.

These tokens will live in a single shared `.qss` file under
`Editor/styles/` (new folder) and be loaded via `CustomStyle.cpp` at
startup.

## First-iteration target order

1. **`Editor/guis/BiQuadFilterGUI.ui`** — the style anchor. Small
   form, common controls (spinboxes, labels, dropdowns). Establish
   the design tokens here.
2. **`Editor/FilterTableRow.ui`** — repeated N times in the main
   window, so restyling it produces high visual impact.
3. **`Editor/MainWindow.ui`** — shell, toolbar, table integration.
4. **Remaining `Editor/guis/*.ui`** — sweep with the established
   tokens. Mostly mechanical.

DeviceSelector is **not** in this first iteration. It revisits later.

## Deferred work

These items are explicitly deferred but tracked so they don't get
lost:

- **Native deps install** (FFTW 3.3.10, libsndfile 1.2.2, muParserX
  3.0.1) to enable full Editor builds. Pick this up once a meaningful
  first-pass restyling is in place and we want to validate end-to-end
  in the running app. Estimated 30–45 min. Procedure documented in
  `docs/getting-started.md` and `docs/dependencies.md`.
- **`build.bat` Qt path update** — currently hardcodes `C:\Qt\6.7.2`;
  ours is `6.7.3`. Not blocking dev (we don't run `build.bat`), but
  worth a one-line fix when we touch packaging.
- **DeviceSelector restyling** — second iteration, after the Editor
  refresh.
- **QSS live-reload** — declined for now (user preference: Qt Designer
  preview is enough). Revisit if the Designer loop ever feels slow.

## Success criteria

The setup is considered complete when:

- User can open `Editor.pro` in Qt Creator without manual configuration.
- User can open any `.ui` file in Designer and trigger a preview with
  `Ctrl+Alt+R`.
- A first commit on `ui-modernization` lands a restyled
  `BiQuadFilterGUI.ui` + the initial `.qss` design-token sheet, and
  Designer preview shows the new look.
- Both parties agree on the ping-pong workflow and have done at least
  one full hand-off cycle.

## Risks

- **Designer preview ≠ runtime appearance.** Some QSS rules behave
  differently when `QApplication`'s palette and `CustomStyle` are
  active vs. in Designer's bare preview. Mitigation: do an end-to-end
  validation run once native deps are installed; expect small
  follow-up tweaks.
- **Qt 6.7.3 vs 6.7.2 source assumptions.** Patch-level differences
  could surface API or behavior nits. Mitigation: stay alert during
  the first build; deltas at this level are rare.
- **Edit conflicts on the shared working tree.** Both parties writing
  the same file simultaneously corrupts state. Mitigation: explicit
  ping-pong handoffs (see Collaboration workflow).
