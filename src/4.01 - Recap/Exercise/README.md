# 🌌 Lab 4.01 – Bounty Base Construction 🧱

### **Mission Briefing – Recap Protocol Alpha**

The Guild’s bounty operations require a secure, isolated base from which to manage contracts, targets, and intel. Over the past days, you've acquired the skills necessary to build such an environment using Kubernetes and Azure Bicep modules.

This mission will validate your command over both **Kubernetes infrastructure** and **Azure IaC best practices**.

> _"A bounty hunter is only as good as their deployment infrastructure."_ – **Fennec Shand**

---

## 🎯 Mission Objectives

You will:

- Define a reusable database module using Bicep
- Integrate and parameterize the database into a second main Bicep deployment, dedicated to the bounty application
- Set up Kubernetes configuration and application resources
- Enforce application availability via Pod Disruption Budgets

---

## 🧭 Step-by-Step

---

### 🧱 Section 1: Azure Infrastructure – Define the PostgreSQL Module

#### 1. Deploy the existing `bicep_aks` folder

- Edit the parameters file for setting unique values for your deployment.
- While deploying, you can continue to step 2.

#### 2. Clone the existing `bicep_aks` folder

- Do not edit the existing `bicep_aks` folder; instead, create a new folder named `bicep_app` for your Bicep files. You can use the existing `bicep_aks` folder as a reference.
- Remove all references that are not needed in an IaC application repository for the bounty application.

#### 3. Create a `database.bicep` module with the following properties:

- **Target scope**: `resourceGroup`

#### Parameters:

| Name                    | Type            | Notes                                                            |
| ----------------------- | --------------- | ---------------------------------------------------------------- |
| `location`              | `string`        | The Azure location to deploy into                                |
| `tags`                  | `object`        |                                                                  |
| `dbName`                | `string`        | Name of the Postgres flexible server                             |
| `administratorLogin`    | `string`        | Administrator username                                           |
| `administratorPassword` | `secure string` | Administrator password. Mark it with `@secure()`                 |
| `skuTier`               | `string`        | Must be one of: `Burstable`, `GeneralPurpose`, `MemoryOptimized` |
| `skuName`               | `string`        | SKU type (e.g. `Standard_B1ms`)                                  |

#### Postgres Flexible Server specs:

- A **PostgreSQL Flexible Server** (API version: `2025-01-01-preview`)
- Enable both **Active Directory** and **Password Auth**
- Set `publicNetworkAccess` to `Enabled`
- Set `storageSizeGB` to `32`
- Set `iops` to `120`
- Set `autoGrow` to `Enabled`
- Set `tier` to `P4`
- Set all parameters you defined above

#### Firewall rules

Additionally, you need to create a firewall rule to allow all Azure services to access the database.

```bash
resource allowAllWindowsAzureIps 'Microsoft.DBforPostgreSQL/flexibleServers/firewallRules@2025-06-01-preview' = {
  name: 'AllowAllWindowsAzureIps' // don't change the name
  parent: db
  properties: {
    endIpAddress: '0.0.0.0'
    startIpAddress: '0.0.0.0'
  }
}
```

> [!NOTE] In production, you would use a more restrictive rule or enable vnet integration, or work with private endpoints.

---

#### 4. Modify your new `main.bicep` file to include the database module (and a new resource group):

- Use the module keyword to reference the `database.bicep` file
- Provide values for all required parameters
- Set `administratorLoginPassword` to a Key Vault reference:
  - Use the `keyVaultResourceGroupName` and `keyVaultName` parameters to reference an existing Key Vault
  - Use the secret name `dbAdminPassword` for the password
  - Make sure to use the `@secure()` function to mark it as a secure string
  - And make sure you create the secret in the Azure Key Vault (created in step 1) before running the application IaC Bicep deployment
- Use logical, unique names (e.g., `bounty-db`) where applicable

---

#### 5. Update your `midSector.bicepparam` file:

- Add values for:

  - `databaseResourceGroupName`
  - `dbName`
  - `administratorLogin`
  - `skuTier`
  - `skuName`

> [!TIP] Store sensitive values securely (e.g., use Key Vault references in a production environment).

#### 6. Deploy the Bicep files

- Use the Azure CLI to deploy your `main.bicep` file with the `midSector.bicepparam` file. Make sure to use a different name than the infra deployment.

- Verify the deployment in the Azure portal. You should see the PostgreSQL Flexible Server created in the resource group specified in your parameters file.

---

### ✅ Section 2: Create the Namespace

- Create a Kubernetes **namespace** named: `bounty`

---

### 🔐 Section 3: Configuration Resources

#### Secret – Database Connection

- Type: `Secret`
- Name: `secret-bounty-db-connectionstring`
- Key: `connectionstring`
- Value: `Server=<<Database Server FQDN>>;Database=bounty;Port=5432;User Id=<<Administrator Name>>;Password=<<Administrator Password>>;Ssl Mode=Require;`

> [!TIP] Since we never store sensitive data in git, we'll create this secret via kubectl. In production we would use a more secure method, such as External Secrets Operator.

> [!NOTE] Bonus: For those that are ahead, take a look back at lab 3.04 to setup External Secrets Operator again and to create the secret correctly.

#### ConfigMaps

1. **Persistence Manager ConfigMap**

   - Name: `cm-bounty-api`
   - Keys:

     - `Migration__ApplyMigrations`: `"true"`
     - `Migration__RecreateDatabase`: `"false"`
     - `Migration__SeedDatabase`: `"true"`

2. **Client ConfigMap**

   - Name: `cm-bounty-client`
   - Key: `Apis__BountyApiUrl`
   - Value: Should point to the API's internal service name and port

---

### 💽 Section 4: Persistent Storage Setup

#### StorageClass

- Name: `bounty-azure-ssd`
- Provisioner: `kubernetes.io/azure-disk`
- Parameters:

  - `skuName`: `Premium_LRS`
  - `kind`: `Managed`

- Enable `allowVolumeExpansion`
- Use `volumeBindingMode`: `Immediate`

#### PersistentVolumeClaim

- Name: `images-claim`
- Namespace: `bounty`
- Storage Class: `bounty-azure-ssd`
- Size: `1Gi`
- Access Mode: `ReadWriteOnce`

---

### 🚀 Section 5: Deployments & Services

Deploy two applications in the `bounty` namespace with the following specs:

#### Deployment – `bounty-api`

- Init container: `bountypersistence` image

  - Pulls from secret and configmap for DB setup
  - Image: `cloudfuelimages.azurecr.io/bountypersistence:latest`
  - Map the `ConnectionStrings__Database` environment variable with the `secret-bounty-db-connectionstring`

- Main container: `bountyapi` image

  - Same env sources
  - CPU request: `0.5`, limit: `0.75`
  - Memory request: `256Mi`, limit: `512Mi`
  - Image: `cloudfuelimages.azurecr.io/bountyapi:latest`
  - Map the `ConnectionStrings__Database` environment variable with the `secret-bounty-db-connectionstring`

#### Service – `svc-bounty-api`

- Port: `5000`
- TargetPort: `8080`
- Selector: `app: bounty-api`

---

#### Deployment – `bounty-client`

- Container: `bountyclient` image

  - Uses `cm-bounty-client` for environment
  - Adds liveness probe on `/` (port `8080`)
  - Mounts volume `images-claim` at `/app/wwwroot/uploads`
  - CPU request: `0.5`, limit: `0.75`
  - Memory request: `256Mi`, limit: `512Mi`
  - Image: `cloudfuelimages.azurecr.io/bountyclient:1.1.0-official`

#### Service – `svc-bounty-client`

- Port: `8080`
- TargetPort: `8080`
- Selector: `app: bounty-client`

---

### 🛡️ Section 6: Pod Disruption Budgets

Ensure high availability during voluntary disruptions (e.g., node upgrades):

#### PDB – `bounty-api`

- `minAvailable`: `1`
- Match label: `app: bounty-api`

#### PDB – `bounty-client`

- `minAvailable`: `1`
- Match label: `app: bounty-client`

---

## 📦 Deliverables

You should produce the following artifacts/resources by the end of this lab:

| Resource Type         | Name                                | Namespace |
| --------------------- | ----------------------------------- | --------- |
| Bicep Folder          | `bicep_app`                         | -         |
| Bicep Module File     | `database.bicep`                    | -         |
| Module Call in Bicep  | `bounty-db`                         | -         |
| Namespace             | `bounty`                            | -         |
| Secret                | `secret-bounty-db-connectionstring` | `bounty`  |
| ConfigMap             | `cm-bounty-api`                     | `bounty`  |
| ConfigMap             | `cm-bounty-client`                  | `bounty`  |
| StorageClass          | `bounty-azure-ssd`                  | -         |
| PersistentVolumeClaim | `images-claim`                      | `bounty`  |
| Deployment            | `bounty-api`                        | `bounty`  |
| Deployment            | `bounty-client`                     | `bounty`  |
| Service               | `svc-bounty-api`                    | `bounty`  |
| Service               | `svc-bounty-client`                 | `bounty`  |
| PodDisruptionBudget   | `pdb-bounty-api`                    | `bounty`  |
| PodDisruptionBudget   | `pdb-bounty-client`                 | `bounty`  |

---

## 🚦 Validation

To validate your work:

1. Ensure all resources are created as specified.
2. Verify the configuration of the PostgreSQL Flexible Server and its firewall rules.
3. All pods in the `bounty` namespace are running.
4. Port-forward the `svc-bounty-client` and try to access the client application on `http://localhost:8080`.
5. You should see a list of bounty items.
6. Try to create a new one. For the image, use an SVG (like this [one](https://commons.wikimedia.org/wiki/File:Kubernetes_logo_without_workmark.svg))
7. Verify that the new bounty item appears in the list with its image.
8. Exec into the client pod and browse to `/app/wwwroot/uploads`.
9. Do you see the uploaded image file?
10. Restart the deployments and verify if the image is still there.

## 📚 Resources

[Key Vault references](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/key-vault-parameter?tabs=azure-cli)

[PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/templates/microsoft.dbforpostgresql/flexibleservers?pivots=deployment-language-bicep)

[Kubernetes docs](https://kubernetes.io/docs/home/)
