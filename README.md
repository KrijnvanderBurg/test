# Azure Data Factory CI/CD
Automated CI/CD for Azure Data Factory spoke deployments using Azure DevOps, following Microsoft best practices.

## Architecture
```
┌─────────────────────────────────────────────────────────────────────────┐
│                           ON-PREMISES NETWORK                           │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      On-Premises Data Source                    │   │
│   │             (SQL Server, File Shares, Database, etc.)           │   │
│   └──────────────────────────────┬──────────────────────────────────┘   │
│                                  │                                      │
│                  ┌───────────────┴───────────────┐                      │
│                  │                               │                      │
│   ┌──────────────┴──────────────┐ ┌──────────────┴──────────────┐       │
│   │     SHIR Host Node 1        │ │     SHIR Host Node 2        │       │
│   │    ┌───────────────┐        │ │        ┌───────────────┐    │       │
│   │    │  SHIR Agent   │        │ │        │  SHIR Agent   │    │       │
│   │    └───────────────┘        │ │        └───────────────┘    │       │
│   └──────────────┬──────────────┘ └──────────────┬──────────────┘       │
│                  │                               │                      │
│                  └───────────────┬───────────────┘                      │
│                                  │                                      │
└──────────────────────────────────┼──────────────────────────────────────┘
                                   │
                                   │ 
                                   │
┌──────────────────────────────────┼──────────────────────────────────────┐
│                                AZURE                                    │
│                                  │                                      │
│   ┌──────────────────────────────┴──────────────────────────────────┐   │
│   │                            Hub ADF                              │   │
│   │                       (PlaceholderSHIR)                         │   │
│   └──────────────────────────────┬──────────────────────────────────┘   │
│                                  │                                      │
│                  ┌───────────────┴───────────────┐                      │
│                  │                               │                      │
│                  ▼                               ▼                      │
│   ┌─────────────────────────┐     ┌─────────────────────────┐           │
│   │       Spoke 1 ADF       │     │       Spoke N ADF       │           │
│   │       (Linked IR)       │     │       (Linked IR)       │           │
│   └────────────┬────────────┘     └────────────┬────────────┘           │
│                │                               │                        │
│                ▼                               ▼                        │
│   ┌─────────────────────────┐     ┌─────────────────────────┐           │
│   │    Storage Account      │     │    Storage Account      │           │
│   │    (Landing Zone)       │     │    (Landing Zone)       │           │
│   └─────────────────────────┘     └─────────────────────────┘           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Repository Structure

```
├── adf/                          # ADF resources (auto-populated by ADF Studio)
│   └── arm-template-parameters-definition.json  # Custom ARM parameterization
├── .azuredevops/
│   ├── package.json              # npm package for ARM template generation
│   ├── azure-pipelines-ci.yml    # Build pipeline (validate + generate ARM)
│   ├── azure-pipelines-cd.yml    # Release pipeline (uses template)
│   ├── templates/
│   │   └── deploy-adf.yml        # Reusable deployment template
│   ├── scripts/
│   │   └── PrePostDeploymentScript.Ver2.ps1  # Microsoft trigger management script
│   ├── variables/
│   │   ├── int-dev.yml    # int-dev environment variables
│   │   └── int-acc.yml    # int-acc environment variables
│   └── parameters/
│       ├── int-dev.parameters.json   # int-dev ARM parameters
│       └── int-acc.parameters.json   # int-acc ARM parameters
└── README.md
```

## Components

| Component | Purpose | Documentation |
|-----------|---------|---------------|
| **package.json** | Uses `@microsoft/azure-data-factory-utilities` npm package to validate ADF resources and generate ARM templates without manual publish | [Automated publishing](https://learn.microsoft.com/en-us/azure/data-factory/continuous-integration-delivery-improvements) |
| **azure-pipelines-cd.yml** | Unified CI/CD pipeline triggered on merge to `main`. Validates, generates ARM templates, and deploys | [CI/CD improvements](https://learn.microsoft.com/en-us/azure/data-factory/continuous-integration-delivery-improvements) |
| **templates/deploy-adf.yml** | Reusable deployment template. Handles blob storage upload for linked templates, uses `AzurePowerShell@5` with managed identity for deployment | [YAML templates](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/templates) |
| **PrePostDeploymentScript.Ver2.ps1** | Microsoft's official script. Stops only modified triggers before deployment, starts them after, and cleans up deleted resources | [Pre/post deployment script](https://github.com/Azure/Azure-DataFactory/tree/main/SamplesV2/ContinuousIntegrationAndDelivery) |
| **arm-template-parameters-definition.json** | Located in `adf/` folder. Customizes which properties are parameterized in ARM templates (Microsoft default template) | [Parameterization](https://learn.microsoft.com/en-us/azure/data-factory/continuous-integration-delivery-resource-manager-custom-parameters) |
| **Linked Self-Hosted IR** | Spoke factories reference a shared SHIR in the hub factory via linked integration runtime | [Shared SHIR](https://learn.microsoft.com/en-us/azure/data-factory/create-shared-self-hosted-integration-runtime-powershell) |

## Setup
### 1. Prerequisites
- Hub ADF with `PlaceholderSHIR` deployed and registered
- Spoke factory's managed identity granted **Contributor** role on hub's SHIR
- Azure Key Vault
- Storage account for linked ARM templates (required for factories >256 resources)

### 2. Azure RBAC Requirements
The Azure DevOps service connection's service principal requires the following role assignments:

| Resource | Role | Purpose |
|----------|------|---------|
| **Resource Group** | Contributor | Deploy ADF and related resources via ARM templates |
| **Data Factory** | Data Factory Contributor | Stop/start triggers via PrePostDeploymentScript |

### 3. Configure Git Integration
1. Open spoke ADF in Azure Portal → ADF Studio
2. Go to **Manage** → **Git configuration**
3. Configure:
   - Repository type: Azure DevOps Git
   - Repository: This repository
   - Collaboration branch: `main`
   - Root folder: `/adf`
4. **Edit** configuration and check **"Disable publish button"**
5. Go to **Manage** → **ARM template** → Enable **"Include global parameters in ARM template"**

> **Note:** Enabling "Include global parameters in ARM template" ensures global parameters are automatically included in the generated ARM template and can be parameterized per environment. This is the current Microsoft-recommended approach and not via the previous powershell script.

> [Disable manual publish documentation](https://learn.microsoft.com/en-us/azure/data-factory/source-control#best-practices-for-git-integration) | [Global parameters CI/CD](https://learn.microsoft.com/en-us/azure/data-factory/author-global-parameters#cicd)

### 4. Configure Variable Files
Update the variable files:

**.azuredevops/variables/<env>.yml:**
```yaml
variables:
  azureServiceConnection: <azure-service-connection>
  subscriptionId: <subscription-id>
  resourceGroupName: <int-dev-resource-group>
  dataFactoryName: <int-dev-adf-name>
  location: <azure-region>
  sharedIRResourceId: /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.DataFactory/factories/<hub-adf>/integrationRuntimes/PlaceholderSHIR
  storageAccountName: <storage-account>
  storageAccountResourceGroup: <storage-rg>
```

> **Linked Templates (>256 resources):** If data factory grows beyond 256 resources, the npm package automatically generates linked templates. These require a storage account because [Azure Resource Manager must access linked templates via HTTP/HTTPS URL](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/linked-templates) during deployment. The pipeline auto-detects linked templates and handles the upload using managed identity authentication.

### 5. Update ARM template parameters
- `.azuredevops/parameters/int-dev.parameters.json` — int-dev ARM parameters (Key Vault URL, etc.)
- `.azuredevops/parameters/int-acc.parameters.json` — int-acc ARM parameters (Key Vault URL, etc.)

## CI/CD Flow
```
Feature Branch → PR → main branch → Pipeline: Build → Deploy int-dev → Deploy int-acc
```

1. Developer creates resources in spoke ADF Studio
2. Changes saved to feature branch automatically
3. PR merged to `main` triggers pipeline
4. Build stage validates resources and generates ARM templates
5. Deploy stage deploys to int-dev environment
6. Deploy stage deploys to int-acc environment

## Best Practices Implemented

| Practice | Implementation | Reference |
|----------|----------------|-----------|
| Automated ARM generation | npm package, no manual publish | [Link](https://learn.microsoft.com/en-us/azure/data-factory/continuous-integration-delivery-improvements) |
| Disabled manual publish | Enforced via Git configuration | [Link](https://learn.microsoft.com/en-us/azure/data-factory/source-control#best-practices-for-git-integration) |
| Smart trigger management | Ver2 script stops only modified triggers | [Link](https://github.com/Azure/Azure-DataFactory/tree/main/SamplesV2/ContinuousIntegrationAndDelivery) |
| Shared SHIR (hub-spoke) | Linked IR references hub's PlaceholderSHIR | [Link](https://learn.microsoft.com/en-us/azure/data-factory/create-shared-self-hosted-integration-runtime-powershell#create-a-shared-self-hosted-integration-runtime-in-azure-data-factory-1) |
| Global parameters | Included in ARM template via ADF Studio setting; parameterized per environment | [Link](https://learn.microsoft.com/en-us/azure/data-factory/author-global-parameters#cicd) |
| Linked templates support | Auto-detects >256 resources, uploads to blob storage with managed identity auth | [Link](https://learn.microsoft.com/en-us/azure/data-factory/continuous-integration-delivery-linked-templates) |
