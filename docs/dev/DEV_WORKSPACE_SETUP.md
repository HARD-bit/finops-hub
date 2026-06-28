# Dev Workspace Setup — Operational Changelog

> This file tracks all changes applied to the dev Power BI workspace that cannot be committed to git (PBI Service credential configuration, manual Power Query edits applied before DevOps integration). It serves as the source of truth for replicating the dev environment and for eventual DevOps pipeline documentation.

---

## Code Changes (committed to git)

| Commit | File | Description |
|---|---|---|
| `53c13fe` | `SP - Azure VMs.tmdl` | Added `Table.SelectRows` to filter blank `ResID` rows — resolves Power BI error 104305 (column on "one" side of relationship cannot have null values). Root cause: SharePoint column `Creation` (mapped to `ResID`) is empty for VMs not yet linked to an Azure Resource ID. |
| Manual (2026-06-24) | `SP - Azure VMs` Power Query | Same fix applied manually to `Progetto Reservations and Saving Plans` .pbix (downloaded from prod AVM). Step name: `Filtered Blank RedID`. Not yet in a git commit — to align with PBIP when PBIP is promoted to main project. |
| `88d9005` | `SP - Azure VMs.tmdl` | Added `Text.Lower` normalization on ResID column to handle casing inconsistencies from SharePoint manual entries. |
| `b55e51d` | `SP - Azure VMs.tmdl` | Added `Text.Contains` filter to exclude RG-level ResIDs (missing `/providers/microsoft.compute/virtualmachines/` path). Root cause: 5 SharePoint entries added 24/06/2026 with ResID pointing to resource group instead of VM. |
| `67c0372` | `Costs.tmdl`, `pages/f1c8a4e7b3925d6e0418/page.json` | Added 4 SP DAX measures (SP Used Cost, SP Unused Cost, VM On-Demand Cost, SP Coverage %) and new hidden page VM_SavingPlans. |
| `90b2d78` | `SP_DataQuality.tmdl` | New table for SharePoint data quality monitoring: flags blank ResID, RG-level ResID, DeletionDate inconsistencies. No model relationships — additive only. |
| `8adf0be` | `SP_DataQuality.tmdl` | Added 5 DAX measures for general model validation: DQ SP Issues Count, DQ Stale VMs, DQ ARG VMs Not in SP, DQ RI Without Usage Summary, DQ Last Cost Date. |
| `0fcc483` | `pages/b3d2e1f4a5906c7d8e9f/page.json` | New hidden page SP_DataQuality for data quality monitoring. Visuals to be added in Power BI Desktop. |

---

## Manual Changes — Power BI Service (no git commit)

These must be applied manually in **every new workspace** (dev, staging, prod) via:  
`Dataset settings → Data source credentials → Edit credentials`

### 1. `isf` table — Web source (aka.ms/isf) ✅ Verified 2026-06-17

| Setting | Value |
|---|---|
| Source | `https://aka.ms/isf` |
| Authentication method | **Anonymous** |
| Privacy level | **Public** |

**Why:** Power BI Service defaults to OAuth for any URL on first load in a new workspace. `aka.ms/isf` is a public Microsoft CSV file (Instance Size Flexibility ratios) and requires no authentication. Setting Anonymous prevents the `DMTS_OAuthFailedToGetResourceIdError` (error code: `OAuthFailedToGetResourceIdError`) that blocks the entire `VmQueryTOTAL` dependency chain (`Base Size`, `Ratio`, `RatioDivided` columns).

**⚠️ Important:** table load must be enabled in Power BI Desktop before publishing — if `isf` was disabled during source isolation testing, re-enable it (`Transform data → right-click isf → Enable load`) and republish before configuring this credential.

---

### 2. `RI transactions` — Azure Cost Management EA

| Setting | Value |
|---|---|
| Source | AzureCostManagement — Enrollment Number `${EA_BILLING_ACCOUNT_ID}` |
| Authentication method | **OAuth2 / Organizational account** |
| Account | ${PBI_SERVICE_ACCOUNT} ([COMPANY] Azure AD) |
| Privacy level | **Organizational** |

**Required permissions (verified 2026-06-24):**
- **Billing Account Reader** — assigned in Azure Portal → Cost Management + Billing → Billing Account `${EA_BILLING_ACCOUNT_ID}` → Access control (IAM). This is the role that unlocks the PBI connector via OAuth2.
- **BENEFITS READER** — assigned on ea.azure.com by EA Admin. Gives visibility of reservations on both the primary tenant (infineum.com) and CBCP tenant (infineum.net).
- **LICENSE POSITION READER** — assigned on ea.azure.com. License position data — not directly used by the report but part of the EA role bundle.

**Why Reservations Reader (Azure RBAC) is NOT required:** the `AzureCostManagement.Tables` connector reads `/ritransactions` and `/riusagesummary` via the Cost Management billing API, not via the Azure Reservations API. Reservations Reader would only be needed for direct queries to the Azure Reservations API or portal access to Reservations → IAM.

**Note:** Recent Power BI Desktop versions no longer accept the EA API Key for this connector — OAuth2 / Organizational account is required.

---

### 3. `RI usage summary` — Azure Cost Management EA

Same credential as `RI transactions` — same enrollment, same connector. Configure once, applies to both tables.

**Known issue:** This endpoint (`/reservationsummaries`) returns HTTP 429 (Too Many Requests) if multiple refresh attempts are made in a short time window. Wait 5–10 minutes between retries. Unrelated to credentials.

**Note:** Reservations Reader (Azure RBAC) is NOT required for this connector. The `/reservationsummaries` endpoint is accessed via the billing API — Billing Account Reader is sufficient. Reservations Reader would only be needed for direct queries to the Azure Reservations API.

---

### 4. `SP - Azure VMs` — SharePoint Online ✅ Verified 2026-06-17

| Setting | Value |
|---|---|
| Source | `${SHAREPOINT_SITE_URL}` |
| Authentication method | **OAuth2** |
| Privacy level | **Organizational** |

Sign in with an account that has at least **Read** access to the `SP - Azure VMs` list on the EA SharePoint site.

---

### 5. `VmQueryUpdate` / `VmQueryTOTAL` / `SQL_ManagedIstances` — Azure Resource Graph ✅ Verified 2026-06-17

| Setting | Value |
|---|---|
| Source | Azure Resource Graph (`AzureResourceGraph.Query`) |
| Authentication method | **OAuth2** |
| Privacy level | **Organizational** |

Sign in with an account that has at least **Reader** on subscription `${AZURE_SUBSCRIPTION_ID}`.

---

### 6. `vmquery_cbcp_automation` — Azure Blob Storage (ADLS Gen2) ✅ Verified 2026-06-17

| Setting | Value |
|---|---|
| Source | `${ADLS_ACCOUNT_NAME}.blob.core.windows.net` |
| Authentication method | **Workspace Identity** (prod) / **OAuth2 o Account Key** (dev senza Premium) |
| Privacy level | **Organizational** |

File path: `/ingestion/vmquery_cbcp_automation.csv`  
Account name: `${ADLS_ACCOUNT_NAME}`

**Errore noto nel workspace dev — `Premium_ASWL_Error`:**  
La connessione usa Workspace Identity a livello tenant. Cambiare le credenziali in "Data source credentials" o in Power BI Desktop non è sufficiente — il dataset si agancia alla cloud connection registrata a livello tenant che forza Workspace Identity.

**Fix verificato (2026-06-17):**
```
Power BI Service → Settings (ingranaggio) → Manage connections and gateways → tab Cloud
→ trova connessione a ${ADLS_ACCOUNT_NAME}.blob.core.windows.net
→ ··· → Settings → cambia Authentication method (OAuth2 o Account Key)
→ Save → torna al dataset → Refresh now
```

**Fix definitivo (dopo upgrade Premium/Fabric):** abilitare Workspace Identity nel workspace dev — vedi Open Items.

**⚠️ Nota prod:** il workspace prod usa Workspace Identity — non modificare la cloud connection prod.

---

### 7. `Costs` — Azure Data Explorer (DirectQuery) ✅ Verified 2026-06-17

| Setting | Value |
|---|---|
| Source | `https://${ADX_CLUSTER_ENDPOINT}` |
| Authentication method | **OAuth2** |
| Privacy level | **Organizational** |

Sign in with an account that has at least **Viewer** on database `Hub` of the ADX cluster.  
Parameter in model: `Cluster URL (3)` — verify value matches cluster endpoint before refreshing.

---

## Recommended Setup Order for a New Workspace

```
1. Publish the .pbix or connect the PBIP project to the workspace
2. Dataset settings → Data source credentials:
   a. isf (aka.ms/isf)          → Anonymous / Public
   b. SharePoint                 → OAuth2 / Organizational
   c. Azure Resource Graph       → OAuth2 / Organizational
   d. Azure Blob Storage (ADLS)  → OAuth2 / Organizational
   e. Azure Data Explorer        → OAuth2 / Organizational
   f. AzureCostManagement (EA)   → Key / EA API Key
3. Trigger a full dataset refresh
4. If 429 on RI usage summary: wait 5–10 min and retry
5. If RI tables still fail: verify Billing Account Reader on Billing Account ${EA_BILLING_ACCOUNT_ID} (Azure Portal → Cost Management + Billing → IAM) and BENEFITS READER on ea.azure.com
```

---

## Future Implementations

### [DRAFT] VM_SavingPlans page

**Current state (2026-06-18):** page exists in the report, visible to users, prefixed `[DRAFT]`. Structurally complete — all 13 visuals are implemented and use only the `Costs` table (ADX DirectQuery). No broken visuals or missing measures identified.

**Current content:**
- Tabella commitment overview: `CommitmentDiscountName`, `CommitmentDiscountType`, count risorse, `x_CommitmentDiscountUtilization`, `EffectiveCost`, `x_CommitmentDiscountSavings` — filtrata su `CommitmentDiscountType = 'Savings Plan'`
- 4× card KPI: `EffectiveCost` per `CommitmentDiscountStatus`
- Tabella dettaglio risorse: `ResourceName`, `x_ResourceGroupName`, `x_ResourceType`, `x_SkuMeterCategory`, `ConsumedQuantity`, `ConsumedUnit`, `x_ConsumedCoreHours`
- Pivot table: `Extimation 1Y` / `Extimation 3Y` (misure DAX: `SUM(ConsumedQuantity) * 12/36`), `ContractedCost`, `EffectiveCost`, `x_CommitmentDiscountSavings` per risorsa

**Pending decisions:**
- [ ] Definire scope delle modifiche / integrazioni alla pagina
- [ ] Decidere se rinominare da `[DRAFT] VM_SavingPlans` a `VM_SavingPlans` e renderla ufficiale
- [ ] Verificare se le stime `Extimation 1Y` / `3Y` (basate su `ConsumedQuantity * 12/36`) riflettono la logica di business corretta o richiedono una formula più accurata

---

## Data Validation — Results (2026-06-24)

Queries run in DAX Query View on report `Progetto Reservations and Saving Plans` (published in dev workspace).

### Area 1 — Row count baseline

| Table | Rows |
|---|---|
| Costs | 503,769 |
| VmQueryTOTAL | 608 |
| VmQueryUpdate | 594 |
| vmquery_cbcp_automation | 14 |
| SP - Azure VMs | 792 |
| RI transactions | 64 |
| RI usage summary | 63 |
| SQL_ManagedIstances | 7 |
| isf | 2,501 |
| Compa-SkuRatio | 311 |
| Sub-id | 9 |
| Regions | 2 |

**Note:** `VmQueryTOTAL` = `VmQueryUpdate` (594) + `vmquery_cbcp_automation` (14) = 608 ✅ — union is correct.

### Area 2 — SP - Azure VMs

| Check | Value |
|---|---|
| SP VMs total | 792 |
| SP VMs Deleted=YES | 129 |
| SP VMs with blank ResID | 0 ✅ |

**ResID fix verified:** no rows with blank ResID — the `Table.SelectRows` filter applied in M Query works correctly.

**SharePoint vs ARG gap:** active VMs in SharePoint registry = 792 − 129 = **663**, while ARG returns **608** (gap of ~55). Possible causes: VMs in transition between states, CBCP VMs not covered by the ARG query scope, SharePoint sync latency. To monitor — investigate if the gap grows over time.

### Area 3 — RI without usage summary

1 `reservationOrderId` found in `RI transactions` but absent from `RI usage summary`:

```
${FABRIC_RESERVATION_ORDER_ID}
```

**Root cause identified (2026-06-25):** this is a **Fabric Capacity reservation**, not a VM/SQL reservation.

| Field | Value |
|---|---|
| armSkuName | `fabric_capacity_cu_hour` |
| reservationOrderName | `[FABRIC_RESERVATION_ORDER_NAME]` |
| eventDate | 2026-04-12 |
| quantity | 64 CU (Compute Units), West Europe, 1 year |
| amount | $6,112 USD, Recurring |
| subscription | [COMPANY] DnA Subscription |

The `/reservationsummaries` endpoint returns utilization data only for specific resource types (VMs, SQL, Cosmos DB, etc.). Fabric Capacity reservations are not included — no usage summary row is ever generated for them. This gap (64 RI transactions vs 63 RI usage summary) is **structural and permanent**: expected API behavior, not a data quality issue. No action required.

### Area 4 — ISF coverage (VM size normalization)

| Check | Value |
|---|---|
| Distinct MinVMSize in VmQueryTOTAL | 16 |
| Distinct MinVMSize in Compa-SkuRatio | 311 |
| VM sizes with no ISF match | 1 (blank MinVMSize) |

**Coverage normal:** `Compa-SkuRatio` (311 entries) contains all VM sizes from the ISF table — not just the 16 currently active. The 16 active sizes are a subset and are all covered.

**ISF coverage: 100% ✅** — follow-up investigation (2026-06-25) confirmed no real VMs with blank MinVMSize. The blank row returned by the original `VALUES()` query was a DAX artifact: when a column participates in a model relationship, `VALUES()` and `DISTINCT()` include a virtual blank row to represent unmatched values from the other side. Direct filter `FILTER('VmQueryTOTAL', ISBLANK([MinVMSize]))` returns empty — all 16 active VM sizes have a valid ISF ratio in `Compa-SkuRatio`.

### Area 5 — Costs freshness (ADX DirectQuery)

| Check | Value |
|---|---|
| First ChargePeriodStart | 2026-05-01 |
| Last ChargePeriodStart | 2026-06-23 |
| Last x_IngestionTime | 2026-06-23 15:44:55 |

**Data is fresh ✅** — ADF pipeline ran on 2026-06-23 at 15:44. The 1-day lag between last charge date (Jun 23) and today (Jun 24) is normal: Azure Cost Management publishes the previous day's data with ~24h delay.

**Data window: ~2 months** — controlled by the `Number of Months (2)` parameter in the ADX M query (`monthsago(lookback)`). Extend this parameter to increase historical coverage.

---

## Open Items / Pending Verification

- [x] **RI tables unblock — resolved 2026-06-24:** OAuth2 / Organizational account (${PBI_SERVICE_ACCOUNT}) now works. Roles required (assigned by [ADMIN]):
  1. **Billing Account Reader** — Azure Portal → Cost Management + Billing → Billing Account `${EA_BILLING_ACCOUNT_ID}` → Access control (IAM)
  2. **BENEFITS READER** + **LICENSE POSITION READER** — ea.azure.com (EA portal roles, cover both infineum.com and CBCP infineum.net tenants)
  Note: Reservations Reader (Azure RBAC) is NOT needed — the PBI connector reads via billing API, not the Reservations API.
- [x] **Error 104305 ResID — resolved 2026-06-24:** `SP - Azure VMs` M query updated to filter blank ResID rows (committed as `53c13fe`, applied manually to `Progetto Reservations and Saving Plans`).
- [x] **Report dev pubblicato e funzionante — 2026-06-24:** tutte le 7 sorgenti autenticate e in refresh, tutte le visual rendering senza errori. Progetto: `Progetto Reservations and Saving Plans`. Prossimo passo: convalida dati RI tables.
- [x] **Data validation — all 5 areas complete (2026-06-24):** row counts baseline, SP VMs, RI gap, ISF coverage, Costs freshness. Full results in *Data Validation* section above.
- [x] **RI order `${FABRIC_RESERVATION_ORDER_ID}...` — resolved (2026-06-25):** Fabric Capacity reservation (`fabric_capacity_cu_hour`, 64 CU, West Europe). The `/reservationsummaries` API does not return usage data for Fabric Capacity — gap is structural and permanent, no action required.
- [x] **MinVMSize blank — resolved (2026-06-25):** DAX artifact from `VALUES()`/`DISTINCT()` on a column in a relationship — no real VM is missing a size. ISF coverage is 100%.
- [x] **SharePoint vs ARG gap — root cause identified (2026-06-25):** all 55 missing VMs are **Citrix VDI machines** (Standard_D8s_v5, Windows) in subscription `${SUBSCRIPTION_ID_SECONDARY}`, decommissioned from Azure but never marked `Deleted=YES` in SharePoint. 5 of them have `DeletionDate` filled in but `Deleted=NO` — inconsistent data.
  - West Europe ([VM_RESOURCE_GROUP_REDACTED]): [VM_NAME_REDACTED]–M20 (20 VMs)
  - West Europe ([VM_RESOURCE_GROUP_REDACTED]): [VM_NAME_REDACTED][_REDACTED][_REDACTED][_REDACTED][_REDACTED][_REDACTED] (6 VMs)
  - North Europe ([VM_RESOURCE_GROUP_REDACTED]): [VM_NAME_REDACTED]–M20 (20 VMs)
  - North Europe ([VM_RESOURCE_GROUP_REDACTED]): [VM_NAME_REDACTED][_REDACTED][_REDACTED][_REDACTED][_REDACTED][_REDACTED][_REDACTED][_REDACTED] (8 VMs)
  - West Europe ([VM_RESOURCE_GROUP_REDACTED]): [VM_NAME_REDACTED] (1 VM)
  **Action required:** SharePoint list `SP - Azure VMs` → Edit in grid view → filter by VM name pattern → set `Deleted = YES` for all 55. Priority: fix the 5 with `DeletionDate` already set first (MC12 AZEU[_REDACTED]/MC07/MC12/MC15 AZNE).
- [x] **SP Coverage measures + VM_SavingPlans page — done (2026-06-25):** 4 new DAX measures added to `Costs.tmdl` (SP Used Cost, SP Unused Cost, VM On-Demand Cost, SP Coverage %). New hidden page `VM_SavingPlans` (id: `f1c8a4e7b3925d6e0418`) created in PBIP. Commit: `67c0372`.
- [x] **AVM git sync — done (2026-06-25):** AVM configured as git clone of Azure DevOps repo with auto-pull; no longer a manual file copy.
- [x] **ResID duplicate / invalid ResID — resolved (2026-06-26):** root cause: SharePoint had 5 entries with ResID pointing to a resource group path (`RG-AZEU-DNA-SPOKE-MANAGEMENT-AVD`) instead of a full VM path, entered with inconsistent casing. Two-step fix in Power Query (commits `88d9005`, `b55e51d`): (1) `Text.Lower` on ResID to normalize casing; (2) filter to only keep rows where ResID contains `/providers/microsoft.compute/virtualmachines/`. Permanent fix (clean up SharePoint entries) deferred to SharePoint governance session.

### Backlog (updated 2026-06-26)

- [x] **EA Renewal — response received (2026-06-26):** True-up already completed, renewal in negotiation. Expected transition: **EA → MCA (Microsoft Customer Agreement)**. Action: verify impact on RI/SP commitments and FinOps report connectors — see backlog item below.
- [ ] **EA → MCA transition — URGENT (cutover ~2026-07-01):** [ADMIN] confirmed: cutover tentatively July 1st (no official confirmation yet), no overlap period expected[_REDACTED]A Billing Account ID not yet available. Action required: reconfigure Power BI EA connector (`RI transactions`, `RI usage summary`) to MCA billing account before cutover. Awaiting MCA Billing Account ID from Enrico.
- [ ] **55 Citrix VMs stale** — mark `Deleted=YES` in SharePoint for 55 decommissioned Citrix VDI machines (full list in root cause finding above). Priority: 5 machines with `DeletionDate` already set (MC12 AZEU[_REDACTED]/MC07/MC12/MC15 AZNE). Note: re-run DAX stale VM check after next refresh to confirm current state (Citrix pool may have been rebuilt in June 2026).
- [x] **SP_DataQuality table + page — done (2026-06-26):** new Power Query table `SP_DataQuality` added to semantic model (commit `90b2d78`). Detects: blank ResID, RG-level ResID, DeletionDate set without Deleted=YES. Five DAX measures added (DQ SP Issues Count, DQ Stale VMs, DQ ARG VMs Not in SP, DQ RI Without Usage Summary, DQ Last Cost Date) — commit `8adf0be`. Hidden page `SP_DataQuality` created (id: `b3d2e1f4a5906c7d8e9f`, commit `0fcc483`). Visuals to be added in Power BI Desktop.
- [ ] **SP_DataQuality page — build visuals** — open PBIP in Power BI Desktop, add cards (DQ counts) and table visual (SP_DataQuality rows) on the hidden SP_DataQuality page.
- [ ] **SharePoint registry — governance & automation** — architecture defined (2026-06-26): three-layer approach (Power Automate ARG sync nightly + Power Apps form for manual input + Power BI monitoring). Open design questions: (1) which service account runs the flow; (2) grace period logic for VMs in review when flagging Deleted=YES. Design session to continue 2026-06-27.
- [ ] **SharePoint manual fix (pending approval):** correct 6 malformed entries ([VM_NAME_REDACTED] to 180, added 24/06/2026): 5 with RG-level ResID missing VM path, 1 with blank ResID. Also fix 4 entries with DeletionDate set but Deleted≠YES.
- [ ] **Savings Plans page — define objective** — clarify what the page must answer/communicate before building visuals.
- [ ] **New VM_SavingPlans page — build visuals** — once objective is defined, build visuals in Power BI Desktop using the new SP measures. Depends on objective definition above.
- [ ] **Service Principal ADF** `${ADF_SP_CLIENT_ID}` — regain control: find existing credentials OR reset secret in App Registrations and re-insert in all usage points. Known usage: ADF parameter `hubDataExplorer_servicePrincipalKey`. Other usage points to map before resetting.
- [ ] **PPU upgrade** workspace dev → Workspace Identity. Steps:
  1. `Workspace Settings → Premium tab → License mode: Premium Per User → Apply`
  2. `Workspace Settings → Advanced → Workspace Identity → Enable`
  3. `Azure Portal → Storage accounts → ${ADLS_ACCOUNT_NAME} → IAM → Add role assignment → Storage Blob Data Reader → Managed Identity → [dev workspace]`
  4. `Manage connections and gateways → ${ADLS_ACCOUNT_NAME} → Settings → Authentication: Workspace Identity`
  Note: PPU is sufficient, no dedicated Premium capacity needed. Fabric Free is NOT sufficient. Prod workspace already has Workspace Identity active.
- [ ] Confirm `vmquery_cbcp_automation.csv` file exists and is up to date in the dev storage account (or confirm dev workspace points to prod ADLS intentionally)
