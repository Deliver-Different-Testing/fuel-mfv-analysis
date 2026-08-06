# FSC_INDEXING_SUPPLEMENT_2026-08-06.md

## Purpose

This document is a supplement to the current FSC implementation spec.

It covers the **future indexed FSC functionality** where a fuel surcharge profile can automatically update from a stored fuel-price index in DFRNT, with the option for that internal index to later be fed from an external independent fuel-price source.

This is **not** a replacement for the current FSC profile work. It is an extension that should be designed now so the FSC table does not need another avoidable redesign later.

---

## Outcome required

DFRNT should support FSC profiles that are either:

- **Fixed** — manually maintained percentage
- **Indexed** — automatically recalculated from a linked fuel-price index

Indexed FSC profiles should support automatic refresh on one of these frequencies:

- **Weekly**
- **Fortnightly**
- **Monthly**

The indexed result should create or update the effective dated FSC row for the relevant period so the existing `Start` / `End` model continues to control history and runtime selection.

---

## Design approach

### Keep `tblFuelSurcharge` as the effective FSC record

`tblFuelSurcharge` already stores:

- FSC percentage (`Rate`)
- effective date range (`Start`, `End`)
- active flag
- client-specific or broader scope

That makes it the correct place to store the **effective calculated FSC result**.

### Add indexing metadata to `tblFuelSurcharge`

While the table is being extended, add the minimum metadata needed to support indexed behaviour later.

Recommended new columns:

```sql
IF COL_LENGTH('dbo.tblFuelSurcharge', 'IsIndexed') IS NULL
BEGIN
    ALTER TABLE dbo.tblFuelSurcharge
    ADD IsIndexed bit NOT NULL
        CONSTRAINT DF_tblFuelSurcharge_IsIndexed DEFAULT (0);
END;
GO

IF COL_LENGTH('dbo.tblFuelSurcharge', 'FuelIndexSourceID') IS NULL
BEGIN
    ALTER TABLE dbo.tblFuelSurcharge
    ADD FuelIndexSourceID int NULL;
END;
GO

IF COL_LENGTH('dbo.tblFuelSurcharge', 'AutoUpdateFrequency') IS NULL
BEGIN
    ALTER TABLE dbo.tblFuelSurcharge
    ADD AutoUpdateFrequency varchar(20) NULL;
END;
GO

IF NOT EXISTS (
    SELECT 1
    FROM sys.check_constraints
    WHERE name = 'CK_tblFuelSurcharge_AutoUpdateFrequency'
)
BEGIN
    ALTER TABLE dbo.tblFuelSurcharge
    ADD CONSTRAINT CK_tblFuelSurcharge_AutoUpdateFrequency
        CHECK (AutoUpdateFrequency IN ('Weekly', 'Fortnightly', 'Monthly') OR AutoUpdateFrequency IS NULL);
END;
GO
```

### Meaning of these fields

- `IsIndexed`
  - `0` = fixed/manual FSC profile
  - `1` = indexed FSC profile

- `FuelIndexSourceID`
  - identifies which stored fuel index should be used
  - allows DFRNT to support multiple different index feeds or house index rules later

- `AutoUpdateFrequency`
  - how often DFRNT should generate the next calculated FSC value
  - allowed values:
    - `Weekly`
    - `Fortnightly`
    - `Monthly`

---

## New supporting tables

## 1. `dbo.FuelIndexSource`

This table defines the available fuel index sources.

```sql
CREATE TABLE dbo.FuelIndexSource
(
    FuelIndexSourceID int IDENTITY(1,1) PRIMARY KEY,
    Name varchar(100) NOT NULL,
    Code varchar(50) NOT NULL,
    Description varchar(500) NULL,
    ExternalProvider varchar(100) NULL,
    ExternalReference varchar(200) NULL,
    Active bit NOT NULL CONSTRAINT DF_FuelIndexSource_Active DEFAULT (1),
    Created datetime NOT NULL,
    CreatedBy varchar(50) NOT NULL,
    LastModified datetime NOT NULL,
    LastModifiedBy varchar(50) NOT NULL
);
```

### Example rows

- `NZ House Diesel Index`
- `MBIE Retail Diesel Index`
- `OTG Contract Fuel Index`

This is intentionally generic so DFRNT can use:

- internal manually loaded fuel prices
- imported CSV values
- API-fed external data later

---

## 2. `dbo.FuelIndexValue`

This table stores the dated fuel-price readings used by the indexed FSC calculation.

```sql
CREATE TABLE dbo.FuelIndexValue
(
    FuelIndexValueID int IDENTITY(1,1) PRIMARY KEY,
    FuelIndexSourceID int NOT NULL,
    EffectiveDate date NOT NULL,
    FuelPrice decimal(18, 4) NOT NULL,
    Notes varchar(500) NULL,
    SourceType varchar(50) NULL,
    Created datetime NOT NULL,
    CreatedBy varchar(50) NOT NULL,
    LastModified datetime NOT NULL,
    LastModifiedBy varchar(50) NOT NULL,
    CONSTRAINT FK_FuelIndexValue_FuelIndexSource
        FOREIGN KEY (FuelIndexSourceID)
        REFERENCES dbo.FuelIndexSource(FuelIndexSourceID)
);
```

### Purpose

This is the internal DFRNT-held fuel price history that indexed FSC profiles will read from.

It allows:

- manual entry now
- independent external feed attachment later
- auditable history of what price caused what FSC result

---

## Calculation model

## Baseline principle

Indexed FSC should not directly read an external provider at quote/runtime.

Instead:

1. an external source (or manual process) updates `FuelIndexValue`
2. DFRNT calculates the FSC percentage from the stored value
3. DFRNT writes the resulting effective FSC percentage into `tblFuelSurcharge`
4. rating continues using `tblFuelSurcharge.Rate` as normal

This keeps runtime rating stable and auditable.

---

## Recommended indexed calculation flow

For each indexed FSC profile:

1. Resolve the linked `FuelIndexSourceID`
2. Read the most recent `FuelIndexValue` for the current cycle
3. Apply the agreed formula for that profile
4. Generate the next dated FSC row in `tblFuelSurcharge`
5. Mark it active for the correct `Start` / `End` window

### Important design rule

The indexed engine should **calculate ahead of the billing period**, not on the fly during job rating.

That means rating still reads a normal dated FSC row and does not need to know whether the value was manually entered or index-generated.

---

## Auto update frequencies

## Weekly

Use when FSC should refresh every 7 days.

Behaviour:

- one calculated FSC row per week
- `Start` = start of week
- `End` = end of week
- next row generated before the new week begins

## Fortnightly

Use when FSC should refresh every 14 days.

Behaviour:

- one calculated FSC row per fortnight
- `Start` / `End` aligned to OTG’s chosen fortnight boundaries
- next row generated before the next fortnight begins

## Monthly

Use when FSC should refresh once per calendar month.

Behaviour:

- one calculated FSC row per month
- `Start` = first day of month
- `End` = last day of month
- next row generated before month rollover

---

## Additional recommended fields

If Kerran is already touching the table and wants to future-proof slightly further, these are also worth considering:

```sql
IF COL_LENGTH('dbo.tblFuelSurcharge', 'LastIndexedAt') IS NULL
BEGIN
    ALTER TABLE dbo.tblFuelSurcharge
    ADD LastIndexedAt datetime NULL;
END;
GO

IF COL_LENGTH('dbo.tblFuelSurcharge', 'IndexLocked') IS NULL
BEGIN
    ALTER TABLE dbo.tblFuelSurcharge
    ADD IndexLocked bit NOT NULL
        CONSTRAINT DF_tblFuelSurcharge_IndexLocked DEFAULT (0);
END;
GO
```

### Meaning

- `LastIndexedAt`
  - last time the row/profile was refreshed by the indexing process

- `IndexLocked`
  - prevents auto-regeneration for that profile when a commercial exception is required

These are optional for phase 1.

---

## UI requirements

Indexed FSC needs an admin maintenance surface.

For each FSC profile, the UI should support:

- **Indexed?** yes/no
- **Fuel index source** dropdown
- **Auto update frequency** dropdown:
  - Weekly
  - Fortnightly
  - Monthly
- current effective fuel price
- last auto-update date
- next scheduled update date
- manual override / lock where required

---

## Scheduler / job behaviour

A scheduled process should run at least daily and:

1. find indexed FSC profiles
2. check whether the next effective period needs to be generated
3. read the latest `FuelIndexValue`
4. calculate the new FSC percentage
5. insert the next dated `tblFuelSurcharge` row
6. log success/failure

### Important rule

The scheduler should **create the next effective row**, not edit already-used historical rows.

That preserves a reliable audit trail.

---

## Formula handling

The actual formula for converting fuel price to FSC % may differ by commercial agreement.

So the design should allow either:

- one standard formula applied to all indexed profiles
- or per-profile formula parameters later

For now, this supplement does **not** require Kerran to build a full formula engine.

It only requires the schema path so indexed FSC can be added cleanly later.

---

## Acceptance criteria

This supplementary work is correctly scaffolded when:

1. `tblFuelSurcharge` can identify whether a profile is fixed or indexed.
2. `tblFuelSurcharge` can store which fuel index source it uses.
3. `tblFuelSurcharge` can store auto-update frequency as:
   - Weekly
   - Fortnightly
   - Monthly
4. DFRNT has a table for defining fuel index sources.
5. DFRNT has a table for storing dated fuel index values.
6. The design assumes index updates write effective dated FSC rows ahead of runtime.
7. The design preserves existing `Start` / `End` behaviour for history and selection.
8. No live rating logic depends directly on an external source call.

---

## Recommendation for current implementation

If Kerran is already altering `tblFuelSurcharge`, the minimum worthwhile future-proofing is:

- `IsIndexed`
- `FuelIndexSourceID`
- `AutoUpdateFrequency`

That is the right balance between:

- avoiding another schema change later
- not forcing the full indexing engine into the current FSC delivery

---

## Scope note

This supplement is for **indexing readiness and future indexed FSC support**.

It does **not** require immediate implementation of:

- external API integration
- automated formula engine
- full scheduler deployment
- retroactive rebuild of historical FSC rows

Those can follow once the indexed FSC commercial rules are confirmed.
