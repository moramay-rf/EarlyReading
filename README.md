# EarlyReading

This document describes a complete preprocessing workflow for diffusion-weighted MRI (DWI) data using **MRtrix3**, **FSL**, **dcm2niix**, and the **DESIGNER** denoising framework.

---

## Requirements

| Software | Version | Purpose |
|---------|---------|---------|
| MRtrix3 | ≥ 3.0.4 | DWI conversion, preprocessing, masking |
| FSL | ≥ 6.0.7 | Eddy correction (motion + distortion) |
| dcm2niix | latest | Convert DICOM → NIfTI |
| Singularity | optional | Run DESIGNER container |
| DESIGNER | 2.0 | MP-PCA denoising |

---

## 1. Load Required Modules

```bash
module load MRTrix
module load fsl
module load dcm2niix
module load singularity
```

---

## 2.Convert DICOM Folders

```bash
dcm2niix /path/to/images/sub_*/
```

Ensure output files include:

```
*.nii  
*.bval  
*.bvec
```

## 3. Convert to MRtrix `.mif` Format

### B2500 shell
```bash
mrconvert -bvalue_scaling false   -fslgrad Ax_DWI_HARDI_HB3_B2500.bvec Ax_DWI_HARDI_HB3_B2500.bval   Ax_DWI_HARDI_HB3_B2500.nii   sub-*-ses-*-dwi-B2500.mif
```

### B800 shell
```bash
mrconvert -bvalue_scaling false   -fslgrad Ax_DWI_HARDI_HB3_B800.bvec Ax_DWI_HARDI_HB3_B800.bval   Ax_DWI_HARDI_HB3_B800.nii   sub-*-ses-*-dwi-B800.mif
```

---
## 4. Concatenate Shells and Re-Export Gradients

```bash
mrcat -axis 3 sub-*B2500.mif sub-*B800.mif - |   mrconvert - -export_grad_fsl dwis.{bvec,bval,nii.gz}
```

Create the final DWIs as `.mif`:

```bash
mrconvert -bvalue_scaling false -fslgrad dwis.bvec dwis.bval dwis.nii.gz   sub-*-ses-*-dwis.mif
```

---

## 5. Convert Reverse Phase-Encoding Image

```bash
mrconvert -bvalue_scaling false   -fslgrad Ax_DWI_HARDI_REPVE.bvec Ax_DWI_HARDI_REPVE.bval   Ax_DWI_HARDI_REPVE.nii   sub-*-ses-*-dwi-revphase.mif
```

---

## 6. Denoising + Eddy + Distortion Correction (DESIGNER)

```bash
module load singularity
designer_container=/home/inb/soporte/lanirem_software/containers/designer2.sif

singularity run --nv $designer_container designer   -eddy   -mask   -denoise   -rpe_pair sub-*-ses-*-dwi-revphase.mif   -pe_dir AP sub-*-ses-*-dwis.mif   sub-*-ses-*-outputdesigner.nii
```

This performs:
- **MP-PCA denoising**
- **Motion + distortion correction using FSL Eddy**
- **Mask generation**

---

## 7. Convert the Output to `.mif`

```bash
mrconvert -bvalue_scaling false   -fslgrad *outputdesigner.bvec *outputdesigner.bval *outputdesigner.nii   sub-*-ses-*_dwi_preproc.mif
```

---

## 8. Remove Gibbs Ringing

```bash
mrdegibbs sub-*-ses-*_dwi_preproc.mif sub-*-ses-*_dwi.nii.gz
```

---

## 9. Create Brain Mask

```bash
bet2 sub-*-ses-*_dwi.nii.gz sub-*-ses-*_ -m -f 0.2
```

This produces:
```
*_.nii.gz      → extracted brain
*_mask.nii.gz → mask used by downstream tools
```

---

## 10. Rename T1 for pyAFQ

```bash
mrconvert T1_3.nii sub-*-ses-*_T1w.nii.gz
```

---

### Document Author
**Moramay Ramos-Flores**  



---

---
