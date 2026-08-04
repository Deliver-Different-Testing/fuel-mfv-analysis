# IMPLEMENTATION.md — OTG fuel logic + Configurator/Admin Manager changes

## 1. Claude Code Steps

```bash
cd /data/.openclaw/workspace/fuel-mfv-analysis
ls -la IMPLEMENTATION.md

# Source repos referenced by this implementation doc
ls -la /data/.openclaw/workspace/gitlab-source/adminmanager/API/Controllers/FuelController.cs
ls -la /data/.openclaw/workspace/gitlab-source/adminmanager/Core/Application/Services/FuelService.cs
ls -la /data/.openclaw/workspace/gitlab-source/adminmanager/Core/Application/Services/ClientSpeedService.cs
ls -la /data/.openclaw/workspace/gitlab-source/adminmanager/Core/Application/Services/VehicleSizeService.cs
ls -la /data/.openclaw/workspace/gitlab-source/adminmanager/Core/Application/Dtos/Fuels/FuelCreateRequest.cs
ls -la /data/.openclaw/workspace/gitlab-source/adminmanager/Core/Application/Dtos/Fuels/FuelUpdateRequest.cs
ls -la /data/.openclaw/workspace/gitlab-source/adminmanager/Core/Application/Dtos/Common/FuelDto.cs
ls -la /data/.openclaw/workspace/gitlab-source/adminmanager/Core/Application/Dtos/Common/ClientSpeedDto.cs
ls -la /data/.openclaw/workspace/gitlab-source/adminmanager/Core/Application/Dtos/Common/VehicleSizeDto.cs
ls -la /data/.openclaw/workspace/gitlab-source/adminmanager/Core/Domain/Despatch/TblFuelSurcharge.cs
ls -la /data/.openclaw/workspace/gitlab-source/adminmanager/Core/Domain/Despatch/VehicleSize.cs
ls -la /data/.openclaw/workspace/Kerran-Configurator/wwwroot/app/react/components/customers/CustomerDetailModal.tsx
ls -la /data/.openclaw/workspace/Kerran-Configurator/wwwroot/app/react/components/customers/AvailableServicesTab.tsx
ls -la /data/.openclaw/workspace/Kerran-Configurator/wwwroot/app/react/data/sampleCustomers.ts

# If Kerran wires the React UI in Kerran-Configurator
cd /data/.openclaw/workspace/Kerran-Configurator
npm run build || npx tsc --noEmit

# If Kerran wires AdminManager backend
cd /data/.openclaw/workspace/gitlab-source/adminmanager
# build command depends on local environment / solution entrypoint
```

## 2. Table of Contents

- [3. Feature Overview](#3-feature-overview)
- [4. Architecture: What's Built vs What Developer Needs to Do](#4-architecture-whats-built-vs-what-developer-needs-to-do)
- [5. Step-by-Step Checklist](#5-step-by-step-checklist)
- [6. Database Tables](#6-database-tables)
- [7. Key Questions Answered](#7-key-questions-answered)
- [8. API Endpoints Summary](#8-api-endpoints-summary)
- [9. Frontend Components](#9-frontend-components)
- [10. Testing Checklist](#10-testing-checklist)

## 3. Feature Overview

- Add support for **driver fuel being calculated from either client base or driver base**.
- Add support for **client-specific driver fuel percentage override**, with fallback to `VehicleSize.FuelPercentage` when blank.
- Keep the existing rule that **if client fuel is not charged, driver fuel is not paid**.
- Surface the configuration in two places:
  - **Admin Manager fuel editing window** for default/global fuel settings
  - **Configurator / Available Services** for client-level visibility and override maintenance
- Preserve current behaviour for existing clients unless an override is explicitly configured.

## 4. Architecture: What's Built vs What Developer Needs to Do

### ✅ Already built

| File | Status | Notes |
|---|---:|---|
| `gitlab-source/adminmanager/API/Controllers/FuelController.cs` | ✅ | Existing CRUD API for `tblFuelSurcharge` fuel rows |
| `gitlab-source/adminmanager/Core/Application/Services/FuelService.cs` | ✅ | Existing service persists `tblFuelSurcharge.Rate`, `ClientId`, `VehicleSizeId`, `PumpPrice` |
| `gitlab-source/adminmanager/Core/Application/Services/ClientSpeedService.cs` | ✅ | Existing CRUD for `tblClientAvailableSpeed`, including `FuelPercentage`, `Mfv`, `Faf` |
| `gitlab-source/adminmanager/Core/Application/Services/VehicleSizeService.cs` | ✅ | Existing CRUD for `VehicleSize.FuelPercentage` |
| `gitlab-source/adminmanager/Core/Domain/Despatch/TblFuelSurcharge.cs` | ✅ | Existing fuel/MFV table entity |
| `gitlab-source/adminmanager/Core/Domain/Despatch/VehicleSize.cs` | ✅ | Existing vehicle-level fallback fuel percentage |
| `Kerran-Configurator/wwwroot/app/react/components/customers/CustomerDetailModal.tsx` | ✅ | Existing mock UI includes **Available Services** tab |
| `Kerran-Configurator/wwwroot/app/react/components/customers/AvailableServicesTab.tsx` | ✅ | Existing mock row/expand UI for per-service settings |
| `Kerran-Configurator/wwwroot/app/react/data/sampleCustomers.ts` | ✅ | Existing mock service model carries `fuelPercentage`, `courierPercent`, etc. patterns |

### 🔧 What developer needs to do

| # | Priority | Est. hrs | Area | Work |
|---|---:|---:|---|---|
| 1 | P1 | 1.5h | DB | Add `DriverFuelPercentage` + `DriverFuelBasis` to `tblFuelSurcharge` |
| 2 | P1 | 1.5h | AdminManager backend | Extend Fuel DTOs/entity/service/controller payloads |
| 3 | P1 | 2h | Rating SQL / proc layer | Read new fields and apply driver fuel basis + fallback logic |
| 4 | P1 | 1.5h | Admin Manager UI | Extend existing fuel editing window with new fields |
| 5 | P1 | 2h | Configurator UI | Show new fields in Client Detail → Available Services |
| 6 | P1 | 2h | Configurator backend/API wiring | Persist client-level overrides to `tblFuelSurcharge` / existing client speed records |
| 7 | P2 | 1h | Audit/history | Record whether client is using default vs override |
| 8 | P1 | 2h | QA | Regression test current clients + OTG override path |

## 5. Step-by-Step Checklist

### Step 1 — Add new columns to `tblFuelSurcharge`

**File to edit:** DB migration in AdminManager / tenant DB migration path

**SQL to run:**

```sql
IF COL_LENGTH('dbo.tblFuelSurcharge', 'DriverFuelPercentage') IS NULL
BEGIN
    ALTER TABLE dbo.tblFuelSurcharge
    ADD DriverFuelPercentage decimal(18,4) NULL;
END;
GO

IF COL_LENGTH('dbo.tblFuelSurcharge', 'DriverFuelBasis') IS NULL
BEGIN
    ALTER TABLE dbo.tblFuelSurcharge
    ADD DriverFuelBasis varchar(20) NOT NULL
        CONSTRAINT DF_tblFuelSurcharge_DriverFuelBasis DEFAULT ('ClientBase');
END;
GO

IF NOT EXISTS (
    SELECT 1
    FROM sys.check_constraints
    WHERE name = 'CK_tblFuelSurcharge_DriverFuelBasis'
)
BEGIN
    ALTER TABLE dbo.tblFuelSurcharge
    ADD CONSTRAINT CK_tblFuelSurcharge_DriverFuelBasis
        CHECK (DriverFuelBasis IN ('ClientBase', 'DriverBase'));
END;
GO
```

**Why this table:**
- `tblFuelSurcharge` is the real MFV/fuel config table already used by `FuelService`.
- Adding the new fields here keeps the override with the existing fuel/MFV configuration instead of inventing a second unrelated table.

### Step 2 — Extend the EF entity for `tblFuelSurcharge`

**File to edit:** `gitlab-source/adminmanager/Core/Domain/Despatch/TblFuelSurcharge.cs`

Add:

```csharp
[Column(TypeName = "decimal(18, 4)")]
public decimal? DriverFuelPercentage { get; set; }

[StringLength(20)]
public string DriverFuelBasis { get; set; } = null!;
```

### Step 3 — Extend fuel DTOs and requests

**Files to edit:**
- `gitlab-source/adminmanager/Core/Application/Dtos/Common/FuelDto.cs`
- `gitlab-source/adminmanager/Core/Application/Dtos/Fuels/FuelCreateRequest.cs`
- `gitlab-source/adminmanager/Core/Application/Dtos/Fuels/FuelUpdateRequest.cs`
- `gitlab-source/adminmanager/Core/Application/Dtos/Fuels/FuelFullDto.cs`

#### `FuelDto.cs`

```csharp
public class FuelDto
{
    public int Id { get; set; }
    public DateTime Start { get; set; }
    public DateTime? End { get; set; }
    public decimal? Rate { get; set; }
    public int? ClientId { get; set; }
    public int? VehicleSizeId { get; set; }
    public bool Active { get; set; }
    public decimal? DriverFuelPercentage { get; set; }
    public string? DriverFuelBasis { get; set; }
}
```

#### `FuelCreateRequest.cs`

```csharp
public class FuelCreateRequest : BaseRequest
{
    public DateTime Start { get; set; }
    public DateTime? End { get; set; }
    public decimal Rate { get; set; }
    public int? ClientId { get; set; }
    public int? VehicleSizeId { get; set; }
    public bool Active { get; set; }
    public decimal? PumpPrice { get; set; }
    public decimal? DriverFuelPercentage { get; set; }
    public string DriverFuelBasis { get; set; } = "ClientBase";
    public string? UserName { get; set; }
}
```

`FuelUpdateRequest.cs` stays as:

```csharp
public class FuelUpdateRequest : FuelCreateRequest
{
    public int Id { get; set; }
}
```

#### `FuelFullDto.cs`

Keep inheritance from `FuelDto`; no extra work beyond ensuring `FuelDto` contains the new fields.

### Step 4 — Persist the new fuel fields in `FuelService`

**File to edit:** `gitlab-source/adminmanager/Core/Application/Services/FuelService.cs`

Update the `Search`, `Get`, `Create`, and `Update` projections/assignments.

#### In `Search(SearchRequest request)`
Add to the select:

```csharp
DriverFuelPercentage = c.DriverFuelPercentage,
DriverFuelBasis = c.DriverFuelBasis,
```

#### In `Get(IdRequest request)`
Add to the projection:

```csharp
DriverFuelPercentage = c.DriverFuelPercentage,
DriverFuelBasis = c.DriverFuelBasis,
```

#### In `Update(FuelUpdateRequest request)`
Add assignments:

```csharp
fuel.DriverFuelPercentage = request.DriverFuelPercentage;
fuel.DriverFuelBasis = string.IsNullOrWhiteSpace(request.DriverFuelBasis)
    ? "ClientBase"
    : request.DriverFuelBasis;
```

#### In `Create(FuelCreateRequest request)`
Add assignments:

```csharp
DriverFuelPercentage = request.DriverFuelPercentage,
DriverFuelBasis = string.IsNullOrWhiteSpace(request.DriverFuelBasis)
    ? "ClientBase"
    : request.DriverFuelBasis,
```

### Step 5 — Leave `VehicleSize.FuelPercentage` as the fallback source

**Files already in place:**
- `gitlab-source/adminmanager/Core/Domain/Despatch/VehicleSize.cs`
- `gitlab-source/adminmanager/Core/Application/Services/VehicleSizeService.cs`

**Decision:** reuse existing `VehicleSize.FuelPercentage` as the generic fallback. Do **not** create a second default percentage table.

**Runtime rule:**
- use `tblFuelSurcharge.DriverFuelPercentage` if populated for the active client fuel row
- otherwise use `VehicleSize.FuelPercentage`

### Step 6 — Keep client/service fuel visibility in `tblClientAvailableSpeed`

**Files already in place:**
- `gitlab-source/adminmanager/Core/Application/Dtos/Common/ClientSpeedDto.cs`
- `gitlab-source/adminmanager/Core/Application/Services/ClientSpeedService.cs`

`tblClientAvailableSpeed.FuelPercentage`, `Mfv`, and `Faf` already exist.

**Recommended use:**
- keep `tblClientAvailableSpeed` as the visible service-level settings row in Configurator
- use `tblFuelSurcharge` as the actual dated MFV / driver fuel override record

This avoids overloading `tblClientAvailableSpeed` with date-ranged fuel rules.

### Step 7 — Update the rating SQL / stored procedure logic

**Files to edit:** the SQL procedure(s) behind API rating, especially the proc Kerran referenced that currently does:

```sql
SELECT @DriverFuelPercentage = FuelPercentage
FROM VehicleSize
WHERE VehicleSizeID = @VehicleSizeID

UPDATE @AvailableRates SET
BaseChargeFuel = CASE WHEN UseBaseFuel = 1 THEN BaseChargeAmount * @MFV ELSE 0 END,
DriverBaseChargeFuel = CASE WHEN UseBaseFuel = 1 THEN BaseChargeAmount * @MFV * @DriverFuelPercentage ELSE 0 END,
ChargedDistanceFuel = CASE WHEN UseDistanceFuel = 1 THEN ChargedDistanceAmount * @MFV ELSE 0 END,
DriverChargedDistanceFuel = CASE WHEN UseDistanceFuel = 1 THEN ChargedDistanceAmount * @MFV * @DriverFuelPercentage ELSE 0 END
FROM @AvailableRates
```

#### Replace the hard-wired vehicle-only source with:

```sql
-- Resolve active client fuel row first
SELECT TOP 1
    @DriverFuelPercentage = ISNULL(fs.DriverFuelPercentage, vs.FuelPercentage),
    @DriverFuelBasis = ISNULL(fs.DriverFuelBasis, 'ClientBase')
FROM dbo.tblFuelSurcharge fs
LEFT JOIN dbo.VehicleSize vs
    ON vs.VehicleSizeID = @VehicleSizeID
WHERE fs.Active = 1
  AND (@ClientID IS NULL OR fs.ClientID = @ClientID)
  AND (fs.VehicleSizeID IS NULL OR fs.VehicleSizeID = @VehicleSizeID)
  AND fs.Start <= @DateTime
  AND (fs.[End] IS NULL OR fs.[End] >= @DateTime)
ORDER BY
    CASE WHEN fs.ClientID = @ClientID THEN 0 ELSE 1 END,
    CASE WHEN fs.VehicleSizeID = @VehicleSizeID THEN 0 ELSE 1 END,
    fs.Start DESC;

IF @DriverFuelPercentage IS NULL
BEGIN
    SELECT @DriverFuelPercentage = FuelPercentage
    FROM dbo.VehicleSize
    WHERE VehicleSizeID = @VehicleSizeID;
END;

SET @DriverFuelBasis = ISNULL(@DriverFuelBasis, 'ClientBase');
```

#### Then change the update block to:

```sql
UPDATE @AvailableRates
SET
    BaseChargeFuel = CASE
        WHEN UseBaseFuel = 1 THEN BaseChargeAmount * @MFV
        ELSE 0
    END,
    DriverBaseChargeFuel = CASE
        WHEN UseBaseFuel = 0 THEN 0
        WHEN @DriverFuelBasis = 'DriverBase' THEN DriverBaseChargeAmount * @DriverFuelPercentage
        ELSE BaseChargeAmount * @MFV * @DriverFuelPercentage
    END,
    ChargedDistanceFuel = CASE
        WHEN UseDistanceFuel = 1 THEN ChargedDistanceAmount * @MFV
        ELSE 0
    END,
    DriverChargedDistanceFuel = CASE
        WHEN UseDistanceFuel = 0 THEN 0
        WHEN @DriverFuelBasis = 'DriverBase' THEN DriverChargedDistanceAmount * @DriverFuelPercentage
        ELSE ChargedDistanceAmount * @MFV * @DriverFuelPercentage
    END
FROM @AvailableRates;
```

**Important:**
- `UseBaseFuel = 0` or `UseDistanceFuel = 0` must still zero out driver fuel.
- That preserves OTG’s requirement: **no client fuel charge = no driver fuel pay**.
- `DriverBaseChargeAmount` / `DriverChargedDistanceAmount` must refer to the real driver-base components already available in the proc. If the proc currently does not materialize those values, add them before this update block.

### Step 8 — Extend Admin Manager fuel editing window

**Real backend/API already exists:**
- `gitlab-source/adminmanager/API/Controllers/FuelController.cs`
- `gitlab-source/adminmanager/Core/Application/Services/FuelService.cs`

**UI work required:**
- add `DriverFuelPercentage` input
- add `DriverFuelBasis` dropdown (`ClientBase`, `DriverBase`)
- helper text: “Leave Driver Fuel % blank to use Vehicle Type fuel % fallback”

**Goal:** Admin Manager remains the place for default/global fuel setup.

### Step 9 — Extend Configurator Client Detail → Available Services

**Files to edit:**
- `Kerran-Configurator/wwwroot/app/react/components/customers/CustomerDetailModal.tsx`
- `Kerran-Configurator/wwwroot/app/react/components/customers/AvailableServicesTab.tsx`
- `Kerran-Configurator/wwwroot/app/react/data/sampleCustomers.ts`

#### Add these fields to the `AvailableService` type:

```ts
export interface AvailableService {
  id: string;
  displayName: string;
  service: string;
  active: boolean;
  salePrice: number;
  cost?: number;
  markup?: number;
  addonPercent?: number;
  courierPercent?: number;
  internetRebate?: number;
  gstInclusive?: boolean;
  minCharge?: number;
  maxCharge?: number;
  activeDays?: ('Mon' | 'Tue' | 'Wed' | 'Thu' | 'Fri' | 'Sat' | 'Sun')[];
  attachedCharges?: AttachedCharge[];
  nzExtras?: NzServiceExtras;

  fuelEnabled?: boolean;
  clientFuelPercentage?: number | null;
  driverFuelBasis?: 'ClientBase' | 'DriverBase';
  driverFuelPercentage?: number | null;
  driverFuelUsesVehicleDefault?: boolean;
}
```

#### Add visible controls inside `AvailableServicesTab.tsx`

Inside the expanded detail panel for each service, add:
- a read/write client fuel toggle / visibility indicator
- a driver fuel basis dropdown
- a driver fuel percentage input
- a read-only hint showing fallback to `VehicleSize.FuelPercentage` when blank

**This is the client-facing configuration surface** Steve referred to in the screenshot.

### Step 10 — Add real API wiring for Configurator

Right now `Kerran-Configurator` uses mock data from `sampleCustomers.ts`.

**Developer needs to do:**
- replace mock `availableServices` persistence with real API calls
- load service settings from `ClientSpeedController.Get(clientId)`
- load dated fuel override rows from `FuelController.Search/Get` filtered by client
- save service settings to `ClientSpeedController.Update/Create`
- save client-specific driver fuel override rows to `FuelController.Update/Create`

**Suggested API helper methods in `Kerran-Configurator`:**

```ts
const API_BASE = '/api';

export async function fetchClientSpeeds(clientId: number) {
  const res = await fetch(`${API_BASE}/ClientSpeed/${clientId}`);
  if (!res.ok) throw new Error(`Failed to load client speeds: ${res.status}`);
  return res.json();
}

export async function fetchFuelRule(id: number) {
  const res = await fetch(`${API_BASE}/Fuel/${id}`);
  if (!res.ok) throw new Error(`Failed to load fuel rule: ${res.status}`);
  return res.json();
}

export async function updateFuelRule(id: number, body: unknown) {
  const res = await fetch(`${API_BASE}/Fuel/${id}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });
  if (!res.ok) throw new Error(`Failed to update fuel rule: ${res.status}`);
  return res.json();
}
```

### Step 11 — History / audit note

Existing `ClientSpeedService.ClientSpeedHistory(...)` already logs available speed changes to `tblHistory`.

**Recommended extension:**
- when a client-level fuel override is created/changed, also write a `tblHistory` row documenting:
  - client id
  - old `DriverFuelPercentage`
  - new `DriverFuelPercentage`
  - old `DriverFuelBasis`
  - new `DriverFuelBasis`

## 6. Database Tables

### `dbo.tblFuelSurcharge`

**Existing columns already in use:**
- `FuelSurchargeID` int PK
- `ClientID` int null
- `VehicleSizeID` int null
- `Start` datetime not null
- `End` datetime null
- `Rate` decimal(18,4) not null
- `PumpPrice` decimal(18,4) null
- `Active` bit not null
- `Created` datetime not null
- `CreatedBy` varchar(50) not null
- `LastModified` datetime not null
- `LastModifiedBy` varchar(50) not null

**New columns to add:**
- `DriverFuelPercentage` decimal(18,4) null
- `DriverFuelBasis` varchar(20) not null default `'ClientBase'`

### `dbo.VehicleSize`

**Existing relevant column:**
- `FuelPercentage` decimal(...) not null

**Use:** generic fallback when client-specific driver fuel % is blank.

### `dbo.tblClientAvailableSpeed`

**Existing relevant columns:**
- `Id` int PK
- `ClientId` int not null
- `SpeedId` int not null
- `DisplayName` nvarchar/... null
- `Active` bit not null
- `SalePrice` decimal null
- `Markup` decimal null
- `AddonPercentage` decimal null
- `CourierPercentage` decimal null
- `Mfv` bit null
- `Faf` bit null
- `FuelPercentage` decimal null

**Use:** service-level visibility / existing available-services settings.

## 7. Key Questions Answered

### Can we reuse existing component X or is new code provided?

- **Reuse existing Admin Manager fuel backend:** yes.
  - `FuelController`, `FuelService`, `TblFuelSurcharge` are the correct existing base.
- **Reuse existing vehicle fallback:** yes.
  - `VehicleSize.FuelPercentage` is already the right default/fallback source.
- **Reuse existing Configurator Available Services UI:** yes.
  - `CustomerDetailModal.tsx` + `AvailableServicesTab.tsx` are the right surfaces.
- **Do we need a brand new fuel table?** no.
  - Extend `tblFuelSurcharge` rather than creating a duplicate config store.

### What tables does this read from / write to?

**Reads:**
- `dbo.tblFuelSurcharge`
- `dbo.VehicleSize`
- `dbo.tblClientAvailableSpeed`
- likely `dbo.TucJobType` via existing client speed joins

**Writes:**
- `dbo.tblFuelSurcharge`
- `dbo.tblClientAvailableSpeed`
- optionally `dbo.tblHistory` for audit trail

### Are there triggers / side effects to watch for?

- The existing rating SQL already uses fuel flags like `UseBaseFuel` / `UseDistanceFuel`; do not break that gate.
- Do not silently change current clients: default all new rows to `DriverFuelBasis = 'ClientBase'`.
- `VehicleSizeService.Create` currently defaults `FuelPercentage` to `1` if missing. Be careful not to treat null/blank override as `1`; blank override must mean “fallback to vehicle value”, not “force 1”.
- `ClientSpeedService` currently treats `CourierPercentage == 0` as null. Do not copy that pattern blindly for `DriverFuelPercentage`; OTG may legitimately want explicit `0` in some scenarios.

## 8. API Endpoints Summary

| Method | Route | Purpose | Tables Read | Tables Written |
|---|---|---|---|---|
| POST | `/API/Fuel/Search` | Search fuel/MFV rows | `tblFuelSurcharge` | - |
| GET | `/API/Fuel/{id}` | Get fuel/MFV row | `tblFuelSurcharge` | - |
| POST | `/API/Fuel` | Create fuel/MFV row | - | `tblFuelSurcharge` |
| POST | `/API/Fuel/{id}` | Update fuel/MFV row | `tblFuelSurcharge` | `tblFuelSurcharge` |
| DELETE | `/API/Fuel/{id}` | Delete fuel/MFV row | `tblFuelSurcharge` | `tblFuelSurcharge` |
| GET | `/API/ClientSpeed/{id}` | Get client available speeds | `tblClientAvailableSpeed`, `TucJobType` | - |
| POST | `/API/ClientSpeed` | Create client service row | - | `tblClientAvailableSpeed` |
| POST | `/API/ClientSpeed/{id}` | Update client service row | `tblClientAvailableSpeed` | `tblClientAvailableSpeed` |
| GET | `/api/VehicleSize` | Get vehicle sizes for fallback display | `VehicleSize` | - |
| POST | `/api/VehicleSize/{id}` | Update vehicle fallback fuel % | `VehicleSize` | `VehicleSize` |

## 9. Frontend Components

### Configurator file tree

```text
Kerran-Configurator/
└── wwwroot/app/react/
    ├── components/customers/
    │   ├── CustomerDetailModal.tsx
    │   └── AvailableServicesTab.tsx
    └── data/
        └── sampleCustomers.ts
```

### Component hierarchy

- `CustomerDetailModal.tsx`
  - owns tab strip
  - renders `AvailableServicesTab` when active tab is `services`
- `AvailableServicesTab.tsx`
  - table of per-service rows
  - expandable detail panel per service
  - correct place to add OTG/client fuel controls
- `sampleCustomers.ts`
  - current mock shape
  - must be replaced or mirrored by real API DTO mapping

### Key behaviours to add

- Display whether service/client is using:
  - default vehicle fuel % fallback
  - explicit client override
- Allow editing:
  - driver fuel basis
  - driver fuel percentage override
- Show helper copy when blank = fallback to vehicle fuel %
- Preserve save/discard behaviour already present in `CustomerDetailModal.tsx`

## 10. Testing Checklist

### Staging / dev first

1. Create a default fuel row in Admin Manager with:
   - `Rate` populated
   - `DriverFuelPercentage = NULL`
   - `DriverFuelBasis = 'ClientBase'`
2. Confirm the runtime still uses `VehicleSize.FuelPercentage` fallback.
3. Create OTG client-specific fuel row with:
   - `DriverFuelPercentage = 0.0500`
   - `DriverFuelBasis = 'DriverBase'`
4. Quote a job for OTG where fuel applies.
5. Confirm driver fuel is calculated from driver base, not client base.
6. Quote a job for OTG where client fuel is off for that component.
7. Confirm driver fuel is `0`.
8. Edit OTG in Configurator Available Services.
9. Confirm UI clearly shows override vs fallback.
10. Update the default vehicle fuel % and confirm clients without override pick up the new fallback.

### Regression checks

11. Existing non-OTG clients continue to behave as before.
12. Existing Admin Manager fuel edit screen still loads old rows with no errors.
13. Existing Vehicle Size maintenance still works.
14. Existing Client Speed maintenance still works.
15. Existing history / audit still records available speed changes.

### Production verification

16. Confirm there is no current tenant using an implicit meaning for blank override other than fallback.
17. Confirm OTG-specific row has the intended `Start` / `End` date range.
18. Confirm no duplicated active `tblFuelSurcharge` rows for the same client/vehicle/date combination are causing ambiguous matches.
19. Confirm driver earnings / settlement downstream still consume the stored courier fuel correctly after rating change.
20. Confirm at least one live OTG quote matches expected manual calculation.
