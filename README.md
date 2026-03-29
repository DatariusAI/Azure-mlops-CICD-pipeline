# Azure MLOps CI/CD Pipeline -- Diabetes Classification

<div align="center">

![Azure ML](https://img.shields.io/badge/Azure%20ML-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.x-3776AB?style=for-the-badge&logo=python&logoColor=white)
![scikit-learn](https://img.shields.io/badge/scikit--learn-F7931E?style=for-the-badge&logo=scikitlearn&logoColor=white)
![MLflow](https://img.shields.io/badge/MLflow-0194E2?style=for-the-badge&logo=mlflow&logoColor=white)

**End-to-end ML pipeline with automated training, hyperparameter tuning, and model registration on Azure ML -- triggered by GitHub Actions CI/CD.**

</div>

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Pipeline Stages](#pipeline-stages)
- [Repository Structure](#repository-structure)
- [Tech Stack](#tech-stack)
- [Setup & Configuration](#setup--configuration)
- [How It Works](#how-it-works)
- [Dataset](#dataset)

---

## Overview

This project implements a production-grade MLOps pipeline that automates the entire machine learning lifecycle for **diabetes prediction** using the Pima Indians Diabetes dataset. On every push or pull request to `main`, GitHub Actions orchestrates Azure ML to provision compute, register data and environments, train a Decision Tree classifier with hyperparameter sweep, and register the best model.

---

## Architecture

```
GitHub (Push / PR to main)
        |
        v
+----------------------------+
|   GitHub Actions Workflow   |
|  deploy-model-training-     |
|  pipeline-classical.yml     |
+----------------------------+
        |
        v  (parallel jobs)
+-------------+  +------------------+  +---------------------+
| Create      |  | Register Dataset |  | Register Conda      |
| Compute     |  | (pima.csv)       |  | Environment         |
| Cluster     |  |                  |  |                     |
+-------------+  +------------------+  +---------------------+
        \               |                /
         \              |               /
          v             v              v
    +------------------------------------+
    |    Azure ML Training Pipeline      |
    |------------------------------------|
    |  1. prep.py  -- split data 80/20   |
    |  2. sweep    -- tune criterion &   |
    |               max_depth (20 trials)|
    |  3. register -- save best model    |
    +------------------------------------+
              |
              v
    +--------------------+
    | Registered Model   |
    | (MLflow format)    |
    +--------------------+
```

---

## Pipeline Stages

| Stage | Script / Config | Description |
|-------|----------------|-------------|
| **Data Preparation** | `data-science/src/prep.py` | Reads raw CSV, performs 80/20 train-test split, logs dataset sizes to MLflow |
| **Hyperparameter Sweep** | `mlops/azureml/train/train.yml` | Sweeps over `criterion` (gini, entropy) and `max_depth` (1, 3, 5, 10) using random sampling -- 20 total trials |
| **Model Training** | `data-science/src/train.py` | Trains a `DecisionTreeClassifier`, evaluates recall on test set, logs metrics to MLflow |
| **Model Registration** | `data-science/src/register.py` | Registers the best sweep model in Azure ML model registry (MLflow format) |

---

## Repository Structure

```
Azure-mlops-CICD-pipeline/
|
|-- config-infra-prod.yml                  # Azure infra config (resource group, workspace)
|-- requirements.txt                       # Dev dependencies (black, flake8, isort)
|
|-- data/
|   +-- pima.csv                           # Pima Indians Diabetes dataset
|
|-- data-science/
|   |-- environment/
|   |   +-- train-conda.yml                # Conda env specification (sklearn, mlflow, pandas)
|   +-- src/
|       |-- prep.py                        # Data preparation & train/test split
|       |-- train.py                       # Model training (DecisionTreeClassifier)
|       +-- register.py                    # Model registration to Azure ML
|
|-- mlops/azureml/train/
|   |-- data.yml                           # Azure ML data asset definition
|   |-- train-env.yml                      # Azure ML environment definition
|   |-- train.yml                          # Training command component (for sweep)
|   +-- newpipeline.yml                    # Full pipeline definition (prep -> sweep -> register)
|
+-- workflows/
    |-- deploy-model-training-pipeline-classical.yml   # Main CI/CD workflow
    |-- custom-create-compute.yml                      # Reusable: provision compute cluster
    |-- custom-register-dataset.yml                    # Reusable: register dataset in Azure ML
    |-- custom-register-environment.yml                # Reusable: register conda environment
    +-- custom-run-pipeline.yml                        # Reusable: submit pipeline job
```

---

## Tech Stack

| Category | Technology |
|----------|-----------|
| **Cloud Platform** | Microsoft Azure (Azure Machine Learning) |
| **CI/CD** | GitHub Actions |
| **ML Framework** | scikit-learn (DecisionTreeClassifier) |
| **Experiment Tracking** | MLflow |
| **Language** | Python 3.x |
| **Data** | Pandas, NumPy |
| **Code Quality** | Black, Flake8, isort, pre-commit |
| **Compute** | Azure ML Compute Cluster (Standard_DS11_v2) |

---

## Setup & Configuration

### Prerequisites

- Azure subscription with an Azure ML workspace
- GitHub repository with Actions enabled
- Azure credentials stored as a GitHub secret (`AZURE_CREDENTIALS`)

### 1. Configure Azure Resources

Edit `config-infra-prod.yml` with your Azure resource group and workspace:

```yaml
resource_group: <your-resource-group-name>
aml_workspace: <your-aml-workspace-name>
```

### 2. Set GitHub Secrets

Add the following secret to your repository:

| Secret | Description |
|--------|-------------|
| `AZURE_CREDENTIALS` | Service principal credentials JSON for Azure authentication |

### 3. Trigger the Pipeline

Push to `main` or open a pull request -- the workflow runs automatically:

```bash
git push origin main
```

---

## How It Works

1. **GitHub Actions** detects a push/PR to `main` and reads the infrastructure config.
2. **Parallel setup** -- compute cluster, dataset, and conda environment are provisioned/registered simultaneously on Azure ML.
3. **Pipeline execution** -- once all resources are ready, the Azure ML pipeline runs:
   - `prep.py` splits the Pima dataset into train (80%) and test (20%) sets.
   - A **random hyperparameter sweep** trains up to 20 Decision Tree models, optimizing for **Recall**.
   - The best model is registered in the Azure ML model registry as `pima_classification_model`.
4. All metrics (recall, dataset sizes, hyperparameters) are tracked in **MLflow**.

---

## Dataset

**Pima Indians Diabetes Dataset**

| Property | Value |
|----------|-------|
| **Task** | Binary Classification |
| **Target** | `class` (0 = no diabetes, 1 = diabetes) |
| **Features** | Pregnancies, Glucose, Blood Pressure, Skin Thickness, Insulin, BMI, Diabetes Pedigree, Age |
| **Primary Metric** | Recall (maximized) |
| **Algorithm** | Decision Tree Classifier |

---

## Author

**DatariusAI** -- [github.com/DatariusAI](https://github.com/DatariusAI)
