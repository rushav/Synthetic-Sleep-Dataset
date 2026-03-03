# Poster Creation Guide
## Synthetic Sleep Environment Dataset Generator
### TECHIN 513 — Team 7: Rushav Dash & Lisa Li

---

## Overview

This guide walks through every design decision for our conference-style poster.
Follow it to produce a poster that earns full marks on all seven rubric criteria
(Relevance, Clarity, Feasibility, Creativity, Soundness, Comprehensiveness, Peer Eval).

---

## Recommended Layout (Portrait, 36" × 48")

```
┌─────────────────────────────────────────────────────────┐
│  [HEADER] Title │ Authors │ UW Logo │ TECHIN 513        │
├──────────────┬──────────────┬──────────────────────────┤
│  1. PROBLEM  │  2. PIPELINE │  3. SIGNAL PROCESSING    │
│   & MOTIVATION│  DIAGRAM    │     (Spectral Plots)     │
├──────────────┼──────────────┼──────────────────────────┤
│  4. ML MODEL │  5. RESULTS  │  6. VALIDATION           │
│   (RF + feats)│  & ABLATION │     (Tier 1/2/3)         │
├──────────────┴──────────────┴──────────────────────────┤
│  7. CONCLUSION, LIMITATIONS, FUTURE WORK │ QR / GitHub  │
└─────────────────────────────────────────────────────────┘
```

Use a **two-column or three-column** layout with clear section headers.
Maximum body font size: **20–22 pt**. Section headers: **28–32 pt bold**.
Title: **40 pt bold**.

---

## Section-by-Section Guide

---

### HEADER

**Content:**
- Full project title (can be shortened to: *Synthetic Sleep Environment Dataset Generator*)
- Author names: Rushav Dash, Lisa Li
- University of Washington wordmark / lockup
- Course: TECHIN 513 — Signal Processing & Machine Learning, Team 7
- GitHub/Kaggle URL as a QR code (bottom-right corner)

**Key talking point:** "We introduce a publicly available synthetic dataset linking bedroom environmental sensor signals to sleep quality metrics — a resource that previously did not exist."

---

### Section 1: Problem & Motivation

**What to include:**
- 2–3 sentence problem statement: *No public dataset links bedroom environment to polysomnographic sleep quality.*
- A simple two-column graphic showing "what exists" vs. "what we need":
  - Left: Sleep datasets (physiological only) | IoT datasets (environmental only)
  - Right: Our dataset (both, linked)
- One compelling statistic: e.g., "Sleep disorders affect 1 in 3 adults — yet no open dataset links bedroom conditions to sleep outcomes."

**Visual suggestion:** A Venn diagram with two non-overlapping circles ("Sleep datasets" / "IoT datasets") and a third overlapping circle labeled "Our dataset."

**Rubric criterion addressed:** Relevance, Creativity.

**Key talking point:** "Smart home devices now collect bedroom data 24/7, but researchers cannot use them to train sleep quality models because there is no labeled dataset. We built the bridge."

---

### Section 2: System Pipeline Diagram

**This is the most important figure on the poster.** It should be large (roughly half of a column) and clearly readable from 3 feet away.

**What to show (as a flow chart):**

```
 ┌───────────────────────────────────────────────────────┐
 │  INPUTS (Real Kaggle Datasets)                        │
 │  Sleep Efficiency (452 records) │ Room Occupancy IoT  │
 │  Smart Home + Weather (HVAC)                          │
 └───────────────────┬───────────────────────────────────┘
                     │  DataLoader: calibration stats
                     ▼
 ┌───────────────────────────────────────────────────────┐
 │  SIGNAL GENERATOR (Signal Processing)                 │
 │  Temperature │ Light │ Sound │ Humidity                │
 │  96 samples × 5 min = 8 h per session                 │
 └───────────────────┬───────────────────────────────────┘
                     │  FeatureExtractor: 30 scalars
                     ▼
 ┌───────────────────────────────────────────────────────┐
 │  SLEEP QUALITY MODEL (Machine Learning)               │
 │  Random Forest × 4 targets → labels                  │
 └───────────────────┬───────────────────────────────────┘
                     │  5,000 sessions, stratified
                     ▼
 ┌───────────────────────────────────────────────────────┐
 │  SYNTHETIC DATASET + 3-TIER VALIDATION                │
 │  KS-test │ ML cross-eval │ Sleep science checks       │
 └───────────────────────────────────────────────────────┘
```

**Rubric criterion addressed:** Clarity, Soundness.

---

### Section 3: Signal Processing Details

**Include two side-by-side plots:**

**Plot A — Temperature signal (one example 8-hour night):**
- X-axis: Time (hours, 0–8)
- Y-axis: Temperature (°C)
- Three colored layers stacked: circadian drift (blue), HVAC sawtooth (orange), pink noise (gray)
- Final filtered signal in black (Butterworth LPF applied)
- Annotate: "Butterworth LPF, order 4, fc = 1/30 min⁻¹"

**Plot B — Light signal (one example night):**
- Show the near-zero background (~3 lux) with 3–4 discrete event spikes
- Annotate: "Poisson process, λ = 2–3 events/night"
- Show Gaussian-smoothed edges

**Key equations to display (large, readable):**

```
T(t) = T_base + A·sin(2πt/480) + T_hvac(t) + T_noise(t)
                                              → Butterworth LPF

L(t) = L_bg(t) + Σ events(t)    [Poisson process, λ adjusted by age/sensitivity]
```

**Signal processing talking points:**
- "We chose pink (1/f) noise because natural processes — temperature drift, wind — follow power-law spectra, not white noise."
- "The Butterworth filter is maximally flat in the passband, so we don't distort the biologically meaningful slow circadian oscillation."
- "Zero-phase filtering (sosfiltfilt) ensures the output is time-aligned — essential when features like rate-of-change are derived from the signal."
- "Poisson processes are the canonical model for rare, memoryless events — exactly what light switches and bathroom trips look like statistically."

**Rubric criterion addressed:** Soundness (SP techniques correct and justified).

---

### Section 4: Machine Learning Model

**Include:**

**Feature Importance Bar Chart (top 8 features):**
- Horizontal bars showing mean decrease in impurity from the RF
- Expected top features: `arousal_index`, `fragmentation_proxy`, `age_numeric`, `fitness_proxy`, `temp_optimal_fraction`, `light_disruption_score`, etc.
- Label bars clearly with feature names

**Small table: CV performance:**

| Target | CV RMSE | CV R² |
|--------|---------|-------|
| Sleep efficiency | 0.075 | 0.45 |
| Awakenings | 1.1 | 0.31 |
| REM % | 7.2 | 0.18 |
| Deep % | 5.4 | 0.22 |

**Why RF talking points:**
- "Random Forest handles the mixed feature types and limited dataset size (n=452) better than linear models."
- "OOB scoring gives us unbiased error estimates without sacrificing data for a held-out split."
- "We train one RF per target variable to allow independent prediction, then renormalize sleep stages to sum to 100%."

**Rubric criterion addressed:** Comprehensiveness (ML justified), Soundness.

---

### Section 5: Results & Ablation Study

**Primary results table:**

| Variable | Synthetic Mean | Real Mean | Match? |
|----------|---------------|-----------|--------|
| Sleep efficiency | 0.79 | 0.79 | ✓ |
| Awakenings | 2.1 | 1.8 | ✓ (~) |
| REM % | 23.5 | 22.0 | ✓ |
| Temp mean | 21.1°C | — | calibrated |
| Light events/night | 2.1 | — | calibrated |

**Ablation results bar chart (Tier 3 pass rate and temperature ACF):**

```
Full pipeline         ████████████ ACF=0.97  [4/6 checks pass]
No Butterworth LPF    ████████     ACF=0.62  [4/6 checks pass]
No Poisson events     ██████       ACF=0.97  [3/6 checks pass]
No seasonal strat.    ██████       ACF=0.97  [3/6 checks pass]
```

**Key talking point for ablation:**
- "Removing the Butterworth filter drops the temperature autocorrelation from 0.97 to 0.62 — the signal becomes too 'jittery' to model realistic thermal inertia."
- "Removing Poisson events eliminates the variability in the light disruption score, making the dataset less useful for modeling light-sleep relationships."

**Rubric criterion addressed:** Comprehensiveness (ablation + baseline), Soundness.

---

### Section 6: Validation Framework (Three-Tier)

**Show a three-row table or three small panels:**

**Tier 1 — Statistical (KS-tests):**
```
KS-test: temperature    D=0.43, p≈0  → FAIL (expected: office ≠ bedroom)
KS-test: light          D=0.61, p≈0  → FAIL (expected: office ≠ bedroom)
KS-test: sleep efficiency D=0.12, p≈0 → FAIL (narrower synthetic range)
```
Note below: "Tier 1 failures are expected given the domain gap between
calibration dataset (office IoT) and our target (bedroom). KS-test has
near-perfect power at n=5,000."

**Tier 2 — ML Predictability:**
```
Real-data baseline RMSE: 0.075
Synthetic in-sample RMSE: 0.061  ✓ within 20% threshold
```

**Tier 3 — Sleep Science:**
Circular PASS/FAIL badges for each of the 6 checks (green = PASS, red = FAIL).

**Rubric criterion addressed:** Comprehensiveness, Soundness.

---

### Section 7: Conclusion, Limitations & Future Work

**Bullet structure (3 columns):**

**We demonstrated:**
- SP pipeline that encodes thermal inertia (LPF), event-driven disruptions (Poisson), and seasonal diversity
- RF label model that transfers sleep relationships from real data to synthetic sessions
- 5,000-session dataset with demographic and seasonal stratification
- 4/6 sleep science sanity checks pass

**Limitations:**
- Calibration IoT data is from an office, not a bedroom
- Tier 2 validation is in-sample (optimistic)
- Independent per-target RFs don't preserve inter-label biological correlations
- Labels are RF predictions, not true measurements

**Future work:**
- Multioutput RF to enforce inter-label correlations
- Bedroom-specific IoT calibration dataset
- Add physiological outputs (heart rate, movement)
- GAN-based generation conditioned on real polysomnography

**QR code:** Link to GitHub repository and Kaggle dataset.

---

## Design Tips

### Color palette
- Use a **sleep-themed** color scheme: deep blue (#1A237E), dark teal (#00695C), off-white background (#FAFAFA)
- Avoid bright reds/greens — use muted PASS/FAIL indicators
- Consistent color coding: temperature = warm orange, light = yellow, sound = purple, humidity = blue

### Typography
- Body text: minimum **20 pt** (readable from 3 feet)
- Section headers: **28–32 pt bold**
- Equations: display in a slightly larger, different background block
- Avoid justified text in narrow columns — use left-aligned

### Figure quality
- All plots: at minimum **300 DPI** for print
- Use vector graphics (PDF/SVG) exported from matplotlib if possible
- Every figure must have a caption and axis labels with units

### Layout flow
- Readers scan from top-left to bottom-right, then down columns
- Put the pipeline diagram in the first or second panel (highest visual impact)
- Group related sections with a light background shading or thin border

### Common mistakes to avoid
- Do NOT put raw code on the poster
- Do NOT use tiny tables with 10+ columns
- Do NOT omit units from axis labels
- Do NOT forget to label your figure numbers even if informal

---

## Rubric Compliance Checklist

| Criterion | Evidence on Poster |
|-----------|-------------------|
| **Relevance (SP + ML)** | Pipeline diagram shows SP → ML integration; equations shown |
| **Clarity** | Problem statement in Section 1; pipeline diagram labeled |
| **Feasibility** | Results table shows working system; dataset generated |
| **Creativity** | Novel dataset filling publicly noted gap; multi-tier validation |
| **Soundness** | SP techniques named, justified (Butterworth, Poisson, pink noise); RF justified |
| **Comprehensiveness** | Baseline comparison (constant + linear); ablation study (4 conditions); 3-tier validation |
| **Peer Eval ready** | Contribution statements per team member visible in header or conclusion |

---

## Poster Presentation Talking Points (60-second pitch)

1. **Hook:** "Have you ever wondered how bedroom temperature affects your sleep? There's no public dataset linking those two things. We built one."
2. **Method:** "We use signal processing — Butterworth filters, Poisson processes, spectral synthesis — to generate realistic bedroom sensor data, then a Random Forest trained on real sleep studies to assign sleep quality labels."
3. **Result:** "We generated 5,000 fully labeled sleep sessions across four seasons, three age groups, and three sensitivity levels — all from a reproducible Python pipeline."
4. **Validation:** "We validate using KS-tests against real IoT data, a linear model benchmark, and sleep science literature checks."
5. **Invitation:** "The dataset is open — you can download it from Kaggle today."
