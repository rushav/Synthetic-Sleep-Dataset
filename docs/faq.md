# FAQ & Interview Preparation Guide
## Synthetic Sleep Environment Dataset Generator
### TECHIN 513 — Team 7: Rushav Dash & Lisa Li

---

This document prepares you for technical presentations, professor Q&A sessions,
and peer evaluations. Questions are grouped by topic. For each question, we
provide a concise answer followed by a deeper explanation for follow-up questions.

---

## PART 1: Signal Processing Theory

---

**Q1. Why did you use a Butterworth filter? Why not a Chebyshev or elliptic filter?**

**Short answer:** The Butterworth filter is maximally flat in the passband —
meaning it doesn't distort signals at frequencies we want to keep. For
temperature, which has a meaningful slow oscillation (circadian drift), we
don't want any passband ripple. Chebyshev and elliptic filters achieve steeper
rolloff by allowing ripple in either the passband or stopband, which is
appropriate for communications applications but would artificially distort the
slow thermal oscillations we care about.

**Deep answer:** Our cutoff is at 1/30 min⁻¹ (one cycle per 30 minutes). The
signals we care about — HVAC cycles (30–70 min period) and circadian drift
(480 min period) — sit at frequencies at or below this cutoff. A Chebyshev
filter's passband ripple (even small) would create fake oscillations in the
temperature signal at those frequencies, which would then be picked up by our
`temp_mean_rate_change` feature extractor. The Butterworth avoids this artifact
entirely.

---

**Q2. What is zero-phase filtering, and why does it matter here?**

**Short answer:** Zero-phase filtering applies the filter forwards and then
backwards through the signal (implemented as `sosfiltfilt`). This cancels
the group delay that a one-pass filter introduces, so the filtered output is
perfectly time-aligned with the input.

**Deep answer:** A causal IIR filter introduces phase shift — different
frequencies are delayed by different amounts. This means the output peaks don't
align with the input peaks. For temperature, this doesn't matter for simple
statistics like mean and std. But our feature extractor computes
`temp_mean_rate_change` = mean of |ΔT/Δt|, which depends on the *timing* of
temperature changes. A time-shifted signal would produce wrong rate-of-change
values. Forward-backward filtering exactly doubles the filter order and squares
the magnitude response while canceling all phase shift, giving us a correct,
time-aligned result.

---

**Q3. Why did you choose pink noise (1/f) instead of white noise for the stochastic components?**

**Short answer:** Natural processes — temperature drift, wind fluctuations,
acoustic environments — universally exhibit pink (1/f) power spectra: lower
frequencies have proportionally more power. White noise is flat across all
frequencies, which is physically unrealistic for environmental signals.

**Deep answer:** In white noise, every frequency is equally likely. Real indoor
environments have much more slow variation (drafts lasting minutes, thermal
cycles) than rapid variation. The 1/f spectrum encodes this: at frequency f,
power ∝ 1/f. This is why environmental recordings have high autocorrelation at
lag-1 (r ≈ 0.97 for our temperature signal), while white noise would give
autocorrelation near zero. Our ablation confirmed that removing the pink noise
component and replacing it with white noise drops the autocorrelation from 0.97
to ~0.70.

---

**Q4. How do you generate pink noise in code?**

**Short answer:** Inverse FFT spectral shaping. We build a frequency-domain
spectrum with amplitude ∝ 1/√f and random phases, then transform back to the
time domain with IFFT.

**Deep answer:**
```python
n_freqs = n_samples // 2 + 1
freqs = np.arange(1, n_freqs + 1)        # avoid DC singularity
amplitudes = 1.0 / np.sqrt(freqs)        # 1/f shaping → power ∝ 1/f
phases = rng.uniform(0, 2π, n_freqs)
spectrum = amplitudes * exp(i·phases)    # complex spectrum
signal = irfft(spectrum, n=n_samples)    # inverse FFT → real signal
```
We then normalise to zero mean, unit variance, and scale to the desired physical
units (e.g., ×0.15°C for temperature noise).

---

**Q5. What is a Poisson process, and why is it the right model for light events?**

**Short answer:** A Poisson process models the arrival of rare, independent,
memoryless events at a constant average rate λ. Light-on events (phone checks,
bathroom trips) are rare (2–3 per night), don't depend on each other (one
bathroom trip doesn't make another more likely), and have no memory of when
the last one occurred. This is exactly the Poisson regime.

**Deep answer:** Formally, if events arrive as a Poisson process with rate λ,
then: (1) the number of events in any interval T ~ Poisson(λT); (2) inter-event
times ~ Exponential(λ); (3) events in non-overlapping intervals are independent.
All three properties are realistic for nocturnal light disruptions. The
exponential duration distribution (mean 8 min) is also natural: a bathroom trip
or phone check that's already been short will tend to end soon (memoryless
property of the exponential distribution).

---

**Q6. How does the FFT appear in your pipeline beyond pink noise generation?**

**Short answer:** We use FFT analysis in two places: (1) to find the dominant
frequency of real IoT temperature readings (for calibration), and (2) to
estimate the HVAC cycle period from the Smart Home dataset.

**Deep answer:** In `data_loader.py`, after loading the room occupancy dataset,
we compute `np.abs(np.fft.rfft(temp - temp.mean()))` and find the peak frequency
excluding DC (index 0). This gives us the dominant oscillation frequency in the
real data, which we use to set our circadian period parameter. Similarly, for
HVAC calibration, we look at the real-valued FFT spectrum and identify the peak
in the 20–120 minute period range — the physically plausible thermostat cycle
band. This calibration process is what makes our synthetic signals statistically
grounded rather than arbitrary.

---

**Q7. Your Tier 1 KS-tests all fail. Doesn't that mean your signals are unrealistic?**

**Short answer:** No — the KS-test failure is caused by a domain mismatch, not
signal quality failure. The Room Occupancy IoT dataset records an office building
during business hours, while our synthetic signals target a bedroom at night.

**Deep answer:** The Kolmogorov-Smirnov test is highly sensitive at large sample
sizes. At n = 5,000 vs. n = 10,000, the test has near-perfect power to detect
even trivial distributional differences — shifts of 0.1°C in mean temperature
will produce p ≈ 0. Our synthetic temperatures average 21.1°C; the office IoT
averages 22.5°C. This 1.4°C difference is physically meaningful (office vs.
bedroom) but the KS-test interprets it as a catastrophic failure. The right
interpretation is: our signals are calibrated for the *correct domain* (bedroom),
while the reference is from the *wrong domain* (office). This is an inherent
limitation of using the only available public IoT dataset.

---

**Q8. What is autocorrelation at lag-1, and why do you use it to evaluate temperature?**

**Short answer:** Autocorrelation at lag-1 (AC-1) measures how correlated a
signal is with itself shifted by one time step. For our 5-minute sampling rate,
it tells us: "knowing the temperature right now, how well can we predict the
temperature in 5 minutes?" A value near 1.0 means temperature changes very slowly
(high inertia), which is physically correct for indoor environments.

**Deep answer:** The lag-1 autocorrelation of our filtered temperature signal
is r ≈ 0.97, matching the r ≈ 0.97 we measured from the real IoT data. Without
the Butterworth filter, it drops to r ≈ 0.62 (our ablation result) — the signal
becomes too "jumpy." This metric is more physically interpretable than the KS-test
for evaluating thermal realism, because it captures the temporal structure of the
signal rather than just its marginal distribution.

---

## PART 2: Machine Learning Methodology

---

**Q9. Why did you choose Random Forest over gradient boosting (XGBoost) or a neural network?**

**Short answer:** Dataset size. With n = 452 training records, gradient boosting
risks overfitting (many hyperparameters to tune) and neural networks are
completely inappropriate. Random Forest is the right tool for small, tabular
datasets with mixed feature types.

**Deep answer:** XGBoost has more hyperparameters than Random Forest and typically
needs more data to tune well. Neural networks need at minimum thousands of
examples for even a shallow MLP to generalise. At n = 452, a 200-tree Random
Forest achieves R² ≈ 0.45 for sleep efficiency — respectable given the inherent
variability in human sleep. OOB scoring gives us unbiased validation without
wasting any of our 452 records on a hold-out set. We also used 5-fold CV to
confirm the OOB scores.

---

**Q10. How did you prevent overfitting in the Random Forest?**

**Short answer:** OOB scoring, 5-fold cross-validation, and by relying on
standard hyperparameters (n_estimators=200, no max_depth limit — RFs are
naturally regularised by averaging).

**Deep answer:** Random Forests are resistant to overfitting because they
average many independent, deep trees trained on bootstrap samples. Each tree
overfits its bootstrap sample, but because different trees overfit in different
directions, the average cancels out. We verified this via the OOB R² (which
estimates generalisation error) and 5-fold CV RMSE. Both agreed to within 5%,
suggesting no significant overfitting. We did not tune max_depth or min_samples_leaf,
choosing to rely on the ensemble's natural regularisation.

---

**Q11. Your R² values are 0.18–0.45 — isn't that poor performance?**

**Short answer:** Not for this domain. Sleep is highly individual and noisy.
Many published models predict sleep efficiency from lifestyle data with R² ≈
0.40–0.55. Our 0.45 for sleep efficiency is competitive, and the other targets
(REM%, awakenings) are inherently harder to predict from lifestyle proxies.

**Deep answer:** The Sleep Efficiency dataset captures lifestyle factors (caffeine,
alcohol, exercise) but misses many determinants of sleep quality: genetic
predisposition, recent illness, stress, prior sleep debt, medication. Even
perfect environmental control cannot fully predict sleep quality for an
individual. An R² of 0.45 on sleep efficiency means our features explain 45% of
the variance — which is meaningful for a dataset this small with this signal
quality. The moderate R² is exactly why we add calibrated residual noise: we
don't claim to perfectly determine sleep outcomes from environment, just to
reproduce the directional relationships.

---

**Q12. What is out-of-bag (OOB) scoring, and how did you use it?**

**Short answer:** Each Random Forest tree trains on a bootstrap sample (random
sample with replacement) of the data. The records *not* selected (~37% on
average) are "out of bag" — they weren't used to train that tree, so they
can be used to evaluate it. Averaging OOB predictions across all trees gives
a free validation set without reserving data.

**Deep answer:** We used OOB predictions in two ways: (1) as a check on model
quality (OOB R² agrees with 5-fold CV R² to within 5%), and (2) to estimate
residual standard deviations for adding calibrated noise. For sleep efficiency,
the OOB residuals have std ≈ 0.08; we add Gaussian noise N(0, 0.08²) to each
synthetic prediction to match the real-data variability. Without this step,
all synthetic efficiencies cluster too tightly around the model mean.

---

**Q13. How do you map environmental features to the Sleep Efficiency training space?**

**Short answer:** We engineer proxy features: an "arousal index" (how much the
environment would disrupt sleep) and a "fragmentation proxy" (how likely are
discrete awakenings), derived from environmental signal features using evidence-
based scaling factors from the sleep science literature.

**Deep answer:** The Sleep Efficiency dataset has no environmental columns.
Its predictors are age, caffeine, alcohol, exercise, gender, smoking. We map:

- `temp_optimal_fraction` → penalises `arousal_index` (poor thermal comfort
  increases arousal, analogous to high caffeine)
- `light_disruption_score` → increases `fragmentation_proxy` (light events
  fragment sleep like alcohol disrupts architecture)
- `sound_above_55db_minutes` → increases `fragmentation_proxy`
- `age_group` → `age_numeric`, `age_young`, `age_middle`, `age_senior` flags

The scaling factors are calibrated from literature (e.g., the thermal comfort
penalty is weighted 0.4 based on Okamoto-Mizuno 2012 showing temperature is the
strongest single environmental predictor). This is a simplified mapping — a
known limitation — but it ensures directional correctness.

---

**Q14. Why do you train separate RFs for each sleep metric rather than one multioutput model?**

**Short answer:** We made an engineering choice for simplicity and found that
the main cost is losing inter-label biological correlations. Our ablation shows
a multioutput RF recovers most of that correlation.

**Deep answer:** A multioutput RF uses the same set of trees but predicts all
targets jointly. The splitting criterion considers all targets simultaneously,
which naturally learns correlations between them (e.g., high deep sleep → fewer
awakenings). Our ablation (Ablation 4) shows the deep-sleep–awakenings correlation
improves from r = −0.002 to r = −0.18 with a multioutput RF. We chose the
per-target approach for the primary pipeline because it is simpler to interpret
and allows independent calibration of each target's residual noise. Future work
should use the multioutput approach.

---

**Q15. How does your train/validation/test split work?**

**Short answer:** For the Random Forest training, we use 5-fold CV (no fixed
held-out set, since n=452 is small). For the final model (used in generation),
we train on all 452 records. For Tier 2 validation, the real-data baseline uses
an 80/20 train-test split.

**Deep answer:** The philosophical issue here is that we need to train the best
possible model (use all data) for generation, but we also need an honest
evaluation of how well the model will generalise. We address this with 5-fold
CV before the final fit: the CV RMSE tells us generalisation error, and we
then fit the final model on all 452 to maximise its accuracy. The final model's
quality is never evaluated directly — only the 5-fold CV model is. This is a
standard practice in small-dataset ML.

---

## PART 3: Dataset Design Decisions

---

**Q16. Why did you choose synthetic data instead of collecting real data?**

**Short answer:** Collecting paired bedroom sensor + polysomnography data requires:
(a) wearable sensors or EEG hardware for each subject, (b) months of overnight
recordings, (c) IRB approval, (d) subject compensation. Our 5,000-session synthetic
dataset would require 5,000 person-nights of real data — roughly 14 person-years.
Synthetic generation makes this feasible for a course project and produces a
larger, more diverse dataset.

**Deep answer:** Beyond feasibility, synthetic data offers controllability: we
can deliberately generate edge cases (very cold rooms, many light events) that
would be rare or unethical in real studies. We can also guarantee exact
stratification across seasons, age groups, and sensitivity levels, whereas real
data would be subject to confounders (e.g., more elderly subjects sleep in winter,
skewing seasonal analyses). The limitation is that synthetic labels are model
predictions, not ground truth measurements — a trade-off we acknowledge explicitly.

---

**Q17. Why did you choose 5,000 sessions? Why not 10,000 or 1,000?**

**Short answer:** The original proposal specified 2,500; we doubled to 5,000 to
ensure adequate statistical power for subgroup analyses. Beyond 5,000, generation
time increases linearly (~8 minutes already) and the additional diversity is
limited by the fixed stratification grid.

**Deep answer:** With 4 seasons × 3 age groups × 3 sensitivity levels = 36
demographic combinations, 5,000 sessions gives ~138 sessions per combination.
This is sufficient for computing group-level statistics with confidence intervals.
At 2,500, some subgroups would have only ~70 sessions, limiting power. Beyond
5,000, the additional sessions would be cyclically repeated from the same
(season, age, sensitivity) combinations but with new random seeds — useful for
augmentation but not adding new demographic diversity.

---

**Q18. Why 5-minute sampling interval instead of, say, 1-minute or 30-second?**

**Short answer:** Sleep staging (standard PSG) uses 30-second epochs, but
environmental sensors in smart home deployments typically sample at 1–5 minute
intervals. We chose 5 minutes as a realistic smart home compromise, giving 96
samples per 8-hour session — enough temporal resolution to observe HVAC cycles
(≥30 min period) while staying computationally efficient.

**Deep answer:** At 5-minute resolution, our Nyquist frequency is 0.1 cycles/min
(one cycle per 10 minutes). Our fastest meaningful signal — HVAC cycles at 30
min period — has frequency 1/30 ≈ 0.033 cycles/min, well below Nyquist. Signal
aliasing is not a concern. A 1-minute resolution would give 480 samples per
session, 5× the data volume, for minimal additional information since we don't
model any sub-5-minute environmental dynamics.

---

**Q19. How does stratification work in your pipeline?**

**Short answer:** We divide the 5,000 sessions equally across 4 seasons (1,250
each). Within each seasonal block, we cycle through all 9 (age, sensitivity)
combinations in order: young-low, young-normal, young-high, middle-low, ...,
senior-high. This gives 138–139 sessions per (season, age, sensitivity) cell.

**Deep answer:** The cycling approach (rather than random assignment) guarantees
exact balance without any randomness. Each session is deterministically linked
to a (season, age, sensitivity) triplet by its index — this means the dataset's
demographic distribution is known and fixed regardless of the random seed. We
then apply a session-specific seed (derived from global seed ⊕ hash of the
triplet) to make signals and labels random within each demographic cell. This
two-level design (fixed structure, random content) is important for ablation
studies: we can change the signal generation parameters and regenerate sessions
with the same demographic structure, making results directly comparable.

---

**Q20. Why did you include sound and humidity signals? Aren't temperature and light sufficient?**

**Short answer:** Sleep science literature shows all four environmental factors
independently affect sleep. WHO guidelines specifically address nighttime noise.
Humidity outside 30–60% range has documented effects on sleep-disordered breathing.
Including them makes the dataset more complete and realistic.

**Deep answer:** Including sound adds a predictor that the RF can use when
`sound_above_55db_minutes` is high — these sessions should have more awakenings.
Including humidity adds `humidity_out_of_range_minutes`, which contributes a
small arousal_index penalty. While the individual effect sizes are smaller than
temperature and light, the multi-signal structure is what makes the dataset
useful for multivariate sleep quality modelling. Researchers using our dataset
can study any subset of the four signals.

---

## PART 4: Results Interpretation

---

**Q21. What does it mean that summer sessions are 4.5°C warmer than winter? Is that realistic?**

**Short answer:** Yes — this is exactly the expected seasonal stratification.
Indoor bedroom temperature in summer (21–25°C) versus winter (17–20°C) reflects
ASHRAE thermal comfort guidelines and real-world thermostat practices.

**Deep answer:** Our `SEASON_TEMP_RANGES` dictionary sets winter base temperatures
at 17–20°C and summer at 21–25°C, calibrated from ASHRAE 55 comfort standard
and the Sleep Efficiency dataset's mean efficiency per season. The 4.5°C seasonal
difference in our dataset (validated in Tier 3) confirms that our stratification
is working correctly and that the signal generator is properly reading the season
parameter. This sanity check passing gives us confidence in the end-to-end
pipeline for the most physically interpretable variable.

---

**Q22. Your Tier 3 check for light events → efficiency fails. What went wrong?**

**Short answer:** The Random Forest label model has a non-linear response that
mutes the light→efficiency penalty at high event counts. Sessions with >4 light
events are rare in the training data, so the RF extrapolates less confidently
in that region.

**Deep answer:** In the training data (n=452), very high light disruption
(>4 events) is uncommon. The RF's arousal index for high-event sessions reaches
a plateau: the feature `light_event_count × 0.08 + light_penalty × 0.4` grows
with event count, but the RF maps this to sleep efficiency through a learned
non-linear function that doesn't necessarily extrapolate linearly. The result is
that sessions with 5+ light events have nearly the same predicted efficiency as
sessions with 3–4 events. Fixing this would require either (a) imposing a hard
constraint on the mapping function, (b) using a multivariate model that
explicitly couples light events to efficiency, or (c) augmenting the training
set with manually labeled high-disruption records.

---

**Q23. Your deep sleep ↔ awakenings correlation is essentially zero (r = −0.002). Is that a serious flaw?**

**Short answer:** It is a known limitation, not a fundamental flaw. The cause is
clearly identified: independent RFs don't share information between outputs. Our
ablation shows that a multioutput RF would improve this to r ≈ −0.18.

**Deep answer:** Biological datasets consistently show r ≈ −0.4 to −0.6 between
slow-wave sleep percentage and number of awakenings (Dijk 2009). Our synthetic
dataset has r ≈ −0.002, essentially no correlation, because the deep% model and
awakenings model are completely independent. This is a design limitation that
affects any downstream analyses that depend on this correlation — for example,
a model trying to predict awakenings from deep% would find no signal in our data.
We disclose this limitation explicitly in the paper and suggest multioutput RF
as the fix. From a grading perspective, the key point is that we identified the
root cause and proposed a concrete solution.

---

**Q24. How would you prove that your pipeline is reproducible?**

**Short answer:** The entire dataset is determined by a single integer seed
(default: 42). Given the same seed, `SleepDatasetGenerator(global_seed=42).setup().generate()` always produces bit-identical output. Our test suite includes explicit reproducibility tests.

**Deep answer:** Each session's seed is derived as
`session_seed = global_seed XOR hash(session_index, season, age_group, sensitivity)`,
making it a deterministic function of the global seed and the session's identity.
The `tests/test_signal_generator.py` file includes `TestReproducibility` tests
that call `generate_temperature(random_seed=123)` twice and assert bit-identical
output with `np.testing.assert_array_equal`. Similarly, the dataset metadata
JSON stores the `random_seed` used for generation, so a reader can regenerate
the exact dataset by running `SleepDatasetGenerator(global_seed=42)`.

---

## PART 5: Code Implementation

---

**Q25. Why did you use `scipy.signal.sosfiltfilt` instead of `scipy.signal.filtfilt`?**

**Short answer:** Second-order sections (SOS) representation is numerically more
stable than transfer function (BA) representation, especially for higher-order
filters. `filtfilt` with a high-order BA transfer function can produce numerical
errors for short signals; `sosfiltfilt` with SOS avoids this.

---

**Q26. How does your UUID generation work for session IDs?**

**Short answer:** We construct a 128-bit integer by combining the low 64 bits
of `session_seed` (for randomness) with the session index shifted into the high
64 bits (for ordering), then cast this to a `uuid.UUID` string.

```python
uuid_int = (session_seed & 0xFFFFFFFFFFFFFFFF) | ((session_idx << 64) & 0xFFFF...FFFF)
session_id = str(uuid.UUID(int=uuid_int))
```

This gives deterministic but unique UUIDs — two sessions with the same seed
but different indices get different UUIDs.

---

**Q27. What does `np.random.default_rng` give you that `np.random.seed` does not?**

**Short answer:** `default_rng` returns a new, isolated `Generator` object
(PCG64 algorithm) that is independent of any global numpy random state.
`np.random.seed` sets a global state that can be contaminated by other library
calls. For reproducibility in a pipeline where multiple modules each need their
own random state, isolated generators are essential.

---

**Q28. How do you handle the edge case where a Poisson draw returns 0 events?**

**Short answer:** The `for _ in range(n_events)` loop simply doesn't execute,
returning a zero event array. This correctly models nights with no light
disruptions (a legitimate and common scenario for young, low-sensitivity individuals).

---

**Q29. What would happen if you set the Butterworth filter cutoff above the Nyquist frequency?**

**Short answer:** `scipy.signal.butter` would raise a `ValueError` because the
normalised cutoff (cutoff / Nyquist) must be in (0, 1). We guard against this
with `np.clip(normalised_cutoff, 1e-4, 0.999)` and a `warnings.warn` if
adjustment was necessary.

---

**Q30. If a professor asked you to add a 5th signal — CO₂ concentration — how would you do it?**

**Short answer:** CO₂ in a bedroom follows a slow accumulation curve: it rises
during sleep (human exhalation) and drops when windows are opened or HVAC
ventilation runs. We would model it as a rising exponential baseline (from
~400 ppm outdoor to 800–1200 ppm peak) with HVAC-correlated drops (sawtooth-
like dips when ventilation runs) and small measurement noise. Feature extraction
would add `co2_mean_ppm`, `co2_above_1000_minutes`, and `co2_max_ppm`. The
feature mapping to arousal index would add a CO₂ penalty (high CO₂ → reduced
slow-wave sleep, documented in Strøm-Tejsen et al. 2016). This would require
no architectural changes — just a new `generate_co2()` method in `SignalGenerator`
and new features in `FeatureExtractor`.

---

## Quick Reference: Key Numbers

| Parameter | Value | Why |
|-----------|-------|-----|
| Session duration | 480 min (8 h) | Standard adult sleep opportunity |
| Sampling interval | 5 min | Realistic IoT sensor rate |
| Samples per session | 96 | 480 / 5 |
| Total sessions | 5,000 | Power for subgroup analyses |
| Seasons | 4 (1,250 each) | Seasonal stratification |
| Butterworth order | 4 | −80 dB/decade rolloff (4×20) |
| Butterworth cutoff | 1/30 min⁻¹ | Preserves HVAC cycles (≥30 min) |
| Pink noise std | ~0.15°C (temp) | Calibrated from IoT recordings |
| Light Poisson λ | 2–3 events/night | Calibrated from occupancy dataset |
| RF n_estimators | 200 | Standard; more = diminishing returns |
| RF random_state | 42 | Fixed for reproducibility |
| CV folds | 5 | Standard; balanced bias-variance |
| Sleep efficiency range | [0.50, 0.99] | Clinical definition |
| Awakenings range | [0, 12] | Clinical range from dataset |
| Optimal temp zone | 18–21°C | Okamoto-Mizuno (2012) |
| Sound disturbance threshold | 55 dB | WHO indoor guidelines (2009) |
| Humidity comfort range | 30–60% | ASHRAE 55 / Nuckton (2006) |
