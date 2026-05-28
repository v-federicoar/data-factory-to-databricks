# Contoso's Data Processing Journey: A Project Setup Guide

Welcome to the project setup guide for Contoso's data processing pipeline. This guide will walk you through the steps needed to set up a simple, yet effective, data processing pipeline. The example is simple from a data point of view, but it's designed to help you learn about the technology and how to build the pipeline.

## Prerequisites

Before we embark on this adventure, ensure you have the following tools ready:

- **An Azure subscription**  [free account](https://azure.microsoft.com/pricing/purchase-options/azure-account?cid=msft_learn)
- **Azure CLI**: Version 2.75.0 or higher. Install from [Azure CLI's official page](https://learn.microsoft.com/cli/azure/install-azure-cli).
- **Bash or WSL**: A Bash-compatible shell environment is crucial. If you're on Windows, check out [Windows Subsystem for Linux (WSL)](https://learn.microsoft.com/windows/wsl/install).
- **Databricks CLI**: Optional, but recommended for cluster manipulation. Install instructions are available [here](https://learn.microsoft.com/azure/databricks/dev-tools/cli/tutorial), version 0.258.0 or higher.

## The Contoso Data Pipeline Adventure

### Step 1: Clone repository

Navigate to the directory where you want to download the code.

```bash
  git clone https://github.com/Azure-Samples/data-factory-to-databricks.git
  cd data-factory-to-databricks
```

### Step 2: Azure Login

Our journey begins with logging into Azure. Use the command below:

```bash
az login
# Optionally, set the default subscription:
# az account set --subscription <subscription_id>
```

### Step 3: Environment Setup

Like any good adventure, we need to prepare our environment:

```bash
export LOCATION=centralus
export RESOURCEGROUP=rg-medallion-lakehouse-${LOCATION}
export USERNAME=$(az ad signed-in-user show --query mail -o tsv)
export USER_OBJECTID=$(az ad signed-in-user show --query id -o tsv)
export USER_TENANTID=$(az account show --query tenantId -o tsv)
```

### Step 4: Resource Group Creation

With our map in hand, we create a resource group in our chosen location:

```bash
  az group create -n $RESOURCEGROUP -l $LOCATION
```

### Step 5: Deploying Resources

Using a Bicep template, we deploy the resources needed for our data processing quest:

```bash
  az deployment group create -f ./main.bicep -g ${RESOURCEGROUP} -p username=${USERNAME} userObjectId=${USER_OBJECTID} userTenantId=${USER_TENANTID} secretsExpirationDate=$(date -d "+1 year" +"%s")
```

If you are using macOS, replace `date -d "+1 year" +"%s"` with `date -v+1y +%s`.

![Contoso's Created Resources](./Resources.jpg "Contoso's Created Resources")

The Bicep template creates:

- User identity for Azure Data Factory
- Azure Data Lake, the previous identity is a collaborator.
- Azure Databricks Workspace, the previous identity is a collaborator.
- A SQL Database that allows access only to Microsoft Entra users; the previous identity is a user.
- Azure Data Factory. The previous identity is associated
  - The Azure Data Factory contains a Pipeline
- A Databricks Key Vault. It includes Azure Data Lake secrets used by Databricks.

The Tale of Data Transformation:

Our pipeline, depicted below, is a tale of transformation:
![ADF Pipeline](adf-pipeline.gif "ADF Pipeline")

The pipeline consumes New York Health data. This example works with baby names https://health.data.ny.gov/Health/Baby-Names-Beginning-2007/jxy9-yhdk/data_preview.

1. Retrieve the file from New York Health Data and store it in the data lake landing container
1. LandingToBronze: A Databricks Notebook moves data to a Delta Table on bronze container.The process **appends** information and adds control metadata, including processing time and file name.
1. BronzeToSilver: Cleaning the data, removing duplicates, and **merging** into the silver container.
1. SilverToGold: Populating a star model in the gold container.
1. Transfer the star model (including dimension tables and fact table) from the gold container to a SQL Database.

### Step 6: The Chronicles of Databricks Notebooks

In the ./notebooks directory, you'll find the scripts of our chronicles. Upload them to Databricks using the CLI or manually via the Azure portal.

You can also do this manually inside Databricks. Import notebooks from the Workspace section in the Azure Databricks UI. Azure Data Factory assumes the notebooks are inside a _myLib_ folder in the user workspace.

Using Azure Databricks CLI, you need a token to authenticate the CLI to the workspace. [Azure Databricks personal access token authentication](https://learn.microsoft.com/azure/databricks/dev-tools/cli/authentication#--azure-databricks-personal-access-token-authentication)  
To create a personal access token, do the following: 

1. In your Azure Databricks workspace, click your Azure Databricks username in the top bar, and then select Settings from the dropdown.
1. Click Developer.
1. Next to Access tokens, click Manage.
1. Click Generate new token.
1. (Optional) Enter a comment that helps you to identify this token in the future, and change the token's default lifetime of 90 days. To create a token with no lifetime (not recommended), leave the Lifetime (days) box empty (blank).
1. Click Generate.
1. Copy the displayed token to a secure location, and then click Done.

```bash
    # Upload Databricks notebooks using Databricks CLI

    # Authenticate Databricks CLI
    export DATABRICKS_WORKSPACE_URL=$(az deployment group show -g ${RESOURCEGROUP} --name main --query properties.outputs.databricksWorkspaceUrl.value --output tsv)
    databricks configure --host $DATABRICKS_WORKSPACE_URL
    # For the prompt Personal Access Token, enter the Azure Databricks personal access token for your workspace

    # Upload the local notebooks to your workspace
    databricks sync ./notebooks/ /Users/${USERNAME}/myLib
```

### Unity Catalog one-time setup (required when using external Azure Data Lake Storage -ADLS- paths)

If your workspace uses Unity Catalog and you keep bronze/silver/gold in ADLS paths, run this one-time setup before executing the pipeline. Without this setup, you might get errors like `[NO_PARENT_EXTERNAL_LOCATION_FOR_PATH]`.

1. Open Azure Databricks in the browser.
2. Go to **SQL**.
3. Start (or select) a SQL Warehouse.
4. Open **SQL Editor** and create a new query.
5. In Bash/WSL, prepare variables and Azure resources (copy and paste this whole block):

```bash
# Current subscription and resource group
export SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# Avoid interactive prompts when az needs extensions
az config set extension.use_dynamic_install=yes_without_prompt
az config set extension.dynamic_install_allow_preview=true

# Access Connector name and ID
export ACCESS_CONNECTOR_NAME=${ACCESS_CONNECTOR_NAME:-adb-access-connector-${LOCATION}}
export ACCESS_CONNECTOR_ID=$(az resource list -g ${RESOURCEGROUP} --resource-type Microsoft.Databricks/accessConnectors --query "[0].id" -o tsv)
export ACCESS_CONNECTOR_PRINCIPAL_ID=""

# If no connector exists, create one with SystemAssigned identity
if [ -z "$ACCESS_CONNECTOR_ID" ]; then
  az databricks access-connector create -g ${RESOURCEGROUP} -n ${ACCESS_CONNECTOR_NAME} -l ${LOCATION} --identity-type SystemAssigned
  export ACCESS_CONNECTOR_ID=$(az databricks access-connector show -g ${RESOURCEGROUP} -n ${ACCESS_CONNECTOR_NAME} --query id -o tsv)
  export ACCESS_CONNECTOR_PRINCIPAL_ID=$(az databricks access-connector show -g ${RESOURCEGROUP} -n ${ACCESS_CONNECTOR_NAME} --query identity.principalId -o tsv)
else
  # Connector exists. Validate that it has a managed identity.
  export ACCESS_CONNECTOR_NAME=$(basename ${ACCESS_CONNECTOR_ID})
  export ACCESS_CONNECTOR_PRINCIPAL_ID=$(az databricks access-connector show -g ${RESOURCEGROUP} -n ${ACCESS_CONNECTOR_NAME} --query identity.principalId -o tsv)

  # If principalId is empty, connector was created without managed identity. Recreate it correctly.
  if [ -z "$ACCESS_CONNECTOR_PRINCIPAL_ID" ] || [ "$ACCESS_CONNECTOR_PRINCIPAL_ID" = "null" ]; then
    az databricks access-connector delete -g ${RESOURCEGROUP} -n ${ACCESS_CONNECTOR_NAME} --yes
    az databricks access-connector create -g ${RESOURCEGROUP} -n ${ACCESS_CONNECTOR_NAME} -l ${LOCATION} --identity-type SystemAssigned
    export ACCESS_CONNECTOR_ID=$(az databricks access-connector show -g ${RESOURCEGROUP} -n ${ACCESS_CONNECTOR_NAME} --query id -o tsv)
    export ACCESS_CONNECTOR_PRINCIPAL_ID=$(az databricks access-connector show -g ${RESOURCEGROUP} -n ${ACCESS_CONNECTOR_NAME} --query identity.principalId -o tsv)
  fi
fi

# Storage account name used by the sample
export STORAGE_ACCOUNT=$(az resource list -g ${RESOURCEGROUP} --resource-type Microsoft.Storage/storageAccounts --query "[0].name" -o tsv)
export STORAGE_ACCOUNT_ID=$(az storage account show -g ${RESOURCEGROUP} -n ${STORAGE_ACCOUNT} --query id -o tsv)

# Grant connector access to ADLS (required)
az role assignment create \
  --assignee-object-id ${ACCESS_CONNECTOR_PRINCIPAL_ID} \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Contributor" \
  --scope ${STORAGE_ACCOUNT_ID} \
  --only-show-errors || true

# Identity that runs notebooks/jobs for this sample.
# This sample uses the ADF User Assigned Managed Identity as Databricks submitter identity.
# Databricks identifies service principals by application/client ID.
export ADF_UAMI_NAME="dataFactoryUserIdentity"
export ADF_UAMI_CLIENT_ID=$(az identity show -g ${RESOURCEGROUP} -n ${ADF_UAMI_NAME} --query clientId -o tsv)

# Quick verification
echo "SUBSCRIPTION_ID=$SUBSCRIPTION_ID"
echo "RESOURCEGROUP=$RESOURCEGROUP"
echo "ACCESS_CONNECTOR_NAME=$ACCESS_CONNECTOR_NAME"
echo "ACCESS_CONNECTOR_ID=$ACCESS_CONNECTOR_ID"
echo "ACCESS_CONNECTOR_PRINCIPAL_ID=$ACCESS_CONNECTOR_PRINCIPAL_ID"
echo "STORAGE_ACCOUNT=$STORAGE_ACCOUNT"
echo "ADF_UAMI_CLIENT_ID=$ADF_UAMI_CLIENT_ID"
```

6. Generate a ready-to-paste SQL script (copy and paste this in Bash/WSL):

```bash
cat > uc_external_locations_setup.sql <<EOF
CREATE STORAGE CREDENTIAL IF NOT EXISTS adls_cred
WITH AZURE_MANAGED_IDENTITY '${ACCESS_CONNECTOR_ID}';

CREATE EXTERNAL LOCATION IF NOT EXISTS landing_ext_loc
URL 'abfss://landing@${STORAGE_ACCOUNT}.dfs.core.windows.net/'
WITH (STORAGE CREDENTIAL adls_cred);

CREATE EXTERNAL LOCATION IF NOT EXISTS bronze_ext_loc
URL 'abfss://bronze@${STORAGE_ACCOUNT}.dfs.core.windows.net/'
WITH (STORAGE CREDENTIAL adls_cred);

CREATE EXTERNAL LOCATION IF NOT EXISTS silver_ext_loc
URL 'abfss://silver@${STORAGE_ACCOUNT}.dfs.core.windows.net/'
WITH (STORAGE CREDENTIAL adls_cred);

CREATE EXTERNAL LOCATION IF NOT EXISTS gold_ext_loc
URL 'abfss://gold@${STORAGE_ACCOUNT}.dfs.core.windows.net/'
WITH (STORAGE CREDENTIAL adls_cred);

GRANT READ FILES, WRITE FILES, CREATE EXTERNAL TABLE ON EXTERNAL LOCATION landing_ext_loc TO `${ADF_UAMI_CLIENT_ID}`;
GRANT READ FILES, WRITE FILES, CREATE EXTERNAL TABLE ON EXTERNAL LOCATION bronze_ext_loc TO `${ADF_UAMI_CLIENT_ID}`;
GRANT READ FILES, WRITE FILES, CREATE EXTERNAL TABLE ON EXTERNAL LOCATION silver_ext_loc TO `${ADF_UAMI_CLIENT_ID}`;
GRANT READ FILES, WRITE FILES, CREATE EXTERNAL TABLE ON EXTERNAL LOCATION gold_ext_loc TO `${ADF_UAMI_CLIENT_ID}`;
EOF
```

```bash
cat uc_external_locations_setup.sql
```

7. Validate the setup:

```sql
SHOW EXTERNAL LOCATIONS;
DESCRIBE EXTERNAL LOCATION landing_ext_loc;
DESCRIBE EXTERNAL LOCATION bronze_ext_loc;
DESCRIBE EXTERNAL LOCATION silver_ext_loc;
DESCRIBE EXTERNAL LOCATION gold_ext_loc;
```

8. Re-run the notebooks/pipeline.

__NOTE:__  [Notebooks](https://learn.microsoft.com/azure/databricks/notebooks/) are the primary tool for creating data science and machine learning workflows on Azure Databricks. Databricks notebooks provide real-time coauthoring in multiple languages, automatic versioning, and built-in data visualizations for developing code and presenting results. You can see and read the notebooks using Visual Studio Code, the notebooks have comments explaining what they are doing. In this example we are using mainly Python and SQL.  

### Step 7: [Databricks Secret Scope Creation](https://learn.microsoft.com/azure/databricks/security/secrets/secret-scopes#create-an-azure-key-vault-backed-secret-scope)

Create an Azure Key Vault-backed secret scope to allow Databricks to access the Data Lake. The notebook will get the secrets from a Databricks Secret Scope.

1. Go to https://-databricks-instance-/**#secrets/createScope**. Replace -databricks-instance- with the workspace URL of your Azure Databricks deployment. Note: The scope name in the URL must be uppercase.

2. Enter the name of the secret scope. Our notebook expect **dataLakeScope**

3. Set Managed Principal to 'All workspace users'

4. Complete dns name and resource id

```bash
  # Get the values from here
  export DATABRICKS_KEY_VAULT_DNS_NAME=$(az deployment group show -g ${RESOURCEGROUP} --name main --query properties.outputs.databricksKeyVaultUrl.value --output tsv)
  export DATABRICKS_KEY_VAULT_RESOURCE_ID=$(az deployment group show -g ${RESOURCEGROUP} --name main --query properties.outputs.databricksKeyVaultResourceId.value --output tsv)
  echo $DATABRICKS_KEY_VAULT_DNS_NAME
  echo $DATABRICKS_KEY_VAULT_RESOURCE_ID
```

### Step 8: The SQL Database Saga

Our data analyst, armed with insights, creates a star model in the SQL database to be populated by the pipeline.

1. Navigate to the resource group using the Azure Portal.
2. Select the SQL Database
3. Select the Query Editor
4. Enter with your Microsoft Entra user provided to the script. The first time you do this, you'll need to configure the firewall by following the portal instructions.
5. Copy the code from ./sql/star_model.sql, and paste on the Query Editor
6. Execute
7. Review the tables that were created and explore any [stored procedures](https://learn.microsoft.com/azure/data-factory/connector-sql-server?tabs=data-factory#invoke-a-stored-procedure-from-a-sql-sink)
8. **Grant permissions to the Azure Data Factory Managed Identity inside the Database**. Copy the code from ./sql/UserManageIdentity.sql and paste it into the Query Editor.
9. Execute the script.

### Step 9: Execute the Azure Data Factory Pipeline

- Go to Azure Data Factory,
- Launch Azure Data Factory studio
- Go to Author/Pipeline -> IngestNYBabyNames_PL
- Add Trigger-> Trigger Now

### Step 10: Monitoring

You can [natively monitor all of your pipeline runs](https://learn.microsoft.com/azure/data-factory/monitor-visually#monitor-pipeline-runs) in the Azure Data Factory user experience. To access the monitoring feature, select the 'Monitor' tile in the Data Factory Studio, and then 'Pipeline runs'.

By default, all data factory runs are displayed in the browser's local time zone. If you change the time zone, all date/time fields adjust to the one you've selected.  

Azure Databricks does not send logs to Azure Monitor by default, but you can enable [diagnostic log delivery](https://learn.microsoft.com/azure/databricks/admin/account-settings/audit-log-delivery) (for example, to Log Analytics). For pipeline-level troubleshooting in this sample, you can still select the notebook execution activity (it may take some time to appear), click on the glasses icon, and follow the [databricks link to check the notebook execution log](https://learn.microsoft.com/en-us/azure/data-factory/transform-data-using-databricks-notebook#monitor-the-pipeline-run).  

Wait for the pipeline success.

The solution uses [Azure Data Lake Storage](https://learn.microsoft.com/azure/storage/blobs/data-lake-storage-introduction). A data lake is a single, centralized repository where you can store all your data, both structured and unstructured. Azure Data Lake Storage is a set of capabilities dedicated to big data analytics, built on Azure Blob Storage. It is possible to check it. Navigate to the resource group, select the Storage Account and see the containers. You will be able to find a 'landing' container where the .csv from api was stored, or bronze, silver and gold containers with the [delta tables](https://learn.microsoft.com/azure/databricks/delta/). All new tables in Databricks are, by default created as Delta tables. A Delta table stores data as a directory of files in cloud object storage and registers that table's metadata to the metastore within a catalog and schema. 

### Step 11: The Quest for Insights

After the pipeline populates the database, you can execute queries in the SQL Database to uncover the most popular names and trends. To do this, navigate to the resource group and open the SQL Database Query Editor.

```sql
-- most common female names used in New York in 2019
SELECT top 10  n.first_name, SUM(f.count) AS total_count
FROM [data].[fact_babynames] f
JOIN [data].[dim_names] n ON f.nameSid = n.sid
JOIN [data].[dim_years] y ON f.yearSid = y.sid
WHERE n.sex = 'F' and y.year = 2019
GROUP BY n.first_name
ORDER BY total_count DESC

-- Abel by year
SELECT y.year, sum(f.count) as total_count
FROM [data].[fact_babynames] f
JOIN [data].[dim_names] n ON f.nameSid = n.sid
JOIN [data].[dim_years] y ON f.yearSid = y.sid
WHERE n.first_name= 'ABEL'
GROUP BY y.year
ORDER BY total_count DESC
```

### Step 12: The Journey's End

When you're done, delete the resources and the resource group. After the delete finishes, purge the soft-deleted Key Vaults created by this reference implementation so you can redeploy with the same resource group name:

```bash
# First Navigate to resource group locks, then delete of all them. Next, execute the script.

az group delete -n $RESOURCEGROUP -y

# Purge only the Key Vaults created by this sample.
for KV in $(az keyvault list-deleted --query "[?properties.location=='${LOCATION}' && (starts_with(name, 'dbricksKV'))].name" -o tsv); do
  az keyvault purge --name $KV --location $LOCATION
done
```

## Contributions

Please see our [Contributor guide](./CONTRIBUTING.md).

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/). For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact <opencode@microsoft.com> with any additional questions or comments.

With :heart: from Microsoft Patterns & Practices, [Azure Architecture Center](https://aka.ms/architecture).