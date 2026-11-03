# OpenADMET CYP Challenge Submission Overview

This document describes our submissions to the [OpenADMET CYP Inhibition Blind Challenge](https://huggingface.co/spaces/openadmet/cyp-challenge), covering direct inhibition prediction, time-dependent inhibition classification, and protein–ligand binding pose prediction.

## 1. Direct inhibition prediction

### Model

We combine a pretrained molecular graph model with RankNet pairwise ranking to predict direct-inhibition pIC50 for **CYP1A2, CYP2C9, CYP2D6, and CYP3A4**. The model takes molecular graphs and RDKit 2D descriptors as inputs. A shared encoder learns molecular representations, while separate task outputs distinguish data sources, CYP isoforms, and assay endpoints.

Training combines numerical regression, interval supervision, and the RankNet ranking loss to learn both activity values and the relative ordering of compounds. The combined data define **47 tasks**. The four OpenADMET pIC50 tasks correspond to the challenge prediction targets, while the remaining tasks provide auxiliary supervision.

### Data used

The official data come from the [OpenADMET challenge dataset](https://huggingface.co/datasets/openadmet/cyp-challenge-train-test). We also use internal data and CYP activity data from public databases.

| Data source | Labels used | Number of tasks |
|---|---|---:|
| OpenADMET | Direct-inhibition pIC50, single-concentration screening log2fc, and direct-inhibition Emax for the four target CYP isoforms | 12 |
| Internal data | pIC50 for five CYP isoforms | 5 |
| ChEMBL | pIC50 for ten CYP isoforms | 10 |
| PubChem | pIC50 for five CYP isoforms | 5 |
| BindingDB | pIC50 for seven CYP isoforms | 7 |
| Tox21 | pAC50 for five CYP isoforms | 5 |
| NCATS | pAC50 for three CYP isoforms | 3 |
| **Total** | | **47** |

### Data processing and training

1. We standardize molecular structures with RDKit, including salt removal, parent-fragment selection, charge neutralization, and structure normalization. Canonical SMILES that preserve stereochemistry are used to match identical molecules.
2. Labels are organized by data source, CYP isoform, and assay endpoint. Duplicate point measurements within the same source and task are averaged, while labels from different sources remain separate tasks.
3. We retain interval information from OpenADMET and ChEMBL. Inspired by the [interval-labeling approach in Arnab28122000/openadmet-cyp-challenge](https://github.com/Arnab28122000/openadmet-cyp-challenge#2-every-dose-response-label-is-a-selected-label), we use single-concentration screening **log2fc ≥ log2(0.9) (approximately −0.152, corresponding to at least 90% residual activity)** to identify weakly or non-inhibitory samples for CYP1A2 and CYP2D6 in OpenADMET. In this work, these samples are assigned the pIC50 interval **[1.97, 3.1]**. For ChEMBL, exact measurements are averaged when available; groups containing only upper-limit records are represented as **[0, upper bound]**. Predictions within an interval incur zero interval regression loss for that observation. Missing labels are masked.
4. Labels from all sources are merged by unique molecule, followed by random five-fold cross-validation at the molecule level. All labels for a given molecule are assigned to the same partition. Label standardization parameters are fitted using only the training set of each fold.

## 2. Time-dependent inhibition classification

### Data and processing

We use the official `cyp-challenge-TRAIN_TDI.csv`, supplemented with **109 CYP2D6 records and 393 CYP3A4 records from ChEMBL 37**, and **481 CYP3A4 records from a [published study](https://doi.org/10.3389/fphar.2024.1451164)**.

For samples in the official dataset where **both** the CYP2D6 TDI-condition pIC50 and direct-inhibition pIC50 are missing, we assume a non-TDI label as an imputation step. This adds 4,648 CYP2D6 training samples. The resulting training datasets contain **6,145 samples for CYP2D6 and 4,458 samples for CYP3A4**.

Training uses **five-fold cross-validation based on Butina clustering** of molecular similarity.

### Models

| Prediction target | Features | Model |
|---|---|---|
| CYP2D6 TDI status | ECFP4 fingerprints and RDKit molecular descriptors | LightGBM classifier |
| CYP3A4 TDI status | Pretrained molecular embeddings from Chemprop | TabICL |

## 3. Structure prediction

We use **AutoDock Vina** to predict inhibitor binding poses in CYP3A4 for the challenge structure task. The receptor is the publicly available **[PDB structure 9BV5](https://www.rcsb.org/structure/9BV5)**. Up to **1,000 candidate poses** are generated per inhibitor.

For each candidate pose, we calculate the distance between the heme iron and ligand nitrogen or oxygen atoms. Poses with **Fe–N/O distances below 3 Å** are retained for manual inspection. Inspection considers two binding modes: the ligand displaces the water molecule coordinated to the heme iron and is stabilized by interactions with surrounding residues; or the ligand additionally coordinates the heme iron through a nitrogen atom or carbonyl oxygen.

Among poses consistent with these binding modes, the pose with the **lowest Vina score** is selected for submission.
