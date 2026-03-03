# Project Explainer: Synthetic Sleep Environment Dataset Generator
### TECHIN 513 — Team 7: Rushav Dash & Lisa Li — University of Washington

---

## What We Built and Why

### The Problem

Imagine you are building a smart home system that can recommend bedroom settings
(thermostat temperature, blackout curtain control, white noise level) to improve
your sleep. To build such a system, you need a machine learning model that
predicts: "If my bedroom is 20°C with no light disturbances tonight, will I
sleep well?"

Training that model requires paired data: sensor readings from the bedroom
(temperature, light, sound, humidity) alongside sleep quality measurements
(how long you slept, how many times you woke up, how much deep sleep you got).
**That paired dataset does not exist publicly.** Sleep research datasets contain
only physiological measurements. Smart home IoT datasets contain only sensor
readings. Neither bridges the gap.

We built a software pipeline — called `SynthSleep` — that generates this
dataset synthetically. Our pipeline produces 5,000 simulated 8-hour sleep
nights, each with:
- Realistic sensor time-series (temperature, light, sound, humidity)
- Physiologically plausible sleep quality labels (sleep efficiency, awakenings,
  sleep stage percentages)
- Rich metadata (season, age group, sensitivity level)

This enables researchers to immediately begin training sleep-optimization models
without purchasing sensors or recruiting subjects.

---

## How We Generate the Sensor Signals (Signal Processing)

### The Big Idea

A real bedroom sensor does not produce random numbers. Temperature drifts slowly
because of thermal inertia. Light turns on briefly when someone checks their
phone. Sound spikes when a car drives by. We need our synthetic signals to behave
the same way — not just have the right average value, but the right *shape over
time*.

We achieve this through **spectral synthesis** and **event-driven models**,
using the mathematical tools of signal processing.

---

### Temperature Signal

**Why it's complex:**
Indoor temperature is governed by three overlapping physical processes:

1. **Overnight thermal drift** (slow): As the night progresses, the body radiates
   heat and the ambient temperature outside drops. This creates a gentle sinusoidal
   oscillation over the full 8-hour night.

2. **HVAC cycling** (medium): A thermostat heats the room, which then cools down,
   triggering the heater again — a classic control cycle. This creates a sawtooth
   wave repeating every 30–70 minutes.

3. **Micro-fluctuations** (fast): Air currents, door drafts, and sensor noise
   create rapid small variations. These follow a *pink noise* (1/f) power spectrum
   — the same statistical structure as wind, traffic noise, and many other natural
   signals.

**The formula we use:**
```
T(t) = T_base + A·sin(2πt/480) + sawtooth(t) + pink_noise(t)
```

**Why we apply a Butterworth filter:**
Real indoor temperature cannot change instantaneously — the room has *thermal
inertia* (like a large thermal mass that takes time to heat up or cool down).
After combining our three components, we apply a **4th-order Butterworth
low-pass filter** (cutoff at 1 cycle per 30 minutes) to enforce this physical
constraint. The filter smooths out any oscillation faster than 30 minutes while
preserving the HVAC and circadian components.

A Butterworth filter is specifically chosen because it has a **maximally flat
passband** — meaning it doesn't distort the signal within the frequency range
we care about (slow thermal oscillations), only attenuating the high-frequency
components we want to remove.

We implement this as a **zero-phase filter** (`sosfiltfilt`): it applies the
filter forwards and then backwards through the signal, so the output is perfectly
time-aligned with the input. This matters because we later compute *rate-of-change*
features — a time-shifted signal would introduce a systematic error.

---

### Light Signal

**Why it's event-driven:**
A bedroom light is either on or off — it doesn't drift continuously. The right
model is a **Poisson process**: a mathematical model for random, rare, independent
events.

In a Poisson process, the number of events per night follows a Poisson
distribution with rate parameter λ. We set λ based on age group and sensitivity:
- Young adults: λ ≈ 1.4 events/night (few disruptions)
- Senior adults: λ ≈ 3.2 events/night (more bathroom trips, lighter sleep)

Each event has:
- A **random start time** (uniformly distributed across the night)
- A **random duration** (exponentially distributed, mean 8 minutes)
- A **random intensity** (bimodal: dim 10–60 lux for phone checks, bright
  60–150 lux for lamp/bathroom)

We add a **Gaussian smoothing step** (convolution with a Gaussian kernel, σ = 2 min)
to soften the abrupt edges of each pulse. A real light level ramps up and down
slightly as your eyes adapt — it doesn't jump instantaneously in the recorded
sensor data.

---

### Sound Signal

Sound follows a similar event-driven model, but with a continuous pink noise
background (30–42 dB, representing a quiet bedroom ambient level). Disturbance
events (snoring, traffic, a phone ringing) are modelled with exponential
decay envelopes, mimicking the gradual fade of an acoustic impulse.

---

### Humidity Signal

Humidity is the simplest signal: it changes slowly overnight based on outdoor
conditions and HVAC operation. We model it as a sinusoidal baseline (seasonal
mean ± small amplitude) plus mild Gaussian noise smoothed with a 3-sample
moving average. It doesn't require event modelling because humidity has no
discrete disruption mechanism.

---

## How We Connect Signals to Sleep Quality (Machine Learning)

### The Challenge

Our signal processing pipeline produces realistic-looking sensor data. But what
sleep quality should we assign to each session? We cannot simply make up numbers
— the labels need to reflect real physiological relationships.

The answer is **transfer learning**: we train a machine learning model on real
human sleep data, then use it to label our synthetic sessions.

---

### The Real Dataset We Use

We train on the **Sleep Efficiency Dataset** (Kaggle), which contains 452 records
of real subjects with measurements including:
- Age, gender, smoking status, caffeine/alcohol consumption, exercise frequency
- Sleep efficiency (fraction of time in bed actually asleep)
- Number of awakenings per night
- REM sleep percentage, deep sleep percentage

This dataset was collected with actigraphy and/or polysomnography — the gold
standard in sleep research.

---

### Why Random Forest?

We train a **Random Forest Regressor** — one per target variable (sleep efficiency,
awakenings, REM %, deep %). Here's why:

**1. Handles small datasets well.** With only 452 training records, we cannot
afford a deep neural network. Random Forest bootstraps the data and averages
predictions across 200 trees, giving us a robust ensemble even at small n.

**2. Handles mixed feature types.** The Sleep Efficiency dataset has both numeric
(age, caffeine mg) and binary (gender, smoking) features. Random Forest handles
these without requiring feature scaling or dummy encoding.

**3. Provides OOB error estimation.** Instead of wasting records on a held-out
validation set, we use **out-of-bag (OOB) scoring**: each tree only sees a
bootstrap sample, so the records left out serve as a natural validation set.
This is crucial when n = 452.

**4. Captures non-linear relationships.** Sleep efficiency is not a linear
function of caffeine intake — there are interaction effects. Decision trees
naturally capture these interactions.

---

### The Feature Mapping Problem

Here is the core challenge: the Sleep Efficiency dataset contains *lifestyle
features* (caffeine, alcohol, exercise), not bedroom sensor readings. Our
synthetic sessions have *environmental features* (temperature, light events),
not lifestyle data.

We bridge this gap with **proxy feature engineering**:

| Environmental feature | → | Proxy in training space |
|----------------------|---|------------------------|
| Low `temp_optimal_fraction` | → | High `arousal_index` (discomfort = arousal) |
| High `light_disruption_score` | → | High `fragmentation_proxy` (light events = sleep fragmentation) |
| High `sound_above_55db_minutes` | → | High `fragmentation_proxy` |
| `age_group` = senior | → | `age_numeric` = 68, `age_senior` = 1 |

These mappings are grounded in sleep science literature:
- Okamoto-Mizuno (2012): thermal comfort → sleep consolidation
- Zeitzer (2000): nighttime light → melatonin suppression → sleep disruption
- Walker (2017): aging → lighter sleep, more awakenings

After prediction, we:
1. Add **calibrated residual noise** (Gaussian, std derived from OOB residuals)
   to prevent labels from appearing too regular
2. **Clip** all labels to physiologically valid ranges
3. **Renormalise** sleep stage percentages (REM + deep + light = exactly 100%)

---

## How Everything Connects: End-to-End

Here is the data flow for generating one sleep session:

```
Input: (season="winter", age_group="senior", sensitivity="high", seed=42)

Step 1 — SignalGenerator:
  → T(t): base 19°C + 0.7°sin(circadian) + sawtooth(45 min) + pink noise → LPF
  → L(t): 3 lux background + Poisson(λ=3.2) events
  → S(t): 38 dB pink background + Poisson events
  → H(t): 38% base + sin + noise

Step 2 — FeatureExtractor:
  → temp_mean=19.1°C, temp_optimal_fraction=0.83, light_event_count=3,
     light_disruption_score=412, sound_above_55db_min=15, ...

Step 3 — SleepQualityModel.predict():
  → arousal_index = 0.24 (senior, high sensitivity, few light events)
  → fragmentation_proxy = 0.19
  → RF(Sleep efficiency) → 0.81 + noise → clipped to [0.50, 0.99] → 0.80
  → RF(Awakenings) → 2.3 + noise → rounded to 2
  → RF(REM %) → 22.1 → rem_pct = 22.1
  → RF(Deep %) → 19.4 → deep_pct = 19.4, light_pct = 58.5

Step 4 — Record assembled:
  → {session_id, season, age_group, sensitivity, seed,
     temp_mean, light_event_count, ...,
     sleep_efficiency=0.80, awakenings=2, rem_pct=22.1,
     deep_pct=19.4, light_pct=58.5,
     ts_temperature=[19.2, 19.1, ...]}
```

This process repeats 5,000 times with different seeds, seasons, ages, and
sensitivity levels.

---

## Validation: How We Know the Dataset is Realistic

We validate along three independent tiers:

### Tier 1 — Statistical Tests (KS-test)
We compare the distribution of synthetic temperature means against real IoT
sensor readings using a **Kolmogorov-Smirnov test**. The KS statistic measures
the maximum difference between two empirical CDFs. A p-value > 0.05 means the
two distributions are statistically indistinguishable.

*Result:* All three Tier 1 KS-tests fail (p ≈ 0). This is expected: the real
IoT dataset records an office building (office HVAC, daytime), not a bedroom at
night. The domain gap is fundamental, and at n = 5,000, the KS-test has
essentially perfect power to detect even trivial differences. We discuss this
as a design limitation.

### Tier 2 — ML Predictability
We train a simple linear regression on our synthetic data and compare its
in-sample RMSE to a baseline linear regression trained on real data.

*Result:* Synthetic in-sample RMSE = 0.061 vs. real baseline = 0.075. The
synthetic model is within the 20% threshold — indicating that the environmental
features we generate are meaningfully predictive of the sleep labels we assign.

### Tier 3 — Sleep Science Sanity Checks
Six domain-knowledge assertions from the literature:
1. Sessions with >80% time in optimal temperature zone (18–21°C) should average
   sleep efficiency ≥ 0.78. **Pass** (actual: 0.823).
2. Sessions with >4 light events should average efficiency ≤ 0.72. **Fail** (0.833).
3. Deep sleep % should negatively correlate with awakenings (r < −0.2). **Fail** (r = −0.002).
4. Senior sessions should have more awakenings than young sessions. **Pass** (Δ = 0.25).
5. Sleep stage percentages should sum to exactly 100%. **Pass** (error = 0.00).
6. Summer sessions should have higher mean temperature than winter. **Pass** (Δ = 4.5°C).

4/6 checks pass. The two failures are discussed as known limitations.

---

## Limitations

**1. Calibration data domain mismatch.**
We calibrate signal parameters (light event λ, temperature baseline) from an
office IoT dataset because no public bedroom IoT dataset exists at scale. This
is circular — we are building a bedroom dataset precisely because it doesn't
exist — but it means our calibration carries some office-environment bias.

**2. Independent Random Forests don't preserve inter-label correlations.**
We train separate RFs for sleep efficiency, awakenings, REM%, and deep%.
The biological correlation between deep sleep and awakenings (more deep sleep
→ fewer awakenings) is not preserved, giving r = −0.002 instead of the
expected r < −0.2.

**3. Tier 2 is in-sample, not cross-domain.**
True cross-domain validation (train on synthetic, test on real, same features)
is impossible because real data doesn't include co-located environmental sensors.

**4. Labels are model predictions, not measurements.**
The sleep quality labels are predictions from a 452-record Random Forest model,
not actual polysomnographic measurements. They carry model uncertainty and
inherit the model's biases.

---

## Future Work

- Use a **multioutput Random Forest** or **conditional copula** to enforce
  inter-label biological correlations
- Add a **bedroom-specific IoT calibration dataset** if one becomes available
- Extend to **non-8-hour sessions** (shift workers, nap studies)
- Add **physiological outputs** (heart rate variability, accelerometry) for
  a more complete sleep model
- Apply **GAN-based augmentation** conditioned on real polysomnography to
  improve label realism

---

## Glossary of Key Terms

| Term | Plain English Definition |
|------|--------------------------|
| **Butterworth filter** | A signal filter that smoothly removes high-frequency "jitter" while keeping slow trends intact. Used to enforce that temperature can't change too fast. |
| **Pink noise (1/f noise)** | Random noise where lower frequencies have more power — like ocean waves or wind. More realistic than white noise for natural signals. |
| **Poisson process** | A mathematical model for random events that occur independently at a constant average rate. Used to model light switches and sound spikes. |
| **FFT (Fast Fourier Transform)** | An algorithm that decomposes a signal into its frequency components — tells you "how much" of each oscillation frequency is in the signal. |
| **Random Forest** | An ensemble of decision trees. Each tree votes on a prediction; the average is used. Robust to overfitting and works well on small datasets. |
| **OOB score** | Out-of-bag score: uses data not seen by each tree (due to bootstrapping) as a free validation set. |
| **KS-test** | Kolmogorov-Smirnov test: compares two distributions to see if they could have come from the same population. |
| **Sleep efficiency** | Fraction of time in bed actually sleeping (0.50–0.99). |
| **Spectral synthesis** | Building a signal by combining mathematical functions (sines, sawtooth, noise) — rather than recording or sampling from real data. |
| **Zero-phase filtering** | Applying a filter forwards and backwards so the output has no time delay relative to the input. |
| **Stratification** | Deliberately dividing data across subgroups (seasons, age groups) to ensure balanced representation. |
