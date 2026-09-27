# Pharmacogenomics-Drug-Response-ML

## Pharmacogenomics-Based Prediction of Lung Cancer Drug Response Using Machine Learning

### Overview

This project investigates whether **gene-expression profiles of lung cancer cell models can be used to predict their response to anticancer drugs** using machine learning.

The project integrates:

* **RNA-seq gene-expression data** from Cell Model Passports
* **Cancer cell-model information** from the Cell Model Passports model list
* **Drug-response data** from the Cell Model Passports Drug sensitivity

The prediction target used in the machine-learning analysis is **LN_IC50**, representing the natural logarithm of the half-maximal inhibitory concentration (IC50).

The complete workflow was developed to transform large public datasets into a lung cancer-specific pharmacogenomics dataset and then use machine learning to investigate drug-response patterns.

---

## Project Workflow

```text
====================================================================================================
                        COMPLETE END-TO-END PHARMACOGENOMICS ML WORKFLOW
====================================================================================================

                                ┌──────────────────────────────────────┐
                                │ Cell Model Passports (Model List)    │
                                └──────────────────┬───────────────────┘
                                                   │
                                                   ▼
                                    Filter Only Lung Models (N=317)
                                                   │
                            ┌──────────────────────┴──────────────────────┐
                            │                                             │
                            ▼                                             ▼
             ┌──────────────────────────────┐              ┌──────────────────────────────┐
             │   RNA-Seq Expression Data    │              │   GDSC2 Drug Response Data   │
             └──────────────┬───────────────┘              └──────────────┬───────────────┘
                            │                                             │
                            ▼                                             ▼
                  Extract Lung RNA-seq                          Filter GDSC2 for Lung
                (N = 247 Unique Models)                       (N = 190 Unique Models)
                            │                                             │
                            ▼                                             │
                  Clean RNA-seq Matrix                                    │
              (Drop Duplicates & Transpose)                               │
                            │                                             │
                            ▼                                             │
                 Gene Variance Filtering                                  │
          (Threshold > 0.01 ➔ Top 5,000 Genes)                            │
                            │                                             │
                            └──────────────────────┬──────────────────────┘
                                                   │
                                                   ▼
                                        [ SET INTERSECTION ]
                                       171 Common Model IDs
                                                   │
                                                   ▼
                                    [ DRUG COMPLETENESS FILTER ]
                               Keep drugs present across ALL 171 models
                                        (33 Complete Drugs)
                                                   │
                                                   ▼
                                        [ FINAL MATRIX MERGE ]
                          Inner Join: 33 Drugs x 171 Models x 5,000 Genes
                                    (5,082 Rows x 5,008 Columns)
                                                   │
                                                   ▼
                                      Leak-Free Data Partitioning
                                 (GroupShuffleSplit by SANGER_MODEL_ID)
                                                   │
                                                   ▼
                                        Preprocessing Pipeline
                               (Imputation, Scaling, OneHotEncoding)
                                                   │
                                       ┌───────────┴───────────┐
                                       ▼                       ▼
                             Random Forest Pipeline    XGBoost Pipeline
                              (SelectKBest + Tuning)  (SelectKBest + Tuning)
                                       │                       │
                                       └───────────┬───────────┘
                                                   │
                                                   ▼
                                    Cross-Validation Model Selection
                                           (Winner: XGBoost)
                                                   │
                                                   ▼
                                 Final Evaluation on Untouched Test Set
                                                   │
                            ┌──────────────────────┼──────────────────────┐
                            ▼                      ▼                      ▼
                   SHAP Interpretation   Permutation Importance    Residual Analysis
                  (Summary & Bar Plots)    (Mean R² Decrease)      (Actual vs Pred)

====================================================================================================
```

---

# 1. Data Sources
## Cell Model Passports

RNA-seq expression data and cancer cell-model information were obtained from **Cell Model Passports**.

The main RNA-seq dataset used in the project is:

```text
rnaseq_merged_rsem_tpm_20260323.csv
```

This dataset contains RNA-seq expression measurements represented as **RSEM TPM** values.

The Cell Model Passports model list used in the project is:

```text
model_list_20260616.csv
```

The model list was used to identify models annotated as **lung**.

---

## GDSC2

Drug-response data were obtained from the **Genomics of Drug Sensitivity in Cancer (GDSC2)** dataset.

The original GDSC2 file used was:

```text
GDSC2_fitted_dose_response_27Oct23.xlsx
```

The dataset contains drug-response measurements and associated drug/model information.

The primary response variable used for machine learning is:

```text
LN_IC50
```

---

# 2. Identify Lung Cancer Models

The first step was to use the Cell Model Passports model list to identify cancer models belonging to the lung tissue type.

The model list was filtered using:

```text
tissue = lung
```

This produced 317 unique lung models. The resulting lung-specific model information was saved as:

```text
only_lung_model_final.csv
```

This lung model list was subsequently used to extract the corresponding RNA-seq samples and GDSC2 drug-response records.

---

# 3. Extract Lung RNA-seq Data

The complete Cell Model Passports RNA-seq dataset was then processed to extract only the models identified as lung models.

The original expression dataset:

```text
rnaseq_merged_rsem_tpm_20260323.csv
```

contains a large number of cancer models and genes.

The workflow:

1. Loaded the RNA-seq dataset.
2. Identified the gene identifier column.
3. Used the lung model IDs to select relevant RNA-seq columns.
4. Removed duplicated gene identifiers.
5. Transposed the expression matrix.
6. Removed duplicated model identifiers.
7. Added `model_id` as the model identifier column.

The resulting processed lung RNA-seq dataset (247 lung models) was saved as:

```text
RNA_FULL_processeeD.csv
```

The resulting structure is approximately:

```text
model_id | Gene_1 | Gene_2 | Gene_3 | ... | Gene_n
```

where each row represents a lung cancer cell model and each gene column represents its expression value.

---

# 4. RNA-seq Feature Filtering

The lung RNA-seq dataset contains thousands of gene-expression features.

Using every gene directly would create a very high-dimensional feature space, so gene filtering was performed before constructing the final machine-learning dataset.

## Variance Filtering

A `VarianceThreshold` filter was applied:

```text
threshold = 0.01
```

Genes with variance below this threshold were removed. This reduced the gene set from 41,145 to 27,343 genes.

## Selection of Top Variable Genes

After variance filtering, the remaining genes were ranked according to their variance.

The **5,000 genes with the highest variance** were retained, producing the final gene-expression feature set for downstream integration and machine learning.

---

# 5. Process GDSC2 Drug-Response Data

The GDSC2 fitted dose-response dataset was loaded and filtered using the lung model IDs identified from Cell Model Passports.

The original dataset:

```text
GDSC2_fitted_dose_response_27Oct23.xlsx
```

was filtered using:

```text
SANGER_MODEL_ID
```

to retain only lung cancer models (190 unique lung models). The resulting lung-specific drug-response dataset was saved as:

```text
LUNG_GDSC2.csv
```

---

# 6. Identify Common Lung Models

The RNA-seq dataset and GDSC2 dataset use different column names for their model identifiers:

```text
RNA-seq:
model_id
```

```text
GDSC2:
SANGER_MODEL_ID
```

These identifiers were converted to a common string format and compared.

The intersection of the two datasets was used to identify lung models for which both:

* RNA-seq expression data, and
* GDSC2 drug-response data

were available.

The workflow identified:

```text
171 common lung models
```

These common models were used for the final pharmacogenomics integration.

---

# 7. Select Drugs with Complete Model Coverage

After identifying the 171 common lung models, drug coverage was examined in the GDSC2 dataset.

For each drug, the number of unique lung models with available drug-response measurements was calculated.

Only drugs represented across all 171 common lung models were retained, which produced **33 complete drugs** (e.g. Dinaciclib, Ibrutinib, Niraparib, AZD5582, among others). This step created a more consistent drug-response dataset for the machine-learning analysis.

---

# 8. Create the Final Pharmacogenomics Dataset

The processed RNA-seq data and filtered GDSC2 drug-response data were merged.

The datasets were joined using:

```text
GDSC2:
SANGER_MODEL_ID
```

and:

```text
RNA-seq:
model_id
```

The final integrated dataset contains **5,082 rows × 5,008 columns**, including:

* Lung cancer cell-model information
* Gene-expression features (5,000)
* Drug information
* Drug targets
* Pathway information
* LN_IC50
* Other original GDSC2 response fields

The final dataset was saved as:

```text
Pharmacogenomics_Finale.csv
```

This dataset is the main input for the machine-learning analysis.

---

# 9. Machine-Learning Objective

The machine-learning component aims to predict:

```text
LN_IC50
```

from molecular and drug-related features.

The general relationship investigated is:

```text
Gene Expression
      +
Drug Information
      ↓
Machine Learning Model
      ↓
Predicted LN_IC50
```

---

# 10. Feature Preparation

The final pharmacogenomics dataset contains both numerical and categorical information.

### Numerical features

The main numerical predictors are the 5,000 selected gene-expression features.

### Categorical features

The following categorical variables are included:

```text
DRUG_NAME
PUTATIVE_TARGET
PATHWAY_NAME
```

Categorical variables are converted into numerical representations using one-hot encoding.

---

# 11. Avoiding Target Leakage

Several columns are deliberately excluded from the machine-learning predictors.

The target itself is removed:

```text
LN_IC50
```

The following response-related columns are also excluded:

```text
AUC
RMSE
```

Model identifiers are excluded:

```text
SANGER_MODEL_ID
model_id
```

Therefore, the machine-learning feature matrix is constructed without directly providing the model with the target value or additional drug-response measurements that could introduce target leakage.

---

# 12. Train/Test Split

A major consideration in this dataset is that one cancer cell model can have measurements for many different drugs.

For example:

```text
Model A → Drug 1
Model A → Drug 2
Model A → Drug 3
```

A simple random row-based split could place some observations from Model A in the training set and other observations from Model A in the test set.

To reduce this problem, the project uses:

```text
GroupShuffleSplit
```

with `SANGER_MODEL_ID` as the grouping variable. This ensures that observations from the same cell model remain within the same partition.

The data were split into:

```text
Training: 4,059 rows across 123 models
Testing:  1,023 rows across 31 models
```

approximately an 80/20 split by model group. The test set is reserved for final model evaluation.

---

# 13. Cross-Validation

Model development uses:

```text
GroupKFold
```

with **5 folds**. The grouping variable remains `SANGER_MODEL_ID`, which prevents the same cancer cell model from being distributed across different cross-validation folds. This grouped strategy is important because the final dataset contains multiple drug-response observations for individual cell models.

---

# 14. Preprocessing Pipeline

The machine-learning preprocessing is performed inside the model pipelines.

## Numerical Features

Numerical features undergo:

```text
Median Imputation
        ↓
Standard Scaling
```

## Categorical Features

Categorical variables undergo:

```text
Most-Frequent Imputation
        ↓
One-Hot Encoding
```

The encoder uses `handle_unknown = "ignore"` so that unseen categories in validation or testing do not cause preprocessing errors.

---

# 15. Feature Selection During Machine Learning

Because the integrated dataset contains thousands of gene-expression features, additional feature selection is performed inside the machine-learning pipeline.

The method used is `SelectKBest` with `f_regression`. The number of selected features is treated as a hyperparameter, with values explored: **500, 1000, 1500**.

Because feature selection is included inside the pipeline, it is fitted during the training process rather than directly using the held-out test set.

---

# 16. Machine-Learning Models

Two regression algorithms are evaluated.

## Random Forest Regressor

The model search explores parameters including number of trees, maximum tree depth, minimum samples for splitting, minimum samples per leaf, maximum number of features, and number of selected features.

## XGBoost Regressor

The search explores parameters including number of estimators, maximum depth, learning rate, subsample, column subsampling, minimum child weight, L1 regularization, L2 regularization, gamma, and number of selected features.

---

# 17. Hyperparameter Optimization

Hyperparameters are optimized using `RandomizedSearchCV` with `GroupKFold`. The primary optimization metric is **R²**. The best configuration identified through grouped cross-validation is retained for each model.

---

# 18. Model Evaluation

The primary regression metric is **R²**, measuring the proportion of variation in the target explained by the model.

Additional evaluation includes:

* **Actual vs Predicted LN_IC50** — compares experimentally measured values with model predictions.
* **Residual Plot** — examines prediction errors for systematic patterns.

---

# 19. Model Interpretation

## Permutation Importance

Evaluates how model performance changes when individual features are randomly shuffled. Features whose permutation substantially reduces model performance are considered important to the model's predictions.

## SHAP

**SHAP (SHapley Additive exPlanations)** is used to examine feature contributions to model predictions, providing overall feature importance, contribution direction, distribution of effects, and individual prediction explanations.

---

# 20. Results

## Cross-Validated Performance

| Model | Best CV R² | Key hyperparameters |
|---|---|---|
| Random Forest | 0.612 | `n_estimators=100`, `max_depth=None`, `min_samples_leaf=2`, `max_features=0.5`, `k=1000` |
| **XGBoost** | **0.624** | `n_estimators=300`, `max_depth=7`, `learning_rate=0.05`, `subsample=0.8`, `colsample_bytree=0.6`, `k=1500` |

**XGBoost was selected as the final model** based on grouped cross-validation performance.

## Final Held-Out Test Performance

```text
Final Test R²: 0.647
```

The model generalizes to unseen cell models (31 held-out models, never seen during training or hyperparameter search) with an R² comparable to — and slightly higher than — its cross-validated score, indicating no meaningful overfitting to the training groups.

## Feature Importance

Permutation importance identified a small number of features with substantially higher importance than the rest, with the top feature's importance (~0.50) far exceeding the second-ranked feature (~0.13) — consistent with a small number of gene-expression or drug-identity features dominating prediction, with SHAP used to further characterize these effects (direction and distribution) across the model.

Figures generated (Actual vs Predicted, Residuals, Permutation Importance, SHAP summary and bar plots) are available in the analysis notebook.

---

# 21. Reproducibility

The analysis can be reproduced by following the workflow in the same order:

```text
1. Obtain Cell Model Passports model list
2. Obtain Cell Model Passports RNA-seq data
3. Identify lung models
4. Extract lung RNA-seq samples
5. Process the RNA expression matrix
6. Apply variance filtering
7. Select the top 5,000 variable genes
8. Obtain GDSC2 drug-response data
9. Filter GDSC2 for lung models
10. Identify common RNA/GDSC2 models
11. Select drugs with complete model coverage
12. Merge RNA-seq and GDSC2 data
13. Create the final pharmacogenomics dataset
14. Prepare machine-learning features
15. Perform grouped train/test splitting
16. Perform grouped cross-validation
17. Train Random Forest and XGBoost
18. Optimize model hyperparameters
19. Evaluate the final model
20. Perform permutation-importance analysis
21. Perform SHAP analysis
22. Generate figures
```

Python dependencies are listed in `requirements.txt`.

---

# 22. Limitations

* **Cell-model data, not patient data.** Results should not be interpreted as direct clinical predictions.
* **Pre-split gene selection.** The top-5,000-variable-gene selection was performed on the full dataset before the train/test split; for strictly leakage-free benchmarking, this step could instead be learned only from each training partition.
* **No external validation yet.** Generalization beyond this cell-model panel has not been tested on an independent dataset.

---

# 23. Future Work

Possible extensions include:

* Testing additional machine-learning algorithms (LightGBM, CatBoost)
* Exploring `FeatureUnion`-based combination of gene-selection methods. PCA was evaluated as a dimensionality-reduction approach but did not perform well on the RNA-seq expression features, so combining selection strategies via `FeatureUnion` is a more promising direction than PCA alone.
* Building drug-specific prediction models
* Integrating mutation data. Copy-number alteration (CNV) data was evaluated but did not improve pharmacogenomic prediction in initial testing; this would need re-investigation with different integration approaches rather than being pursued as-is.
* Performing external validation
* Investigating biologically relevant genes identified by SHAP against known pharmacogenomic mechanisms

---

# 24. Project Goal

The central idea of this project is to connect molecular characteristics of cancer cell models with their experimentally measured drug responses.

```text
Lung Cancer Cell Model
          │
          ▼
   Gene Expression
          │
          ▼
  Molecular Features
          │
          ▼
 Machine Learning Model
          │
          ▼
 Predicted Drug Response
          │
          ▼
Feature Interpretation
```

The project demonstrates a complete computational workflow from **public biological datasets to an integrated pharmacogenomics dataset and machine-learning analysis**, achieving a final held-out test R² of 0.647 with XGBoost.

---

# Author

**Maryam Sattar**
BS Bioinformatics

The data-processing workflow, lung-model selection, RNA-seq processing, GDSC2 integration, final dataset construction, machine-learning workflow, evaluation strategy, and model-interpretation analysis were developed as part of this project.

---

# Acknowledgements

This project uses publicly available data and resources from:

* **Cell Model Passports** — cancer cell-model information and RNA-seq expression data.
* **Genomics of Drug Sensitivity in Cancer (GDSC)** — experimental cancer drug-response data.
* The developers and maintainers of the Python scientific-computing and machine-learning libraries used in this project.

The original datasets are not redistributed in this repository.

---
# License

This project is released under the **MIT License**.

See the [`LICENSE`](LICENSE) file for details.
