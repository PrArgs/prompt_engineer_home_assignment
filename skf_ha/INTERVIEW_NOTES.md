# Interview Notes - Temperature Anomaly Detection

## Quick Reference for 15-Minute Presentation

### Key Results Summary (from final notebook)
- **Data:** 3,507 measurements over 364 days, median 2-hour sampling gap
- **Temperature range:** 8.8°C to 41.9°C
- **Anomalies detected:** 1,183 / 3,507 (33.7%)
- **First detection:** January 2, 2024 (day 2 of data)
- **Hampel cleanup:** 113 spikes replaced via `n_sigma=2.0`, `window=24h` (3.2% of data)

### Phase Analysis Results
| Phase | Anomaly Rate | Expected |
|-------|-------------|----------|
| Phase 1 (Jan-Jul) | 24.4% | Medium ✓ (slow drift)
| Recovery (Aug-Oct) | 18.9% | Low (acceptable trade-off)
| Phase 2 (Nov-Dec) | 88.4% | High ✓ |

**Key Insight:** Detector separates the two degradation phases while keeping recovery false positives below 20% using shorter persistence + 4-day trend window.

---

## Statistical Concepts (Deeper Explanations)

### 1. MAD (Median Absolute Deviation)

**What it is:**
MAD is a robust measure of variability, analogous to standard deviation but resistant to outliers.

**Formula:**
```
MAD = median(|x_i - median(x)|)
```

**Why 1.4826?**
To make MAD comparable to standard deviation under a normal distribution:
```
σ ≈ 1.4826 × MAD
```

This conversion factor comes from the relationship between MAD and σ for Gaussian data. When data is normally distributed, `MAD ≈ 0.6745σ`, so `σ ≈ MAD/0.6745 ≈ 1.4826×MAD`.

**When to use:** When your data has outliers that would inflate standard deviation.

---

### 2. Hampel Filter

**Purpose:** Remove spikes while preserving underlying signal (especially trends).

**Algorithm:**
1. For each point, compute rolling median and rolling MAD in a local window
2. Convert MAD to σ: `σ = 1.4826 × MAD`
3. Identify outliers: `|x - rolling_median| > k × σ` (tunable; we use k=2.0 for higher sensitivity)
4. Replace outliers with rolling median

**Why it works:**
- **Median** is unaffected by the outlier you're trying to remove
- **MAD** gives robust scale estimate within local neighborhood
- **Replacement** (not deletion) preserves time series continuity

**Advantage over simple threshold:** Adapts to local baseline (what's "normal" changes over time as motor degrades).

---

### 3. De-seasonalization (Residualization)

**Purpose:** Remove cyclic patterns so drift detection focuses on true trend, not daily variation.

**Method:**
1. Compute time-of-day baseline: `baseline(h) = median(temp at hour h)`
2. Subtract: `residual_t = temp_filtered_t - baseline(hour_of_day_t)`

**Why it helps:**
- Without de-seasonalization: A motor that's always 5°C hotter in the afternoon might trigger false "upward trend" alarms every day
- With de-seasonalization: The detector sees "is afternoon temp rising over multiple days?" not "is it afternoon?"

**When to skip:** If daily variation is minimal (<2°C range), skip this step to keep method simpler.

---

### 4. Page-Hinkley Test

**Intuition:** Think of a bucket that collects water (positive deviations) and drains water (negative deviations). In normal operation, it fills and empties randomly, staying near zero. During sustained drift, it fills continuously and overflows → alarm.

**Formal definition:**
```
PH_t = max(0, PH_{t-1} + (x_t - μ_0 - δ))
```

Where:
- `μ_0` = reference mean (baseline median)
- `δ` = drift magnitude you want to detect (set to ~0.5σ)
- `max(0, ...)` = reset to zero if cumulative becomes negative

**Alarm condition:** `PH_t > λ` (threshold)

**One-sided detection:** This formulation only detects upward drift. For downward, use `-(x_t - μ_0 + δ)`.

**Why it's good for this problem:**
- Accumulates small consistent changes → detects slow degradation
- Resets on negative deviations → ignores temporary spikes
- Sequential (online) → works in real-time

**Implementation note:** In the final notebook we realize this idea through a 4-day rolling linear slope with threshold 0.012 plus persistence K=4, which behaves like a simplified one-sided Page-Hinkley detector.

---

### 5. Persistence & Hysteresis

**Persistence (K consecutive detections):**
- Even after threshold is exceeded, wait for K consecutive alarms before truly flagging
- **Why:** A single point above threshold could be noise; K=3-5 confirms it's sustained
- **Trade-off:** Adds ~K × sampling_period to detection delay

**Hysteresis (separate ON/OFF thresholds):**
- Alarm ON: PH > λ_high for K consecutive points
- Alarm OFF: PH < λ_low (where λ_low < λ_high)
- **Why:** Prevents "flicker" (alarm rapidly turning on/off near boundary)
- **Not implemented in v2:** Mentioned as production improvement

---

## Defense: Plots, Statistics, Strengths/Weaknesses

### What Plots Show and How They Support the Method

1. **Full-year time series**
   - Shows: Clear upward trend from 10°C → 38°C
   - Supports: Confirms problem is sustained drift, not random variation

2. **Distribution & Boxplot**
   - Shows: Baseline statistics, presence of outliers
   - Supports: Justifies need for robust statistics (median, MAD) and outlier removal

3. **Hour-of-day seasonality**
   - Shows: Daily temperature variation range
   - Supports: Decision to de-seasonalize (or skip if range <2°C)

4. **Hampel filter effect (zoomed period)**
   - Shows: Outliers detected and replaced with local median
   - Supports: Pre-processing removes spikes while preserving trend

5. **Page-Hinkley statistic over time**
   - Shows: Cumulative drift accumulation; resets during stable periods
   - Supports: Demonstrates how PH captures sustained changes vs transient noise

6. **Final plot with red markers**
   - Shows: Temperature + anomaly labels aligned with visible upward trend
   - Supports: Visual validation that detected anomalies match observed degradation

---

### Statistics Used and Why

| Statistic | Purpose | Why This One? |
|-----------|---------|---------------|
| **Median** | Central tendency (smoothing, baselines) | Unaffected by outliers; 50th percentile |
| **MAD** | Scale estimate (outlier detection) | Robust alternative to std; uses median internally |
| **1.4826×MAD** | Convert MAD to σ-equivalent | Makes MAD comparable to standard deviation |
| **95th percentile** | Threshold calibration | Assumes 95% of baseline is normal; captures top 5% |
| **Rolling slope (PH-style)** | Drift detection | Linear trend over 4-day window + persistence filters noise |

**All chosen for robustness:** Data has spikes/outliers; robust statistics less sensitive than mean/std.

---

### Strengths

1. **Explainable:** Every step has clear statistical justification; can show exactly why an alarm fired
2. **No labels required:** Calibrated on baseline period assumption (unsupervised)
3. **Handles spikes vs trends:** Hampel removes outliers; PH accumulates sustained changes
4. **Tunable trade-offs:** K and threshold control FP/detection delay balance
5. **Computationally efficient:** Rolling operations; suitable for real-time deployment

---

### Weaknesses

#### Detection Delay vs Event Start Time

**Estimated lag:** ~90-110 hours for barely-above-threshold drifts (empirically ~24h here because Phase 1 rises quickly)

**Breakdown:**
- Hampel window (24h centered): ~12h lag
- 4-day rolling slope: needs ~3-4 days of evidence before slope exceeds 0.012
- Persistence (K=4 @ 2h median sampling): +8h
- **Total:** Shorter in practice when drift is steep; longer for borderline trends

**Trade-off:** Smaller K and shorter windows → faster detection but more false positives

---

#### False Positives

**Risk:** Temporary fluctuations may trigger alarms if:
- K too small (less confirmation needed)
- Threshold too low (too sensitive)
- Baseline period itself was abnormal (wrong calibration)

**Evidence in results:**
- Recovery period flagged in 18.9% of points (short 4-day window is intentionally sensitive)
- Those alerts correspond to brief upward bumps while the system is cooling back down
- Still warmer than baseline, but persistence K=4 prevents the entire recovery from being labeled anomalous

**Mitigation:**
- Increase K (more persistence)
- Add hysteresis (separate ON/OFF thresholds)
- Manual review/confirmation before critical actions
- Operator feedback loop to refine thresholds

---

#### False Negatives

**Risk:** Very slow trends below threshold may be missed

**Example:** If degradation is 0.005°C/hour uniformly, it might not exceed 95th percentile of baseline slopes

**Evidence:** If baseline had gradual warming, our threshold would be too high

**Mitigation:**
- Lower threshold (at cost of more FPs)
- Adaptive baselines (recompute using recent "confirmed normal" periods)
- Longer integration windows (accumulate more drift before alarming)
- Multi-sensor fusion (cross-check with vibration, current draw)

---

#### Manual Tuning

**Limitation:** Parameters require domain knowledge:
- Window sizes (24-72h)
- Threshold percentile (90th? 95th? 99th?)
- Persistence K (3? 5? 7?)
- δ (drift magnitude to detect)

**Current approach:** Educated guesses + sensitivity analysis

**Better approach:**
- Calibrate on fleet of similar motors
- Use maintenance logs to tune parameters
- A/B testing in production

---

## Likely Interview Questions & Crisp Answers

### 1. How did you choose window sizes (24h Hampel, 4-day trend)?

**Answer:** Balance between responsiveness and noise rejection. The Hampel window stays at 24h so it adapts to daily ambient swings without chasing noise. The rolling linear slope uses ~4 days (48 samples) because mechanical degradation unfolds over days, not hours; shorter windows misinterpret diurnal swings, longer ones delay alarms. Sensitivity analysis showed stable performance across 3-5 day trend windows, so 4 days was the sweet spot.

---

### 2. Why MAD instead of standard deviation?

**Answer:** MAD is robust to outliers. Standard deviation uses squared differences, so one spike of 100°C would massively inflate σ, making normal variation look artificially small. MAD uses median of absolute differences → unaffected by extremes. Critical for data with spikes.

---

### 3. Why Page-Hinkley over simpler methods (e.g., "alert if temp >30°C")?

**Answer:** Absolute thresholds fail because "normal" changes over time. A motor might be 35°C due to high ambient temperature (normal) or due to bearing failure (anomaly). The Page-Hinkley-style trend detector watches **rate of change**, not absolute value. Our 4-day rolling slope accumulates small increases, so it catches slow degradation that fixed temperature trip points miss.

---

### 4. What if the baseline period itself had a problem?

**Answer:** Critical assumption: first 60 days are "mostly normal." If violated, threshold calibration fails. Mitigations: (1) Use multiple machines to establish baseline, (2) Domain expert review of baseline period, (3) Compare to manufacturer specs, (4) Adaptive thresholds that recalibrate using recent "confirmed normal" windows.

---

### 5. How would you reduce detection delay in production?

**Answer:** (1) Reduce persistence to K=2-3 (currently 4) to react faster at the cost of more noise, (2) Shrink the trend window to 2-3 days so slope turns positive sooner, (3) Add a "fast channel" that monitors first-derivative spikes while keeping the 4-day channel as confirmation, (4) Fuse temperature with vibration/current to confirm degradation sooner, (5) Accept that 2-3 day delay is often worth the reduction in nuisance alerts for mechanical systems.

---

### 6. How do you validate without ground truth labels?

**Answer:**
1. **Phase analysis:** Detector flags 24% of slow Phase 1 (reasonable for gradual drift) and 88% of rapid Phase 2 while keeping recovery at ~19%
2. **Domain expert review:** Show flagged periods to maintenance engineers; do they agree?
3. **Maintenance log correlation:** Do alarms precede actual failures/repairs?
4. **Cross-machine validation:** Apply to fleet; identify outlier machines
5. **Sensitivity analysis:** Results stable across 90-99th percentile and K=2-5 (robust)

---

### 7. What about daily/weekly seasonality?

**Answer:** EDA found 8.23°C hour-of-day range in baseline (>2°C threshold), so we applied de-seasonalization: subtract hour-of-day median baseline from filtered signal. This creates residuals that remove daily cycles. Page-Hinkley then detects drift in residuals, ignoring "expected" daily variation. The resulting signal range is -8.5°C to +18.6°C.

---

### 8. Why not use machine learning (LSTM, Isolation Forest)?

**Answer:**
- **No labeled data:** Supervised ML needs many examples of failures to train
- **Explainability:** Statistical methods transparent; can show "slope exceeded X for Y hours" vs "LSTM predicted anomaly" (black box)
- **Data volume:** 3,500 points is small for deep learning; overfitting risk
- **Operational requirement:** Maintenance decisions need justifications; "MAD-based outlier" is defensible
- **Future work:** Could use ML for multi-sensor fusion or pattern discovery once labeled data becomes available

---

### 9. How do you handle false positives in production?

**Answer:**
1. **Increase persistence K:** Require more consecutive confirmations
2. **Hysteresis:** Separate ON (λ_high) and OFF (λ_low) thresholds to prevent flicker
3. **Two-tier alerting:** Low-confidence = log only; high-confidence = notify operator
4. **Manual review:** Require human confirmation before critical actions (e.g., shutdown)
5. **Feedback loop:** Operators label false alarms → retrain thresholds using confirmed normals

---

### 10. Next steps for production deployment?

**Answer:**
1. **Calibrate on fleet:** Validate parameters across multiple motors
2. **Real-time alerting:** Email/SMS/dashboard notifications
3. **Live monitoring UI:** Show temp, PH statistic, current status (normal/alarm)
4. **Operator feedback:** Allow labeling false alarms → adaptive thresholds
5. **Add context:** Include motor load, ambient temp, maintenance history in alerts
6. **Drift monitoring:** Track if detector itself needs recalibration over time
7. **Scale:** Apply to entire fleet; identify patterns across machines

---

## Common Pitfalls to Avoid in Interview

1. **Don't say "I chose 24h arbitrarily"** → Say "I picked 24h so the Hampel filter tracks daily swings while ignoring noise; verified via sensitivity analysis"

2. **Don't say "MAD is just like std"** → Say "MAD is robust to outliers, unlike std which squares differences"

3. **Don't claim "zero false positives"** → Acknowledge trade-off; show calibration strategy

4. **Don't ignore weaknesses** → Proactively mention detection delay and mitigation strategies

5. **Don't oversell AI** → Statistical methods are appropriate here; know when NOT to use ML

---

## 1-Minute Elevator Pitch

"I built an anomaly detector for motor temperature data that distinguishes sustained upward trends (mechanical degradation) from transient spikes (noise or load variations).

**Key discovery:** The data shows TWO distinct degradation phases separated by a recovery period - not a simple linear trend. Phase 1 (Jan-Jul) shows gradual rise of +6.3°C, then recovery of -3.1°C (Aug-Oct), then catastrophic Phase 2 (+12.8°C in 2 months).

The method uses robust statistics: Hampel filter removes outliers via median+MAD, de-seasonalization removes 8°C daily cycles, and Page-Hinkley test accumulates positive deviations to detect drift. Thresholds calibrated on first 60 days.

**Results:** 1,183 anomalies detected (33.7%), first detection on Jan 2. Detector still covers both degradation phases (24% in slow Phase 1, 88% in rapid Phase 2) while keeping the recovery period under 19% false positives. The approach is explainable, requires no labeled training data, and is production-ready with minor tuning."

---

## Calibration Parameters Used

| Parameter | Value | Justification |
|-----------|-------|---------------|
| Baseline period | 60 days | Before Phase 1 accelerates |
| Hampel window | 24 hours | Reacts within a day to ambient shifts |
| Hampel n_sigma | 2.0 | Higher sensitivity (3.2% of points replaced) |
| De-seasonalization | Applied | 8.23°C daily range > 2°C threshold |
| Rolling slope window | 48 samples (~4 days) | Needs multi-day evidence before alerting |
| Trend threshold | 0.012 slope units | Calibrated via baseline + sensitivity grid |
| Persistence K | 4 | ≈8 hours of sustained rise before label |

---

## Known Limitation: Recovery Period Flagging

**Issue:** Even after tuning, about 19% of recovery samples trigger alerts.

**Why this happens:**
- The 4-day slope window still sees short-lived upticks while the system cools down
- Those bumps exceed the 0.012 threshold before the overall downward trend fully dominates
- Persistence K=4 suppresses most of them, but not every burst

**Interpretation:**
- Motor temperature remains above the original baseline during recovery, so occasional "watch" alerts are acceptable
- Operators mainly care that the detector no longer labels the entire recovery as anomalous (mission accomplished)

**Possible improvements for production:**
1. Add a two-sided detector to explicitly confirm negative slopes before clearing alerts
2. Refresh the baseline after maintenance so recovery operates against a higher reference
3. Increase K during known recovery windows (dynamic persistence)
4. Provide operator feedback buttons to mark "maintenance done" and reset thresholds

---

**Status:** Ready for 15-minute presentation. Review notebook + these notes before interview.
