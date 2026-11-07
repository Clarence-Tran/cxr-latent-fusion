# Data

> _What the pipeline expects on disk, how the study-level embedding caches must be built, and how to point a config at them._

---

## 🔒 Access and Licensing

This repository ships no data. MIMIC-CXR-JPG is a credentialed PhysioNet dataset, and its [data use agreement](https://physionet.org/content/mimic-cxr/view-dua/2.1.0/) prohibits redistributing the images or anything derived from them, embeddings included. Complete PhysioNet credentialing (CITI training + DUA), download [MIMIC-CXR-JPG v2.1.0](https://physionet.org/content/mimic-cxr-jpg/2.1.0/), and build the caches described below yourself.

---

## 🧊 Building the Study-Level Caches

The pipeline never reads raw images. It consumes two precomputed, study-level embedding caches, one per frozen encoder. The encoder inference code that produces them is not part of this repository. Build the caches once, following Section 4.1 of the paper; the pipeline applies no further view handling.

| Step | Rule |
|---|---|
| Preprocessing | Crop each image to 559 × 559, then resize to 518 × 518 |
| Encoders | [RAD-DINO](https://huggingface.co/microsoft/rad-dino) (768-d) and [BioViL-T](https://huggingface.co/microsoft/BiomedVLP-BioViL-T) (128-d), both frozen. BioViL-T receives only the current image, with no prior image or temporal input |
| View groups | `AP`, `PA` → frontal; `LATERAL`, `LL` → lateral. Average the embeddings within each group |
| Multi-view aggregation | Both groups present: `0.6 · frontal + 0.4 · lateral`. One group present: use that group's mean directly |
| Labels | The 14 CheXpert labels, binarized with U-Ones (positive and uncertain → 1, negative and unmentioned → 0) |
| Standardization | None. `StandardScaler` is fitted on the training partition inside the pipeline |

> **Labels must arrive already binary.** `build_study_base` coerces labels to numeric, fills missing values with 0 and clips to `[0, 1]`. A raw uncertain value (`-1`) would therefore silently become `0` (U-Zeros). Apply U-Ones when building the cohort table, not afterwards.

---

## 📁 Expected Files

Every input is named in the `paths:` block of the config. Relative paths are resolved against the directory that contains the config file.

### RAD-DINO cache — `raddino_dir`

| File | Config key | Contents |
|---|---|---|
| `v2_embedding.npy` (fallback `v1_onehot.npy`) | — | Feature matrix, one row per study. The RAD-DINO dimensions are the columns named `x_cxr_0` … `x_cxr_767` |
| `columns.json` | `columns_json` | Column schema per variant (`variants.<name>.columns`), the 14 label names (`labels`), and the row-index column name (`row_col`, default `row`) |
| `index.parquet` | `index_path` | Maps each matrix row (`row`) to its `study_id` |
| `cohort_study_level.parquet` | `cohort_path` | One row per study with `subject_id`, `study_id`, `index_time`, and the 14 binary label columns. Label columns are matched to `columns.json` by name, ignoring case, spaces, hyphens and underscores |
| `support_devices_update_audit.json` | `support_audit_path` | Optional provenance for the Support Devices label (`official_chexpert_source`, `study_coverage`, `support_policy`). When absent, the label is treated as native to the cohort table |
| `v2_embedding.parquet` | — | Optional parquet export of the same matrix, used only to audit the `.npy` against it |

### BioViL-T cache — `bio_cache_dir`

| File | Config key | Contents |
|---|---|---|
| `biovilt_study_embeddings.npy` | `bio_emb_path` | Feature matrix of shape `(n_studies, 128)` |
| `biovilt_study_manifest.parquet` | `bio_manifest_path` | `subject_id`, `study_id`, `bio_row` (row in the `.npy`), `row` (matching RAD-DINO row, kept for bookkeeping) |
| `cache_spec.json` | `bio_cache_spec` | `version`, plus optional `study_manifest_hash` and `source_jpg_accessed` |

The two caches are joined on `(subject_id, study_id)`, and only studies present in both enter the cohort. [`src/cxr_fusion/data.py`](../src/cxr_fusion/data.py) (`build_study_base`, `load_biovilt_cache`, `align_rad_and_bio`) is the authoritative reader.

---

## ⚙️ Pointing the Pipeline at Your Caches

```bash
cp configs/default.yaml configs/my_env.yaml
# edit the paths: block of configs/my_env.yaml
python scripts/prepare_data.py --config configs/my_env.yaml
```

`prepare_data.py` aligns the two caches, writes the aligned cohort and its audits under `<out_root>/checkpoints/<case_name>/`, and runs a linear-probe diagnostic on the raw embeddings as a sanity check before any training.

---

## 🔀 Other Cohort Variants

Caches built with a different aggregation rule (for example a frontal-only cohort) run unchanged: create one config per variant and point its `paths:` block at that cache. Only the aggregation above reproduces the paper's primary numbers.
