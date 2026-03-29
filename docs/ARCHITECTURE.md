# Architecture -- Azure MLOps CI/CD Pipeline (Diabetes Classification)

## Overview

This project implements an end-to-end MLOps CI/CD pipeline for diabetes prediction using the Pima Indians Diabetes dataset. GitHub Actions orchestrates Azure Machine Learning to automate compute provisioning, data registration, environment setup, model training with hyperparameter sweep, and model registration.

## Pipeline Architecture

```
+===========================================================================+
|                          GitHub Repository                                |
|   Push / PR to main                                                       |
+===========================================================================+
            |
            v
+===========================================================================+
|              GitHub Actions Workflow (CI/CD Orchestrator)                  |
|              deploy-model-training-pipeline-classical.yml                  |
|---------------------------------------------------------------------------|
|  1. get-config    -- reads config-infra-prod.yml for Azure settings       |
+===========================================================================+
            |
            v  (three parallel jobs)
+-----------------+  +---------------------+  +------------------------+
| create-compute  |  | register-dataset    |  | register-environment   |
| Standard_DS11_v2|  | pima.csv as         |  | train-conda.yml as     |
| (0-1 nodes)     |  | pima-diabetes-data  |  | pima-train-env         |
+-----------------+  +---------------------+  +------------------------+
         \                    |                       /
          \                   |                      /
           v                  v                     v
+===========================================================================+
|                Azure ML Training Pipeline (newpipeline.yml)               |
|---------------------------------------------------------------------------|
|                                                                           |
|  +------------------+     +---------------------+     +-----------------+ |
|  |  prep_data       |---->|  sweep_step         |---->|  register_model | |
|  |  (prep.py)       |     |  (train.yml x 20)   |     |  (register.py)  | |
|  +------------------+     +---------------------+     +-----------------+ |
|  | - Read pima.csv  |     | - Random sampling    |     | - Load best     | |
|  | - 80/20 split    |     | - criterion:         |     |   sweep model   | |
|  | - Save train.csv |     |   gini / entropy     |     | - Register as   | |
|  |   and test.csv   |     | - max_depth:         |     |   pima_classif- | |
|  | - Log sizes to   |     |   1, 3, 5, 10       |     |   ication_model | |
|  |   MLflow         |     | - 20 total trials    |     | - Write model   | |
|  |                  |     | - Maximize Recall    |     |   info JSON     | |
|  +------------------+     +---------------------+     +-----------------+ |
|                                                                           |
+===========================================================================+
            |
            v
+===========================================================================+
|  Azure ML Model Registry                                                  |
|  - pima_classification_model (MLflow format)                              |
|  - Versioned, tracked, ready for deployment                               |
+===========================================================================+
```

## Pipeline Stages in Detail

### Stage 1: Data Preparation (`prep.py`)

- **Input:** Raw `pima.csv` (Pima Indians Diabetes dataset)
- **Process:** Reads CSV, performs 80/20 train-test split with `random_state=42`
- **Output:** `train.csv` and `test.csv` saved to Azure blob storage
- **Tracking:** Logs `train size` and `test size` metrics to MLflow

### Stage 2: Hyperparameter Sweep (`train.yml` + `train.py`)

- **Algorithm:** `DecisionTreeClassifier` from scikit-learn
- **Sweep Strategy:** Random sampling over the search space
- **Search Space:**
  - `criterion`: choice of `gini` or `entropy`
  - `max_depth`: choice of `1`, `3`, `5`, or `10`
- **Limits:** 20 total trials, 10 concurrent, 7200s timeout
- **Objective:** Maximize `Recall` on the test set
- **Tracking:** Each trial logs `criterion`, `max_depth`, and `Recall` to MLflow

### Stage 3: Model Registration (`register.py`)

- **Input:** Best model from the sweep (MLflow sklearn format)
- **Process:** Loads model, logs to MLflow, registers in Azure ML model registry
- **Output:** `pima_classification_model` with version number, plus `model_info.json`

## Azure ML Components Used

| Component | Purpose |
|-----------|---------|
| **Compute Cluster** | `Standard_DS11_v2` (0-1 nodes, dedicated tier) for training |
| **Data Asset** | `pima-diabetes-data` -- registered URI file pointing to `pima.csv` |
| **Environment** | `pima-train-env` -- Conda environment with sklearn 0.24.1, mlflow, pandas |
| **Pipeline Job** | Orchestrates prep, sweep, and register steps sequentially |
| **Sweep Job** | Random hyperparameter search with 20 trials |
| **Model Registry** | Stores versioned MLflow models |

## GitHub Actions Workflow

The main workflow (`deploy-model-training-pipeline-classical.yml`) is structured as:

```
get-config (reads YAML)
    |
    +---> create-compute   (parallel)
    +---> register-dataset (parallel)
    +---> register-environment (parallel)
    |
    v
run-pipeline (depends on all three above)
```

Each step uses a **reusable workflow** from the `workflows/` directory:

| Workflow File | Purpose |
|---------------|---------|
| `custom-create-compute.yml` | Provisions Azure ML compute cluster |
| `custom-register-dataset.yml` | Registers dataset as Azure ML data asset |
| `custom-register-environment.yml` | Registers conda environment in Azure ML |
| `custom-run-pipeline.yml` | Submits the Azure ML pipeline job |

## Authentication

All workflows authenticate to Azure using the `AZURE_CREDENTIALS` GitHub secret, which contains a service principal JSON with contributor access to the Azure ML workspace.

## Key Configuration Files

| File | Purpose |
|------|---------|
| `config-infra-prod.yml` | Azure resource group and workspace names, VM image, location |
| `data-science/environment/train-conda.yml` | Python 3.7.5 + sklearn + mlflow + pandas dependencies |
| `mlops/azureml/train/data.yml` | Azure ML data asset definition |
| `mlops/azureml/train/train-env.yml` | Azure ML environment definition |
| `mlops/azureml/train/train.yml` | Training command component (used by sweep) |
| `mlops/azureml/train/newpipeline.yml` | Full pipeline definition |
