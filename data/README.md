# Data

The raw data are **not included** in this repository (tens of GB).

## Dataset

**SeizeIT2** – wearable behind-the-ear EEG recorded in epilepsy monitoring units, in BIDS format.

* OpenNeuro accession: **ds005873** – https://openneuro.org/datasets/ds005873
* DOI: [10.18112/openneuro.ds005873.v1.0.0](https://doi.org/10.18112/openneuro.ds005873.v1.0.0)
* The licence and terms of use are given on the OpenNeuro page and in `dataset_description.json`.

## Download

Download the dataset from the OpenNeuro page (or with the OpenNeuro command-line tools) and place it here, so that
the folder looks like this:

```
data/
└── seizeit2/
    ├── participants.tsv
    ├── sub-001/
    │   └── ses-01/
    │       └── eeg/
    │           ├── sub-001_ses-01_task-szMonitoring_run-01_eeg.edf
    │           └── sub-001_ses-01_task-szMonitoring_run-01_events.tsv
    ├── sub-002/
    └── ...
```

Only the `eeg/` folders are used (the `ecg/`, `emg/` and `mov/` folders are not needed).
If the data are stored elsewhere, set the environment variable `SEIZEIT2_DATA` to that folder.
