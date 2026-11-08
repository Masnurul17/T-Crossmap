# T-Crossmap
# T-CrossMap

**A Taguchi-Based Design-of-Experiments Framework for Point-Cloud Preprocessing of a Cross-Modal Mapper in RGB-3D Defect Detection**
<p align="center">
  <img src="figures/fig1_framework_illustr.png" width="90%" alt="T-CrossMap framework">
</p>

**Figure 1.** Overview of T-CrossMap. RGB and organised XYZ inputs pass through the preprocessing factors A–C (background removal, RANSAC threshold, XYZ interpolation) and the Point-MAE factors D–F (PFA neighbours, FPS centres, points per group). Frozen DINO ViT-B/8 and Point-MAE features are mapped across modalities by two MLP mappers; their disagreement gives the anomaly map Ψ = Ψ2D · Ψ3D and the image score s = max Ψ. I-AUROC, AUPRO@30% and latency from each run feed the Taguchi–grey relational analysis, which selects the optimum levels of A–F. *The example images and maps are illustrative, not detector output.*
> Manuscript under preparation for the *Journal of Engineering and Technological Sciences* (JETS, ITB).
> Code, configuration files and trained mappers will be released after publication.

---

## Overview

Industrial visual inspection has to find rare defects on products whose appearance and shape vary, usually without defect samples for training. Cross-modal detectors that combine a colour image (RGB) with a 3D point cloud from a depth camera handle this well, but their accuracy depends heavily on how the point cloud is preprocessed. In practice these settings are chosen by trial and error.

**T-CrossMap** turns that choice into a systematic procedure:

- **T** stands for **Taguchi**. A design-of-experiments framework (orthogonal arrays, signal-to-noise ratio, grey relational analysis and ANOVA) selects the preprocessing settings.
- **CrossMap** is the **cross-modal mapper** whose input is being prepared. It is a detector built on frozen DINO ViT and Point-MAE features and trained only on defect-free samples, following the CFM approach.

The framework tunes the **preprocessing**. The detector architecture stays fixed.

## Contributions

1. **A systematic way to choose 3D preprocessing settings.** A staged Taguchi design (18 screening runs plus 9 refinement runs per operating mode) replaces a full search over 486 combinations. Engineers can repeat it when a new product is introduced.
2. **Insight into which settings matter.** ANOVA shows that XYZ interpolation dominates the result. A two-way response table reveals a strong interaction between background removal and interpolation, so one-factor-at-a-time tuning can lead to a poor choice.
3. **A balance between accuracy and speed.** Grey relational analysis combines detection accuracy, localisation accuracy and latency into one score. Two operating points result: an accuracy setting and a fast, all-GPU setting.
4. **Deployment feasibility.** Both settings are benchmarked on a GPU server and on an NVIDIA Jetson AGX Orin edge device, including latency, power and energy.

## Method

### Detector (CrossMap)

```
RGB  ──► DINO ViT-B/8 (frozen) ──► E2D (768×28×28) ──┐
                                                     ├─► mapper 2D→3D / 3D→2D (MLP) ──► discrepancy maps
XYZ  ──► preprocessing ──► Point-MAE (frozen) + PFA ──► E3D (1152×28×28) ┘
                                                                        │
                    fusion Ψ = Ψ2D · Ψ3D ──► Gaussian (σ = 4) ──► anomaly map, image score = max
```

- **Training**: only defect-free samples. Two MLP mappers (768→960→1152 and 1152→960→768) learn to predict one modality's features from the other with a cosine loss on valid 3D patches. Features are standardised per channel with statistics from the nominal training set.
- **Inference**: defects appear where the mapped and actual features disagree. The discrepancy is the L2 distance between L2-normalised features.
- **PFA (Point Feature Alignment)**, from M3DM: each valid pixel takes the inverse-squared-distance weighted mean of its *k* nearest Point-MAE group features, then the features are average-pooled to the 28×28 grid.

### Preprocessing factors

| | Factor | Level 1 | Level 2 | Level 3 |
|---|---|---|---|---|
| A | Background removal | Offline (CPU) | Online (GPU) | – |
| B | RANSAC inlier threshold τ | 2.5 mm | 5 mm | 10 mm |
| C | XYZ resize interpolation | Nearest | Bilinear + mask | Bilinear |
| D | PFA neighbours *k* | 1 | 3 | 8 |
| E | FPS centres *G* | 512 | 1024 | 2048 |
| F | Points per group *S* | 32 | 64 | 128 |

- **Offline (A1)**: RANSAC (1000 iterations) on a 10-pixel border strip, plane removal, then the largest connected component is kept. This runs once per part on the CPU.
- **Online (A2)**: RANSAC (50 iterations) on all valid points, inside the GPU pipeline.

### Staged Taguchi framework

1. **Screening (L18)**: all six factors, each run on 10 MVTec 3D-AD categories. Product categories act as the noise factor in the S/N ratio (larger-the-better for accuracy, smaller-the-better for latency).
2. **Grey relational analysis**: I-AUROC, AUPRO@30% and latency are combined into a single grade (ζ = 0.5, equal weights).
3. **ANOVA**: quantifies each factor's contribution to the grade.
4. **Refinement (L9)**: the strong A × C interaction makes the additive L18 prediction unreliable. C is therefore fixed to *nearest*, and B, D, E, F are refined with an L9 array, separately for A1 and A2.
5. **Confirmation**: each optimum is rerun with three seeds.

## Results

All values are means over the 10 MVTec 3D-AD categories.

### ANOVA of the L18 grade (end-to-end latency)

| Factor | Contribution (%) |
|---|---|
| C XYZ interpolation | **55.5** |
| B RANSAC threshold | 10.4 |
| A × B | 9.9 |
| D PFA neighbours | 5.2 |
| E FPS centres | 0.9 |
| A Background removal | 0.8 |
| F Points per group | 0.1 |
| Error | 17.1 |

A × C cannot be estimated orthogonally in L18. It is shown by the two-way response table: online removal is best with nearest interpolation, but AUPRO@30% collapses to 0.21–0.27 with bilinear interpolation.

### Confirmation (L9 optimum, three seeds)

| Setting | Levels | Predicted grade | Observed grade | Best L9 run |
|---|---|---|---|---|
| **T-CrossMap-Acc** (A1) | τ 2.5 mm, k 1, G 512, S 128 | 0.939 | 0.905 ± 0.006 | 0.888 |
| **T-CrossMap-Fast** (A2) | τ 5 mm, k 1, G 512, S 128 | 0.940 | 0.942 ± 0.011 | 0.877 |

### Comparison of settings (GPU server, three seeds)

| Setting | I-AUROC | AUPRO@30% | GPU latency (ms) | End-to-end (ms) |
|---|---|---|---|---|
| CFM default (bilinear XYZ) | 0.775 | 0.903 | 14.8 | 130.2 |
| Baseline | 0.824 | 0.960 | 14.7 | 130.1 |
| **T-CrossMap-Acc** (A1) | **0.859** | **0.968** | 12.2 | 128.2 |
| **T-CrossMap-Fast** (A2) | 0.845 | 0.958 | 19.3 | **19.3** |

The standard deviation across seeds is ≤ 0.002 for every setting.

### Edge deployment (NVIDIA Jetson AGX Orin, MAXN, FP32)

| Setting | GPU (ms) | CPU prep (ms) | Total (ms) | FPS | Module power (W) | GPU energy (J/img) |
|---|---|---|---|---|---|---|
| Baseline | 129.8 | 154.9 | 284.6 | 3.5 | 50.0 | 6.48 |
| T-CrossMap-Acc (A1) | 104.6 | 158.2 | 262.8 | 3.8 | 51.1 | 5.34 |
| **T-CrossMap-Fast** (A2) | 146.8 | 0 | **146.8** | **6.8** | **41.4** | 6.08 |

- On the AGX, CPU preprocessing takes more than half of the baseline latency. The all-GPU A2 setting is therefore 1.94× faster than the baseline and has the most uniform latency across categories (141–155 ms, against 174–399 ms for the baseline).
- Energy = mean module power (VDD_GPU_SOC + VDD_CPU_CV + VIN_SYS_5V0) × GPU latency. It excludes the unmetered CPU preprocessing of the baseline and A1, so the comparison is conservative toward A2.
- Accuracy on the AGX matches the server.
### Deployment scenario

<p align="center">
  <img src="figures/fig_deploy.png" width="95%" alt="T-CrossMap deployment scenario">
</p>

**Figure 2.** Proposed deployment of T-CrossMap with one top-down Intel RealSense RGB-D camera and one GPU server. (a) Inline inspection of bagels on a conveyor: after plane removal, each product becomes one 3D blob that is cropped and scored separately. (b) Station inspection in a bolt workshop, one part per scan on a flat surface. **Inspection loop:** capture → preprocessing (optimum A–C) → T-CrossMap inference (optimum D–F) → anomaly map and score → decision against a threshold. **Commissioning** is done once per product with defect-free parts only: capture a few hundred normal samples, run the L18 screening and L9 refinement, train the mappers, and set the decision threshold on nominal validation scores. *This is a proposed scheme; the scenes are illustrative and the system was evaluated on MVTec 3D-AD, not on a production line.*
## Limitations

- The selected settings are the best compromise **within the tested factor space**, not a global optimum.
- The additive prediction held for A2 but overestimated A1 by 0.034.
- Gains are not uniform across categories: A1 improves 8 of 10 categories, A2 6 of 10.
- The absolute accuracy of our reproduced detector is below values reported in the literature. The claims concern improvement over the baseline and CFM default within the same pipeline.
- Experiments use the MVTec 3D-AD benchmark. Validation on a production line with an industrial depth camera is future work.

## Dataset

[MVTec 3D-AD](https://www.mvtec.com/company/research/datasets/mvtec-3d-ad), 10 categories: bagel, cable gland, carrot, cookie, dowel, foam, peach, potato, rope, tire. The dataset is not redistributed here. Please follow its licence (CC BY-NC-SA 4.0).

## Acknowledgements

This work builds on:

- **CFM**: Costanzino et al., *Multimodal Industrial Anomaly Detection by Crossmodal Feature Mapping*, CVPR 2024.
- **M3DM**: Wang et al., *Multimodal Industrial Anomaly Detection via Hybrid Fusion*, CVPR 2023 (Point Feature Alignment, preprocessing protocol).
- **DINO**: Caron et al., *Emerging Properties in Self-Supervised Vision Transformers*, ICCV 2021.
- **Point-MAE**: Pang et al., *Masked Autoencoders for Point Cloud Self-supervised Learning*, ECCV 2022.
- **MVTec 3D-AD**: Bergmann et al., VISAPP 2022.

## Citation

The citation will be added after publication.

## Licence

To be decided before the code release.
