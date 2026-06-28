# FinOps Hub — Reservations & Savings Plans Tracker

A Power BI solution for Azure cost management, built on the [Microsoft FinOps Toolkit](https://github.com/microsoft/finops-toolkit). Tracks Reserved Instance and Savings Plan coverage, utilisation, and purchasing recommendations across the Azure fleet.

---

## Architecture

```
Azure Cost Management Exports
  └─► ADLS Gen2 /msexports/
        └─► ADF (msexports_ExecuteETL) — EA→FOCUS schema conversion
              └─► ADLS Gen2 /ingestion/
                    └─► ADF (ingestion_ExecuteETL)
                          └─► Azure Data Explorer (Hub database)
                                └─► Power BI — Costs table (DirectQuery)

SharePoint / Resource Graph / Cost Management
  └─► Power BI Import refresh (all other tables)
```

Full documentation: [`docs/architecture/FinOps_Architecture_Documentation.md`](docs/architecture/FinOps_Architecture_Documentation.md)

---

## Repo Structure

```
finops-hub/
├── docs/
│   ├── architecture/
│   │   ├── FinOps_Architecture_Documentation.md   # Full technical reference
│   │   └── DATA_FLOW_EXPLAINED.md                 # Plain-English data flow walkthrough
│   └── dev/
│       ├── DEV_WORKSPACE_SETUP.md                 # Dev workspace changelog & credential setup
│       └── DEVOPS_WIKI.md                         # DevOps wiki (architecture summary)
├── semantic-model/
│   └── ReservationsTracker.SemanticModel/         # PBIP semantic model (TMDL)
│       └── definition/
│           ├── expressions.tmdl                   # Shared parameters (cluster URL, lookback)
│           ├── model.tmdl                         # Model definition
│           └── tables/                            # One .tmdl file per table
└── .env.example                                   # Required environment variables (template)
```

> **Note:** The semantic model TMDL files are version-controlled here. The Azure infrastructure (ADF pipelines, ADLS, ADX) is deployed and managed via Azure DevOps.

---

## Setup

### 1. Clone and configure

```bash
git clone https://github.com/<your-org>/finops-hub.git
cd finops-hub
cp .env.example .env
# Fill in .env with your actual resource names and IDs
```

### 2. Configure Power BI data source credentials

After publishing the PBIP project to a Power BI workspace, configure credentials manually via **Dataset settings → Data source credentials**:

| Source | Auth method | Privacy level |
|---|---|---|
| `isf` (aka.ms/isf) | Anonymous | Public |
| SharePoint | OAuth2 | Organizational |
| Azure Resource Graph | OAuth2 | Organizational |
| Azure Blob Storage (ADLS) | OAuth2 or Workspace Identity | Organizational |
| Azure Data Explorer | OAuth2 | Organizational |
| Azure Cost Management (EA) | OAuth2 / Organizational account | Organizational |

See [`docs/dev/DEV_WORKSPACE_SETUP.md`](docs/dev/DEV_WORKSPACE_SETUP.md) for full setup order and known issues.

### 3. Required Azure roles

| Resource | Role |
|---|---|
| ADLS Gen2 | Storage Blob Data Reader |
| Azure Data Explorer | Database Viewer |
| EA Billing Account | Billing Account Reader |
| EA Portal (ea.azure.com) | Benefits Reader |
| Azure Resource Graph | Reader on subscription |
| SharePoint list | Read |

---

## Report Pages

| Page | Description |
|---|---|
| **All VMs** | Full VM inventory: sizes, power state, owner, coverage |
| **VM_ReservedInstances** | RI detail per VM family: utilisation %, units to buy |
| **SQL MI_ReservedInstances** | Same analysis scoped to SQL Managed Instances |
| **[DRAFT] VM_SavingPlans** | Savings Plans overview: 1Y/3Y projections |
| **vm list query** *(hidden)* | Internal VM list used by other pages |

---

## Key Business Logic

The report answers **"how many Reserved Instances should we buy?"** using ISF (Instance Size Flexibility) normalisation:

```
vmSize → ISF ratio → normalised units (RatioDivided)
Sum_RatioDivided (fleet demand) − RI quantity owned = ResToHave ("To Buy")
```

Detailed explanation: [`docs/architecture/DATA_FLOW_EXPLAINED.md`](docs/architecture/DATA_FLOW_EXPLAINED.md)

---

## Known Issues & Deprecations

- **EA → MCA transition:** the `AzureCostManagement.Tables` EA connector will need to be reconfigured when the billing account migrates from EA to MCA.
- **EA connector deprecation:** Microsoft is deprecating the EA connector. Plan migration to Cost Management Export connector or FOCUS dataset.
- **Table name typos** (`SQL_ManagedIstances`, `Extimation`): do not rename without updating all DAX dependencies.

---

## References

- [Microsoft FinOps Toolkit](https://github.com/microsoft/finops-toolkit)
- [FOCUS Specification](https://focus.finops.org/)
- [Azure Instance Size Flexibility](https://aka.ms/isf)
- [Power BI PBIP format](https://learn.microsoft.com/power-bi/developer/projects/projects-overview)
