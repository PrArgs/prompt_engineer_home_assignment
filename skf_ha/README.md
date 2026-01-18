# Temperature Anomaly Detection - Quick Start Guide

## 📦 Files Created

### Main Deliverable
- **home_assignment.ipynb** - Complete presentation-ready notebook (48 cells, executes in ~2-3 minutes)

### Output Files
- **labeled_output.csv** - Final results with timestamp, temperature, and binary labels (0=normal, 1=anomaly)
- **figures/final_anomaly_detection.png** - Production plot with red anomaly markers

### Input Files (Provided)
- **Machine_1_temp 1.csv** - Original temperature data
- **home_assignment.docx** - Assignment requirements

---

## 🚀 How to Run

### Option 1: Open in Jupyter Notebook
```bash
cd "c:\Users\argam\Documents\GitHub\home_assignments\skf_ha"
jupyter notebook home_assignment.ipynb
```
Then: Click "Cell" → "Run All" (or press Shift+Enter through each cell)

### Option 2: Execute from Command Line
```bash
cd "c:\Users\argam\Documents\GitHub\home_assignments\skf_ha"
jupyter nbconvert --to notebook --execute --inplace home_assignment.ipynb
```

### Option 3: View Without Executing
Open `home_assignment.ipynb` in any notebook viewer (VS Code, JupyterLab, Google Colab)
All outputs are already embedded from the successful execution.

---

## 📖 Notebook Structure

### 10 Sections for 15-Minute Presentation

1. **Introduction & Setup** (2 min)
   - Problem definition: sustained upward trend vs spikes
   - Import libraries, set random seed

2. **Data Loading & Standardization** (1 min)
   - Parse timestamps, sort, check for duplicates
   - 3,507 measurements, irregular sampling

3. **Exploratory Data Analysis** (3 min)
   - Full-year time series (shows 10°C → 38°C trend)
   - Zoomed views, distribution plots
   - Rolling statistics, seasonality check
   - Baseline period definition (first 60 days)

4. **Detector A: Rolling-Slope Trend** (2 min)
   - Smooth → calculate slope → threshold → persistence
   - Calibrated to 95th percentile of baseline slopes

5. **Detector B: Page-Hinkley with Pre-processing** (3 min)
   - Hampel filter (de-spike) → optional de-seasonalization
   - Page-Hinkley cumulative drift detection
   - More robust to outliers

6. **Fair Comparison** (2 min)
   - Alert burden, spike sensitivity, detection delay
   - Sensitivity analysis heatmap

7. **Final Recommendation** (1 min)
   - **Chosen: Detector B** for better spike rejection
   - Trade-off table, justification

8. **Final Combined Plot** (Mandatory) (0.5 min)
   - Temperature vs time with RED anomaly markers
   - Annotated first detection

9. **Defense Section** (2 min)
   - Plots explained, statistics used (median, MAD, slopes, PH)
   - Strengths/weaknesses, detection delay (~2-3 days)
   - False positives/negatives discussion

10. **Presentation & Q&A** (0.5 min)
    - 10 interview questions with crisp answers
    - Production deployment next steps

---

## 🎯 Key Results

### Detector A (Rolling-Slope)
- **Method**: Median smoothing + linear regression slope
- **Threshold**: 95th percentile of baseline slopes
- **Pros**: Simple, 3 parameters, easy to explain
- **Cons**: Vulnerable to outliers affecting slopes

### Detector B (Page-Hinkley) ⭐ RECOMMENDED
- **Method**: Hampel filter → Page-Hinkley cumulative sum
- **Threshold**: 95th percentile of baseline PH values
- **Pros**: Better spike rejection, handles seasonality, lower false positives
- **Cons**: More complex (5+ parameters)

### Detection Performance
- **Detection delay**: ~60-70 hours after sustained trend starts
- **False positives**: <2-5% on baseline period (calibrated)
- **False negatives**: May miss very slow trends below threshold

---

## 🛠️ Dependencies

All standard Python libraries:
- pandas
- numpy
- matplotlib
- seaborn
- scipy
- sklearn (minimal usage)

No external APIs or pre-built anomaly detection packages.

---

## 📊 Assignment Requirements Checklist

✅ Single reproducible Jupyter Notebook (runs top-to-bottom, no errors)
✅ Clear markdown for ~15-minute presentation
✅ Purposeful EDA visualizations (inform detector design)
✅ Relevant statistical analysis (all stats used for thresholds/robustness)
✅ Detection method outputting timestamp-aligned labels
✅ Combined plot with temperature + RED anomaly markers
✅ Defense section: plots, statistics, strengths/weaknesses, detection delay, FP/FN
✅ No external APIs or pre-built anomaly services

---

## 📝 Interview Preparation

### Top 3 Questions You'll Likely Face

**1. "Why two detectors instead of one?"**
→ To demonstrate trade-off understanding and make evidence-based choice. Detector A is simpler; Detector B is more robust. Comparison shows which better meets goals.

**2. "How did you handle the detection delay vs false positive trade-off?"**
→ Used persistence filter (K=3 consecutive detections required). Larger K = fewer FPs but slower detection. Current setting: ~2-3 day delay, <5% FP rate.

**3. "How would you validate this without ground-truth labels?"**
→ (1) Semi-synthetic ramp injection, (2) domain expert review, (3) maintenance log correlation, (4) A/B testing in production shadow mode.

See notebook Section 10 for 10 questions + answers.

---

## 🎓 Statistical Methods Used

| Method | Purpose | Why Robust? |
|--------|---------|-------------|
| **Median** | Central tendency (smoothing) | 50th percentile, unaffected by outliers |
| **MAD** | Scale estimate (outlier detection) | Median-based, unlike std |
| **Linear regression slope** | Trend rate (°C/hour) | Direct interpretable measure |
| **95th percentile** | Threshold calibration | Assumes 95% of baseline is normal |
| **Page-Hinkley** | Cumulative drift detection | Accumulates sustained changes, rejects spikes |

All chosen for **robustness to outliers** (data has spikes).

---

## 🚀 Next Steps (If Presenting)

1. **Practice timing**: Run through once, aim for 15 minutes
2. **Know your parameters**: Be ready to justify window sizes, K=3, threshold choices
3. **Prepare for "what if"**: Seasonality, baseline shift, faster/slower detection needs
4. **Have backup**: If asked about Detector A, explain why Detector B is better but keep A as validation

---

## 📞 Questions?

The notebook is fully documented with:
- Markdown explanations for each step
- Inline comments in code cells
- Statistical background for all methods
- Parameter justifications
- 10 interview Q&A pairs

**Ready to present!** ✅
