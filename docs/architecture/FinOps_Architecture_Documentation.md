# [COMPANY] FinOps Hub — Technical Architecture Documentation

> **Version:** 1.0 | **Date:** June 2026 | **Author:** [AUTHOR] — Data Architecture Team  
> **Status:** CONFIDENTIAL — Internal Use Only

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Layer 1 — Data Sources](#3-layer-1--data-sources)
4. [Layer 2 — Storage (ADLS Gen2)](#4-layer-2--storage-adls-gen2)
5. [Layer 3 — ETL Orchestration (Azure Data Factory)](#5-layer-3--etl-orchestration-azure-data-factory)
6. [Layer 4 — Analytics Store (Azure Data Explorer)](#6-layer-4--analytics-store-azure-data-explorer)
7. [Layer 5 — Semantic Model (Power BI Dataset)](#7-layer-5--semantic-model-power-bi-dataset)
8. [Layer 6 — Power BI Report](#8-layer-6--power-bi-report)
9. [Ownership & Access Requirements](#9-ownership--access-requirements)
10. [Testing & Correctness Checklist](#10-testing--correctness-checklist)
11. [Known Issues & Considerations](#11-known-issues--considerations)
12. [Glossary](#12-glossary)

---

## 1. Executive Summary

This document describes the end-to-end data flow of the [COMPANY] FinOps solution, built on the **Microsoft FinOps Toolkit (FinOps Hub)**. The solution collects Azure costs from Cost Management, processes them through Azure Data Factory, stores them in Azure Data Lake Storage Gen2 and Azure Data Explorer, and exposes them via a Power BI semantic model to the **"Reservations and Saving Plans Tracker"** report.

**Document objectives:**
- Document the complete architecture and data flow, layer by layer
- Identify data sources, transformations, and dependencies
- Provide a checklist for data correctness verification and source testing
- Support solution ownership transfer

---

## 2. Architecture Overview

### 2.0 Azure Resource Hierarchy

All FinOps Hub resources live within the following Azure organisational structure:

```
Tenant ([COMPANY])
└── Management Group
    └── Subscription — DT Enterprise Architecture (${AZURE_SUBSCRIPTION_ID})
        └── Resource Group — ${RESOURCE_GROUP}
            ├── Storage Account (ADLS Gen2)  ${ADLS_ACCOUNT_NAME}
            ├── Azure Data Factory           ${ADF_NAME}
            └── Azure Data Explorer cluster  ${ADX_CLUSTER_NAME}
```

Roles assigned at Management Group level are **inherited** by all child subscriptions and resources. Billing Account and Reservations are **tenant-level scopes**, separate from the subscription hierarchy — IAM on those must be configured independently.

**Key portal entry points:**

| **Resource**<br> | **How to reach it**<br> |
| --- | --- |
| ADLS Gen2 — browse files<br> | Search `${ADLS_ACCOUNT_NAME}` → Storage Browser<br> |
| ADF — pipeline runs and triggers<br> | Search `${ADX_CLUSTER_NAME}-engine` → Launch Studio → Monitor<br> |
| ADX — query cost data<br> | Search `${ADX_CLUSTER_NAME}` → Query → database `Hub`<br> |
| Reserved Instances<br> | Search `Reservations` → tenant-level view<br> |
| Billing Account (EA)<br> | Search `Cost Management + Billing` → Billing Account `${EA_BILLING_ACCOUNT_ID}`<br> |
| My role assignments on a resource<br> | Open resource → Access Control (IAM) → Check access → search your name<br> |

**IAM roles relevant to this project:**

| **Role**<br> | **Scope**<br> | **Required for**<br> |
| --- | --- | --- |
| Reader<br> | Subscription / Management Group<br> | Read Azure resources (VMs, storage, etc.)<br> |
| Billing Reader<br> | Management Group<br> | Read subscription cost data<br> |
| Billing Account Reader<br> | Billing Account (EA `${EA_BILLING_ACCOUNT_ID}`)<br> | Read RI transactions and usage data in Power BI<br> |
| Reservations Reader<br> | Tenant<br> | Direct access to Azure Reservations API/portal — **not required** for Power BI RI tables (those read via billing API)<br> |
| Storage Blob Data Reader<br> | Storage Account<br> | Read files in ADLS Gen2<br> |
| Contributor<br> | Subscription / Resource<br> | Create and modify resources (not manage access)<br> |

> **Note:** time-bound assignments (PIM) expire automatically. Permanent assignments remain until explicitly removed.

### 2.1 Solution Layers

| **Layer**<br> | **Technology**<br> | **Azure Resource / Identifier**<br> |
| --- | --- | --- |
| 1. Cost source<br> | Azure Cost Management Exports<br> | EA Billing Account + Subscription scope<br> |
| 1b. Operational VM data<br> | SharePoint Online (OData)<br> | List "SP - Azure VMs" (ownership, keep, role)<br> |
| 1c. VM/SQL inventory<br> | Azure Resource Graph / ARM API<br> | Subscription `${AZURE_SUBSCRIPTION_ID}`<br> |
| 2. Storage<br> | ADLS Gen2<br> | `${ADLS_ACCOUNT_NAME}.dfs.core.windows.net`<br> |
| 3. ETL Orchestration<br> | Azure Data Factory<br> | `${ADF_NAME}` (RG: `${RESOURCE_GROUP}`)<br> |
| 4. Analytics store<br> | Azure Data Explorer<br> | `${ADX_CLUSTER_ENDPOINT}`<br> |
| 5. Semantic Model<br> | Power BI Service Dataset<br> | `${PBI_DATASET_ID}`<br> |
| 6. Report<br> | Power BI Report<br> | `${PBI_REPORT_ID}` (workspace `${PBI_WORKSPACE_ID}`)<br> |

### 2.2 Sequential Data Flow

**Path A — Cost data (ADF ingestion pipeline)**

Cost data follows a multi-stage pipeline before reaching Power BI. Each step is triggered automatically by the previous one.

| **Step**<br> | **Component**<br> | **Trigger**<br> | **What happens**<br> |
| --- | --- | --- | --- |
| 1<br> | Azure Cost Management<br> | ADF scheduled trigger (daily / monthly)<br> | Exports Parquet/CSV + `manifest.json` to ADLS Gen2 `/msexports/`<br> |
| 2<br> | ADF — `msexports_ExecuteETL`<br> | BlobEvent: `msexports_ManifestAdded`<br> | Schema mapping, EA/MCA channel detection, converts to Parquet Snappy<br> |
| 3<br> | ADLS Gen2 `/ingestion/`<br> | Output of step 2<br> | Stores processed Parquet files + `manifest.json`<br> |
| 4<br> | ADF — `ingestion_ExecuteETL`<br> | BlobEvent: `ingestion_ManifestAdded`<br> | Loads Parquet files into Azure Data Explorer<br> |
| 5<br> | Azure Data Explorer — `Hub`<br> | After step 4<br> | Cost records available for query in FOCUS schema<br> |
| 6<br> | Power BI — `Costs` table<br> | DirectQuery on every report interaction<br> | Live cost data served to the semantic model and report<br> |

**Path B — Operational data (Power BI Import refresh)**

All other tables are loaded directly by Power BI on a scheduled refresh — no intermediate pipeline.

| **Source**<br> | **Protocol**<br> | **Tables loaded**<br> |
| --- | --- | --- |
| SharePoint Online<br> | OData<br> | `SP - Azure VMs`<br> |
| Azure Resource Graph / ARM<br> | OAuth2<br> | `VmQueryTOTAL`, `VmQueryUpdate`, `SQL_ManagedIstances`<br> |
| Azure Cost Management<br> | OAuth2<br> | `RI transactions`, `RI usage summary`<br> |

---

## 3. Layer 1 — Data Sources

### 3.1 Azure Cost Management Exports

| Parameter | Value |
|---|---|
| Contract type | Enterprise Agreement (EA) |
| Supported export types | Actual Cost, Amortized Cost, FOCUS 1.x, Reservation Details, Reservation Transactions |
| File format | Parquet (Snappy) or CSV (Gzip) |
| Frequency | Daily (MTD) and Monthly (previous month) |
| Folder structure | `Container/Directory/ExportName/[YYYYMMDD-YYYYMMDD]/[RunID]/` |
| Manifest file | `manifest.json` per run: partitions list, byteCount, dataRowCount, exportConfig |
| Historical data | Up to 13 months via portal, up to 7 years via REST API |

#### EA Export Schema (v2023-12-01-preview) — Key Columns

| Field | Description |
|---|---|
| `SubscriptionId` / `SubscriptionName` | Azure subscription |
| `ResourceGroup` / `ResourceLocation` / `Date` | Resource group, region, charge date |
| `MeterCategory` / `ProductName` / `MeterName` | Service and product classification |
| `Quantity` / `EffectivePrice` / `CostInBillingCurrency` | Quantity, blended price, billed cost |
| `ChargeType` | `Usage` / `Purchase` / `Refund` |
| `PricingModel` | `On Demand` / `Reservation` / `SavingsPlan` / `Spot` |
| `ReservationId` / `ReservationName` | Applied Reserved Instance identifier |
| `benefitId` / `benefitName` | Applied Savings Plan identifier |
| `BillingAccountId` / `BillingAccountName` | EA enrollment billing account |
| `IsAzureCreditEligible` | Eligible for Azure credits (`True`/`False`) |
| `CostAllocationRuleName` | Applied cost allocation rule |

#### FOCUS Schema (v1.2-preview) — Key Columns Used in Report

| FOCUS Field | Description |
|---|---|
| `BilledCost` | Base cost for invoicing (no RI/SP amortization) |
| `EffectiveCost` | Amortized cost (includes RI/SP pro-rated discount) |
| `ContractedCost` | Cost at negotiated prices, excluding RI/SP |
| `CommitmentDiscountId/Name/Type/Status` | Applied Reserved Instance or Savings Plan data |
| `ConsumedQuantity` / `ConsumedUnit` | Consumed quantity and unit of measure |
| `ResourceId` / `ResourceName` / `ResourceType` | Target Azure resource |
| `ServiceName` / `ServiceCategory` | Service (e.g., Virtual Machines) and category |
| `SubAccountId` / `SubAccountName` | Subscription |
| `Tags` | Resource JSON tags |
| `x_CommitmentDiscountSavings` | Savings generated by RI/SP vs. list price |
| `x_ConsumedCoreHours` | Core-hours consumed (VM-specific) |
| `x_ResourceGroupName` / `x_ResourceType` | ARM resource group and type |
| `x_SkuMeterCategory` | Meter category (e.g., Virtual Machines, SQL Database) |
| `x_PricingUnitDescription` | Pricing unit description |

> **Note:** The FinOps Hub preferentially uses the FOCUS format for compatibility with the international FinOps Open Cost and Usage Specification standard.

### 3.2 SharePoint Online — SP - Azure VMs Table

Populated from a SharePoint list via OData connector. Contains operational VM metadata managed manually by the team.

| Power BI Column | OData Native Field | Content |
|---|---|---|
| `OData_1` (VM Name) | VM Name | VM name |
| `Owner` / `Role` | Role | VM owner / responsible person |
| `OData_2` (Keep) | Keep | Flag: keep this VM (yes/no) |
| `AO` | AO | Application Owner |
| `Location` | Location | Azure region |
| `RG` | RG | Resource Group |
| `Sku` | Sku | VM size (e.g., `Standard_D4s_v3`) |
| `CreationDate` | CreationDate | Creation date |
| `Deleted` | Deleted | Logical deletion flag |
| `Notes` | Notes | Free text notes |

### 3.3 Azure Resource Graph / ARM API

| Power BI Table | Source | Key Fields |
|---|---|---|
| `VmQueryTOTAL` | Azure Resource Graph (VMs) | `name`, `vmSize`, `powerState`, `resourceGroup`, `timeCreated`, `MinVMSize` (calc.) |
| `VmQueryUpdate` | Azure Resource Graph (VMs) | `location` |
| `SQL_ManagedIstances` | Azure Resource Graph (SQL MI) | `name`, `resourceGroup`, `location`, `status`, `subscriptionId`, `Cores`, `FamilySize` |

### 3.4 Reference / Lookup Tables

| Table | Content |
|---|---|
| `Regions` | Azure region mapping (standard name ↔ aliases) |
| `Sub-id` | Subscription ID → readable name mapping (`SubAccountName`) |
| `isf` | Instance Size Flexibility ratios per VM family |
| `Compa-SkuRatio` | SKU sizing ratios (`Family Ratio`, `MinVMSize`, `Sku Size`) |
| `Recommendation` | DAX-calculated table with RI/SP recommendations (`TOPN 100`) |

---

## 4. Layer 2 — Storage (ADLS Gen2)

### 4.1 Storage Account

| Parameter | Value |
|---|---|
| Account name | `${ADLS_ACCOUNT_NAME}` |
| DFS endpoint | `${ADLS_ACCOUNT_NAME}.dfs.core.windows.net` |
| Subscription | `${AZURE_SUBSCRIPTION_ID}` |
| Resource Group | `${RESOURCE_GROUP}` |
| ADF access | Managed Identity (linked service `${ADLS_ACCOUNT_NAME}`) |

### 4.2 Containers and Folder Structure

| Container | Path | Content | Format |
|---|---|---|---|
| `msexports` | `ExportName/[YYYYMMDD-YYYYMMDD]/[RunID]/` | Raw Cost Management exports | CSV, CSV+GZip, Parquet+Snappy |
| `msexports` | `.../_manifest.json` | Run manifest (ADF trigger) | JSON |
| `ingestion` | `<table>/<month>/` | Post-ETL Parquet files ready for ADX | Parquet + Snappy |
| `ingestion` | `.../_manifest.json` | Ingestion manifest (ADF trigger) | JSON |
| `config` | `settings.json` | Hub configuration (scopes, retention, version) | JSON |

> **Note:** Trigger `msexports_ManifestAdded` monitors pattern `/msexports/blobs/...manifest.json` on `BlobCreated` event. Trigger `ingestion_ManifestAdded` monitors `/ingestion/blobs/...manifest.json`.

---

## 5. Layer 3 — ETL Orchestration (Azure Data Factory)

### 5.1 ADF Resources

| Parameter | Value |
|---|---|
| Factory name | `${ADF_NAME}` |
| Resource Group | `${RESOURCE_GROUP}` |
| Subscription | `${AZURE_SUBSCRIPTION_ID}` |
| Tenant | `${AZURE_TENANT_ID}` |
| Based on | Microsoft FinOps Toolkit (`github.com/microsoft/finops-toolkit`) |

### 5.2 Linked Services (3)

| Name | Type | Target / Notes |
|---|---|---|
| `ftkRepo` | HTTP (Anonymous) | `https://github.com/microsoft/finops-toolkit/` — schema and release file downloads |
| `${ADLS_ACCOUNT_NAME}` | ADLS Gen2 (Azure Blob FS) | Hub primary storage account (all containers) |
| `hubDataExplorer` | Azure Data Explorer (Kusto) | Endpoint: `${ADX_CLUSTER_ENDPOINT}` \| SP ID: `${ADF_SP_CLIENT_ID}` |

### 5.3 Triggers (5)

| Trigger | Type | Schedule / Condition | Pipeline Activated |
|---|---|---|---|
| `config_DailySchedule` | ScheduleTrigger | Every 24h, 01:01 W.Europe (from 2023-01-01) | `config_StartExportProcess` (Recurrence=Daily) |
| `config_MonthlySchedule` | ScheduleTrigger | Days 2, 5, 19 of month, 01:11 W.Europe | `config_StartExportProcess` (Recurrence=Monthly) |
| `config_SettingsUpdated` | BlobEventsTrigger | BlobCreated on `/config/blobs/...settings.json` | `config_ConfigureExports` |
| `msexports_ManifestAdded` | BlobEventsTrigger | BlobCreated on `/msexports/blobs/...manifest.json` | `msexports_ExecuteETL` (folderPath, fileName) |
| `ingestion_ManifestAdded` | BlobEventsTrigger | BlobCreated on `/ingestion/blobs/...manifest.json` | `ingestion_ExecuteETL` (folderPath) |

### 5.4 Pipelines — Config & Export Flow

#### `config_StartExportProcess` (param: `Recurrence`)
Main export start pipeline. Reads configuration (scopes, recurrence), filters invalid scopes, and for each scope starts `config_RunExportJobs`.
- Activities: `Get Config` → `Set/Save Scopes` → `Filter Invalid Scopes` → `ForEach Export Scope` → `config_RunExportJobs`

#### `config_ConfigureExports`
Creates or updates Cost Management exports for each configured scope. Also triggered by `config_SettingsUpdated` when `settings.json` changes.

#### `config_InitializeHub`
Initialization pipeline run on first deploy or after an update. Reads configuration, sets version/scopes/retention, waits for ADX capacity to be available (`Until` loop → `Fail` on timeout).

#### `config_StartBackfillProcess` / `config_RunBackfillJob`
Enable historical data recovery: `StartBackfillProcess` iterates month by month over `[StartDate, EndDate]`; `config_RunBackfillJob` starts exports for each scope and period.

### 5.5 Pipelines — msexports Flow (Raw → Ingestion)

#### `msexports_ExecuteETL` (params: `folderPath`, `fileName`)
- Reads `manifest.json`: dataset type, schema version, scope, dates, EA/MCA channel
- Switch `Detect Channel`: handles schema differences between EA and MCA
- Verifies existence of schema mapping file (from `ftkRepo` on GitHub)
- Determines destination folder in `/ingestion/` and Hub table name
- `ForEach blob`: starts `msexports_ETL_ingestion`
- Copy activity: copies manifest to `ingestion` container

#### `msexports_ETL_ingestion` (params: `blobPath`, `destinationFile`, `destinationFolder`, `schemaFile`, `exportDatasetType`, `exportDatasetVersion`)
- Gets list of existing Parquet files for same period
- Loads schema mapping from FinOps Toolkit repository (`ftkRepo`)
- Adds calculated columns (e.g., `x_CommitmentDiscountSavings`)
- Switch `Convert to Parquet`: converts CSV/GZip → Parquet Snappy via Copy activity
- If raw export retention is not enabled: deletes original raw file

### 5.6 Pipelines — Ingestion Flow (Parquet → ADX)

#### `ingestion_ExecuteETL` (param: `folderPath`)
- Initial `Wait` to ensure complete file write
- `GetMetadata`: list Parquet files in folder
- `Filter`: exclude subfolders
- `Set Ingestion Timestamp`
- `ForEach` file: starts `ingestion_ETL_dataExplorer`
- `IfCondition`: if no files present, exits without error

#### `ingestion_ETL_dataExplorer` (params: `folderPath`, `fileName`, `originalFileName`, `ingestionId`, `table`)
- `Read Hub Config`: reads retention configuration from ADX
- `Set Final Retention Months`
- `Until Capacity Is Available`: waits for ADX capacity before ingesting
- Loads Parquet file into ADX target table (parameter `table`)

---

## 6. Layer 4 — Analytics Store (Azure Data Explorer)

### 6.1 Cluster and Database

| Parameter | Value |
|---|---|
| Cluster endpoint | `${ADX_CLUSTER_ENDPOINT}` |
| Region | West Europe |
| ADF authentication | Service Principal `${ADF_SP_CLIENT_ID}` |
| Tenant | `${AZURE_TENANT_ID}` |
| ADF secret | `hubDataExplorer_servicePrincipalKey` (secure ADF parameter) — verify expiry |
| Database | Verify exact name with: `.show databases` in ADX Studio |

### 6.2 Expected ADX Tables (FinOps Hub Standard)

| ADX Table (expected) | Content | Power BI Table |
|---|---|---|
| `Costs` | FOCUS amortized cost data with `x_*` extension columns | `Costs` |
| `ReservationTransactions` | RI purchases, refunds, exchanges | `RI transactions` |
| `ReservationDetails` / `ReservationUsageSummary` | RI usage (avgUtilizationPercentage) | `RI usage summary` |
| Hub internal config tables | Config, manifest, ingestion log | n/a |

> ⚠️ **ACTION REQUIRED:** Connect to the ADX cluster and verify exact table names and schema:
> ```kusto
> .show tables
> .show table Costs schema as json
> ```

---

## 7. Layer 5 — Semantic Model (Power BI Dataset)

### 7.1 Identifiers

| Parameter | Value |
|---|---|
| Dataset ID | `${PBI_DATASET_ID}` |
| Workspace ID | `${PBI_WORKSPACE_ID}` |
| Report ID | `${PBI_REPORT_ID}` |
| Connection type | Published dataset on Power BI Service (live connection from .pbix) |
| Power BI Desktop release | 2025.08 |

> **Note:** Power Query (M) scripts and DAX formulas are fully documented in sections 7.4–7.9 of this document, reconstructed from the TMDL source files in the git repository (`Finops/Progetto Reservation/*.SemanticModel/definition/`).

### 7.2 Model Tables (13)

| Table | Source | Key Fields |
|---|---|---|
| `Costs` | ADX / FOCUS | `CommitmentDiscountName/Type/Status`, `ConsumedQuantity/Unit`, `ContractedCost`, `EffectiveCost`, `ResourceId/Name`, `x_CommitmentDiscountSavings`, `x_ConsumedCoreHours`, `x_PricingUnitDescription`, `x_ResourceGroupName`, `x_ResourceType`, `x_SkuMeterCategory` |
| `RI transactions` | Cost Management / ADX | `armSkuName`, `purchasingSubscriptionName`, `quantity`, `QuantityNoRefund`, `reservationOrderName` |
| `RI usage summary` | Cost Management / ADX | `avgUtilizationPercentage` |
| `SP - Azure VMs` | SharePoint OData | `OData_1` (VM Name), `OData_2`/Keep, `Owner`/Role, `AO`, `Location`, `RG`, `Sku`, `CreationDate`, `Deleted`, `Notes` |
| `VmQueryTOTAL` | Resource Graph / ARM | `name`, `vmSize`, `powerState`, `resourceGroup`, `timeCreated`, `MinVMSize` |
| `VmQueryUpdate` | Resource Graph / ARM | `location` |
| `SQL_ManagedIstances` | Resource Graph / ARM | `name`, `resourceGroup`, `location`, `status`, `subscriptionId`, `Cores`, `FamilySize` |
| `Sub-id` | Lookup | `SubAccountName` |
| `Regions` | Lookup | `Regions` |
| `isf` | Lookup | ISF ratios per VM family |
| `Compa-SkuRatio` | Lookup | `Family Ratio`, `MinVMSize`, `Sku Size` |
| `Recommendation` | DAX calculated table | RI/SP recommendations (`TOPN 100`) |
| `vmquery_cbcp_automation` | Resource Graph / automation | VM data for CBCP automation |

### 7.3 Identified DAX Measures

| Measure | Table | Visual Label | Expected Purpose |
|---|---|---|---|
| `Extimation 1Y` | `Costs` | — | 1-year cost/savings projection |
| `Extimation 3Y` | `Costs` | — | 3-year cost/savings projection |
| `x_CommitmentDiscountUtilization` | `Costs` | — | Commitment discount utilization % (RI/SP) |
| `ResToHave` | `RI transactions` | **To Buy** | Number of RIs to purchase to cover VM demand |
| `ResToHaveSQL` | `RI transactions` | — | Number of SQL RIs to purchase |
| `Sum_RatioDivided` | `VmQueryTOTAL` | **Needed** | Aggregated ISF-based VM sizing ratio |
| `SumRatio` | `VmQueryUpdate` | — | Updated aggregated ratio |
| `NeededSQL` | `SQL_ManagedIstances` | — | Number of SQL MI RIs needed |

### 7.4 Model Parameters (Shared Expressions)

Parameters defined in `expressions.tmdl` and referenced in table queries:

| Parameter | Value | Used by |
|---|---|---|
| `Cluster URL (3)` | `https://${ADX_CLUSTER_ENDPOINT}` | `Costs` (DirectQuery) |
| `Number of Months (2)` | `1` | `Costs` — lookback window in months |
| `Default Granularity (2)` | `"Daily"` | `Costs` — `x_ReportingDate` grain: `startofday` vs `startofmonth` |
| `Cluster URL` / `Cluster URL (2)` | `https://${ADX_CLUSTER_ENDPOINT}` | Legacy parameters — not used by current queries |
| `Number of Months` | `13` | Legacy parameter — not used |
| `Default Granularity` | `"Daily"` | Legacy parameter — not used |

---

### 7.5 Power Query M Transformations — Per Table

#### `Costs` (DirectQuery — ADX)

**Source:** `AzureDataExplorer.Contents("https://${ADX_CLUSTER_ENDPOINT}", "Hub", "<dynamic KQL>", ...)`  
**Mode:** DirectQuery — no data is imported; every visual interaction generates and sends a live KQL query to ADX.

The M function dynamically builds a KQL query with these stages:

| Stage | KQL operation | Purpose |
|---|---|---|
| Date filter | `where ChargePeriodStart >= monthsago(1)` | Lookback = `Number of Months (2)` = 1 month |
| Date columns | `extend x_ChargeMonth`, `x_ReportingDate` | Month bucket; daily or monthly granularity switch |
| VM properties | `extend x_SkuVMProperties`, `x_CapacityReservationId` | Extract VM-specific fields from `x_SkuDetails` JSON |
| Azure Hybrid Benefit | `extend x_SkuCoreCount`, `x_SkuImageType`, `x_SkuType`, `x_SkuLicenseStatus/Type/Quantity/Unit` | AHB detection: Windows Server BYOL, SQL AHB flag |
| Core-hours | `extend x_ConsumedCoreHours = x_SkuCoreCount * ConsumedQuantity` | Total core consumption (VM usage rows only) |
| Commitment tracking | `extend x_CommitmentDiscountKey`, `x_CommitmentDiscountUtilizationPotential/Amount` | Numerator and denominator for RI/SP utilisation |
| Amortisation | `extend x_AmortizationCategory` | `"Principal"` (purchase row) vs `"Amortized Charge"` (usage row) |
| Savings | `extend x_CommitmentDiscountSavings`, `x_NegotiatedDiscountSavings`, `x_TotalSavings`, `x_*DiscountPercent` | Three discount layers: RI/SP, negotiated, total |
| Toolkit metadata | `extend x_ToolkitTool/Version`, `x_ResourceParentId/Name/Type` | FinOps Hub version tag; parent resource from `cm-resource-parent` tag |
| Unique name suffixes | `extend CommitmentDiscountNameUnique`, `ResourceNameUnique`, `x_ResourceGroupNameUnique`, `SubAccountNameUnique` | Disambiguate identical names by appending type or subscription |
| Free-cost reason | `extend x_FreeReason` | Why cost is 0: Trial, Preview, Low/No Usage, Committed, Unknown |
| Tag promotion | `extend tag_Application`, `tag_BusinessUnit`, `tag_Env`, `tag_Owner`, `tag_Product`, `tag_Project`, `tag_Purpose`, `tag_Service` | Flatten 8 selected tags from the JSON `Tags` bag (case-insensitive key matching + trim) |
| Top-1K flag | `extend x_ResourceTop1K` | Boolean: resource is in top 1,000 by `EffectiveCost` |
| M post-processing | `Table.ReorderColumns`, `Table.RemoveColumns` | Sort columns alphabetically; remove ~40 unused FOCUS columns |

---

#### `SP - Azure VMs` (Import — SharePoint Online)

**Source:** `SharePoint.Tables("${SHAREPOINT_SITE_URL}", [ApiVersion = 15])` → list ID `${SHAREPOINT_LIST_ID}`

| Step | M function | What changes |
|---|---|---|
| 1 | `Source{[Id="${SHAREPOINT_LIST_ID}"]}[Items]` | Access the specific SharePoint list by GUID |
| 2 | `Table.RenameColumns` | `ID` → `ID.1` |
| 3 | `Table.RenameColumns` | `Creation` → `ResID`; `OData_2` → `Keep` |
| 4 | `Table.Sort` | Sort by `OData_1` (VM name) ascending |
| 5 | `Table.RenameColumns` | `Owner` → `Role` |
| 6 | `Table.SelectRows` | **Filter**: drop rows where `ResID` is null or `""` *(added in commit `53c13fe`)* |

> **Why step 6 matters:** the `SP - Azure VMs.ResID` → `VmQueryTOTAL.id` relationship has cardinality `one` from `SP - Azure VMs`. Power BI requires the "one" side to be free of blank values — null `ResID` rows trigger error 104305. The SharePoint `Creation` field (mapped to `ResID`) is empty for VMs not yet linked to an Azure Resource ID.

---

#### `VmQueryUpdate` (Import — Azure Resource Graph)

**Source:** `AzureResourceGraph.Query("<KQL>", null, null, null, [resultTruncated=null])`

**KQL query:**
```kusto
resources
| where type == "microsoft.compute/virtualmachines"
| extend props = parse_json(properties)
| project
    id, name, location, resourceGroup, subscriptionId,
    provisioningState = tostring(props.provisioningState),
    priority          = tostring(props.priority),
    timeCreated       = tostring(props.timeCreated),
    vmSize            = tostring(props.hardwareProfile.vmSize),
    osType            = tostring(props.storageProfile.osDisk.osType),
    diskSizeGB        = todouble(props.storageProfile.osDisk.diskSizeGB),
    storageAccountType = tostring(props.storageProfile.osDisk.managedDisk.storageAccountType),
    publisher         = tostring(props.storageProfile.imageReference.publisher),
    offer             = tostring(props.storageProfile.imageReference.offer),
    sku               = tostring(props.storageProfile.imageReference.sku),
    version           = tostring(props.storageProfile.imageReference.version),
    networkInterfaceId = tostring(props.networkProfile.networkInterfaces[0].id),
    licenseType       = tostring(props.licenseType),
    computerName      = tostring(props.osProfile.computerName),
    adminUsername     = tostring(props.osProfile.adminUsername),
    bootDiagnosticsEnabled = tostring(props.diagnosticsProfile.bootDiagnostics.enabled),
    hyperVGeneration  = tostring(props.extended.instanceView.hyperVGeneration),
    powerState        = tostring(props.extended.instanceView.powerState.displayStatus),
    osName            = tostring(props.extended.instanceView.osName),
    osVersion         = tostring(props.extended.instanceView.osVersion)
```

**M steps after query:**

| Step | What changes |
|---|---|
| Sort | `timeCreated` DESC, `name` ASC |
| Remove columns | Drop storage, publisher, provisioning, network, OS detail columns — keep: `id`, `name`, `location`, `resourceGroup`, `subscriptionId`, `timeCreated`, `vmSize`, `osType`, `computerName`, `powerState` |
| Cast type | `timeCreated` → `datetime` |

---

#### `vmquery_cbcp_automation` (Import — ADLS Gen2 CSV)

**Source:** `Csv.Document(AzureStorage.BlobContents("https://${ADLS_ACCOUNT_NAME}.blob.core.windows.net/ingestion/vmquery_cbcp_automation.csv"), [Delimiter=",", Columns=25, Encoding=65001, ...])`

| Step | What changes |
|---|---|
| Read CSV | 25 columns, comma-delimited, UTF-8 |
| Promote headers | First row → column names |
| Cast types | `timeCreated` → datetime; `diskSizeGB` → Int64; `bootDiagnosticsEnabled` → logical; all others → text |

> This table contains VMs managed by a separate automation pipeline (CBCP). It carries the same schema as `VmQueryUpdate` but is sourced from a CSV snapshot in ADLS rather than a live Resource Graph query. It retains full detail columns (publisher, OS image, network interface, etc.) that are stripped when the table is merged into `VmQueryTOTAL`.

---

#### `VmQueryTOTAL` (Import — Union of two sources)

**Source:** M combination — not an external source.

```powerquery
Source = Table.Combine({VmQueryUpdate, vmquery_cbcp_automation})
```

| Step | What changes |
|---|---|
| Union | Append all rows from `VmQueryUpdate` and `vmquery_cbcp_automation` |
| Remove columns | Drop 16 detail columns present in `vmquery_cbcp_automation` but not needed in the unified table: `computerName`, `provisioningState`, `priority`, `diskSizeGB`, `storageAccountType`, `publisher`, `offer`, `sku`, `version`, `networkInterfaceId`, `licenseType`, `adminUsername`, `bootDiagnosticsEnabled`, `hyperVGeneration`, `osName`, `osVersion` |

> **Result:** the single unified VM inventory used throughout the model. Contains both live Resource Graph VMs (`VmQueryUpdate`) and automation-pipeline VMs (`vmquery_cbcp_automation`).

---

#### `SQL_ManagedIstances` (Import — Azure Resource Graph)

**Source:** `AzureResourceGraph.Query("<KQL>", null, null, null, [resultTruncated=null])`

**KQL query:**
```kusto
resources
| where type =~ "microsoft.sql/managedinstances"
| extend subnetResourceId = properties.subnetId
| extend vNetName  = tostring(split(subnetResourceId, "/", 8)[0])
| extend subnetName = tostring(split(subnetResourceId, "/", 10)[0])
| extend status   = tostring(properties.state)
| extend SkuName  = tostring(sku.family)   -- VM family, e.g. "GP_Gen5"
| extend Cores    = tostring(sku.capacity) -- number of vCores
```

No additional M steps. `Cores` is cast to `double` per the column type definition in the model.

---

#### `isf` (Import — Public Microsoft CSV)

**Source:** `Csv.Document(Web.Contents("https://aka.ms/isf"), [Delimiter=",", Columns=3, Encoding=65001, ...])`

| Step | What changes |
|---|---|
| Read CSV | 3 columns: `InstanceSizeFlexibilityGroup`, `ArmSkuName`, `Ratio` |
| Promote headers | First row → column names |
| Cast types | Group/ArmSkuName → text; Ratio → number |
| Filter | Remove rows where `ArmSkuName` contains `"Provisioned"` (removes provisioned IOPS storage entries that appear in the same file) |

---

#### `RI transactions` (Import — Azure Cost Management EA)

**Source:** `AzureCostManagement.Tables("Enrollment Number", "${EA_BILLING_ACCOUNT_ID}", 1, ...)` → `{[Key="ritransactions"]}[Data]`

| Step | What changes |
|---|---|
| Extract table | `ritransactions` table from the EA connector |
| Deduplicate | `Table.Distinct` on `reservationOrderId` — one row per RI order |
| Filter event type | Keep only `eventType = "Purchase"` rows (exclude refunds, exchanges) |
| Exclude specific orders | Remove rows where `reservationOrderName` contains `"[EXCLUDED_RI_ORDER_1]"` or `"[EXCLUDED_RI_ORDER_2]"` |

> The two excluded order names are specific RIs that appear to be duplicated or superseded — explicitly excluded to avoid inflating `ResToHave`.

---

#### `RI usage summary` (Import — Azure Cost Management EA)

**Source:** `AzureCostManagement.Tables("Enrollment Number", "${EA_BILLING_ACCOUNT_ID}", 1, ...)` → `{[Key="riusagesummary"]}[Data]`

| Step | What changes |
|---|---|
| Extract table | `riusagesummary` table from the EA connector |
| Deduplicate | `Table.Distinct` on `reservationOrderId` |
| Filter SKU | Remove rows where `skuName` contains `"fabric_capacity_cu_hour"` (Fabric capacity commitments, not RI/SP) |
| Sort | By `avgUtilizationPercentage` ascending (worst-utilised reservations first) |

---

#### `Regions` and `Sub-id` (Import — Embedded lookup data)

Both tables are **embedded binary data** compiled into the model at design time — not refreshed from an external source.

```powerquery
Source = Table.FromRows(
    Json.Document(Binary.Decompress(Binary.FromText("<base64>", BinaryEncoding.Base64), Compression.Deflate)),
    ...
)
```

- **`Regions`**: single column `Regions` — Azure region codes and display names
- **`Sub-id`**: two columns — `SubscriptionId` (GUID) + `SubAccountName` (readable name)

> To update these values the model must be opened in Power BI Desktop and the embedded data regenerated. They do not sync automatically with Azure.

---

#### `Compa-SkuRatio` (DAX calculated table)

Not an M source — computed entirely in DAX at model refresh time from the `isf` table.

**DAX logic:**
1. From `isf`: trim whitespace from `InstanceSizeFlexibilityGroup` and `ArmSkuName`; filter out blank `Ratio` rows
2. Get distinct ISF groups
3. For each group compute:
   - `MinRatio` = minimum ratio value in the group (= ratio of the smallest VM)
   - `MinVMSize` = `ArmSkuName` of the VM with the minimum ratio (the "base" VM of the family)
4. Append a hard-coded row: `("SQLMI_GP_Compute_Gen5", 0, "SQLMI_GP_Compute_Gen5")` — SQL MI SKUs are absent from the ISF file, so this row is manually added to make the `SQL_ManagedIstances.FamilySize` → `Compa-SkuRatio` relationship work

**Output:** one row per ISF group with the name of its base VM (`MinVMSize`) and its ratio (`MinRatio`).

---

### 7.6 DAX Calculated Columns

Columns computed in DAX at refresh time — not loaded from any external source:

| Table | Column | DAX formula | Purpose |
|---|---|---|---|
| `VmQueryTOTAL` | `Base Size` | `LOOKUPVALUE(ISF[InstanceSizeFlexibilityGroup], ISF[ArmSkuName], VmQueryTOTAL[vmSize])` | ISF flexibility group for this VM |
| `VmQueryTOTAL` | `Ratio` | `LOOKUPVALUE(ISF[Ratio], ISF[ArmSkuName], VmQueryTOTAL[vmSize])` | ISF ratio for this VM |
| `VmQueryTOTAL` | `MinVMSize` | `LOOKUPVALUE(Compa-SkuRatio[MinVMSize], Compa-SkuRatio[InstanceSizeFlexibilityGroup], VmQueryTOTAL[Base Size])` | Smallest VM in the same ISF group |
| `VmQueryTOTAL` | `MinRatio` | `LOOKUPVALUE(Compa-SkuRatio[MinRatio], Compa-SkuRatio[MinVMSize], VmQueryTOTAL[MinVMSize])` | Ratio of the smallest VM in the group |
| `VmQueryTOTAL` | `RatioDivided` | `DIVIDE(VmQueryTOTAL[Ratio], VmQueryTOTAL[MinRatio])` | **Normalised size units** — how many base units this VM represents |
| `VmQueryUpdate` | `Base Size` | `LOOKUPVALUE(ISF[InstanceSizeFlexibilityGroup], ISF[ArmSkuName], VmQueryUpdate[vmSize])` | ISF group for delta inventory |
| `VmQueryUpdate` | `Ratio` | `LOOKUPVALUE(ISF[Ratio], ISF[ArmSkuName], VmQueryUpdate[vmSize])` | ISF ratio for delta inventory |
| `vmquery_cbcp_automation` | `Base Size` | `LOOKUPVALUE(ISF[InstanceSizeFlexibilityGroup], ISF[ArmSkuName], vmquery_cbcp_automation[vmSize])` | ISF group for automation VMs |
| `vmquery_cbcp_automation` | `Ratio` | `LOOKUPVALUE(ISF[Ratio], ISF[ArmSkuName], vmquery_cbcp_automation[vmSize])` | ISF ratio for automation VMs |
| `SQL_ManagedIstances` | `FamilySize` | `CALCULATE(FIRSTNONBLANK('RI transactions'[armSkuName], 1), FILTER('RI transactions', CONTAINSSTRING('RI transactions'[armSkuName], SQL_ManagedIstances[SkuName])))` | Match SQL MI SKU family to RI transaction `armSkuName` via substring search |

---

### 7.7 DAX Measures — Complete Reference

| Measure | Table | Formula | Business meaning |
|---|---|---|---|
| `Extimation 1Y` | `Costs` | `SUM(Costs[ConsumedQuantity]) * 12` | 1-year annualised consumption estimate |
| `Extimation 3Y` | `Costs` | `SUM(Costs[ConsumedQuantity]) * 36` | 3-year annualised consumption estimate |
| `x_CommitmentDiscountUtilization` | `Costs` | `IFERROR(SUM([x_CommitmentDiscountUtilizationAmount]) / SUM([x_CommitmentDiscountUtilizationPotential]), "")` | RI/SP utilisation rate: actual use divided by potential |
| `x_EffectiveSavingsRate` | `Costs` | `IFERROR(SUMX(FILTER(Costs, [x_AmortizationCategory] <> "Principal"), [x_TotalSavings]) / SUMX(FILTER(Costs, ...), [ListCost]), "")` | Overall savings rate; excludes Principal (purchase) rows from both numerator and denominator |
| `ResToHave` | `RI transactions` | `SUM(VmQueryTOTAL[RatioDivided]) - SUM('RI transactions'[quantity])` | **"To Buy"**: normalised VM demand minus existing RI quantity |
| `ResToHaveSQL` | `RI transactions` | `SUM(SQL_ManagedIstances[Cores]) - SUM('RI transactions'[quantity])` | SQL MI cores needed minus existing SQL RI quantity |
| `Sum_RatioDivided` | `VmQueryTOTAL` | `SUM(VmQueryTOTAL[RatioDivided])` | **"Needed"**: total normalised VM demand in base units |
| `SumRatio` | `VmQueryUpdate` | `SUM(VmQueryUpdate[Ratio])` | Raw ISF ratio sum across delta inventory |
| `NeededSQL` | `SQL_ManagedIstances` | `SUM(SQL_ManagedIstances[Cores])` | Total vCore demand across all SQL Managed Instances |
| `Total Rows SP` | `SP - Azure VMs` | `COUNTROWS('SP - Azure VMs')` | Row count of SharePoint VM registry (after ResID filter) |
| `Total Deleted VMs` | `SP - Azure VMs` | `CALCULATE(COUNTROWS('SP - Azure VMs'), 'SP - Azure VMs'[Deleted] = "YES")` | Count of logically deleted VMs in SharePoint |
| `Total Rows` | `VmQueryTOTAL` | `COUNTROWS('VmQueryTOTAL')` | Total VMs in unified inventory |
| `SumRatioCBCP` | `vmquery_cbcp_automation` | `SUM('vmquery_cbcp_automation'[Ratio])` | Raw ISF ratio sum for automation-pipeline VMs |

---

### 7.8 Core Business Logic — ISF Normalisation Chain

The central analytical logic of the report: compute how many Reserved Instances of the **smallest base size** in each VM family must be purchased to cover the entire fleet.

```
vmSize (e.g. "Standard_D8s_v3")
  → isf lookup by ArmSkuName
  → Ratio    = 4.0           (this VM = 4× the base unit in its family)
  → Base Size = "DSv3 Type1" (ISF flexibility group)

Base Size = "DSv3 Type1"
  → Compa-SkuRatio lookup by InstanceSizeFlexibilityGroup
  → MinVMSize = "Standard_D2s_v3"  (smallest VM in the group)
  → MinRatio  = 1.0                (its ISF ratio)

RatioDivided = Ratio / MinRatio = 4.0 / 1.0 = 4.0
  → this D8s_v3 counts as 4 Standard_D2s_v3-equivalent RI units

Sum_RatioDivided = SUM(RatioDivided across all VMs)
  → total RI demand expressed in D2s_v3-equivalent units

ResToHave = Sum_RatioDivided - SUM(RI transactions[quantity])
  → how many Standard_D2s_v3-equivalent RIs still to purchase
```

**Why this matters for data validation:** if the `isf` table has stale ratios, or a `vmSize` value does not match any `ArmSkuName` in the ISF file, `Ratio` will be blank, `RatioDivided` will compute as 0, and `ResToHave` will undercount required reservations. The validation step must cross-check `Sum_RatioDivided` against the expected number of VMs × their known ISF ratios.

---

### 7.9 Model Relationships Summary

| From table / column | To table / column | Cardinality | Filter direction | Notes |
|---|---|---|---|---|
| `SP - Azure VMs.ResID` | `VmQueryTOTAL.id` | 1:many | Both | **Core join**: links SharePoint VM registry to live Azure inventory |
| `RI usage summary.reservationOrderId` | `RI transactions.reservationOrderId` | many:1 | Single | Links daily utilisation records to purchase transactions |
| `VmQueryTOTAL.Base Size` | `Compa-SkuRatio.InstanceSizeFlexibilityGroup` | many:1 | Single | ISF group lookup for VM normalisation |
| `VmQueryUpdate.Base Size` | `Compa-SkuRatio.InstanceSizeFlexibilityGroup` | many:1 | Single | ISF group lookup for delta inventory |
| `vmquery_cbcp_automation.Base Size` | `Compa-SkuRatio.InstanceSizeFlexibilityGroup` | many:1 | Single | ISF group lookup for automation VMs |
| `RI transactions.armSkuName` | `Compa-SkuRatio.MinVMSize` | many:1 | Single | Links RI purchase SKU to ISF base size |
| `SQL_ManagedIstances.FamilySize` | `Compa-SkuRatio.InstanceSizeFlexibilityGroup` | many:1 | Single | SQL MI family → ISF group (via hard-coded `SQLMI_GP_Compute_Gen5` row) |
| `VmQueryUpdate.subscriptionId` | `Sub-id.SubscriptionId` | many:1 | Single | Subscription name lookup |
| `VmQueryTOTAL.subscriptionId` | `Sub-id.SubscriptionId` | many:1 | Single | Subscription name lookup |
| `SQL_ManagedIstances.subscriptionId` | `Sub-id.SubscriptionId` | many:1 | Single | Subscription name lookup |
| `RI transactions.purchasingSubscriptionGuid` | `Sub-id.SubscriptionId` | many:1 | Single | Purchasing subscription lookup |
| `vmquery_cbcp_automation.subscriptionId` | `Sub-id.SubscriptionId` | many:1 | Single | Subscription name lookup |
| `VmQueryUpdate.location` | `Regions.Regions` | many:1 | Single | Region display name lookup |
| `VmQueryTOTAL.location` | `Regions.Regions` | many:1 | Single | Region display name lookup |
| `SQL_ManagedIstances.location` | `Regions.Regions` | many:1 | Single | Region display name lookup |
| `vmquery_cbcp_automation.location` | `Regions.Regions` | many:1 | Single | Region display name lookup |
| `RI transactions.region` | `Regions.Regions` | many:1 | Single | Region display name lookup |
| Date auto-tables | `VmQueryUpdate.timeCreated`, `VmQueryTOTAL.timeCreated`, `vmquery_cbcp_automation.timeCreated`, `RI usage summary.usageDate` | 1:many | Single | Auto-generated date hierarchies |

---

## 8. Layer 6 — Power BI Report

**Report name:** [COMPANY] - Reservations and Saving Plans Tracker  
**File:** `[COMPANY] - Reservations and Saving Plans Tracker.pbix`

| Page | # Visuals | Content |
|---|---|---|
| All VMS | 11 | Aggregated view of all VMs: inventory, sizes, power state, RI/SP coverage |
| VM_ReservedInstances | 10 | Reserved Instances detail per VM: utilization %, RIs to buy (`To Buy = ResToHave`) |
| SQL MI_ReservedIstances | 7 | Reserved Instances for SQL Managed Instances: needed, utilization, transactions |
| [DRAFT] VM_SavingPlans | 13 | **DRAFT** — Savings Plans per VM: 1Y/3Y projections, coverage, recommendations |
| vm list query | 7 | VM list with inventory and current state query |

> ⚠️ Page **[DRAFT] VM_SavingPlans** is in draft state. Verify whether it should be hidden or completed before distributing to end users.

---

## 9. Ownership & Access Requirements

### 9.1 Required Access for Full Ownership

| Resource | Role / Permission | How to Obtain |
|---|---|---|
| Azure Data Factory | Data Factory Contributor | Azure Portal → `${RESOURCE_GROUP}` → IAM |
| ADLS Gen2 | Storage Blob Data Contributor | Azure Portal → Storage Account → IAM |
| Azure Data Explorer | Database Admin | ADX Portal → cluster → Permissions |
| EA Billing Account | Enterprise Administrator (Read-only) | `ea.azure.com` → role management |
| Cost Management | Cost Management Reader | Azure Portal → Subscriptions → IAM |
| EA Billing Account (RI data) | **Billing Account Reader** (Azure Portal IAM) + **BENEFITS READER** (ea.azure.com) | Azure Portal → Cost Management + Billing → Billing Account `${EA_BILLING_ACCOUNT_ID}` → IAM; and ea.azure.com for EA portal roles |
| Power BI Workspace | Workspace Admin or Member | Power BI Service → Workspace → Access |
| Power BI Dataset | Dataset Owner | Power BI Service → Dataset → Manage permissions |
| SharePoint (SP - Azure VMs list) | Read or Contribute | SharePoint Admin / site owner |

### 9.2 ADF → ADX Service Principal

| Parameter | Value |
|---|---|
| Client ID (SP ID) | `${ADF_SP_CLIENT_ID}` |
| Tenant | `${AZURE_TENANT_ID}` |
| Secret | Stored as secure ADF parameter `hubDataExplorer_servicePrincipalKey` |

> ⚠️ **ACTION REQUIRED:** Verify secret expiry at: Azure Portal → App Registrations → `${ADF_SP_CLIENT_ID}` → Certificates & secrets  
> Required ADX role: at minimum **Database Ingestor** + **Table Admin** on target tables.

---

## 10. Testing & Correctness Checklist

### 10.1 Source Verification

| # | Test | How to Verify | Status |
|---|---|---|---|
| 1 | Cost Management exports active and scheduled | Azure Portal → Cost Management → Exports: verify export list and last successful run | `[ ]` |
| 2 | Files present in `/msexports/` | Storage Explorer → container `msexports`: verify `YYYYMMDD-YYYYMMDD` folders with `manifest.json` | `[ ]` |
| 3 | Parquet files present in `/ingestion/` | Storage Explorer → container `ingestion`: verify post-ETL `.parquet` files | `[ ]` |
| 4 | SharePoint list accessible | Browser → SharePoint → SP - Azure VMs list: verify content and last update date | `[ ]` |
| 5 | Resource Graph query working | Azure Portal → Resource Graph Explorer: run query for VMs and SQL MI in tenant | `[ ]` |
| 6 | `settings.json` valid in `/config/` | Storage Explorer → container `config`: open `settings.json`, verify scopes and version | `[ ]` |

### 10.2 ADF Verification

| # | Test | How to Verify | Status |
|---|---|---|---|
| 7 | All 5 triggers active | ADF Studio → Monitor → Triggers: verify "Started" status for all triggers | `[ ]` |
| 8 | Last `msexports_ExecuteETL` run successful | ADF Studio → Monitor → Pipeline runs → `msexports_ExecuteETL`: last run Succeeded | `[ ]` |
| 9 | Last `ingestion_ExecuteETL` run successful | ADF Studio → Monitor → Pipeline runs → `ingestion_ExecuteETL`: last run Succeeded | `[ ]` |
| 10 | Service Principal secret not expired | Azure Portal → App Registrations → `${ADF_SP_CLIENT_ID}` → Certificates & secrets | `[ ]` |

### 10.3 ADX Verification

| # | Test | KQL Command | Status |
|---|---|---|---|
| 11 | Tables present in database | `.show tables` | `[ ]` |
| 12 | Recent data in Costs | `Costs \| where ChargePeriodStart > ago(7d) \| count` | `[ ]` |
| 13 | RI transactions present | `ReservationTransactions \| take 10` | `[ ]` |
| 14 | FOCUS schema correct | `.show table Costs schema as json` | `[ ]` |
| 15 | No recent ingestion errors | `.show ingestion failures \| where FailedOn > ago(7d)` | `[ ]` |

### 10.4 Power BI Verification

| # | Test | How to Verify | Status |
|---|---|---|---|
| 16 | Dataset refreshed with recent data | PBI Service → Dataset → Refresh history: last refresh successful and recent | `[ ]` |
| 17 | Costs table not empty | PBI Service → Dataset → Explore data: count `Costs` rows for current month > 0 | `[ ]` |
| 18 | RI transactions consistent | Compare RI transactions row count PBI vs ADX for same period | `[ ]` |
| 19 | "To Buy" measure plausible | Open page `VM_ReservedInstances`: verify `ResToHave` is numeric and reasonable | `[ ]` |
| 20 | SharePoint data up to date | Compare `VM Name` in `SP - Azure VMs` with `VmQueryTOTAL`: verify coverage | `[ ]` |
| 21 | DRAFT page not exposed to users | PBI Service → Report: verify `[DRAFT] VM_SavingPlans` is hidden or report not distributed | `[ ]` |

---

## 11. Known Issues & Considerations

### 11.1 Pre-Existing Issues

- **RI usage / RI transactions — resolved 2026-06-24:** required roles are Billing Account Reader (Azure Portal → Cost Management + Billing → Billing Account `${EA_BILLING_ACCOUNT_ID}` → IAM) and BENEFITS READER on ea.azure.com. Reservations Reader (Azure RBAC) is NOT required — the `AzureCostManagement.Tables` connector reads via billing API, not Reservations API. Auth method: OAuth2 / Organizational account (${PBI_SERVICE_ACCOUNT}).
- **EA API key potential expiry:** the legacy `AzureCostManagement.Tables` (EA) connector requires an enrollment number and access key renewable annually from `ea.azure.com`.
- **Table name typos:** `SQL_ManagedIstances` (one 's' in 'Istances') and report page `VM_ReservedIstances` — keep unchanged for compatibility.
- **Measure name typo:** `Extimation` (should be `Estimation`) — do not correct without updating all dependent visualizations.

### 11.2 Deprecations to Monitor

- **EA Cost Management connector (`AzureCostManagement.Tables`):** Microsoft is deprecating this connector. Plan migration to the Cost Management Export connector or FOCUS dataset.
- **FinOps Toolkit version:** verify updates at `github.com/microsoft/finops-toolkit` and plan hub upgrade.
- **FOCUS 1.2-preview:** version is in preview. Monitor GA release to update schema mapping in ADF.

### 11.3 Remaining Documentation Gaps

- **Actual ADX KQL schema:** must be extracted directly from the cluster with `.show table <name> schema as json`.
- **Cost Management monitored scopes:** list of monitored subscriptions in `config/settings.json` in ADLS storage — read to know coverage.

> **Closed gaps (2026-06-23):** Power Query M transformations and DAX measure formulas have been fully reconstructed from TMDL source files and documented in sections 7.4–7.9 of this document.

---

## 12. Glossary

| Term | Definition |
|---|---|
| ADX / Kusto | Azure Data Explorer, columnar analytical database for large data volumes |
| ADLS Gen2 | Azure Data Lake Storage Gen2, scalable Big Data storage |
| ADF | Azure Data Factory, Azure ETL/ELT orchestration service |
| EA | Enterprise Agreement, Microsoft enterprise contract for Azure billing |
| FinOps Hub | Microsoft open-source solution (FinOps Toolkit) for Azure cost management |
| FOCUS | FinOps Open Cost and Usage Specification, international standard for cloud cost data |
| ISF | Instance Size Flexibility: RI ability to cover different VM sizes in the same family |
| MCA | Microsoft Customer Agreement, alternative contract type to EA |
| RI | Reserved Instance, long-term Azure capacity purchase commitment (1 or 3 years) |
| Savings Plan (SP) | Flexible compute spend commitment for Azure |
| Parquet Snappy | Columnar compressed file format, optimal for analytics |
| Semantic Model | Power BI data layer with tables, relationships, and DAX measures |

---

*Document generated automatically — [COMPANY] Data Architecture Team — June 2026*
