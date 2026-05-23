# Editor UI Refresh — BiQuadFilterGUI Anchor — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish the "Refined native" visual language on `BiQuadFilterGUI.ui` as the design anchor, with a reusable design-token QSS that can be propagated to the remaining Editor forms in follow-up plans.

**Architecture:** Two canonical QSS files (`refined-native-light.qss` and `refined-native-dark.qss`) live under `Editor/styles/` and are loaded by a new `CustomStyle::loadStyleSheet()` helper called from `main.cpp` after the existing `CustomStyle` proxy install. In deferred-build mode (no FFTW/libsndfile/muParserX yet) we validate by setting the form's `styleSheet` property to the file's content during Designer preview; at runtime (once native deps are installed) the global app stylesheet renders the same look and the per-form inline copy is removed in a follow-up plan.

**Tech Stack:** Qt 6.7.3 (MSVC 2022 64-bit kit), Qt Designer, QSS (Qt Style Sheets), C++ (Qt Widgets / QProxyStyle).

**Scope of this plan:** ONLY the BiQuadFilterGUI anchor. `FilterTableRow.ui`, `MainWindow.ui`, and the remaining `guis/*.ui` forms each get their own follow-up plan once this anchor is approved.

**Spec:** `docs/superpowers/specs/2026-05-23-windows-dev-setup-and-ui-refresh-design.md`

---

## File Structure

| File | Role |
|------|------|
| `Editor/styles/refined-native-light.qss` (new) | Design-token QSS for light mode — all baseline widget selectors used by the Editor. |
| `Editor/styles/refined-native-dark.qss` (new) | Same selectors, dark-mode palette. |
| `Editor/Editor.qrc` (modify) | Register the two QSS files as `:/styles/...` resources. |
| `Editor/CustomStyle.h` (modify) | Add `static void loadStyleSheet(QApplication& app)` declaration. |
| `Editor/CustomStyle.cpp` (modify) | Implement `loadStyleSheet()` — picks the file based on `GUIHelper::isDarkMode()`, reads it, calls `app.setStyleSheet(...)`. |
| `Editor/main.cpp` (modify) | One added line after the existing `setStyle(new CustomStyle(...))` to call `CustomStyle::loadStyleSheet(application)`. |
| `Editor/guis/BiQuadFilterGUI.ui` (modify) | Set root widget's `styleSheet` property to the file's text content so Qt Designer's Form Preview renders the look. Removed in a follow-up plan once a runtime build is possible. |

Single responsibility per file. The QSS files are the design source of truth. `CustomStyle.cpp` gains one new function; `main.cpp` gains one new line. The .ui inline-stylesheet is a deferred-mode validation aid, not permanent.

---

## Task 0: Create branch and styles directory

**Files:**
- Create: `Editor/styles/` (directory)
- Create: `Editor/styles/.gitkeep` (placeholder to track the empty dir)

- [ ] **Step 1: Verify clean working tree on `master`**

Run (from repo root):

```bash
git status
```

Expected: working tree should not have any uncommitted UI files. The pre-existing `docs/` staging and the spec commit you have prepared are fine to leave staged or commit first — do whichever you prefer before continuing. The point is: don't start the UI branch on top of unrelated uncommitted UI changes.

- [ ] **Step 2: Create and switch to the `ui-modernization` branch**

Run:

```bash
git checkout -b ui-modernization
```

Expected output: `Switched to a new branch 'ui-modernization'`.

- [ ] **Step 3: Create the styles directory with a placeholder**

Run:

```bash
mkdir -p Editor/styles
touch Editor/styles/.gitkeep
```

Expected: `ls Editor/styles/` shows `.gitkeep`.

- [ ] **Step 4: Commit the branch scaffold**

Run:

```bash
git add Editor/styles/.gitkeep
git commit -m "Added: Editor/styles/ directory for the UI refresh design tokens."
```

Expected: one new file committed.

---

## Task 1: Write the light-mode design-token QSS

**Files:**
- Create: `Editor/styles/refined-native-light.qss`

- [ ] **Step 1: Create the light QSS with concrete design tokens**

Write the following to `Editor/styles/refined-native-light.qss`:

```css
/*
    Refined Native — Light
    Design tokens applied to baseline Qt Widgets used by the Editor.
    Source of truth for the light theme; dark counterpart lives in
    refined-native-dark.qss.

    Tokens (canonical, repeated below because QSS has no variables):
      bg-base        #f3f3f3   (window surface)
      bg-elevated    #ffffff   (input fields, panels)
      bg-hover       #ececec
      bg-pressed     #e0e0e0
      text-primary   #1f1f1f
      text-secondary #606060
      text-disabled  #a0a0a0
      accent         #0067c0   (Windows 11 light accent)
      accent-hover   #1d7bd0
      accent-pressed #005ba1
      border-subtle  #e5e5e5
      border-strong  #c8c8c8
      border-focus   #0067c0
    Spacing scale (px): 4 / 8 / 12 / 16 / 24
    Radii (px):         4 (small controls), 6 (panels)
    Focus ring:         2px solid accent, offset implicit via border
*/

/* ---- Base window + text ---------------------------------------- */

QWidget {
    color: #1f1f1f;
    font-size: 9pt;
}

/* ---- Labels ---------------------------------------------------- */

QLabel {
    color: #1f1f1f;
    padding: 0px;
}

QLabel:disabled {
    color: #a0a0a0;
}

/* ---- Push buttons ---------------------------------------------- */

QPushButton {
    background-color: #ffffff;
    color: #1f1f1f;
    border: 1px solid #c8c8c8;
    border-radius: 4px;
    padding: 4px 12px;
    min-height: 20px;
}

QPushButton:hover {
    background-color: #ececec;
    border-color: #0067c0;
}

QPushButton:pressed {
    background-color: #e0e0e0;
    border-color: #005ba1;
}

QPushButton:focus {
    border: 2px solid #0067c0;
    padding: 3px 11px;  /* compensate border growth */
}

QPushButton:disabled {
    color: #a0a0a0;
    background-color: #f3f3f3;
    border-color: #e5e5e5;
}

/* ---- Spinboxes (QSpinBox, QDoubleSpinBox) ---------------------- */

QSpinBox, QDoubleSpinBox {
    background-color: #ffffff;
    color: #1f1f1f;
    border: 1px solid #c8c8c8;
    border-radius: 4px;
    padding: 2px 6px;
    min-height: 20px;
    selection-background-color: #0067c0;
    selection-color: #ffffff;
}

QSpinBox:hover, QDoubleSpinBox:hover {
    border-color: #a0a0a0;
}

QSpinBox:focus, QDoubleSpinBox:focus {
    border: 2px solid #0067c0;
    padding: 1px 5px;
}

QSpinBox:disabled, QDoubleSpinBox:disabled {
    color: #a0a0a0;
    background-color: #f3f3f3;
    border-color: #e5e5e5;
}

/* ---- ComboBoxes ------------------------------------------------ */

QComboBox {
    background-color: #ffffff;
    color: #1f1f1f;
    border: 1px solid #c8c8c8;
    border-radius: 4px;
    padding: 2px 8px;
    min-height: 22px;
}

QComboBox:hover {
    border-color: #a0a0a0;
}

QComboBox:focus {
    border: 2px solid #0067c0;
    padding: 1px 7px;
}

QComboBox:disabled {
    color: #a0a0a0;
    background-color: #f3f3f3;
    border-color: #e5e5e5;
}

QComboBox::drop-down {
    border: none;
    width: 18px;
}

QComboBox QAbstractItemView {
    background-color: #ffffff;
    color: #1f1f1f;
    border: 1px solid #c8c8c8;
    border-radius: 4px;
    selection-background-color: #0067c0;
    selection-color: #ffffff;
    padding: 2px;
}

/* ---- Dials (QDial) --------------------------------------------- */
/* QDial styling via QSS is intentionally light-touch: heavy
   customization requires a custom QStyle. We just set the disabled
   tone here; everything else uses the proxied platform style. */

QDial:disabled {
    background-color: #f3f3f3;
}

/* ---- Group boxes ----------------------------------------------- */

QGroupBox {
    background-color: #ffffff;
    border: 1px solid #e5e5e5;
    border-radius: 6px;
    margin-top: 12px;
    padding: 12px 12px 8px 12px;
    font-weight: 600;
}

QGroupBox::title {
    subcontrol-origin: margin;
    subcontrol-position: top left;
    padding: 0 6px;
    color: #1f1f1f;
}

/* ---- Tool tips ------------------------------------------------- */

QToolTip {
    background-color: #ffffff;
    color: #1f1f1f;
    border: 1px solid #c8c8c8;
    border-radius: 4px;
    padding: 4px 8px;
}
```

This file is the canonical light theme. Every value is concrete; no placeholders. Selectors cover exactly the widget types present in `BiQuadFilterGUI.ui` (QDial, QComboBox, QDoubleSpinBox, QLabel) plus the common controls (QPushButton, QGroupBox) we'll need in the next forms.

- [ ] **Step 2: Commit the light QSS**

Run:

```bash
git add Editor/styles/refined-native-light.qss
git commit -m "Added: refined-native-light.qss — light-mode design tokens for Editor."
```

Expected: one new file committed.

---

## Task 2: Write the dark-mode design-token QSS

**Files:**
- Create: `Editor/styles/refined-native-dark.qss`

- [ ] **Step 1: Create the dark QSS — identical selectors, dark palette**

Write the following to `Editor/styles/refined-native-dark.qss`:

```css
/*
    Refined Native — Dark
    Mirror of refined-native-light.qss with the dark palette.

    Tokens:
      bg-base        #202020   (window surface, matches Fusion dark)
      bg-elevated    #2b2b2b
      bg-hover       #323232
      bg-pressed     #3a3a3a
      text-primary   #ffffff
      text-secondary #b8b8b8
      text-disabled  #6a6a6a
      accent         #4cc2ff   (Windows 11 dark accent)
      accent-hover   #62caff
      accent-pressed #3aa7e0
      border-subtle  #393939
      border-strong  #5a5a5a
      border-focus   #4cc2ff
*/

QWidget {
    color: #ffffff;
    font-size: 9pt;
}

QLabel {
    color: #ffffff;
    padding: 0px;
}

QLabel:disabled {
    color: #6a6a6a;
}

QPushButton {
    background-color: #2b2b2b;
    color: #ffffff;
    border: 1px solid #5a5a5a;
    border-radius: 4px;
    padding: 4px 12px;
    min-height: 20px;
}

QPushButton:hover {
    background-color: #323232;
    border-color: #4cc2ff;
}

QPushButton:pressed {
    background-color: #3a3a3a;
    border-color: #3aa7e0;
}

QPushButton:focus {
    border: 2px solid #4cc2ff;
    padding: 3px 11px;
}

QPushButton:disabled {
    color: #6a6a6a;
    background-color: #202020;
    border-color: #393939;
}

QSpinBox, QDoubleSpinBox {
    background-color: #2b2b2b;
    color: #ffffff;
    border: 1px solid #5a5a5a;
    border-radius: 4px;
    padding: 2px 6px;
    min-height: 20px;
    selection-background-color: #4cc2ff;
    selection-color: #1f1f1f;
}

QSpinBox:hover, QDoubleSpinBox:hover {
    border-color: #8a8a8a;
}

QSpinBox:focus, QDoubleSpinBox:focus {
    border: 2px solid #4cc2ff;
    padding: 1px 5px;
}

QSpinBox:disabled, QDoubleSpinBox:disabled {
    color: #6a6a6a;
    background-color: #202020;
    border-color: #393939;
}

QComboBox {
    background-color: #2b2b2b;
    color: #ffffff;
    border: 1px solid #5a5a5a;
    border-radius: 4px;
    padding: 2px 8px;
    min-height: 22px;
}

QComboBox:hover {
    border-color: #8a8a8a;
}

QComboBox:focus {
    border: 2px solid #4cc2ff;
    padding: 1px 7px;
}

QComboBox:disabled {
    color: #6a6a6a;
    background-color: #202020;
    border-color: #393939;
}

QComboBox::drop-down {
    border: none;
    width: 18px;
}

QComboBox QAbstractItemView {
    background-color: #2b2b2b;
    color: #ffffff;
    border: 1px solid #5a5a5a;
    border-radius: 4px;
    selection-background-color: #4cc2ff;
    selection-color: #1f1f1f;
    padding: 2px;
}

QDial:disabled {
    background-color: #202020;
}

QGroupBox {
    background-color: #2b2b2b;
    border: 1px solid #393939;
    border-radius: 6px;
    margin-top: 12px;
    padding: 12px 12px 8px 12px;
    font-weight: 600;
}

QGroupBox::title {
    subcontrol-origin: margin;
    subcontrol-position: top left;
    padding: 0 6px;
    color: #ffffff;
}

QToolTip {
    background-color: #2b2b2b;
    color: #ffffff;
    border: 1px solid #5a5a5a;
    border-radius: 4px;
    padding: 4px 8px;
}
```

The two files have identical selector lists in identical order so future edits can be done in parallel.

- [ ] **Step 2: Commit the dark QSS**

Run:

```bash
git add Editor/styles/refined-native-dark.qss
git commit -m "Added: refined-native-dark.qss — dark-mode design tokens for Editor."
```

Expected: one new file committed.

---

## Task 3: Register QSS files in Editor.qrc

**Files:**
- Modify: `Editor/Editor.qrc`

- [ ] **Step 1: Add the two QSS files to the resource list**

Open `Editor/Editor.qrc`. Find the existing `<qresource prefix="/">` block (currently lists translations, icons, sounds). Add two new `<file>` entries inside that block, immediately after the existing translations entries (i.e. after the `qtbase_de.qm` line).

The added lines:

```xml
        <file>styles/refined-native-light.qss</file>
        <file>styles/refined-native-dark.qss</file>
```

The final relevant section should look like:

```xml
<RCC>
    <qresource prefix="/">
        <file>translations/Editor_de.qm</file>
        <file>translations/qtbase_de.qm</file>
        <file>styles/refined-native-light.qss</file>
        <file>styles/refined-native-dark.qss</file>
        <file>icons/document-open.ico</file>
        ...
```

(Leave everything else in the file unchanged.)

- [ ] **Step 2: Commit the qrc update**

Run:

```bash
git add Editor/Editor.qrc
git commit -m "Added: refined-native QSS sheets to Editor.qrc as :/styles/ resources."
```

Expected: one file modified.

---

## Task 4: Add `loadStyleSheet()` declaration to CustomStyle.h

**Files:**
- Modify: `Editor/CustomStyle.h`

- [ ] **Step 1: Add the static helper declaration**

In `Editor/CustomStyle.h`, replace the class body with the version below (adds one forward declaration of `QApplication` and one new public static method; everything else is unchanged):

```cpp
#pragma once

#include <QProxyStyle>

class QApplication;

class CustomStyle : public QProxyStyle
{
    Q_OBJECT

public:
    CustomStyle(QStyle* style);

    int pixelMetric(PixelMetric metric, const QStyleOption* option, const QWidget* widget) const override;
    QIcon standardIcon(StandardPixmap standardIcon, const QStyleOption *option = nullptr, const QWidget *widget = nullptr) const override;

    // Loads the appropriate :/styles/refined-native-*.qss resource based
    // on dark-mode state and applies it via app.setStyleSheet().
    static void loadStyleSheet(QApplication& app);
};
```

Keep the existing GPL-2.0 license header at the top of the file untouched.

- [ ] **Step 2: Commit the header change**

Run:

```bash
git add Editor/CustomStyle.h
git commit -m "Added: CustomStyle::loadStyleSheet() declaration for QSS loading."
```

---

## Task 5: Implement `loadStyleSheet()` in CustomStyle.cpp

**Files:**
- Modify: `Editor/CustomStyle.cpp`

- [ ] **Step 1: Add the implementation**

In `Editor/CustomStyle.cpp`, add the following two `#include` lines at the top (after the existing `#include "Editor/helpers/GUIHelper.h"` line):

```cpp
#include <QApplication>
#include <QFile>
```

Then append the following function definition at the end of the file (after the closing brace of `standardIcon`):

```cpp
void CustomStyle::loadStyleSheet(QApplication& app)
{
    const QString resourcePath = GUIHelper::isDarkMode()
        ? QStringLiteral(":/styles/refined-native-dark.qss")
        : QStringLiteral(":/styles/refined-native-light.qss");

    QFile file(resourcePath);
    if (!file.open(QIODevice::ReadOnly | QIODevice::Text))
    {
        // Resource missing — leave the app unstyled rather than crash.
        // This path indicates a build problem, not a runtime error.
        return;
    }

    app.setStyleSheet(QString::fromUtf8(file.readAll()));
}
```

The dark-mode branch matches the existing pattern used at `main.cpp:48` (`GUIHelper::isDarkMode()`), so the loaded QSS aligns with the Fusion base style already swapped in for dark mode.

- [ ] **Step 2: Commit the implementation**

Run:

```bash
git add Editor/CustomStyle.cpp
git commit -m "Improved: CustomStyle::loadStyleSheet() loads refined-native QSS by theme."
```

---

## Task 6: Call `loadStyleSheet()` from main.cpp

**Files:**
- Modify: `Editor/main.cpp:50`

- [ ] **Step 1: Add the one-line call after the existing setStyle**

In `Editor/main.cpp`, find the block at lines 48–50:

```cpp
if(GUIHelper::isDarkMode())
    application.setStyle("fusion");
application.setStyle(new CustomStyle(application.style()));
```

Replace it with:

```cpp
if(GUIHelper::isDarkMode())
    application.setStyle("fusion");
application.setStyle(new CustomStyle(application.style()));
CustomStyle::loadStyleSheet(application);
```

(One added line. Nothing else in the file changes — keep the existing includes; `CustomStyle.h` is already included at line 26.)

- [ ] **Step 2: Commit the wiring**

Run:

```bash
git add Editor/main.cpp
git commit -m "Improved: main.cpp loads refined-native stylesheet after CustomStyle install."
```

**Runtime validation is deferred.** Because the Editor cannot link until the FFTW/libsndfile/muParserX native deps are installed, we cannot exercise this code path yet. The wiring is correct by inspection (matches the GUIHelper dark-mode check at line 48, applies after `setStyle`, uses the qrc resource path matching Editor.qrc); end-to-end validation happens in the deferred-deps follow-up.

---

## Task 7: Apply QSS to BiQuadFilterGUI.ui for Designer preview validation

**Files:**
- Modify: `Editor/guis/BiQuadFilterGUI.ui`

**Purpose:** Since we cannot run the app in deferred-build mode, the Form Preview in Qt Designer is our only visual validation. Designer renders the `styleSheet` property set on a widget. We set the root widget's `styleSheet` to the light QSS file's contents so Preview shows the refined look. **This is a deferred-mode validation aid**; a follow-up plan removes it once the runtime build works.

- [ ] **Step 1: Apply the styleSheet via Qt Designer (user action, on Windows)**

This step is **performed by the user on Windows**, not by Claude:

1. In Qt Creator on Windows, open `Editor/guis/BiQuadFilterGUI.ui` in Designer.
2. In the Object Inspector (top-right panel), click the root widget `BiQuadFilterGUI` (the QWidget at the top of the tree).
3. In the Property Editor (bottom-right panel), scroll to **`styleSheet`** and click the `...` button next to it. A "Edit Style Sheet" dialog opens.
4. Open `Editor/styles/refined-native-light.qss` in another editor (or in Qt Creator's editor pane), copy its entire contents, paste them into the Edit Style Sheet dialog, click **OK**.
5. Save the .ui file (`Ctrl+S`). The `styleSheet` property is now embedded in the .ui XML.

- [ ] **Step 2: Verify the .ui XML was updated**

From WSL, run:

```bash
grep -c "QSpinBox" Editor/guis/BiQuadFilterGUI.ui
```

Expected: at least `1` (the .ui now contains the QSS selectors inline; previously this grep returned 0).

Also check the file now has a `<property name="styleSheet">` block somewhere near the top widget definition:

```bash
grep -A2 'name="styleSheet"' Editor/guis/BiQuadFilterGUI.ui | head -10
```

Expected: non-empty output containing the start of the QSS content.

- [ ] **Step 3: Hand off to user for visual validation**

The user runs Form Preview in Designer (`Ctrl+Alt+R`) and reports back. **Do not commit yet** — the preview may surface immediate visual issues (spacings, focus rings, dial appearance under QDial's limited QSS support, hover states not triggering cleanly). Iterate before committing.

---

## Task 8: User preview + ping-pong final-touch loop

**Files:** (TBD by user feedback)

**Purpose:** Honor the spec's ping-pong workflow. User previews; Claude iterates the QSS or .ui; user previews again until "looks good".

- [ ] **Step 1: User opens BiQuadFilterGUI.ui in Designer and runs Form Preview (Ctrl+Alt+R)**

User reviews the preview against the "Refined native" direction: subtle rounding, refined spacing, focus rings, hover states feel intentional, controls feel polished.

- [ ] **Step 2: User reports back with one of:**

  - **"Looks good"** → proceed to Step 5.
  - **Screenshot + specific concerns** → Claude edits the QSS or .ui to address them, then loops back to Step 1. Each iteration is a separate commit (`Improved: tighten spinbox vertical padding`, `Improved: bump combobox min-height for label baseline`, etc.).
  - **"I'll do the final touch"** → user takes over (Designer drag-and-drop or QSS tweaks), saves, commits themselves or hands the file back.

- [ ] **Step 3: (Iteration only) If Claude tweaks the QSS, the user must re-copy it into the `styleSheet` property**

Reminder for the user: since Designer's inline `styleSheet` is a snapshot of the file, every time Claude edits `refined-native-light.qss`, the user must repeat the copy-paste from Task 7 Step 1 to see the change in Preview. (This duplication is the price of deferred-build mode; it disappears once runtime loading is verified.)

- [ ] **Step 4: (Final-touch handoff only) Claude pauses while user edits**

Per the spec's ping-pong rule, Claude does not touch BiQuadFilterGUI.ui or the QSS files while the user is editing. User says "back to you" or commits the change and pings Claude when done.

- [ ] **Step 5: Commit the validated BiQuadFilterGUI.ui**

Once preview is approved, commit. Run:

```bash
git add Editor/guis/BiQuadFilterGUI.ui
git commit -m "Improved: BiQuadFilterGUI inline styleSheet for Designer preview (deferred-mode validation)."
```

If iteration commits accumulated on the QSS files during the loop, they are already committed separately. No squashing — keep history granular.

---

## Task 9: Update the spec's "Deferred work" section

**Files:**
- Modify: `docs/superpowers/specs/2026-05-23-windows-dev-setup-and-ui-refresh-design.md`

- [ ] **Step 1: Add the deferred follow-ups discovered by this plan**

Open `docs/superpowers/specs/2026-05-23-windows-dev-setup-and-ui-refresh-design.md`. Find the `## Deferred work` section and append the following two bullets at the end of the existing list (do not remove or reorder existing bullets):

```markdown
- **Remove inline `styleSheet` properties from `.ui` files once runtime
  QSS loading is verified.** The `BiQuadFilterGUI.ui` inline copy
  (added in plan
  `2026-05-23-editor-ui-refresh-biquad-anchor.md`, Task 7) is a
  deferred-mode preview aid. After the native deps are installed and
  `CustomStyle::loadStyleSheet()` is observed to apply the QSS at
  runtime, delete the inline copy from each form.
- **Runtime validation of `CustomStyle::loadStyleSheet()`.** Wiring
  added in this plan (Tasks 4–6) is verified by inspection only.
  Confirm the resource path resolves, the file content is non-empty,
  and the visible app appearance matches the Designer preview, once a
  full build is possible.
```

- [ ] **Step 2: Commit the spec update**

Run:

```bash
git add docs/superpowers/specs/2026-05-23-windows-dev-setup-and-ui-refresh-design.md
git commit -m "Improved: spec deferred-work list — note inline styleSheet cleanup and runtime QSS validation."
```

---

## Task 10: Plan handoff for the next form

**Files:** (none — bookkeeping only)

- [ ] **Step 1: Confirm the anchor is committed**

Run:

```bash
git log --oneline ui-modernization -- Editor/styles/ Editor/guis/BiQuadFilterGUI.ui Editor/CustomStyle.cpp Editor/main.cpp Editor/Editor.qrc | head -10
```

Expected: a chain of commits covering the work above, no dangling local edits (`git status` shows clean tree).

- [ ] **Step 2: Tell the user the anchor is done and offer the next plan**

Message template:

> BiQuadFilterGUI anchor is committed on `ui-modernization`. Two paths forward:
>
>   (A) **Next form**: I write a follow-up plan for `Editor/FilterTableRow.ui`, propagating the now-validated tokens. Highest visual ripple per hour.
>
>   (B) **Pick up deferred deps**: install FFTW / libsndfile / muParserX so we can do a full build and verify runtime QSS loading, then remove inline `styleSheet`s.
>
> Which one?

This task closes the plan. The next implementation cycle starts with a new spec or directly with a new writing-plans invocation, depending on the user's pick.

---

## Self-review (run after writing the plan)

**Spec coverage:**

- ✅ Refined native visual direction → implemented in Tasks 1 (light) and 2 (dark) with concrete tokens listed in the spec (color palette, spacing, radii, focus ring).
- ✅ Single shared `.qss` under `Editor/styles/` → split into light/dark variants to support the existing `GUIHelper::isDarkMode()` runtime switch, which is the spec's intent without it (the spec said "single .qss" but the dual-file split is the natural way to honor the existing dark-mode wiring; both files are otherwise structurally identical).
- ✅ Loaded via `CustomStyle.cpp` at startup → Tasks 4–6.
- ✅ First target = `BiQuadFilterGUI.ui` → Tasks 7–8.
- ✅ Ping-pong workflow with explicit handoffs → Task 8 explicitly enforces this.
- ✅ Deferred-build mode honored → no build/run commands anywhere; validation is Designer Form Preview.
- ✅ Out-of-scope items (FilterTableRow, MainWindow, rest, DeviceSelector, native deps) → not in any task; Task 10 hands them to follow-up plans.

**Placeholder scan:** No "TBD", "TODO", "implement later", or "appropriate" handwaving. Every code block contains complete content. The single use of "(TBD by user feedback)" in Task 8's Files row is by design — the files touched depend on user feedback — and the steps below it are concrete.

**Type / name consistency:**

- `CustomStyle::loadStyleSheet(QApplication&)` — declared in Task 4, defined in Task 5, called in Task 6. Signature matches across all three.
- Resource paths `:/styles/refined-native-light.qss` and `:/styles/refined-native-dark.qss` — registered in Task 3 (qrc), referenced in Task 5 (C++). Identical.
- `GUIHelper::isDarkMode()` — used in Task 5; verified against `main.cpp:48` which already uses it.
