
---

# Development of Affordable Non-Invasive Anemia Screening Device

An end-to-end repository utilizing multi-wavelength optical sensing (PPG), edge machine learning, and adaptive skin-tone calibration to accurately predict Hemoglobin (Hb) levels and classify anemia on ultra-low-power resource-constrained microcontrollers.

The core embedded pipeline implements a optimized **Dual-Model System** (combining a Random Forest Regressor and a Random Forest Classifier) utilizing `emlearn` to compress the machine learning models down to less than **70 KB**.

---

## 🚀 Key Features

* **Multi-Wavelength Feature Extraction**: Integrates raw sensor data and derived metrics spanning vital wavelengths (530nm, 660nm, 850nm, and 940nm).
* **Dual-Model Predictive Pipeline**: Features simultaneous regression (Hb g/dL) and classification (Anemia flag) execution.
* **Adaptive Skin-Tone Calibration**: Dynamically clusters skin properties via a Melanogenic Index (MI = $DC_{530} / DC_{940}$) into 3 localized profiles (Light, Medium, Dark) to rectify optical shift bias.
* **WHO Compliance & Fallback Mechanism**: Cross-checks physiological thresholds (Male: 13.0 g/dL, Female: 12.0 g/dL) to generate cross-validation alerts and handle low-confidence safety nets.
* **Ultra-Lightweight Embedded Footprint**: Fully inlined, optimized fixed-point integer comparative execution requiring no floating-point matrix dependencies or runtime memory overhead.

---

## 📁 Repository Structure

```bash
├── Final (1).ipynb          # Python Research Notebook (Data processing, training, and compression)
├── hb_dual_model.h          # Final deployed C Header file (Dual Regression + Classification)
├── hb_rf_model (3).h        # Standalone Random Forest Model export (Iterative generation)
└── hb_rf_model (4).h        # Standalone Random Forest Model export (Iterative generation)

```

---

## 🛠️ Model Architecture & Specifications

### 1. Feature Order (`int16_t features[16]`)

To prevent configuration mismatches, features must map to the extraction array passed to `hb_dual_predict()` in this exact sequence:

| Index | Feature Key | Description / Type |
| --- | --- | --- |
| `[00]` | `hb_rf_corr` | Corrected initial Hb estimate |
| `[01]` | `hb_rf_raw` | Raw initial Hb estimate |
| `[02]` | `hb_pred_corrected` | Segmented structural validation factor |
| `[03]` | `hb_pred_raw` | Raw predicted hemoglobin metric |
| `[04]` | `std_ac_crest_660` | Peak-to-trough crest variability at 660nm |
| `[05]` | `pulse_width_50` | Pulse amplitude width at half-maximum (FWHM) |
| `[06]` | `spectral_curve` | Curvature profile derived from multi-spectral analysis |
| `[07]` | `iqr_apg_ba_ratio` | Interquartile range of the Accelerated PPG b/a wave ratio |
| `[08]` | `iqr_ac_ratio_530_660` | Optical ratios across green and red bands |
| `[09]` | `iqr_hrv_rmssd` | Heart Rate Variability RMSSD stability metric |
| `[10]` | `iqr_spectral_linear` | Feature matching frequency domain linearity |
| `[11]` | `demo_sex` | Demographic Boolean parameter (0 = Female, 1 = Male) |
| `[12]` | `hrv_sdnn` | Standard deviation of NN intervals |
| `[13]` | `ac_crest_850` | Crest parameter evaluation at 850nm |
| `[14]` | `iqr_hrv_sdnn` | Interquartile dispersion of SDNN intervals |
| `[15]` | `demo_age` | Patient age parameter |

### 2. Micro-Deployment Bounds

* **Regression Module**: 15 Decision Trees $\times$ Depth 5 (~44.1 KB).
* **Classification Module**: 10 Decision Trees $\times$ Depth 6 (~24.7 KB).
* **Total Resource Profile**: **~68.8 KB**, highly optimized for deployment targets under 100 KB.

---

## 💻 Edge C/C++ Implementation Example

The primary file `hb_dual_model.h` operates inside pure C/C++ environments (e.g., ESP32, STM32, or Arduino).

```c
#include "hb_dual_model.h"
#include <stdio.h>

void process_screening() {
    // 1. Prepare raw sensor inputs scaled to fixed-point (Value * 1000)
    int16_t sample_features[HB_N_FEATURES] = {
        13719, 13682, 13593, 13554, 152, 32767, 108, 1883, 27, 32767, 1, 0, 32767, 2083, 32767, 32767
    };

    // 2. Execute inference
    HbResult prediction;
    prediction = hb_dual_predict(sample_features, HB_N_FEATURES);

    // 3. Evaluate results
    printf("--- Diagnostic Screening Results ---\n");
    printf("Raw Predicted Hb      : %.2f g/dL\n", prediction.hb_raw);
    printf("Calibrated Hb Level   : %.2f g/dL\n", prediction.hb_calibrated);
    printf("Skin-Tone Category    : Group %d\n", prediction.skin_group);
    printf("Anemia Probability    : %.2f%%\n", prediction.anemia_prob * 100.0f);
    printf("Clinical Evaluation   : %s\n", prediction.anemia_flag ? "ANEMIC (ALERT)" : "NORMAL");
    printf("Confidence Interval   : Level %d\n", prediction.confidence);
}

int main() {
    process_screening();
    return 0;
}

```

---

## 🔬 Calibration Details

To handle skin tone variations across diverse patient demographics, the implementation scales and manages calibration maps directly within the firmware:

* **Group 1 (Light Skins)**: Slope scale factor $0.9648$ | Intercept $+0.5176$
* **Group 2 (Medium Skins)**: Slope scale factor $1.0649$ | Intercept $-0.8760$
* **Group 3 (Dark Skins)**: Slope scale factor $1.0000$ | Intercept $0.0000$

---

## 📈 Model Training and Generation Workflow

The model was built and verified within the `Final (1).ipynb` notebook following these structural steps:

1. Data ingestion and preprocessing of synchronized multi-wavelength PPG signal tracks.
2. Training of robust Python-based Random Forest estimators.
3. Feature scaling and transformation to `int16_t` metrics to remove floating-point overhead on the device.
4. Export and quantization using `emlearn` to generate ultra-light inline execution trees.
