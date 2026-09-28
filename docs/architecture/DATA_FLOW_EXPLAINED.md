# FinOps Hub — How the Data Flows

> A plain-English walkthrough of how data moves from Azure sources to Power BI visuals.  
> For the full technical reference (formulas, KQL queries, M code) see `FinOps_Architecture_Documentation.md` sections 7.4–7.9.

---

## What the report is trying to answer

The "Reservations and Saving Plans Tracker" answers three business questions:

1. **What are we running?** — A live inventory of every Azure VM and SQL Managed Instance, enriched with owner, role, and environment metadata
2. **Are our reservations being used?** — Utilisation analysis for existing Reserved Instances and Savings Plans
3. **What should we buy next?** — A data-driven recommendation: how many RIs of each VM family we still need to purchase to cover our fleet

Everything else in the model — the transformations, the DAX columns, the relationships — exists to compute these three answers correctly.

---

## Two data paths

The solution has two completely separate data paths depending on the type of data.

---

## Path A — Cost data

Cost data never goes directly from Azure to Power BI. It passes through a multi-stage pipeline first.

```
Azure Cost Management
  ↓  (ADF scheduled trigger — daily and on days 2, 5, 19 of each month)
ADLS Gen2  /msexports/
  [Raw export files: Parquet + manifest.json]
  ↓  (ADF pipeline: msexports_ExecuteETL — triggered by manifest file creation)
  Schema mapping from EA format → FOCUS standard
  Channel detection: EA vs MCA
  Conversion to Parquet Snappy
  ↓
ADLS Gen2  /ingestion/
  [Processed Parquet files + manifest.json]
  ↓  (ADF pipeline: ingestion_ExecuteETL — triggered by manifest file creation)
  Load into Azure Data Explorer
  ↓
Azure Data Explorer — database: Hub — table: Costs_v1_0
  [Cost records in FOCUS schema]
  ↓  (DirectQuery — no data imported into Power BI)
Power BI — Costs table
```

**Key point — DirectQuery:** the `Costs` table does not import data into Power BI. Every time you interact with a visual on the report, Power BI generates a KQL query on the fly and sends it live to the ADX cluster. This means cost data is always up to the last ADF run.

**What ADF does to the data:** the `msexports_ExecuteETL` pipeline reads the Cost Management export, detects whether it comes from an EA or MCA account, downloads the schema mapping file from Microsoft's FinOps Toolkit repository on GitHub, and converts the file from raw EA format to the FOCUS standard (FinOps Open Cost and Usage Specification — the international standard for cloud cost data).

**What happens inside the Power BI M query:** the `Costs` table's M code dynamically builds a KQL query that, on top of the raw FOCUS data in ADX, adds:

- A date filter: only data from the last 1 month (`ChargePeriodStart >= monthsago(1)`)
- `x_ConsumedCoreHours`: total CPU core-hours consumed by each VM row (`vCPU count × consumed quantity`)
- Azure Hybrid Benefit detection: identifies Windows Server BYOL and SQL AHB usage, computes the number of licences involved
- Three savings layers:
  - `x_CommitmentDiscountSavings`: what the RI/Savings Plan saves vs. the contracted price
  - `x_NegotiatedDiscountSavings`: what the negotiated price saves vs. the public list price
  - `x_TotalSavings`: total savings from list price to effective cost
- Tag promotion: eight resource tags (`Application`, `Env`, `Owner`, `Product`, etc.) are extracted from the JSON tag bag into dedicated columns, with case-insensitive key matching
- `x_ResourceTop1K`: a boolean flag marking whether a resource is in the top 1,000 by effective cost — used to limit visual row counts
- An explanation for zero-cost rows (`x_FreeReason`): Trial, Preview, Low Usage, No Usage, Unknown

After the KQL query runs, M reorders columns alphabetically and removes about 40 FOCUS columns that are not used in any visual.

---
## Path B — Operational data

All other tables are loaded directly by Power BI on a scheduled refresh. No ADF pipeline is involved.

### `SP - Azure VMs` — the team's VM registry (SharePoint Online)

This is a manually maintained SharePoint list. For each VM it records: who owns it, whether it should be kept, its application role, its resource group, and — critically — its **Azure Resource ID** (`ResID`).

**What M does:**
1. Connects to the SharePoint list via its internal GUID (`${SHAREPOINT_LIST_ID}...`)
2. Renames the `Creation` column to `ResID` (the column was named incorrectly in SharePoint but contains the Azure Resource ID)
3. Renames `OData_2` to `Keep` and `Owner` to `Role`
4. Sorts by VM name
5. **Filters out rows where `ResID` is null or empty**

Step 5 is critical: the relationship `SP - Azure VMs → VmQueryTOTAL` is a one-to-many join on `ResID`. Power BI requires the "one" side of the relationship to have no blank values — if a VM in SharePoint has no Resource ID yet, it would crash the model with error 104305. Rows without a `ResID` are VMs that exist in SharePoint but have not yet been associated with an actual Azure resource.

**Key columns used in the report:** `OData_1` (VM name), `Role` (owner/responsible), `Keep`, `AO` (application owner), `Deleted`, `ResID` (join key to live Azure inventory)

---

### `VmQueryUpdate` — live VM inventory (Azure Resource Graph)

Queries Azure Resource Graph in real time for all virtual machines in subscription `${AZURE_SUBSCRIPTION_ID}`. The KQL query extracts from the `properties` object: `vmSize`, `powerState`, `timeCreated`, `osType`, and identifiers (`id`, `name`, `location`, `resourceGroup`, `subscriptionId`).

M then sorts the result by creation date descending and removes the detail columns not needed for the report.

---

### `vmquery_cbcp_automation` — automation pipeline VMs (ADLS Gen2 CSV)

A CSV file in Blob Storage produced by a separate automation pipeline (CBCP). It contains VMs managed by that pipeline, with the same schema as `VmQueryUpdate` but with 25 columns — including extra detail (publisher, OS image, network interface, etc.) that is stripped when the table is merged.

---

### `VmQueryTOTAL` — the unified VM inventory

This is not an external data source. It is a **union** in Power Query M of the two tables above:

```powerquery
Source = Table.Combine({VmQueryUpdate, vmquery_cbcp_automation})
```

After the union, M removes the 16 detail columns that `vmquery_cbcp_automation` carries but that are not needed in the combined table.

The result is the single VM inventory used throughout the report — containing both live Resource Graph VMs and automation-managed VMs.

On top of this combined table, DAX then computes five calculated columns that drive the ISF normalisation logic (see below).

---

### `isf` — Instance Size Flexibility table (public Microsoft CSV)

The file at `https://aka.ms/isf` is a public Microsoft CSV that defines how VM sizes relate to each other within the same family. For example:

| ArmSkuName | InstanceSizeFlexibilityGroup | Ratio |
|---|---|---|
| Standard_D2s_v3 | DSv3 Type1 | 1 |
| Standard_D4s_v3 | DSv3 Type1 | 2 |
| Standard_D8s_v3 | DSv3 Type1 | 4 |
| Standard_D16s_v3 | DSv3 Type1 | 8 |

A ratio of 4 means: one `Standard_D8s_v3` RI can cover the same capacity as four `Standard_D2s_v3` VMs (the base size of the family). This is called Instance Size Flexibility — Microsoft allows RIs to be applied flexibly across sizes in the same family.

M filters out rows where `ArmSkuName` contains "Provisioned" (those are storage IOPS entries that share the same file format but are not VM families).

---

### `Compa-SkuRatio` — the base-size reference table (DAX calculated table)

A DAX table computed at refresh time from `isf`. For each ISF group it identifies:
- `MinVMSize`: the ARM SKU name of the smallest VM in the group (ratio = 1)
- `MinRatio`: the ratio of that smallest VM (almost always 1.0)

It also appends a hard-coded row for `SQLMI_GP_Compute_Gen5` — because SQL Managed Instances are not included in the ISF file, but the model needs a record to make the `SQL_ManagedIstances → Compa-SkuRatio` relationship work.

---
### `RI transactions` — reservation purchase history (Azure Cost Management EA)

Connects to EA enrollment `${EA_BILLING_ACCOUNT_ID}` and loads the `ritransactions` table.

**What M does:**
- Deduplicates on `reservationOrderId` — one row per RI order
- Keeps only `eventType = "Purchase"` rows (excludes refunds and exchanges)
- Excludes two specific reservation orders by name (they appear to be duplicates or superseded purchases)

**Key columns:** `reservationOrderId`, `armSkuName` (the VM family the RI covers), `quantity` (number of RI units purchased), `region`, `term` (1 or 3 year), `eventDate`

---

### `RI usage summary` — daily utilisation per reservation (Azure Cost Management EA)

Same EA enrollment, different table key (`riusagesummary`).

**What M does:**
- Deduplicates on `reservationOrderId`
- Filters out rows where `skuName` contains `"fabric_capacity_cu_hour"` (Fabric capacity commitments that appear in the same endpoint but are not Azure RIs)
- Sorts by `avgUtilizationPercentage` ascending — so the worst-performing reservations appear first in the report

**Key columns:** `reservationOrderId` (links back to `RI transactions`), `usageDate`, `avgUtilizationPercentage`, `reservedHours`, `usedHours`

---

### `Regions` and `Sub-id` — lookup tables (embedded data)

Both tables are static data compiled directly into the model file — they are not refreshed from any external source. `Regions` maps Azure region codes to display names; `Sub-id` maps subscription GUIDs to readable subscription names.

---

## The central business logic — ISF normalisation

This is the core calculation that drives the RI recommendation.

**The problem:** Azure VMs come in many sizes within the same family. An RI does not just cover one specific size — it covers all sizes in the same family, proportionally. To know how many RIs you need, you need to convert every VM's size into a common unit: "how many smallest-VM-equivalents is this VM?"

**The calculation chain:**

```
Step 1 — Find the ISF group and ratio for each VM
  vmSize = "Standard_D8s_v3"
  → look up isf table by ArmSkuName
  → Ratio     = 4.0
  → Base Size = "DSv3 Type1"   (the ISF flexibility group)

Step 2 — Find the base VM of that group
  Base Size = "DSv3 Type1"
  → look up Compa-SkuRatio by InstanceSizeFlexibilityGroup
  → MinVMSize = "Standard_D2s_v3"   (smallest VM in the group)
  → MinRatio  = 1.0                  (its ratio)

Step 3 — Normalise
  RatioDivided = Ratio / MinRatio = 4.0 / 1.0 = 4.0
  → this D8s_v3 is equivalent to 4 Standard_D2s_v3 RI units

Step 4 — Aggregate demand
  Sum_RatioDivided = SUM of all RatioDivided across all VMs
  → total fleet demand expressed in D2s_v3-equivalent units
  → shown in the report as "Needed"

Step 5 — Subtract existing coverage
  ResToHave = Sum_RatioDivided − SUM(RI transactions[quantity])
  → how many Standard_D2s_v3-equivalent RIs still to purchase
  → shown in the report as "To Buy"
```

**Example:** 10 VMs of type D8s_v3 (ratio 4 each) = 40 normalised units. You already own 20 RIs → you need 20 more. The report shows `Needed = 40`, `To Buy = 20`.

The same logic applies to SQL Managed Instances, using vCore count instead of ISF ratios:  
`ResToHaveSQL = SUM(Cores) − SUM(RI transactions[quantity])`

---
## DAX measures — what gets displayed in visuals

| Measure | Shown as | Formula in plain English |
|---|---|---|
| `Sum_RatioDivided` | **Needed** | Total normalised VM demand across the fleet |
| `ResToHave` | **To Buy** | Needed minus existing RI quantity |
| `NeededSQL` | SQL cores | Total vCore count across all SQL MIs |
| `ResToHaveSQL` | SQL To Buy | SQL cores minus existing SQL RI quantity |
| `x_CommitmentDiscountUtilization` | Utilisation % | What fraction of purchased RI/SP capacity is actually being used |
| `Extimation 1Y` | 1Y estimate | `SUM(ConsumedQuantity) × 12` — annualised from current month |
| `Extimation 3Y` | 3Y estimate | `SUM(ConsumedQuantity) × 36` |
| `x_EffectiveSavingsRate` | Savings rate | Total savings divided by list cost (excludes principal purchase rows) |

---

## The key model relationship

The most important join in the model is:

```
SP - Azure VMs.ResID  ←→  VmQueryTOTAL.id
```

This is a **bidirectional** one-to-many relationship. It links the team's operational registry (SharePoint — who owns this VM, what is it for) with the live Azure inventory (Resource Graph — what's actually running, what size, what state).

Without this join, the report cannot connect cost and operational data to the correct VM. That is why `ResID` must be non-blank on the SharePoint side — blank values break the relationship.

---

## Report pages — what each one shows

| Page | Data used | Business question |
|---|---|---|
| **All VMs** | `VmQueryTOTAL` + `SP - Azure VMs` | What VMs are running, who owns them, in what size and region? |
| **VM_ReservedInstances** | `VmQueryTOTAL`, `RI transactions`, `RI usage summary` | Per VM family: how many RIs do we have, how well are they used, how many more do we need to buy? |
| **SQL MI_ReservedInstances** | `SQL_ManagedIstances`, `RI transactions`, `RI usage summary` | Same analysis scoped to SQL Managed Instances |
| **[DRAFT] VM_SavingPlans** | `Costs` (DirectQuery only) | What Savings Plans do we have, how are they used, what is the projected cost over 1 and 3 years? |
| **vm list query** *(hidden)* | `VmQueryTOTAL` | Raw VM list used internally by other pages — not visible to end users |

---

## What to verify during data validation

Given this data flow, the key checks are:

1. **`isf` coverage:** every `vmSize` in `VmQueryTOTAL` should have a match in the `isf` table. VMs with no match get `Ratio = blank`, `RatioDivided = 0`, and are silently excluded from `ResToHave`. Check with a DAX query: `COUNTROWS(FILTER(VmQueryTOTAL, ISBLANK([Ratio])))` — this should be 0.

2. **`ResID` join coverage:** every VM in `VmQueryTOTAL` that appears in the report should ideally have a matching entry in `SP - Azure VMs`. Rows in `VmQueryTOTAL` with no SharePoint match will show blank owner/role fields. Check with `Total Rows` (VmQueryTOTAL) vs `Total Rows SP` (SharePoint).

3. **`ResToHave` plausibility:** cross-check `Sum_RatioDivided` against the manual sum of (VM count × expected ratio) for a single VM family. For example: if you have 5 D4s_v3 VMs (ratio 2 each), `Sum_RatioDivided` filtered to that family should equal 10.

4. **`RI usage summary` deduplication:** after the `Table.Distinct` on `reservationOrderId`, the table has one row per order — not one per day. Verify that the `avgUtilizationPercentage` shown in the report is the percentage for that single deduplicated row, not an average of averages.

5. **Cost data freshness:** `Costs` is DirectQuery — its freshness depends on the last successful ADF run. Check ADF Monitor for `ingestion_ExecuteETL` last run status.

---

*Document generated 2026-06-23*
