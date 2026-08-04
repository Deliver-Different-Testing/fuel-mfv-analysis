# OTG fuel logic change outline

## Purpose

This note explains, in plain English, the change OTG want to the fuel logic and a practical way to implement it without disrupting existing clients.

## Current behaviour

From Kerran's current logic:

- the **driver fuel percentage** comes from `VehicleSize.FuelPercentage`
- the **driver fuel amount** is calculated from the **client charge components**, not from the driver pay/base
- fuel is only calculated for charge components where the relevant fuel flag is turned on

In simple terms, the current logic is effectively:

- client fuel = client base or distance charge x MFV
- driver fuel = client base or distance charge x MFV x driver fuel percentage

That means the current driver fuel is a **share of the client fuel calculation**, not a separate calculation based on the driver's own base pay.

## What OTG want

OTG want three things:

1. **Driver fuel should be able to be calculated against the driver base**, not just the client base.
2. **If the client is not charged fuel, the driver should not be paid fuel.**
3. **The default fuel percentage should stay generic/easy to maintain**, with client-specific override behaviour only where needed.

## Recommended approach

### 1. Add a basis setting for driver fuel

Introduce a setting that controls what the driver fuel is calculated against.

Suggested values:

- `ClientBase`
- `DriverBase`

This allows the existing behaviour to remain available, while enabling OTG's preferred behaviour.

### 2. Keep the default driver fuel percentage generic

Add a `DriverFuelPercentage` field to the MFV / client fuel configuration.

Behaviour:

- if `DriverFuelPercentage` is populated for the client configuration, use it
- if `DriverFuelPercentage` is null / not populated, fall back to `VehicleSize.FuelPercentage`

This keeps monthly or periodic updates simple, because the default percentage can still be maintained in one place while still allowing client-specific driver fuel percentages where needed.

### 3. Allow client-specific override where required

If a client needs different behaviour, allow an override for that client.

For OTG, the client override would set:

- driver fuel basis = `DriverBase`
- optional client-specific `DriverFuelPercentage`
- if no client-specific `DriverFuelPercentage` is supplied, fall back to `VehicleSize.FuelPercentage`

If no client override exists, the system should continue using the generic/default setup.

### 4. Keep the client fuel gate in place

Regardless of which basis is chosen, the driver fuel should still only be calculated when fuel is turned on for that charge component.

In plain English:

- if the client is not being charged fuel for that part of the job, the driver fuel for that part should be zero

This is important because OTG explicitly do **not** want driver fuel paid when client fuel is off.

## Proposed rules in plain English

### Default behaviour for most clients

- use the existing fuel flags
- use the generic driver fuel percentage from `VehicleSize.FuelPercentage`
- keep current basis unless overridden

### OTG override behaviour

- still respect the existing fuel flags
- if client fuel is off, driver fuel is zero
- if client fuel is on, calculate driver fuel against the **driver base** instead of the client base
- use either:
  - the client-specific `DriverFuelPercentage`, or
  - if that is blank, the generic `VehicleSize.FuelPercentage`

## Suggested decision hierarchy

When calculating driver fuel:

1. Check whether fuel applies to that charge component at all.
   - if not, driver fuel = 0
2. Check for a client-specific override.
3. Resolve the driver fuel percentage:
   - use client `DriverFuelPercentage` if populated
   - otherwise use `VehicleSize.FuelPercentage`
4. Use the configured basis:
   - `ClientBase` = calculate from client charge amount
   - `DriverBase` = calculate from driver pay/base amount

## Why this is the safest option

This approach gives OTG what they need **without forcing a full redesign** for every other client.

Benefits:

- preserves current behaviour for everyone else by default
- keeps the standard fuel percentage easy to maintain
- allows OTG to have a client-specific rule
- avoids paying driver fuel when client fuel is off
- gives flexibility later if another client wants the same behaviour

## UI / configurator changes required

This change also needs to be visible and maintainable in the UI.

### Client-level configurator UI

In the client detail / available services area of Configurator, add client-visible fields for the fuel behaviour for that service.

Suggested fields:

- client fuel enabled / existing fuel settings
- driver fuel basis
  - `ClientBase`
  - `DriverBase`
- driver fuel percentage override
- clear indication that blank driver fuel percentage means fallback to `VehicleSize.FuelPercentage`

This gives the team a way to see, per client and per available service, whether that client is using default behaviour or an override.

### Configurator settings / admin setup UI

A setup screen is also needed in Configurator settings (or the equivalent admin area) so these settings can be created and maintained.

This should allow:

- setting or changing default MFV / fuel settings
- setting or changing client-specific fuel behaviour
- setting or changing client-specific `DriverFuelPercentage`
- choosing whether driver fuel is based on `ClientBase` or `DriverBase`
- leaving `DriverFuelPercentage` blank to use the vehicle fallback

### Existing fuel editing screen alignment

The new fields should align with the existing fuel editing concepts already present in admin tooling, rather than introducing a completely separate fuel setup pattern.

The goal should be:

- one obvious place to maintain default fuel settings
- one obvious place to see and override client-specific behaviour
- no ambiguity about whether a client is using default or override logic

## What should change in the logic

The key change is that the driver fuel calculation should no longer be hard-wired to client charge amounts only.

Instead, it should:

- read a **driver fuel basis** setting
- resolve a **driver fuel percentage** from client override first, then vehicle fallback
- choose either client base or driver base
- apply the chosen percentage
- only do so when client fuel is enabled for that charge

## Acceptance criteria

The change should be considered correct when all of the below are true:

1. A default client with no override continues to behave as it does today.
2. OTG can be configured so driver fuel is calculated from **driver base**.
3. If fuel is off for the client charge, driver fuel is also **0**.
4. The default driver fuel percentage can still be maintained generically.
5. A client override can change the basis and, if needed, the percentage.
6. If client `DriverFuelPercentage` is blank, the logic falls back to `VehicleSize.FuelPercentage`.
7. Client/service fuel settings are visible in Configurator.
8. Admin users have a settings screen to create or alter these rules.

## Recommended next step

Kerran should implement this as a **configuration-driven change**, not as a special OTG-only hard-code.

That keeps the logic clean:

- generic by default
- override only where needed
- OTG gets the driver-base behaviour they want
