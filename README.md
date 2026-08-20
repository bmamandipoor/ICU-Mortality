# ICU-Mortality

Code for **"Development and external validation of a multimodal artificial intelligence
mortality prediction model of critically ill patients using multicenter data"**
(*Anesthesiology*, 2026).

A multimodal deep learning model that predicts in-hospital mortality risk for critically ill
patients using only data from the **first 24 hours** of their ICU admission. It combines four
modalities — time-invariant variables, time-variant variables (vital signs, laboratory values,
medication administration, ventilator changes, validated severity scores), clinical notes, and
chest X-ray images — and was developed on the MIMIC datasets and externally validated on
HiRID, eICU-CRD, and a temporally separated MIMIC-IV COVID-era population.

Across **203,434 ICU admissions from more than 200 hospitals (2001–2022)**, the model built on
structured data reached an AUROC of 0.92 (95% CI 0.90–0.93), AUPRC of 0.53, and Brier score of
0.19. External validation across eight individual eICU institutions gave AUROCs of 0.84–0.92.
Restricted to patients with notes and imaging available, adding those modalities improved AUROC
from 0.87 to 0.89, AUPRC from 0.43 to 0.48, and the Brier score from 0.37 to 0.17.

## Citation

> Mamandipoor B, Hsu C-N, Krause M, Schmidt UH, Gabriel RA. Development and external validation
> of a multimodal artificial intelligence mortality prediction model of critically ill patients
> using multicenter data. *Anesthesiology*. 2026. doi:10.1097/ALN.0000000000006294

Preprint: [arXiv:2512.19716](https://arxiv.org/abs/2512.19716)

## Data access

**No patient data is included in this repository, and none can be.** All four datasets are
credentialed-access resources distributed by PhysioNet and the HiRID project. To run this code
you must complete the required training and sign the applicable data use agreements, then
download the data yourself:

| Dataset | Version used | Where |
|---|---|---|
| MIMIC-III | v1.4 | https://physionet.org/content/mimiciii/ |
| MIMIC-IV | v2.2 and v3.1 | https://physionet.org/content/mimiciv/ |
| MIMIC-CXR (+ JPG) | — | https://physionet.org/content/mimic-cxr/ |
| MIMIC-IV-Note | — | https://physionet.org/content/mimic-iv-note/ |
| eICU-CRD | v2.0 | https://physionet.org/content/eicu-crd/ |
| HiRID | v1.1.1 | https://physionet.org/content/hirid/ |

Every notebook refers to its input data through a `"PATH TO DATA/..."` placeholder. Point those
at your own local copy before running anything.

> **Notebook outputs.** Stored outputs that rendered patient-level records — DataFrames keyed by
> `subject_id` / `hadm_id` / `stay_id` / `patientunitstayid`, clinical note text, and imaging
> identifiers — have been removed from this release, because redistributing them would breach the
> PhysioNet Credentialed Health Data Use Agreement. Aggregate results, cohort summary tables, and
> figures are retained.

## Repository layout

The pipeline runs in two stages. Within each directory, notebooks execute in the numeric order of
their filenames, and each expects to be run with its own directory as the working directory
(paths are relative, e.g. `../Data/`, `./Results/`).

### `Extraction/` — raw databases to per-ICU-stay tables

One tree per source: `MIMICIII/`, `MIMICIV/`, `MIMICIV(V3)/`, `eICU/`, `HiRID/` (each under `Code/`).

| Step | Purpose |
|---|---|
| `01. <DB>` | Schema exploration of every raw table |
| `02. Cohort_Selection` | Filters on age, ICU length of stay, first hospital and ICU admission, note/CXR availability |
| `03. Comorbidities_Extraction` | 30 Elixhauser comorbidity flags from ICD-9 / ICD-10 |
| `04. ICD_Code_Extraction` *(eICU only)* | Maps ICD-10 codes to the bundled embeddings |
| `04./05. Cohort_CSV_Extraction` | Pulls per-cohort CSVs from each event table |
| `05./06. Feature_Selection_byICU` | Builds per-ICU-stay feature tables |
| `06./07. Timeseries_Data_Extraction` | Outlier removal, hourly binning, categorical/continuous/static split |
| `07./08. Post-processing` | Missingness indicators, time-since-last-measurement, sample-and-hold imputation, and PaO2/FiO2, SIRS, Shock Index, SOFA, SAPS-II, OASIS |
| `09./03. Vitals_Processing` *(eICU, HiRID)* | Dense vital-sign feature engineering — PRSA, PSD, DFA, sample/approximate entropy, Lempel-Ziv complexity, FFT/STFT/DWT, desaturation and hypoxic burden |

HiRID follows a shorter path (`01. Extraction` → `02. Post-processing` → `03. Vitals_Processing`).

### `Prediction/` — cohorts, models, and results

`MIMICIII/`, `MIMICIV/`, `eICU/`, `HiRID/`, plus `External/{eICU,hirid,covid}/`.

| Step | Purpose |
|---|---|
| `01. Processing` | Assembles the 0–24 h matrix, maps race, builds the stratified train/validation/test split, cleans notes, and produces the cohort characteristics table |
| `02. First Step` | EHR-only models |
| `03. Second Step` | Adds clinical notes |
| `04. Third Step` *(MIMIC-IV only)* | Adds chest X-rays. `01.`/`02.` train the DenseNet121 image encoder in two preprocessing variants and export `cxr_densnet_representation.csv`, which `03.`–`06.` consume |
| `04./05. Risk Scores` | SOFA / SAPS-II / OASIS baselines and the deep-learning-versus-risk-score comparison |
| `Results*` | Pools saved prediction frames into combined ROC, PR, calibration, and metric tables |

`Prediction/External/covid/` is **not** a separate hospital cohort: it is MIMIC-IV v3.1
restricted to `anchor_year_group == '2020 - 2022'`.

### Models

Logistic regression and XGBoost, PCA plus autoencoder, a feed-forward MLP, a bidirectional LSTM
(the main EHR baseline), LSTM with attention, RETAIN, BERT over clinical notes, a DenseNet121 CXR encoder (in two
preprocessing variants — a single processed view replicated across channels, and a three-channel
composite of raw, contrast-enhanced, and Sobel edge views), and four multimodal fusion designs — late-fusion pooling (concatenation / addition /
gated / bilinear), multi-head cross-attention, and Deep & Cross.

Every model notebook additionally performs isotonic calibration, bootstrap confidence intervals,
algorithmic fairness auditing with `aequitas` over sex, age group, race, and ethnicity, and
interpretability analysis with `captum` and `bertviz`.

### Reference data included

* `Extraction/eICU/Data/Embeddings_ICD10/` — pretrained 10/20/50-dimensional ICD-10 code
  embeddings and their code-to-label metadata.
* `Prediction/{MIMICIII,MIMICIV}/Data/Abbreviations/Abbreviation.txt` — a 1,667-entry clinical
  abbreviation dictionary used to expand abbreviations during note preprocessing.

## Environment

Python 3.11 with a CUDA-capable GPU (the training notebooks monitor GPU state via `GPUtil` and
`pynvml`).

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

See the provenance note at the top of `requirements.txt`.

## License

MIT — see [LICENSE](LICENSE). The article itself is published under CC BY-NC-ND 4.0.

## Funding

This study was funded by the National Institutes of Health (1OT2OD03799501).
