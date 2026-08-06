# IMPLEMENTATION.md — OTG fuel surcharge profiles in rating

## 1. Summary

This spec replaces the earlier narrower OTG-only driver-fuel override approach.

The client feedback requires Fuel Surcharge to behave as a **named, reusable FSC profile inside rating**, with:

- one **client FSC %**
- one **courier/driver FSC %**
- component inclusion rules for each side
- one rolled-up visible `FSC` total on the client side
- one rolled-up visible `FSC` total on the courier side
- the rule that **courier FSC is not paid when client FSC is not charged**

## 2. Architecture decision

### UI perspective — two separate touchpoints are required

The client request needs two distinct UI layers:

1. **Rate setup / maintenance**
   - this is where FSC profiles are attached to the underlying rate structure
   - this is the default / structural assignment point

2. **Client profile → Available Services in Configurator**
   - this is where OTG can see which FSC profile a client/service is using
   - this is the client-level visibility and override point

That means the missing requirement is not only profile creation, but also being able to assign or override the selected FSC profile in both of those operational surfaces.

### Keep `tblFuelSurcharge` as the profile table

Do **not** build a second fuel-profile table.

Instead, treat each `dbo.tblFuelSurcharge` row as one named FSC configuration/profile, for example:

- `House FSC`
- `20/15% FSC`
- `20/20% FSC`
- `Medical FSC`
- `Distribution FSC`

### Assignment happens in rating

To make profiles reusable and selected inside the rating area, assign the profile from `dbo.tblClientAvailableSpeed`.

That means:

- `tblFuelSurcharge` = the profile definition
- `tblClientAvailableSpeed` = which profile a client/service uses

This is the missing piece that makes the profile reusable instead of hard-wiring it to one client row.

### Reuse existing component-level fuel flags

The current model already has component-level fuel flags in the rating structures, especially:

- `DistanceRate.ApplyBaseChargeFuel`
- `DistanceRate.ApplyDistanceFuel`
- `ExtraCharge.ApplyWeightFuel`
- `ExtraCharge.ApplyWaitTimeFuel`
- `ExtraCharge.ApplyExtraStopFuel`
- `ExtraCharge.ApplyAfterHoursFuel`
- `ExtraCharge.ApplyHolidayFuel`
- `ExtraCharge.ApplyPalletsFuel`
- `ExtraCharge.ApplyDryIceFuel`
- `ExtraCharge.ApplyDangerousGoodsFuel`
- `ExtraCharge.ApplyCubicFuel`
- `ExtraCharge.ApplyItemFuel`

The gap is that these currently do **not** split between:

- client/company FSC basis
- courier/driver FSC basis

So the implementation should extend the existing flags with courier-side companions rather than invent a separate component-selection subsystem.

## 3. Real files / code areas involved

### AdminManager backend

- `gitlab-source/adminmanager/API/Controllers/FuelController.cs`
- `gitlab-source/adminmanager/Core/Application/Services/FuelService.cs`
- `gitlab-source/adminmanager/Core/Application/Services/ClientSpeedService.cs`
- `gitlab-source/adminmanager/Core/Application/Services/RateService.cs`
- `gitlab-source/adminmanager/Core/Application/Dtos/Fuels/FuelCreateRequest.cs`
- `gitlab-source/adminmanager/Core/Application/Dtos/Fuels/FuelUpdateRequest.cs`
- `gitlab-source/adminmanager/Core/Application/Dtos/Common/FuelDto.cs`
- `gitlab-source/adminmanager/Core/Application/Dtos/Common/ClientSpeedDto.cs`
- `gitlab-source/adminmanager/Core/Domain/Despatch/TblFuelSurcharge.cs`
- `gitlab-source/adminmanager/Core/Domain/Despatch/TblClientAvailableSpeed.cs`
- `gitlab-source/adminmanager/Core/Domain/Despatch/DistanceRate.cs`
- `gitlab-source/adminmanager/Core/Domain/Despatch/ExtraCharge.cs`

### Configurator / UI

- `Kerran-Configurator/wwwroot/app/react/components/customers/CustomerDetailModal.tsx`
- `Kerran-Configurator/wwwroot/app/react/components/customers/AvailableServicesTab.tsx`
- `Kerran-Configurator/wwwroot/app/react/data/sampleCustomers.ts`

### Rating / proc layer

- the stored procedure / SQL path Kerran already identified which currently calculates:
  - client base fuel
  - client distance fuel
  - driver base fuel
  - driver distance fuel

That proc must be updated to resolve the selected FSC profile and roll the final result to one FSC total per side.

## 4. Database changes

## 4.1 `dbo.tblFuelSurcharge`

### Existing purpose
Currently this is the dated fuel/MFV configuration table.

### New purpose
Each row becomes a **named FSC profile** that contains both sides of the FSC setup.

### Columns to add

```sql
IF COL_LENGTH('dbo.tblFuelSurcharge', 'Name') IS NULL
BEGIN
    ALTER TABLE dbo.tblFuelSurcharge
    ADD Name varchar(100) NULL;
END;
GO

IF COL_LENGTH('dbo.tblFuelSurcharge', 'DriverFuelPercentage') IS NULL
BEGIN
    ALTER TABLE dbo.tblFuelSurcharge
    ADD DriverFuelPercentage decimal(18,4) NULL;
END;
GO

IF COL_LENGTH('dbo.tblFuelSurcharge', 'IsDefaultHouseProfile') IS NULL
BEGIN
    ALTER TABLE dbo.tblFuelSurcharge
    ADD IsDefaultHouseProfile bit NOT NULL
        CONSTRAINT DF_tblFuelSurcharge_IsDefaultHouseProfile DEFAULT (0);
END;
GO
```

### Column usage

- `Rate` = **client FSC percentage**
- `DriverFuelPercentage` = **courier FSC percentage**
- `Name` = internal admin/profile name only
- `IsDefaultHouseProfile` = the default House FSC profile

### Important decision
Keep the existing `Rate` column as the client-side FSC percentage to minimise churn. Do **not** rename it in SQL for this phase.

### Legacy compatibility
Keep `ClientID` and `VehicleSizeID` on the table for backward compatibility and dated/profile filtering, but **new reusable assignment should not rely on `ClientID` alone**.

## 4.2 `dbo.tblClientAvailableSpeed`

Add the selected FSC profile into the rating configuration row.

```sql
IF COL_LENGTH('dbo.tblClientAvailableSpeed', 'FuelSurchargeID') IS NULL
BEGIN
    ALTER TABLE dbo.tblClientAvailableSpeed
    ADD FuelSurchargeID int NULL;
END;
GO

IF NOT EXISTS (
    SELECT 1
    FROM sys.foreign_keys
    WHERE name = 'FK_tblClientAvailableSpeed_tblFuelSurcharge'
)
BEGIN
    ALTER TABLE dbo.tblClientAvailableSpeed
    ADD CONSTRAINT FK_tblClientAvailableSpeed_tblFuelSurcharge
        FOREIGN KEY (FuelSurchargeID)
        REFERENCES dbo.tblFuelSurcharge(FuelSurchargeID);
END;
GO
```

### Why this column is needed
Without this FK, a named row in `tblFuelSurcharge` is still just a row. With this FK, it becomes a selectable profile in the rating area.

### Runtime rule

- if `tblClientAvailableSpeed.FuelSurchargeID` is populated, use that profile
- otherwise fall back to the active `IsDefaultHouseProfile = 1` row

## 4.3 `dbo.DistanceRate`

Current client-side flags already exist:

- `ApplyBaseChargeFuel`
- `ApplyDistanceFuel`

Add courier-side companions:

```sql
IF COL_LENGTH('dbo.DistanceRate', 'ApplyBaseChargeCourierFuel') IS NULL
BEGIN
    ALTER TABLE dbo.DistanceRate
    ADD ApplyBaseChargeCourierFuel bit NULL;
END;
GO

IF COL_LENGTH('dbo.DistanceRate', 'ApplyDistanceCourierFuel') IS NULL
BEGIN
    ALTER TABLE dbo.DistanceRate
    ADD ApplyDistanceCourierFuel bit NULL;
END;
GO
```

### Backward-compatibility default
For migrated rows, initialise new courier-side flags from the existing client-side flags so existing behaviour does not silently disappear.

## 4.4 `dbo.ExtraCharge`

Current client-side flags already exist:

- `ApplyWeightFuel`
- `ApplyWaitTimeFuel`
- `ApplyExtraStopFuel`
- `ApplyAfterHoursFuel`
- `ApplyHolidayFuel`
- `ApplyPalletsFuel`
- `ApplyDryIceFuel`
- `ApplyDangerousGoodsFuel`
- `ApplyCubicFuel`
- `ApplyItemFuel`

Add courier-side companions:

```sql
IF COL_LENGTH('dbo.ExtraCharge', 'ApplyWeightCourierFuel') IS NULL
    ALTER TABLE dbo.ExtraCharge ADD ApplyWeightCourierFuel bit NULL;
GO
IF COL_LENGTH('dbo.ExtraCharge', 'ApplyWaitTimeCourierFuel') IS NULL
    ALTER TABLE dbo.ExtraCharge ADD ApplyWaitTimeCourierFuel bit NULL;
GO
IF COL_LENGTH('dbo.ExtraCharge', 'ApplyExtraStopCourierFuel') IS NULL
    ALTER TABLE dbo.ExtraCharge ADD ApplyExtraStopCourierFuel bit NULL;
GO
IF COL_LENGTH('dbo.ExtraCharge', 'ApplyAfterHoursCourierFuel') IS NULL
    ALTER TABLE dbo.ExtraCharge ADD ApplyAfterHoursCourierFuel bit NULL;
GO
IF COL_LENGTH('dbo.ExtraCharge', 'ApplyHolidayCourierFuel') IS NULL
    ALTER TABLE dbo.ExtraCharge ADD ApplyHolidayCourierFuel bit NULL;
GO
IF COL_LENGTH('dbo.ExtraCharge', 'ApplyPalletsCourierFuel') IS NULL
    ALTER TABLE dbo.ExtraCharge ADD ApplyPalletsCourierFuel bit NULL;
GO
IF COL_LENGTH('dbo.ExtraCharge', 'ApplyDryIceCourierFuel') IS NULL
    ALTER TABLE dbo.ExtraCharge ADD ApplyDryIceCourierFuel bit NULL;
GO
IF COL_LENGTH('dbo.ExtraCharge', 'ApplyDangerousGoodsCourierFuel') IS NULL
    ALTER TABLE dbo.ExtraCharge ADD ApplyDangerousGoodsCourierFuel bit NULL;
GO
IF COL_LENGTH('dbo.ExtraCharge', 'ApplyCubicCourierFuel') IS NULL
    ALTER TABLE dbo.ExtraCharge ADD ApplyCubicCourierFuel bit NULL;
GO
IF COL_LENGTH('dbo.ExtraCharge', 'ApplyItemCourierFuel') IS NULL
    ALTER TABLE dbo.ExtraCharge ADD ApplyItemCourierFuel bit NULL;
GO
```

### Backward-compatibility default
Set each new courier-side flag to the existing client-side value during migration, then let OTG change them where commercial rules differ.

## 5. EF / DTO changes

## 5.1 `TblFuelSurcharge.cs`

Add:

```csharp
[StringLength(100)]
public string? Name { get; set; }

[Column(TypeName = "decimal(18, 4)")]
public decimal? DriverFuelPercentage { get; set; }

public bool IsDefaultHouseProfile { get; set; }
```

## 5.2 `TblClientAvailableSpeed.cs`

Add:

```csharp
[Column("FuelSurchargeID")]
public int? FuelSurchargeId { get; set; }

[ForeignKey("FuelSurchargeId")]
public virtual TblFuelSurcharge? FuelSurcharge { get; set; }
```

## 5.3 `DistanceRate.cs`

Add:

```csharp
public bool? ApplyBaseChargeCourierFuel { get; set; }
public bool? ApplyDistanceCourierFuel { get; set; }
```

## 5.4 `ExtraCharge.cs`

Add courier-side companion bools matching the SQL above.

## 5.5 Fuel DTOs

Update:

- `FuelDto.cs`
- `FuelCreateRequest.cs`
- `FuelUpdateRequest.cs`
- `FuelFullDto.cs`

Required additions:

```csharp
public string? Name { get; set; }
public decimal? DriverFuelPercentage { get; set; }
public bool IsDefaultHouseProfile { get; set; }
```

## 5.6 Client speed DTOs

Update `ClientSpeedDto.cs` and request models to carry:

```csharp
public int? FuelSurchargeId { get; set; }
```

## 6. Service / API changes

## 6.1 `FuelService.cs`

Update `Search`, `Get`, `Create`, and `Update` so `tblFuelSurcharge` behaves like a profile record, not just a dated percentage row.

Must persist and return:

- `Name`
- `Rate`
- `DriverFuelPercentage`
- `IsDefaultHouseProfile`
- `ClientId`
- `VehicleSizeId`
- `Start`
- `End`
- `Active`
- `PumpPrice`

### Validation rules

- `Name` required for new OTG-style profiles
- only one active `IsDefaultHouseProfile = 1` row per tenant/date window
- allow `DriverFuelPercentage = 0`
- do **not** convert explicit `0` to null

## 6.2 `ClientSpeedService.cs`

Extend create/update/search/get paths to read and write `FuelSurchargeId`.

This is what makes the FSC profile visible and editable inside rating.

## 6.3 `RateService.cs`

If rate maintenance APIs expose `DistanceRate` / `ExtraCharge` flags, extend DTOs and request models for the new courier-side fields.

Do not create a separate OTG-only API path. Reuse the existing rate maintenance pipeline.

## 7. Rating SQL / stored procedure changes

## 7.1 Resolve the selected FSC profile

The proc should stop assuming a single implicit fuel percentage path.

At runtime it should resolve:

1. the `tblClientAvailableSpeed` row for the job/client/service
2. `FuelSurchargeID` from that row
3. the matching `tblFuelSurcharge` row
4. if no profile is assigned, fall back to the active `IsDefaultHouseProfile = 1` row

Example pattern:

```sql
SELECT TOP 1
    @FuelSurchargeId = cas.FuelSurchargeID
FROM dbo.tblClientAvailableSpeed cas
WHERE cas.ClientID = @ClientID
  AND cas.SpeedID = @SpeedID
  AND cas.Active = 1;

SELECT TOP 1
    @ClientFuelPercentage = fs.Rate,
    @CourierFuelPercentage = fs.DriverFuelPercentage,
    @FuelProfileName = fs.Name
FROM dbo.tblFuelSurcharge fs
WHERE fs.FuelSurchargeID = @FuelSurchargeId
  AND fs.Active = 1
  AND fs.Start <= @DateTime
  AND (fs.[End] IS NULL OR fs.[End] >= @DateTime);

IF @FuelSurchargeId IS NULL
BEGIN
    SELECT TOP 1
        @ClientFuelPercentage = fs.Rate,
        @CourierFuelPercentage = fs.DriverFuelPercentage,
        @FuelProfileName = fs.Name
    FROM dbo.tblFuelSurcharge fs
    WHERE fs.Active = 1
      AND fs.IsDefaultHouseProfile = 1
      AND fs.Start <= @DateTime
      AND (fs.[End] IS NULL OR fs.[End] >= @DateTime)
    ORDER BY fs.Start DESC;
END;
```

## 7.2 Calculate each side independently

### Client FSC basis
Calculate from the selected client-side component flags:

- base charge if `DistanceRate.ApplyBaseChargeFuel = 1`
- distance charge if `DistanceRate.ApplyDistanceFuel = 1`
- extra charge components where the matching `ExtraCharge.Apply*Fuel = 1`

### Courier FSC basis
Calculate from the selected courier-side component flags:

- courier base pay if `DistanceRate.ApplyBaseChargeCourierFuel = 1`
- courier distance pay if `DistanceRate.ApplyDistanceCourierFuel = 1`
- courier extra-pay components where the matching `ExtraCharge.Apply*CourierFuel = 1`

### Commercial gate
If client FSC basis total is zero, courier FSC must be zero even if courier-side flags are on.

```sql
IF ISNULL(@ClientFscBasisTotal, 0) = 0
BEGIN
    SET @CourierFscTotal = 0;
END;
```

## 7.3 Output one FSC line only per side

Do **not** emit separate FSC records for:

- base
- distance
- weight
- wait time
- extra stop
- after-hours
- holiday
- etc.

Instead:

- sum all selected client-side components first
- apply client FSC % once
- sum all selected courier-side components first
- apply courier FSC % once

Visible outputs:

- client: `FSC` / `Fuel Surcharge`
- courier: `FSC` / `Fuel Surcharge`

The profile name is admin-only and must not appear on job/invoice/settlement output.

## 8. UI changes

## 8.1 Admin Manager fuel screen

Extend the existing fuel maintenance screen to support profile maintenance.

Add fields:

- `Name`
- `Client FSC %` (existing `Rate` field relabelled in UI)
- `Courier FSC %`
- `Default House Profile`
- `Start`
- `End`
- `Active`

Actions:

- create profile
- copy profile
- activate/deactivate profile
- edit percentages quickly

## 8.2 Rate setup / maintenance UI

This is the first operational assignment point.

Where rate structures are maintained, the UI needs to support both:

- attaching the applicable FSC profile to the rate/service structure
- maintaining the component-level inclusion rules for each side

Required UI behaviour:

- FSC profile selector on the applicable rate/service maintenance surface
- visible indication of which profile is structurally assigned by default
- courier-side companion flags exposed alongside the existing client-side fuel flags

Examples:

- `Fuel Surcharge Profile: [House FSC ▼]`
- `Apply Base FSC (Client)`
- `Apply Base FSC (Courier)`
- `Apply Weight FSC (Client)`
- `Apply Weight FSC (Courier)`

This is the structural/default setup point.

## 8.3 Configurator → Client Detail → Available Services

This is the second operational assignment point.

Inside `AvailableServicesTab.tsx`, add:

- FSC profile dropdown sourced from `tblFuelSurcharge`
- visible label for `House FSC` fallback if none explicitly selected
- read-only preview of:
  - client FSC %
  - courier FSC %
  - selected profile name
- clear indication of whether the client/service is:
  - inheriting the rate/default FSC profile, or
  - using a client-specific override

This is the client-level visibility and override point.

## 9. Step-by-step checklist

1. Add `Name`, `DriverFuelPercentage`, `IsDefaultHouseProfile` to `tblFuelSurcharge`.
2. Add `FuelSurchargeID` FK to `tblClientAvailableSpeed`.
3. Add courier-side FSC flags to `DistanceRate`.
4. Add courier-side FSC flags to `ExtraCharge`.
5. Extend EF models for all of the above.
6. Extend fuel DTOs and controller/service payloads.
7. Extend client speed DTOs and controller/service payloads.
8. Extend rate maintenance DTOs/payloads for new courier-side flags.
9. Update Admin Manager fuel/profile UI.
10. Update rate setup / maintenance UI so the applicable FSC profile can be attached to the rate/service structure.
11. Update Configurator Available Services UI for client-level visibility and override of FSC profile.
12. Update the rating proc to:
    - resolve selected profile
    - calculate client basis
    - calculate courier basis
    - zero courier FSC when client FSC is not charged
    - roll to one FSC line per side
13. Update downstream invoice / settlement presentation if it currently shows component-level FSC lines.
14. Add audit/history for profile assignment changes.
15. Regression test non-OTG behaviour.

## 10. Acceptance criteria

The work is only complete when all of the following are true:

1. Admin can create multiple named FSC profiles in the existing fuel area.
2. One row in `tblFuelSurcharge` can represent `House FSC`, `20/15% FSC`, etc.
3. A client/service can be assigned an FSC profile from the rating area.
4. Client FSC uses `tblFuelSurcharge.Rate`.
5. Courier FSC uses `tblFuelSurcharge.DriverFuelPercentage`.
6. Base and distance can be included separately for client and courier sides.
7. Extra/accessorial charge components can be included separately for client and courier sides.
8. Client and courier FSC are calculated independently.
9. Courier FSC is zero when client FSC is not charged.
10. Jobs/invoices show only one client FSC line.
11. Driver earnings/settlements show only one courier FSC line.
12. Existing clients retain current behaviour after migration unless profile assignment or courier-side flags are explicitly changed.

## 11. Open item / scope note

The client also mentioned assigning alternate profiles to specific drivers or driver groups.

That is **not yet tied to a confirmed existing courier/group assignment table in the source reviewed here**.

For this implementation:

- **Phase 1**: assign FSC at the client/service rating level via `tblClientAvailableSpeed`
- the selected profile contains both the client and courier side rules

If OTG later needs courier/group-specific profile assignment independent of client/service, Kerran should add that as a separate follow-up once the canonical courier/group assignment table is confirmed.

## 12. Build / verification commands

```bash
cd /data/.openclaw/workspace/fuel-mfv-analysis
ls -la IMPLEMENTATION.md

cd /data/.openclaw/workspace/Kerran-Configurator
npm run build || npx tsc --noEmit

cd /data/.openclaw/workspace/gitlab-source/adminmanager
# build command depends on local solution entrypoint / local environment
```
