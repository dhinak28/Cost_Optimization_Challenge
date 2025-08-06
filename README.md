# Cost_Optimization_Challenge
Solution to the challenge
# 📦 Azure Billing Archive & Retrieval System

## **Overview**
This solution provides a **cost‑optimized**, **zero‑downtime**, and **highly available** way to store and retrieve billing records in Azure.  

It uses:
- **Azure Cosmos DB** for recent data (**Hot Tier**)
- **Azure Blob Cold Tier** for archival storage (**Cold Tier**)

Data older than **90 days** is **automatically moved** to cheaper cold storage while remaining **instantly retrievable**.

---

## **Architecture**

[Architecture Diagram]
<img width="1024" height="1536" alt="Image Aug 7, 2025, 01_56_54 AM" src="https://github.com/user-attachments/assets/795cf824-03d2-4372-96cd-149cb4debed3" />


### **Retrieval Flow**
```
User Request
    ↓
HTTP API (Retrieve Function)
    ↓
Check Cosmos DB (Hot Data, < 90 days)
↓               ↘
Found?            Check Blob Archive (Cold Data, > 90 days)
↓                      ↓
Return Data         Return Data
```

### **Archival Process**
```
Daily Timer Trigger (Archive Function)
    ↓
Query Cosmos DB for records older than 90 days
    ↓
Upload to Blob Storage (Cold Tier) + Store Checksum
    ↓
Delete from Cosmos DB after successful archive
```

---

## **Azure Resources Used**
| Resource | Purpose | Tier / Notes |
|----------|---------|--------------|
| **Resource Group** (`BillingRG`) | Logical grouping | East US |
| **Cosmos DB** (`billingcosmosdb`) | Hot storage | 400 RU/s autoscale |
| **Azure Blob Storage** (`billingarchivestorage`) | Cold archive | `Standard_GZRS` |
| **Function App** (`BillingArchiveApp`) | Archive & retrieve | Python 3.10 |
| **Azure Monitor** | Alerts | Latency, Failures |
| **Azure Automation** | Backup verification | Monthly run |

---

## **Deployment Instructions**

### **1. Prerequisites**
- [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)
- [Terraform](https://developer.hashicorp.com/terraform/downloads) *(optional, if using IaC)*
- [Azure Functions Core Tools](https://learn.microsoft.com/azure/azure-functions/functions-run-local)

---

### **2. Login to Azure**
```bash
az login
az account set --subscription "<YOUR_SUBSCRIPTION_ID>"
```

---

### **3. Create Resource Group**
```bash
az group create --name BillingRG --location "East US"
```

---

### **4. Create Cosmos DB**
```bash
az cosmosdb create   --name billingcosmosdb   --resource-group BillingRG   --locations regionName="East US" failoverPriority=0   --default-consistency-level Session

az cosmosdb sql database create   --account-name billingcosmosdb   --name BillingDB   --resource-group BillingRG

az cosmosdb sql container create   --account-name billingcosmosdb   --database-name BillingDB   --name BillingContainer   --partition-key-path "/partitionKey"
```

---

### **5. Create Storage Account & Container**
```bash
az storage account create   --name billingarchivestorage   --resource-group BillingRG   --location "East US"   --sku Standard_GZRS   --kind StorageV2

az storage container create   --account-name billingarchivestorage   --name billing-archive   --auth-mode login
```

---

### **6. Deploy Function App**
```bash
az functionapp create   --resource-group BillingRG   --consumption-plan-location "East US"   --runtime python   --runtime-version 3.10   --functions-version 4   --name BillingArchiveApp   --storage-account billingarchivestorage
```

---

### **7. Configure Environment Variables**
```bash
COSMOS_URI=$(az cosmosdb show --name billingcosmosdb --resource-group BillingRG --query documentEndpoint -o tsv)
COSMOS_KEY=$(az cosmosdb keys list --name billingcosmosdb --resource-group BillingRG --query primaryMasterKey -o tsv)
AZURE_STORAGE_CONN=$(az storage account show-connection-string --name billingarchivestorage --resource-group BillingRG --query connectionString -o tsv)

az functionapp config appsettings set   --name BillingArchiveApp   --resource-group BillingRG   --settings COSMOS_URI=$COSMOS_URI              COSMOS_KEY=$COSMOS_KEY              COSMOS_DB="BillingDB"              COSMOS_CONTAINER="BillingContainer"              AZURE_STORAGE_CONN="$AZURE_STORAGE_CONN"
```

---

### **8. Deploy Functions**
#### Install Azure Functions Core Tools:
```bash
npm install -g azure-functions-core-tools@4 --unsafe-perm true
```

#### Create function project:
```bash
func init functions --python
cd functions
func new --name ArchiveFunction --template "Timer trigger"
func new --name RetrieveFunction --template "HTTP trigger"
```

Replace `__init__.py` with your archive/retrieve scripts.

#### Update `requirements.txt`:
```
azure-cosmos
azure-storage-blob
```

#### Deploy:
```bash
func azure functionapp publish BillingArchiveApp
```

---

### **9. Schedule Archival**
```bash
az functionapp config appsettings set   --name BillingArchiveApp   --resource-group BillingRG   --settings "TIMER_SCHEDULE=0 0 * * *"
```

---

### **10. Monitoring**
```bash
az monitor metrics alert create   --name HighLatencyAlert   --resource-group BillingRG   --scopes $(az storage account show --name billingarchivestorage --resource-group BillingRG --query id -o tsv)   --condition "avg BlobServiceLatency > 2000"   --description "Alert if retrieval latency exceeds 2s"
```

---

### **11. Monthly Backup Verification**
```bash
az automation account create   --resource-group BillingRG   --name BillingAutomation   --location "East US"

az automation runbook create   --resource-group BillingRG   --automation-account-name BillingAutomation   --name VerifyBackup   --type Python3   --runbook-file ../scripts/verify_offline_backup.py
```

---

## **Fail‑Safe Against Global Azure Outage**
For **full resilience**, extend this system to also store archives in:
- **AWS Glacier** (Ultra‑low‑cost deep archive)
- **GCP Nearline** (Low‑latency cold storage)

The archive function can be extended to write to all three clouds.  
Retrieval logic will check:
```
Azure → AWS → GCP
```
This guarantees **zero downtime** even if a full cloud provider outage occurs.

---
