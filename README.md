# Automated seizure detection from behind-the-ear EEG (SeizeIT2)

Scientific Programming project – Ioannis Vakkas, October 2026

## Purpose

Wearable EEG with two electrodes behind the ears could monitor people with epilepsy at home, but every
false alarm wakes the patient or a carer. This project builds a seizure detector for **two behind-the-ear EEG
channels** and evaluates it **on patients the model has never seen**, reporting sensitivity together with
false alarms per hour.

**Research questions**

1. **RQ1** – Does detection performance depend on the **seizure type**?
2. **RQ2** – Does detection performance depend on the **seizure duration**?

## Main results (125 patients, nested patient-wise 10-fold cross-validation)

| Operating point | Event sensitivity | False alarms / hour |
|---|---|---|
| **Threshold for ≤ 1 false alarm/h (main)** | **55.9 %** (494 / 883) [44.3–68.6 %] | **0.93** [0.79–1.07] |
| Fixed threshold 0.5 (as in the SeizeIT2 paper) | 57.6 % | 0.99 |
| Threshold for ≤ 2 false alarms/h | 69.4 % | 1.94 |
| Threshold for ≤ 3 false alarms/h | 77.6 % | 2.91 |

95 % confidence intervals: bootstrap over patients (2000 resamples).

* **RQ1:** focal-to-bilateral tonic-clonic seizures are detected in 98 % of cases (54 / 55); the focal types
  (44–61 %) have overlapping confidence intervals. Tonic-clonic seizures are also the longest, so type and
  duration are linked.
* **RQ2:** seizures of ≤ 30 s are detected in 32 % of cases, seizures longer than 60 s in 84 %; the intervals
  do not overlap. Median detection latency is 19 s.

The full results, figures and limitations are in the notebook (Sections 7–11) and in [`figures/`](figures/).

## Method in short

1. **Data:** SeizeIT2 (BIDS), first two EEG channels of every recording (behind-the-ear), 256 Hz.
2. **Pre-processing (per recording, label-free):** volts → µV, band-pass 1–25 Hz, 2 s windows with 1 s step.
3. **Features:** 21 per channel (time domain, band powers, 40–80 Hz muscle band, entropies) = 42; log transform
   and a causal running normalisation per recording; plus 42 *context* features (each feature minus its mean over
   the previous 60 s) = **84 features**.
4. **Artifact rule:** a window is clean if the RMS of both channels is 2–150 µV (amplitude only, never labels).
5. **Model:** XGBoost, **nested patient-wise cross-validation**:
   * outer: 125 patients → 10 folds (seed 42); each fold is the test set once;
   * inner: training patients split again by patient (80 % fit / 20 % validation);
   * Optuna (15 trials, AUPR) and the decision threshold are chosen on the validation patients;
   * training uses all seizure windows + 5 × as many clean background windows (undersampling, training patients only);
   * the final model is retrained on all training patients and evaluated **once** on the test patients.
6. **Alarm:** ≥ 8 of the last 10 windows above the threshold.
7. **Scoring:** event-based SzCORE rules (30 s before / 60 s after tolerance, 90 s merge, 5 min split),
   cross-checked with the official `timescoring` library.

## Repository structure

```
seizure-detection-seizeit2/
├── README.md                 this file
├── requirements.txt          Python packages (versions used for the results)
├── notebooks/
│   └── seizure_detection_xgboost.ipynb   complete analysis, with outputs
├── data/
│   └── README.md             how to download SeizeIT2 (data are not included)
├── figures/                  figures produced by the notebook
├── results/                  generated tables / checkpoints (created when the notebook runs)
└── presentation/
    └── Seizure_Detection_SeizeIT2.pptx   final presentation
```

Generated files (`processed_v3/`, `results/`, `figures/`) are kept separate from the raw data and the code.

## How to run

### 1. Software

* Python **3.11** (results were produced with Python 3.11.9 on Windows 10, 4 CPU cores, 4 GB RAM)
* Install the packages:

```bash
python -m venv .venv
# Windows: .venv\Scripts\activate      macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
```

### 2. Data

Download SeizeIT2 into `data/seizeit2/` – see [`data/README.md`](data/README.md).

### 3. Run the notebook

```bash
cd notebooks
jupyter notebook seizure_detection_xgboost.ipynb
```

Run all cells from top to bottom. No manual steps are needed.

* **Paths.** By default the project folder is the repository root and the data are read from
  `data/seizeit2/`. Other locations can be set with environment variables:
  `SEIZEIT2_DATA` (data folder) and `SEIZURE_PROJECT` (where processed data, results and figures are written).
* **Run time.** Feature extraction (Cell 3) and the cross-validation (Cell 6) take several hours on a laptop.
  Both steps are **resumable**: processed subjects (`processed_v3/*.npz`) and finished folds
  (`results/loso_folds/fold_XX.pkl`) are skipped when the cell is run again.
* **Quick test.** Set the environment variable `QUICK_TEST=1` to run the whole pipeline on 10 subjects with
  4 Optuna trials (minutes instead of hours).
* **Optional cells.** The comparison with the 42-feature baseline (Sections 9b, 10c-1) needs a second run with
  `CONTEXT_SEC = 0` saved in `results_baseline/`; it is skipped if that folder does not exist. The SzCORE
  cross-check (Section 10d) needs the `timescoring` package.

## Reproducibility

* All parameters are defined in **Cell 1** (window length, filter, artifact rule, folds, Optuna settings,
  alarm rule, scoring tolerances).
* A fixed random seed (**42**) is used for the patient folds, the inner splits, undersampling, Optuna and XGBoost.
* Cell 1a writes `results/environment.json` (software versions), `results/config_used.json` (all parameters)
  and `requirements_lock.txt`.
* The patient-to-fold assignment is saved in `results/folds.csv`.

## Limitations

* Hospital (epilepsy monitoring unit) recordings, not home use; no external test set.
* Only two EEG channels; channel 2 is the cross-head channel in most recordings and the contralateral
  behind-the-ear channel in the others.
* Two design choices (2 µV artifact limit, context features) were made after seeing earlier cross-validation
  results on the same patients, so a small optimistic bias cannot be excluded.
* One random seed; a different patient split could change the numbers slightly.

## References

* SeizeIT2 dataset: OpenNeuro **ds005873**, doi:[10.18112/openneuro.ds005873.v1.0.0](https://doi.org/10.18112/openneuro.ds005873.v1.0.0)
* Bhagubai et al. (2025) – SeizeIT2 dataset and baseline seizure detection (alarm rule 8 of 10, 13–150 µV artifact rule).
* Dan et al. (2024) – SzCORE: a framework for evaluating seizure detection algorithms; `timescoring` library.

## Practical notes for a full run

* **Dataset size.** The full SeizeIT2 dataset is about **120 GB**. Only the `eeg/` folders are used
  (the `ecg/`, `emg/` and `mov/` folders are not needed).
* **Disk and memory.** In addition to the raw data, about **15 GB** of free disk space is needed for the
  extracted features (`processed_v3/` and `X_all.npy`), and about **4 GB of RAM**.
* **Data in another folder.** If the data are not in `data/seizeit2/`, set the path in the same terminal
  before starting Jupyter (replace the example path with your own):

```bash
  # Windows (PowerShell)
  $env:SEIZEIT2_DATA = "C:\path\to\seizeit2"
  # macOS / Linux
  export SEIZEIT2_DATA=/path/to/seizeit2
```

  The quick test is switched on in the same way (`$env:QUICK_TEST = "1"` or `export QUICK_TEST=1`).
* **Run time.** A full run takes several hours on a laptop; keep the computer from going to sleep.
  If it stops, run the notebook again: finished patients and folds are skipped.
* **Expected output.** A full run reproduces the results table above: **125 patients, 883 seizures,
  494 detected (55.9 %) at 0.93 false alarms per hour**, train AUROC 0.969 and test AUROC 0.894.
