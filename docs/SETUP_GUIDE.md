# Setup Guide -- Azure MLOps CI/CD Pipeline

Step-by-step instructions to configure and run the diabetes classification pipeline.

## Prerequisites

### Azure Requirements

1. **Azure Subscription** -- an active Azure account with billing enabled
2. **Azure Resource Group** -- a resource group to hold all ML resources
3. **Azure Machine Learning Workspace** -- an Azure ML workspace created within the resource group
4. **Service Principal** -- with Contributor role on the resource group

### Local Requirements

- Git
- Python 3.7+ (for local testing, optional)
- Azure CLI (optional, for manual resource management)

### GitHub Requirements

- A GitHub repository with Actions enabled
- Permission to add repository secrets

## Step 1: Create Azure Resources

If you have not already created the required Azure resources:

```bash
# Install Azure CLI (if needed)
# https://docs.microsoft.com/en-us/cli/azure/install-azure-cli

# Login to Azure
az login

# Create a resource group
az group create --name myResourceGroup --location eastus

# Create an Azure ML workspace
az ml workspace create \
  --name myMLWorkspace \
  --resource-group myResourceGroup \
  --location eastus
```

## Step 2: Create a Service Principal

The CI/CD pipeline authenticates using a service principal. Create one with Contributor access:

```bash
az ad sp create-for-rbac \
  --name "mlops-cicd-sp" \
  --role contributor \
  --scopes /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP> \
  --sdk-auth
```

This outputs a JSON block like:

```json
{
  "clientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "clientSecret": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  ...
}
```

Save this entire JSON output -- you will need it in the next step.

## Step 3: Configure GitHub Secrets

1. Go to your GitHub repository
2. Navigate to **Settings > Secrets and variables > Actions**
3. Click **New repository secret**
4. Add the following secret:

| Secret Name | Value |
|-------------|-------|
| `AZURE_CREDENTIALS` | The full JSON output from the service principal creation step |

## Step 4: Update Infrastructure Configuration

Edit `config-infra-prod.yml` in the repository root with your Azure resource names:

```yaml
variables:
  resource_group: myResourceGroup        # <-- your resource group name
  aml_workspace: myMLWorkspace           # <-- your Azure ML workspace name
  location: eastus                       # <-- your preferred Azure region
```

## Step 5: Trigger the Pipeline

The pipeline runs automatically on:
- **Push** to the `main` branch
- **Pull request** targeting the `main` branch

To trigger manually:

```bash
git add .
git commit -m "Configure Azure resources"
git push origin main
```

Then monitor the workflow in the **Actions** tab of your GitHub repository.

## Step 6: Monitor Pipeline Execution

1. **GitHub Actions:** Watch the workflow progress in the Actions tab. The jobs run in this order:
   - `get-config` (read YAML settings)
   - `create-compute`, `register-dataset`, `register-environment` (parallel)
   - `run-pipeline` (after all three complete)

2. **Azure ML Studio:** Open your Azure ML workspace at [ml.azure.com](https://ml.azure.com) to:
   - View the pipeline run under **Jobs**
   - Inspect the hyperparameter sweep trials
   - Check the registered model under **Models**
   - Review MLflow metrics under **Experiments > pima-classification-training**

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `AZURE_CREDENTIALS` authentication failure | Invalid or expired service principal | Regenerate the service principal and update the GitHub secret |
| `Resource group not found` | Incorrect name in `config-infra-prod.yml` | Verify the resource group name matches your Azure subscription |
| `Workspace not found` | Workspace name mismatch or not created | Create the workspace or fix the name in `config-infra-prod.yml` |
| Compute cluster creation timeout | Quota limits in the selected region | Try a different VM size or Azure region; request quota increase |
| `Environment registration failed` | Conda dependency conflict | Check `data-science/environment/train-conda.yml` for version compatibility |
| Pipeline job stays in `Preparing` state | Compute cluster scaling up from 0 nodes | Wait 5-10 minutes for the cluster to provision; check Azure ML compute status |
| `Module not found` error in pipeline | Environment not registered correctly | Re-run the workflow or manually register the environment via Azure CLI |

### Checking Azure Resources Manually

```bash
# List compute clusters
az ml compute list --resource-group myResourceGroup --workspace-name myMLWorkspace

# List registered environments
az ml environment list --resource-group myResourceGroup --workspace-name myMLWorkspace

# List registered models
az ml model list --resource-group myResourceGroup --workspace-name myMLWorkspace

# View a specific pipeline job
az ml job show --name <job-name> --resource-group myResourceGroup --workspace-name myMLWorkspace
```

### Re-running a Failed Workflow

1. Go to the **Actions** tab in GitHub
2. Click on the failed workflow run
3. Click **Re-run all jobs** (or re-run only the failed job)

Alternatively, push a new commit to trigger a fresh run.
