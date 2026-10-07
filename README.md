# DermaFusion

Deep learning for **skin-lesion classification** from dermoscopic and smartphone images. The final system is a two-model EfficientNet-B4 ensemble over 8 lesion classes (MEL, NV, BCC, AKIEC, BKL, DF, VASC, OTHER), trained on four public dermatology datasets and shipped as both a Gradio research demo and a native iOS app.

This repository holds the **data and evaluation side** of the ML pipeline: dataset loading, colour constancy and hair removal, leakage-safe splits, a lesion-aware class-balanced sampler, augmentations, and the metrics, calibration, fairness and statistics modules, along with training configs, analysis notebooks and the Gradio demo entry points.

> **Full pipeline** (model definitions, trainer, command-line scripts, tests, Core ML export): [ManikantaSirumalla/DermaFusion](https://github.com/ManikantaSirumalla/DermaFusion).
>
> **Project page:** [rahulreddykota.github.io/dermafusion.html](https://rahulreddykota.github.io/dermafusion.html).

> [!WARNING]
> **Research use only — not a medical device.** DermaFusion is an academic research project. It is not FDA/CE cleared, has not been clinically validated, and must not be used to diagnose, screen, or make decisions about any real skin condition. Concerning skin lesions should always be evaluated by a qualified clinician.

---

## Headline results

Final **E0 + E3 soft-vote ensemble** (EfficientNet-B4 × 2, 380 × 380), evaluated on the held-out **TEST** set (*n* = 5,279) with 1,000-iteration bootstrap 95% confidence intervals.

| Metric | Value (95% CI) |
| --- | --- |
| 8-class balanced accuracy | **0.540** [0.509, 0.568] |
| 8-class macro F1 | **0.469** [0.441, 0.495] |
| Top-1 / Top-2 / Top-3 accuracy | 0.778 / 0.907 / 0.954 |
| Binary-malignant AUROC | 0.830 |
| Sensitivity @ 95% specificity | 0.462 |

These are the results of the complete system, which is trained by the end-to-end Colab notebook in the [full pipeline repository](https://github.com/ManikantaSirumalla/DermaFusion). That repository also has the per-class breakdown and the ISIC 2019 leaderboard comparison. The code here comes from the project's HAM10000 / ISIC 2018 Task 3 (7-class) stage.

> **Reading the headline number.** 0.540 balanced accuracy is across **8 unified classes spanning four datasets** (dermoscopic *and* smartphone images), which is a harder setting than a single-dataset 8-class benchmark. Top-2 accuracy of 0.907 and a binary-malignant AUROC of 0.830 are the more clinically meaningful framing of the same model.

---

## What's in this repo

```
DermaFusion-Skin-Cancer/
├── src/
│   ├── data/
│   │   ├── dataset.py          # HAM10000Dataset: image, metadata and label per sample
│   │   ├── preprocessing.py    # Shades-of-Gray, DullRazor, metadata encoding, lesion-grouped splits
│   │   ├── sampler.py          # ClassBalancedSampler: class-balanced and lesion-aware
│   │   └── transforms.py       # Train, validation and 8x test-time augmentation transforms
│   └── evaluation/
│       ├── metrics.py          # Balanced accuracy, macro F1, per-class sensitivity / AUROC / AUPRC
│       ├── calibration.py      # Expected calibration error, reliability diagram, temperature scaling
│       ├── fairness.py         # Demographic parity, equalized odds, per-group accuracy
│       └── statistical.py      # Bootstrap confidence intervals, McNemar test
├── configs/
│   ├── data/ham10000.yaml      # Data paths, 380 px images, split sizes and seed
│   └── training/
│       ├── default.yaml        # AdamW, cosine schedule with warm-up, focal loss
│       └── finetune.yaml       # Short low-learning-rate fine-tuning run
├── notebooks/                  # EDA, training and results analysis, Colab training
├── demo/
│   └── app.py                  # Gradio research demo
├── app/
│   └── gradio_demo.py          # Launcher for the demo
├── COLAB_SETUP.md              # Step-by-step Colab training guide
├── run_train_colab.py          # Colab launcher for scripts/train.py
├── requirements.txt
├── pyproject.toml
├── LICENSE
└── README.md
```

**Not in this repository:** `src/models`, `src/training`, `src/utils`, `src/evaluation/interpretability.py`, `scripts/`, `tests/` and the Core ML export. They are in the [full pipeline repository](https://github.com/ManikantaSirumalla/DermaFusion), which uses the same folder layout.

---

## Architecture summary

The final system, as built by the full pipeline:

| Stage | Description |
| --- | --- |
| **Data** | ISIC 2018 / 2019 / 2020 + PAD-UFES-20 → **61,694** dermoscopic + smartphone images, 8 unified classes |
| **Splits** | Patient-grouped (`GroupShuffleSplit` on `patient_id` → `lesion_id`) with iterative leakage enforcement → **TRAIN 52,483 / VAL 3,932 / TEST 5,279**, zero patient/lesion overlap |
| **Preprocessing** | Shades-of-Gray colour constancy (power = 6) + DullRazor hair removal + resize to 380 × 380 |
| **Backbone** | EfficientNet-B4 (~17.6 M params) — two members trained from different starting points |
| **E0** | `efficientnet_b4`; CB-Focal (γ = 1.5) + MixUp + TrivialAugmentWide + EMA 0.999; √-inverse sampler |
| **E3** | `tf_efficientnet_b4.ns_jft_in1k`; class-balanced focal (γ = 2.0); full-inverse sampler |
| **Ensemble** | Soft-vote with Dirichlet-optimised weights `[0.705, 0.295]`; per-class thresholds tuned on VAL macro-F1 |
| **Deployment** | Gradio (single 142 MB ensemble bundle) + iOS (two FP16 `.mlpackage` files, 34 MB each) |

---

## Setup

```bash
git clone https://github.com/RahulReddyKota/DermaFusion-Skin-Cancer.git
cd DermaFusion-Skin-Cancer
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Requires **Python 3.10 or 3.11** (the pinned scikit-learn has no prebuilt package for 3.12) and **PyTorch 2.1+**.

---

## Use the data and evaluation modules

The modules under `src/data` and `src/evaluation` run on their own. Run Python from the repository root so that `src` is importable.

**Leakage-safe splits and metadata encoding.** `load_metadata` expects a HAM10000-style CSV with `lesion_id`, `image_id`, `dx`, `dx_type`, `age`, `sex` and `localization` columns.

```python
from src.data.preprocessing import create_splits, encode_metadata, load_metadata

df = load_metadata("data/raw/metadata/HAM10000_metadata.csv")

# Fixed test hold-out plus 5 train/val folds, all grouped by lesion_id.
splits = create_splits(df, n_folds=5, seed=42, test_size=0.15)
train_idx, val_idx = splits.train_val_folds[0]

# Fit the encoding statistics on the training fold only, then reuse them.
train_df, stats = encode_metadata(df.iloc[train_idx])
val_df, _ = encode_metadata(df.iloc[val_idx], stats=stats)
```

**Dataset, augmentations and balanced sampling.**

```python
from torch.utils.data import DataLoader

from src.data.dataset import HAM10000Dataset
from src.data.sampler import ClassBalancedSampler
from src.data.transforms import get_train_transforms

label_encoder = {"mel": 0, "nv": 1, "bcc": 2, "akiec": 3, "bkl": 4, "df": 5, "vasc": 6}
train_df["label"] = train_df["dx"].map(label_encoder)

dataset = HAM10000Dataset(train_df, "data/raw/images", transform=get_train_transforms(image_size=380))
sampler = ClassBalancedSampler(dataset.df, batch_size=32)
loader = DataLoader(dataset, batch_size=32, sampler=sampler)

batch = next(iter(loader))  # keys: image, metadata, label, image_id, lesion_id
```

**Evaluation.** `predictions` and `labels` are integer NumPy arrays; `probabilities` has one column per class.

```python
import numpy as np

from src.evaluation import (
    MetricCalculator,
    bootstrap_confidence_intervals,
    expected_calibration_error,
)

calculator = MetricCalculator(num_classes=7, class_names=list(label_encoder))
calculator.update(predictions, labels, probabilities)  # call once per batch
results = calculator.compute()  # balanced_accuracy, macro_f1, per_class, confusion_matrix, ...

ece = expected_calibration_error(probabilities, labels)
low, high = bootstrap_confidence_intervals(labels, predictions, lambda y, p: float(np.mean(y == p)))
```

---

## Gradio demo

`demo/app.py` (also launched through `app/gradio_demo.py`) is the research demo: a dermoscopic image and optional age, sex and lesion location go in; class probabilities and a Grad-CAM or attention overlay come out.

It does not run from this repository alone. It imports the model factory, config loader and Grad-CAM helpers (`src.models`, `src.utils`, `src.evaluation.interpretability`) from the full pipeline repository, and it looks for a trained checkpoint in the `DERMAFUSION_CKPT` environment variable or under `outputs/checkpoints/`.

---

## Training

Training needs the model and trainer code, so both [`COLAB_SETUP.md`](COLAB_SETUP.md) and [`notebooks/Colab_Train_HairRemoved.ipynb`](notebooks/Colab_Train_HairRemoved.ipynb) work from a copy of the full pipeline repository (cloned from GitHub or kept on Google Drive) and run its `scripts/train.py` on a Colab GPU with hair-removed images. `run_train_colab.py` is the launcher for that script.

`configs/` holds the Hydra data and training configs:

- **`configs/data/ham10000.yaml`** — HAM10000 metadata and image paths, 380 px input, 15% test and validation fractions, 5 folds, split seed 42.
- **`configs/training/default.yaml`** — AdamW (backbone 1e-4, head 1e-3), batch 32 with 4-step gradient accumulation, cosine annealing with 5 warm-up epochs, mixed precision, early stopping, and focal loss (γ = 2.0) with extra cost on melanoma and BCC false negatives.
- **`configs/training/finetune.yaml`** — 20-epoch fine-tuning at a lower learning rate with a cost-sensitive loss.

---

## Notebooks

| Notebook | Contents |
| --- | --- |
| [`01_eda.ipynb`](notebooks/01_eda.ipynb) | EDA for ISIC 2018 Task 3 / HAM10000: class distribution, age by diagnosis, images per lesion, metadata correlations, image sizes and a sample grid |
| [`02_Baseline.ipynb`](notebooks/02_Baseline.ipynb) | Outline for the image-only baselines (EfficientNet-B4, Swin-T, ConvNeXt-V2) |
| [`03_training_analysis.ipynb`](notebooks/03_training_analysis.ipynb) | Loss and balanced-accuracy curves from saved training histories |
| [`04_results_analysis.ipynb`](notebooks/04_results_analysis.ipynb) | Ablation comparison table and chart, plus a check for the saved confusion-matrix, ROC, reliability and fairness outputs |
| [`05_Analysis.ipynb`](notebooks/05_Analysis.ipynb) | Outline for the final analysis |
| [`Colab_Train_HairRemoved.ipynb`](notebooks/Colab_Train_HairRemoved.ipynb) | GPU training run on Colab with hair-removed images |

The notebooks are saved without outputs. Notebooks 03 and 04 read files under `outputs/`, which a training run produces.

---

## References

- **ISIC 2018 Task 3** — <https://challenge.isic-archive.com/landing/2018/>
- **ISIC 2019** — <https://challenge.isic-archive.com/landing/2019/>
- **HAM10000** — Tschandl et al., *Scientific Data* (2018)
- **PAD-UFES-20** — Pacheco et al., *Data in Brief* (2020)
- **Shades-of-Gray colour constancy** — Finlayson & Trezzi (2004)
- **DullRazor hair removal** — Lee et al. (1997)
- **Class-balanced focal loss** — Cui et al., *CVPR* (2019)

---

## License

[MIT](LICENSE).
