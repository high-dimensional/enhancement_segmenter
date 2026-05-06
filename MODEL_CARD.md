---
language: en
license: apache-2.0
library_name: nnunet
tags:
  - medical-imaging
  - mri
  - brain-tumour
  - segmentation
  - neuro-oncology
  - nnu-net
  - contrast-enhancement
  - non-contrast-mri
datasets:
  - BraTS
metrics:
  - dice
  - balanced-accuracy
  - sensitivity
  - specificity
pipeline_tag: image-segmentation
---

# Enhancement Segmenter — Model Card

A 3D nnU-Net v2 model that predicts which regions of a brain tumour *would* enhance after gadolinium contrast, using **only non-contrast** MRI sequences (FLAIR, T1, T2) as input.

## Model details

| | |
|---|---|
| **Model name** | Enhancement Segmenter |
| **Version** | v1 (Zenodo deposit, May 2026) |
| **Architecture** | Residual Encoder U-Net (3D fullres), nnU-Net v2 |
| **Trainer** | `nnUNetTrainer_4000epochs` |
| **Plans identifier** | `nnUNetResEncUNetPlans_80G` |
| **Configuration** | `3d_fullres` |
| **Folds** | 5 (full cross-validation ensemble) |
| **Inputs** | 3-channel volumetric MRI: FLAIR, T1 (non-contrast), T2 |
| **Outputs** | 4-class voxel segmentation (0 = background, 1 = brain parenchyma, 2 = non-enhancing abnormality, 3 = predicted enhancing tumour) |
| **License** | [Apache 2.0](LICENSE) |
| **Code** | <https://github.com/jamesruffle/enhancement_segmenter> |
| **Weights** | [![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.20055549.svg)](https://doi.org/10.5281/zenodo.20055549) |
| **Paper** | [arXiv:2508.16650](https://arxiv.org/abs/2508.16650) |

### Authors

Ruffle JK, Mohinta S, Pombo G, Biswas A, Campbell A, Davagnanam I, Doig D, Hamman A, Hyare H, Jabeen F, Lim E, Mallon D, Owen S, Wilkinson S, Brandner S, Nachev P.

## Uses

### Direct use

Research and benchmarking applications in computational neuro-oncology that benefit from a contrast-free estimate of tumour enhancement, including:

- Methodology development for contrast-free MRI in brain tumour imaging.
- Comparative studies of how non-contrast vs. contrast-enhanced segmentation performs on downstream tasks.
- Educational and demonstration use in radiology AI courses.
- Pre-screening or quality-control workflows where no contrast study is available.

### Downstream use

The 4-class output mask can be used as input to downstream pipelines (radiomics feature extraction, longitudinal volumetry, registration targets, treatment-planning auxiliaries). The model is a starting point — downstream applications should be re-validated end-to-end on the target population.

### Out-of-scope use

- **Standalone clinical diagnosis or treatment decisions.** This model is **not** a regulatory-cleared medical device. Outputs must be reviewed by a qualified radiologist and interpreted alongside the full clinical picture.
- **Replacement for contrast-enhanced MRI** in patients where contrast is clinically indicated.
- **Surveillance imaging** in routine practice without prospective validation in the target setting.
- **Populations / scanners / pulse sequences not represented in training** (e.g., very-low-field MRI, unusual contrast weightings, paediatric scanners not represented in BraTS-PEDs).
- **Discrimination of enhancement subtypes** (rim-enhancing vs. solid-enhancing vs. nodular) — the model produces a single binary "would-enhance / would-not-enhance" mask.

## Bias, risks, and limitations

- **The headline metric of interest is the enhancing-tumour class, where Dice is moderate.** Cross-validation Dice for class 3 (predicted enhancing tumour) is approximately 0.55, compared to 0.99 for brain parenchyma and 0.82 for non-enhancing abnormality. On the held-out test set, ~50% of patients have enhancing-tumour Dice > 0.7 and ~32% have Dice ≤ 0.3.
- **Training data was assembled from 10 international research datasets**, predominantly higher-field (1.5–3 T) clinical scanners with research-grade pulse sequences. Performance on out-of-distribution sequences (e.g., 7 T research scans, low-field portable scanners, motion-corrupted clinical scans) is unverified.
- **Adult and paediatric subjects are both represented**, but paediatric coverage is smaller; expect higher uncertainty in paediatric subgroups, especially in rare tumour types.
- **The model was trained on cases with known intracranial pathology.** Behaviour on entirely healthy scans is not formally evaluated — predicted enhancing tumour voxels may appear in some healthy scans and should not be interpreted as a positive finding.
- **Skull-stripping / brain extraction is not performed by this model.** Inputs are expected to follow nnU-Net's standard preprocessing.
- **Predicted enhancement is a learned proxy, not a measurement.** The model has not been compared head-to-head against gadolinium-enhanced ground truth in prospective clinical workflows.
- **Risk of automation bias.** Users — particularly less experienced readers — may over-rely on the segmentation. Outputs should be inspected manually before any clinical interpretation.

## How to get started

```bash
# 1. Install nnU-Net v2
pip install nnunetv2

# 2. Download model weights from Zenodo and extract
#    https://doi.org/10.5281/zenodo.20055549
unzip enhancement_segmenter_weights.zip -d /path/to/nnUNet_results/

# 3. Set the results directory
export nnUNet_results="/path/to/nnUNet_results"

# 4. Run inference (input files: subject_0000 = FLAIR, _0001 = T1, _0002 = T2)
nnUNetv2_predict \
    -d Dataset003_enhance_and_abnormality_batchconfig \
    -i INPUT_FOLDER -o OUTPUT_FOLDER \
    -f 0 1 2 3 4 \
    -tr nnUNetTrainer_4000epochs \
    -c 3d_fullres \
    -p nnUNetResEncUNetPlans_80G \
    -chk checkpoint_best.pth
```

A user-friendly Python wrapper is provided at [`predict.py`](predict.py).

## Training details

### Training data

11,089 brain MRI studies aggregated from 10 international datasets (9,980 used for 5-fold cross-validation training, 1,109 held out as a final test set). Cohorts span glioma, meningioma, brain metastases, and post-resection appearances, in both adult and paediatric populations. See the paper for the full dataset breakdown.

### Training procedure

- **Framework**: nnU-Net v2 (auto-configured pipeline)
- **Architecture**: Residual Encoder U-Net (3D fullres), 6 stages, 32–320 channels, instance normalisation, leaky-ReLU activation
- **Patch size**: 160 × 192 × 160 voxels
- **Spacing**: 1.0 × 1.0 × 1.0 mm (resampled)
- **Normalisation**: Z-score per channel within the brain mask
- **Loss**: nnU-Net default Dice + cross-entropy (deep supervision)
- **Optimiser**: SGD with Nesterov momentum (lr 0.01, decay schedule per nnU-Net default)
- **Epochs**: 4,000 (trainer variant `nnUNetTrainer_4000epochs`)
- **Cross-validation**: 5-fold

### Speeds, sizes, times

- **Checkpoint size**: ~821 MB per fold × 5 folds ≈ 4 GB total
- **Inference time**: ~30–90 s per case on a 48 GB consumer/workstation GPU (5-fold ensemble, 3D fullres)
- **Training time**: multi-week per fold on a single NVIDIA RTX 6000 Ada (48 GB)

## Evaluation

### Testing data, factors, metrics

- **Test set**: 1,109 brain MRI studies held out from the training pool, spanning the same diagnostic categories as the training set.
- **Primary metric**: per-patient Dice score for the predicted enhancing tumour class.
- **Secondary metrics**: balanced accuracy, sensitivity, and specificity for patient-level detection of any enhancement.

### Results

| Metric | Value |
|---|---|
| Balanced accuracy (patient-level enhancement detection) | **83 %** |
| Sensitivity | **91.5 %** |
| Specificity | **74.4 %** |
| Test-set Dice (enhancing tumour, fraction of patients exceeding threshold) | 76.8 % > 0.3; 67.5 % > 0.5; 50.2 % > 0.7 |

Cross-validation Dice per class (mean across the 5 training folds, reported by nnU-Net's internal validation):

| Class | Cross-val Dice |
|---|---|
| Brain parenchyma | 0.99 |
| Non-enhancing abnormality | 0.82 |
| Predicted enhancing tumour | 0.55 |
| (Foreground mean) | 0.79 |

## Environmental impact

Training was carried out on NVIDIA RTX 6000 Ada GPUs (~300 W TDP). Estimated cumulative GPU-hours for the 5-fold ensemble are on the order of several hundred to ~1,000 GPU-hours, depending on data-loader throughput. Inference is comparatively cheap (a few minutes per case on the same hardware). Carbon impact has not been formally quantified; users are encouraged to share an inference container rather than retrain when possible.

## Technical specifications

| Component | Version (verified) |
|---|---|
| Operating system | Ubuntu 22.04.5 LTS, Linux 6.8.0, x86_64 |
| GPU hardware | NVIDIA RTX 6000 Ada Generation (48 GB VRAM) |
| NVIDIA driver | 580.105.08 |
| CUDA toolkit | 13.0 (build `cu130`) |
| cuDNN | 9.1.x |
| Python | 3.10.12 |
| PyTorch | 2.11.0 (`+cu130` build) |
| nnU-Net | v2 (≥ 2.2) |

The Python dependency manifest is in [`requirements.txt`](requirements.txt). Inference is feasible on any CUDA-capable GPU with ≥ 8 GB VRAM (e.g., consumer RTX 30/40-series). Training as configured here requires ≥ 48 GB of GPU VRAM.

## Citation

```bibtex
@article{ruffle2025predicting,
  title   = {Predicting brain tumour enhancement from non-contrast MR imaging with artificial intelligence},
  author  = {Ruffle, James K and Mohinta, Samia and Pombo, Guilherme and
             Biswas, Asthik and Campbell, Alan and Davagnanam, Indran and
             Doig, David and Hamman, Ahmed and Hyare, Harpreet and
             Jabeen, Farrah and Lim, Emma and Mallon, Dermot and
             Owen, Stephanie and Wilkinson, Sophie and Brandner, Sebastian
             and Nachev, Parashkev},
  journal = {arXiv preprint arXiv:2508.16650},
  year    = {2025}
}
```

Please also cite nnU-Net (Isensee et al., *Nature Methods* 2021).

## Model card authors

James K. Ruffle, on behalf of the paper's author list. Open an issue at <https://github.com/jamesruffle/enhancement_segmenter/issues> for questions or feedback.

## More information

- Repository: <https://github.com/jamesruffle/enhancement_segmenter>
- Pretrained weights (Zenodo): <https://doi.org/10.5281/zenodo.20055549>
- Preprint: <https://arxiv.org/abs/2508.16650>
- Public dataset to try the model on: BraTS challenge — <https://www.synapse.org/Synapse:syn53708126/wiki/626320>
