# Deformation Based Morphometry 

## Pipeline overview: 
1. [Pulling scans from the directory](#1-pulling-scans-from-directory)
2. [Conversion from scanner](#2-conversion-from-scanner)
3. [Motion QC](#3-motion-qc)
4. [Preprocessing](#4-preprocessing)
5. [Brief QC](#5-brief-qc)
6. [DBM set up](#6-dbm-set-up)
7. [Running DBM](#7-running-dbm)
8. [DBM QC](#8-dbm-qc)
9. [Set up for R](#9set-up-for-r)
10. [Analyses in R](#10-analyses-in-r)
11. [Display and locating peak voxels](#11-display-and-locating-peak-voxels)
12. [References](#12-useful-references)

## Background


## 1. Pulling scans from directory
### DICOM scans are located in (either PV5 or PV6): 
```
/data/chamal/Bruker_7T_animal_scans/DICOM/
```
### Scans in bruker format from PV5:
```
/data/chamal/Bruker_7T_animal_scans/nmr/
```
### Scans in bruker format from PV6: 
```
/data/chamal/Bruker_7T_animal_scans/pv6/galdan/
```
## 2. Conversion from scanner:
### For **DICOM** to **MINC** conversion follow [this wiki](https://github.com/CobraLab/documentation/wiki/Mouse-DICOM-to-MINC-Conversion).

### For **BRUKER** to **NIFTI** you can follow the wiki [bruker2nifti conversion.](https://github.com/CoBrALab/documentation/wiki/bruker2nifti-conversion)
This organizes your scans according to BIDS format. 
It should look something like this: 
```
/home/m/mchakrav/liz1296/scratch/fmri_pilot/input
├── dataset_description.json
├── participants.json
├── participants.tsv
├── README
├── sub-1
│   ├── ses-1
│   │   ├── anat
│   │   │   └── sub-1_ses-1_FLASH_T1w.nii.gz
│   │   ├── fmap
│   │   │   ├── sub-1_ses-1_fieldmap.nii.gz
│   │   │   └── sub-1_ses-1_magnitude.nii.gz
│   │   └── func
│   │       └── sub-1_ses-1_EPI_bold.nii.gz
│   └── ses-2
│       ├── anat
│       │   └── sub-1_ses-2_FLASH_T1w.nii.gz
│       ├── fmap
│       │   ├── sub-1_ses-2_run-01_fieldmap.nii.gz
│       │   ├── sub-1_ses-2_run-01_magnitude.nii.gz
│       │   ├── sub-1_ses-2_run-02_fieldmap.nii.gz
│       │   └── sub-1_ses-2_run-02_magnitude.nii.gz
│       └── func
│           └── sub-1_ses-2_EPI_bold.nii.gz
```
### For **NIFTI** to **MINC** conversion: you need to load the module ``minc-toolkit`` and use the command ``nii2mnc``.

### For **MINC** to **NIFTI** you need to load the module ``minc-toolkit`` and use the command ``mnc2nii``.

## 3. Motion QC 
Currently, there's a [wiki](https://github.com/CoBrALab/documentation/wiki/Mouse-QC-Manual-(Structural)) created by Mila with some rating examples for your scans. **(This wiki might need to be updated)**

## 4. Preprocessing
There are different scripts that can be used for preprocessing your scans depending on the Paravision version that was used during acquisition. 

Things to consider when using raw dicom scans either from Pv5 or Pv6:
* Scans acquired with Pv5 are reconstructed with a wrong orientation. 
* [Add more if needed]

To avoid any potential issues, it is recommended to use [Bruker raw scans](#scans-in-bruker-format-from-pv5), [convert them to NIFTI](#for-nifti-to-minc-conversion-you-need-to-load-the-module-minc-toolkit-and-use-the-command-nii2mnc) and preprocess them.

In the Bruker MRI scanner at the Douglas we use a surface cryo coil, this means that our scans will have a dorsal to ventral drop of signal in which regions that are closer to the coil will have higher signal intensity. The brightness location changes depending on how the animal is placed inside the scanner (In neonates, due to the small size of the brain, this is less of an issue). 

The preprocessing pipelines created by Gabriel Devenyi use tools from minc-toolkit and ANTs to make the images as uniform as possible before processing. Some of the steps include:
* Fix the orientation of the scans
* Set the coordinate system to align them to the DSURQE atlas 
* Correct for intensity inhomogeneities using the N4 algorithm 
* Generate a mask and use it to cut off signal that is not important
* Denoise
* Register scans using 6 degrees of freedom (3 translations and 3 rotations in x, y and z)

These will generate different outputs:
* Two files whose names include 'lqs6' (a mask and t1w scan). These scans were aligned using 6 degrees of freedom (3 rotations and translations in x , y and z) 
* Two files with the name of your original scan (a mask and a t1w scan). These are preprocessed not realigned scans. 

Differences between lsq6, lsq9 and lsq12:
* lsq6: implicates rigid changes - rotation in 3 axis and translation in 3 directions. Also known as linear or rigid transformations. The goal is to get two images to overlap. This does not change the properties of the brain only its orientation and position.
* lsq9: includes the prev 6. + stretching/scaling 
* lsq12: lsq9 + shearing in 3 angles. 

Non-linear transformations can take into account regional/local manipulations of the voxels or volumetric changes. This include expanding some voxels and contracting others to get a one-to-one map of two images. In order for this to take place you need first to do a linear registration between the two images. 

Registering one scan to another depends on different similarity metrics. This can be influenced by motion, hence it is important to QC your images prior to any processing.

The wiki with the information on how to preprocess scans is found in [here](https://github.com/CobraLab/documentation/wiki/Mouse-Scan-Preprocessing).

The current version of preprocessing script that is recommended is ``mouse-preprocessing-v5.sh``. There are different versions depending on whether your scans were acquired using Pv5 or Pv6, or they are DICOM or NIFTI. You will need to load the following modules: **minc-toolkit ANTs qbatch minc-toolkit-extras**
* For scans in Pv5:``mouse-preprocessing-v5.sh``
* For DICOM scans in Pv6:``mouse-preprocessing-v5_pv6_dicom.sh``
* For NIFTI scans in Pv6: ``mouse-preprocessing-v5_pv6_nifti.sh``

## 5. Brief QC
After your preprocessing is done you will need to QC your images. 


## 6. DBM set up
Deformation based morphometry uses image registration to map one image to the space of another in which the deformation fields are used to evaluate morphometric differences in the brain. The procedure implicates registering each scan onto an atlas brain that represents our population (like the ICBM152 and MNI152 templates in humans), calculate the deformation fields for each subject and use that to measure voxel-wise volumetric differences. To avoid any biases related to the atlas that has been selected or in case it does not exist, we create an average image (template) using our sample. 


In brief, this is achieved by iteratively registering all images to an arbitrary selected image, taking the average of all transforms, use that to resample each image and average them to create a study-population-specific atlas. This is performed first using lsq12 to create the best linear representation of the data. We then use that as a starting template, non-linearly deform each scan onto it and create a template for subsequent non-linear registrations. Once this is achieved, we can obtain the deformation fields (linear or non-linear) from the template to each individual image. These are used to calculate the Jacobian determinants, a measure of local volume expansion and contraction, and the magnitude of the deformation at each voxel. This process allows a one-to-one mapping in which we can make voxel-by-voxel comparisons. 

We can measure absolute displacements (absolute Jacobian determinants) which encode scaling in all directions, or local brain differences (relative Jacobian determinants) by subtracting the linear transformations from the non-linear transform. All jacobian determinants are centered on 1 and range from 0 to infinity (shrinking <1; expanding >1), therefore we  log transform them to get a gaussian distribution so that we don't break any assumptions of any statistical test we would like to perform. 

In Gabriel Devenyi's pipeline, we can decide to implement DBM either on cross-sectional (1 level) or longitudinal data (2 level). For more information you can take a look [here](https://github.com/CobraLab/twolevel_ants_dbm). 

The wiki with everything related to how to run DBM is found [here.](https://github.com/CoBrALab/documentation/wiki/Running-twolevel_ants_dbm-on-Niagara )
You need to copy your T1w NIFTI scans to Niagara and then follow the instructions on the wiki. 
You can use ``screen`` for each DBM run. Take note of the hostname, this way you can logout from Niagara and come back to check it. 
Some commands to work with screen:
* To create a new screen: ``screen``
* To detach from current screen: ctrl+a then d
* To list all current screens: ``screen -ls``
* To reattach to a selected screen: ``screen -r [screen id]``

In case you are logged in to a different host name use:
```
ssh [HOSTNAME example: nia-login06]
```
Now, create a CSV for the scans that will be used as input to DBM: each row is a subject and each column is a timepoint.

## 7. Running DBM
As shown in the [wiki](https://github.com/CoBrALab/documentation/wiki/Running-twolevel_ants_dbm-on-Niagara) you have to create a script to run DBM:
```
#!/bin/bash
module load cobralab/2019b
./twolevel_ants_dbm/twolevel_dbm.py --modelbuild-command ${QUARANTINE_PATH}/twolevel-dbm/niagara2-antsMultivariateTemplateConstruction2.sh \
—-cluster-type=slurm [POSSIBLY OTHER OPTIONS] 2level [CSV WITH INPUT PATHS] 2>&1 | tee -a dbm_logfile.log
```
Make it executable: ``chmod u+x [DBM SCRIPT. SH]``

Run script: `bash [DBM SCRIPT .SH]`


## 8. DBM QC
After the pipeline has been run you need to QC your outputs. One way to do this is using the ``register`` tool from minc-toolkit to see if each subject template is properly registered to the average template. 

## 9.Set up for R
First copy your DBM output from Niagara to CIC. 
Then you need to create a mask to run your stats. You can follow this [wiki.](https://github.com/CoBrALab/documentation/wiki/Making-a-stats-mask-for-2level-dbm-output) 
The DSURQE atlas was used for each mask. This is found in the following directory: 
```/opt/quarantine/resources/Dorr_2008_Steadman_2013_Ullmann_2013_Richards_2011_Qiu_2016_Egan_2015_40micron/ex-vivo/```
Mask and templates need to be converted from [NIFTI to MINC](#for-nifti-to-minc-conversion-you-need-to-load-the-module-minc-toolkit-and-use-the-command-nii2mnc) so we can use them in R.

Using Display, verify if the mask needs to be eroded.
```
Display [AVERAGE TEMPLATE IN MINC] -label [MASK IN MINC]
```
Erode if necessary:
```
mincmorph -erosion [INPUT ORIGINAL MASK .MNC] [OUTPUT ERODED MASK .MNC]
```

## 10. Analyses in R

## 11. Display and locating peak voxels

## 12. Useful references
Here is a shared [paperpile folder](https://paperpile.com/shared/nzPX8K) with some resourcers.
