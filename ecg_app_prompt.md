# ECG Educational Simulator — Coding Agent Prompt

## Project Overview

Build a fully self-contained, single-file **ECG Educational Simulator** as a plain HTML/CSS/JS application (no frameworks, no build step, no external dependencies except Google Fonts). This is a medical education tool designed for medical students and junior doctors to learn about ECG interpretation, physiological changes, and pathological conditions.

---

## Tech Stack & Constraints

- **Single file**: All HTML, CSS, and JavaScript in one `.html` file
- **No frameworks**: Pure vanilla JS only
- **No build step**: Must open directly in a browser
- **External allowed**: Google Fonts CDN only
- **Must work offline** after initial font load

---

## Visual Design

**Theme**: Clean medical white — textbook-professional aesthetic, like a premium clinical atlas.

- Background: `#FAFAF8` (warm off-white, not harsh clinical white)
- ECG paper: classic light pink/cream with standard red grid lines (1mm small squares, 5mm large squares), exactly like real ECG paper
- Primary accent: `#C0392B` (medical red) for highlights, active states, and the live rhythm strip trace
- Secondary accent: `#2C3E50` (deep navy) for text and static trace lines
- Font pairing: **"Libre Baskerville"** (serif) for headings and labels — evokes medical textbooks; **"Source Sans 3"** for UI controls and body text
- Sidebar: white with a subtle `1px` border and gentle box-shadow
- All controls should feel like high-quality medical instrument UI — precise, clean, no rounded-pill buttons
- Subtle ECG paper texture on the graph background (CSS repeating grid lines)

---

## Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  HEADER: App title + subtitle                                       │
├─────────────────┬───────────────────────────────────────────────────┤
│                 │  12-LEAD ECG GRID (3×4 layout: I,II,III /         │
│  LEFT SIDEBAR   │  aVR,aVL,aVF / V1,V2,V3 / V4,V5,V6)             │
│  (320px fixed)  │  Each lead in its own labeled cell with           │
│                 │  ECG paper grid background                         │
│  ─ Physiology   ├───────────────────────────────────────────────────┤
│    Variables    │  RHYTHM STRIP (Lead II, full width, animated,     │
│                 │  scrolling continuously left like a monitor)       │
│  ─ Condition    ├───────────────────────────────────────────────────┤
│    Selector     │  EDUCATION PANEL (collapsible, appears when a     │
│                 │  condition is selected — shows condition info)     │
└─────────────────┴───────────────────────────────────────────────────┘
```

---

## ECG Rendering

### Mathematical Model

Render all ECG waveforms using mathematically computed curves — **do not use static SVG images**. Use the Canvas API for all ECG drawing.

Each beat is a sequence of mathematically defined segments:

**Waveform Components (render in order per beat):**
1. **P wave** — small Gaussian bump: `A_p * exp(-((x - p_center)^2) / (2 * p_width^2))`
2. **PR segment** — flat isoelectric line
3. **Q wave** — small negative dip (Gaussian, negative amplitude)
4. **R wave** — tall sharp positive spike (Gaussian, large positive amplitude)
5. **S wave** — small negative dip after R (Gaussian, negative)
6. **ST segment** — flat line (elevation/depression applied here as a vertical offset)
7. **T wave** — broad rounded positive bump (Gaussian)
8. **QT interval** — determined by combined Q-start to T-end duration
9. **TP segment** — isoelectric baseline before next beat

**Parameters that drive the waveform (all user-adjustable):**

| Parameter | Normal Value | Range | Unit | ECG Effect |
|---|---|---|---|---|
| Heart Rate | 75 | 30–200 | bpm | Controls R-R interval (total beat length) |
| PR Interval | 160 | 80–400 | ms | Gap between P-wave start and Q-wave start |
| QRS Duration | 80 | 40–200 | ms | Width of the QRS complex |
| QT Interval | 400 | 200–600 | ms | Duration from Q to end of T |
| P Wave Amplitude | 0.15 | 0–0.4 | mV | Height of P wave |
| P Wave Duration | 80 | 40–160 | ms | Width of P wave |
| R Wave Amplitude | 1.0 | 0.1–3.0 | mV | Height of R wave (tallest spike) |
| S Wave Depth | 0.2 | 0–1.0 | mV | Depth of S wave below baseline |
| T Wave Amplitude | 0.3 | -0.5–1.0 | mV | Height/inversion of T wave |
| T Wave Symmetry | 0.5 | 0.1–0.9 | ratio | Peaked vs broad T wave shape |
| ST Deviation | 0.0 | -0.5–0.5 | mV | ST elevation (+) or depression (-) |
| ST Shape | normal | normal/upsloping/downsloping/saddle | — | Morphology of ST segment |
| QRS Axis | 60 | -90–270 | degrees | Affects relative amplitudes across leads |
| Delta Wave | 0 | 0–1 | bool/slider | Slurred initial QRS upstroke (WPW) |
| U Wave | 0 | 0–0.3 | mV | Post-T wave bump (hypokalemia) |
| P Wave Morphology | normal | normal/biphasic/absent/retrograde | — | Shape of P wave |

### Lead-Specific Rendering

Each of the 12 leads must display a **physiologically correct** variation of the waveform based on the **QRS axis** and standard lead vectors. Use simplified but medically accurate lead projections:

- **Limb leads** (I, II, III, aVR, aVL, aVF): Apply standard Einthoven/augmented lead projections relative to QRS axis
  - Lead II is the most upright in normal axis → tallest R wave
  - aVR is always negative (inverted P and QRS) in normal sinus rhythm
  - Lead I reflects lateral forces
- **Precordial leads** (V1–V6): Model R-wave progression
  - V1: mostly negative (QS or rS pattern)
  - V2–V3: transitional
  - V4: transition zone (RS pattern)
  - V5–V6: mostly positive (qR or R pattern)
  - Apply correct ST/T changes per lead grouping for conditions

The formula for each lead's amplitude multiplier:
- Use a projection vector table for each lead against the mean QRS axis angle
- Scale R-wave, S-wave, P-wave amplitudes by the lead's projection coefficient

### 12-Lead Grid

- Draw a **3-column × 4-row grid** (standard ECG layout):
  - Row 1: I, aVR, V1, V4
  - Row 2: II, aVL, V2, V5
  - Row 3: III, aVF, V3, V6
  - Row 4: Full-width Lead II rhythm strip (static snapshot)
- Each cell has a red ECG paper grid (0.5px minor lines every ~3px, 1px major lines every ~15px)
- Label each lead in top-left corner of cell
- Show 2.5 seconds of trace per cell (standard calibration)
- Calibration mark: 1mV = 10mm standard

### Rhythm Strip (Live Animated)

- Full-width canvas at the bottom
- Lead II trace scrolling continuously left (like a bedside monitor)
- Speed: 25mm/sec standard
- Dark red/crimson trace on light ECG paper background
- Smooth animation using `requestAnimationFrame`
- Show a moving cursor/beam at the right edge

---

## Left Sidebar

### Section 1: Physiological Variables

- Title: "Physiological Parameters"
- For each parameter listed in the table above, render:
  - Parameter name (bold label)
  - Current value display (right-aligned, with unit)
  - A styled range slider (`<input type="range">`)
  - Normal range shown below slider in small grey text
  - When value is outside normal range, highlight label in amber
  - When value is critically abnormal, highlight in red
- Group parameters into collapsible sub-sections:
  - **Rate & Rhythm** (Heart Rate, P Wave Morphology)
  - **Intervals & Durations** (PR, QRS, QT)
  - **Amplitudes & Morphology** (P amp, R amp, S depth, T amp, T symmetry, ST deviation, ST shape, Delta wave, U wave)
  - **Axis** (QRS Axis with a small circular axis diagram showing the current axis visually)
- A "Reset to Normal" button at the top of this section

### Section 2: Condition Selector

- Title: "Pathological Conditions"
- Organized in collapsible accordion groups:

**Cardiac Conditions:**
- Normal Sinus Rhythm (default)
- Sinus Tachycardia
- Sinus Bradycardia
- 1st Degree AV Block
- 2nd Degree AV Block (Mobitz I — Wenckebach)
- 2nd Degree AV Block (Mobitz II)
- Complete (3rd Degree) AV Block
- Left Bundle Branch Block (LBBB)
- Right Bundle Branch Block (RBBB)
- Left Anterior Fascicular Block (LAFB)
- Atrial Fibrillation
- Atrial Flutter
- Ventricular Tachycardia
- Left Ventricular Hypertrophy (LVH)
- Right Ventricular Hypertrophy (RVH)
- Anterior STEMI (V1–V4)
- Inferior STEMI (II, III, aVF)
- Lateral STEMI (I, aVL, V5–V6)
- Posterior MI
- NSTEMI / UA (ST depression pattern)
- Old Anterior MI (pathological Q waves)
- Pericarditis (diffuse saddle-shaped ST elevation)

**Metabolic / Electrolyte:**
- Hyperkalemia (mild → moderate → severe progression slider)
- Hypokalemia
- Hypercalcemia
- Hypocalcemia
- Hypomagnesemia
- Hypothermia (Osborn/J waves)

**Drug Effects:**
- Digoxin toxicity (reverse tick ST, bradycardia)
- Quinidine (prolonged QT, wide QRS)
- Tricyclic Antidepressant Overdose (wide QRS, R in aVR, right axis)
- Beta-blocker overdose (bradycardia, PR prolongation)
- Cocaine toxicity (vasospasm-like ST changes, tachycardia)
- Amiodarone (prolonged QT, U waves)

**Congenital / Structural:**
- Wolff-Parkinson-White (WPW) — Delta waves, short PR
- Brugada Syndrome (Type 1, 2, 3)
- Long QT Syndrome (congenital)
- Short QT Syndrome
- Hypertrophic Cardiomyopathy (HCM)
- Arrhythmogenic Right Ventricular Dysplasia (ARVD)
- Dextrocardia

**Each condition, when selected, must:**
1. Immediately update all parameter sliders to preset values matching that condition
2. Re-render the full 12-lead ECG and rhythm strip to reflect those values
3. Apply any lead-specific changes (e.g., ST elevation only in V1–V4 for anterior STEMI, reciprocal changes in inferior leads)
4. Show the Education Panel

---

## Education Panel

A collapsible panel that appears below the ECG grid when a condition is selected. Contains four tabs:

**Tab 1 – Overview**
- Condition name as heading
- 2–3 sentence plain-English description of the condition and its clinical significance

**Tab 2 – ECG Findings**
- Bulleted list of key ECG findings to look for, e.g.:
  - "Broad monophasic R wave in V5–V6"
  - "Absent septal Q waves in I and V6"
  - "RSR' pattern (M-shaped) in V1"

**Tab 3 – Pathophysiology**
- 3–5 sentence explanation of *why* the ECG looks this way — the underlying physiology/pathology causing the change (e.g., "In LBBB, the left bundle is blocked, forcing the left ventricle to depolarise late via slow myocyte-to-myocyte conduction rather than the fast Purkinje system. This delayed activation…")

**Tab 4 – Diagnostic Criteria & Pearls**
- Formal diagnostic criteria (where applicable, e.g., Sokolow-Lyon for LVH, Sgarbossa criteria for MI in LBBB)
- 2–3 clinical pearls or exam tips
- Differential diagnoses to consider

All educational text must be **medically accurate and appropriately detailed** for a medical student / junior doctor audience.

---

## Condition Presets — Parameter Values

Each condition must set exact parameter values. Key examples (implement all):

**Anterior STEMI (V1–V4):**
- ST deviation: +0.3 mV in V1–V4
- Reciprocal ST depression: -0.15 mV in II, III, aVF
- T waves: hyperacute (tall, broad) in V1–V4 initially; invert later
- May show pathological Q waves if "evolved" sub-type selected

**LBBB:**
- QRS Duration: 140ms
- R wave in V5–V6: tall, broad, monophasic (no S wave)
- QS or rS in V1
- Discordant ST-T changes (ST and T opposite to QRS direction)
- Absent septal Q in I, V5, V6

**RBBB:**
- QRS Duration: 140ms
- RSR' (M-pattern) in V1
- Wide slurred S in I, V5, V6

**Hyperkalemia (mild → severe):**
- Mild (K 5.5–6.5): Tall peaked T waves, shortened QT
- Moderate (K 6.5–7.5): Prolonged PR, widened QRS, flattened P
- Severe (K >7.5): Absent P waves, very wide QRS, sine wave pattern
- Implement as a sub-slider within the condition

**WPW:**
- Delta wave: enabled
- Short PR: 100ms
- Slightly widened QRS
- Pseudo ST changes from delta wave in some leads

**Atrial Fibrillation:**
- No P waves (P amplitude = 0, replaced by fibrillatory baseline noise)
- Irregularly irregular R-R intervals (use randomised beat timing)
- Otherwise normal QRS unless aberrant

**Atrial Flutter:**
- Sawtooth flutter waves at 300 bpm in inferior leads (II, III, aVF)
- Ventricular rate: typically 150 (2:1 block) — add AV ratio selector (2:1, 3:1, 4:1)

**VTach:**
- Wide complex tachycardia: QRS 160ms+, rate 150–200
- AV dissociation (P waves march through independently if visible)
- LBBB or RBBB morphology depending on origin

**Digoxin:**
- Reverse tick (down-sloping ST depression with upward curve, "Salvador Dalí moustache")
- Bradycardia
- Short QT
- PR prolongation

**Hypothermia:**
- Osborn (J) wave: positive deflection at J point (between QRS end and ST segment)
- Bradycardia
- PR/QRS/QT prolongation

**Brugada Type 1:**
- Coved ST elevation ≥2mm in V1–V2 with negative T wave
- Specific morphology (not simple ST elevation — must model the "coved" downsloping shape)

---

## Real-Time Interactivity

- All slider changes must update the ECG **immediately** (< 16ms, no debounce delay)
- Use `requestAnimationFrame` for smooth re-rendering
- When a slider is moved while a condition is selected, the condition label should change to show "(Modified)" to indicate it's a customised variant
- Smooth interpolation between old and new waveform over ~300ms when switching conditions (not abrupt jump)
- The live rhythm strip must continue scrolling uninterrupted during parameter changes

---

## Additional Features

1. **Calibration marker**: Show a 1mV square pulse at the start of each row in the 12-lead grid (standard ECG calibration)
2. **Paper speed indicator**: Display "25 mm/sec | 10 mm/mV" as standard calibration text on the ECG
3. **Measurements overlay** (toggle button): Click any ECG cell to show a magnified view with interval measurement labels (PR, QRS, QT, R-R annotated with brackets and values)
4. **Print-friendly layout**: CSS `@media print` styles that render the 12-lead grid cleanly as a printable ECG sheet
5. **Comparison mode** (stretch goal): Side-by-side comparison of two conditions

---

## Code Quality Requirements

- Well-commented code with section headers
- All condition presets stored as a clean JSON/object data structure (not scattered inline)
- All educational text stored as a separate data object (easy to update)
- The waveform generation must be a clean, reusable function: `generateBeat(params, lead)` → returns array of `{x, y}` points
- The 12-lead renderer calls `generateBeat` for each lead with the appropriate projection coefficient
- Canvas rendering should be efficient: clear and redraw only when parameters change, not on every animation frame (except the live strip)

---

## File Output

Deliver a single file: `ecg_simulator.html`

The file must open in any modern browser (Chrome, Firefox, Safari, Edge) without errors, without any server, and without any internet connection after the initial Google Fonts load.
