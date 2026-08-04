# OTG fuel change summary

## Purpose

This note summarises the fuel logic change OTG have requested in plain English.

## What OTG need

OTG want the fuel logic to work like this:

1. **Driver fuel should be able to be calculated from the driver base amount**, not only from the client charge amount.
2. **If fuel is not being charged to the client, driver fuel should not be paid.**
3. **The system should stay simple to maintain**, with a generic default setup for most clients and an override only where a client needs different behaviour.

## Current position

The current logic appears to calculate driver fuel from the **client-side charge components**, using a fuel factor and a percentage taken from the vehicle setup.

That means the current driver fuel behaves more like a share of the client fuel calculation, rather than a separate calculation based on the driver's own base pay.

## Requested change

The recommended change is to make the driver fuel logic configurable.

### Default behaviour

For most clients, the system can continue using the existing generic/default setup.

### OTG-specific behaviour

For OTG, the system should support:

- driver fuel being calculated from the **driver base**
- driver fuel only applying when the client fuel rule is on
- use of either the standard default percentage or an OTG-specific percentage if needed

## Recommended approach

Introduce a configurable setting for how driver fuel is calculated.

Suggested options:

- **Client base**
- **Driver base**

This would allow:

- existing clients to keep current behaviour
- OTG to use driver-base fuel calculation

## Important rule

Even when OTG uses driver-base fuel calculation, the following rule should still apply:

- **if the client is not charged fuel, the driver fuel should be zero**

This ensures the driver fuel does not continue to be paid when client fuel has been turned off.

## Why this approach makes sense

This gives OTG the behaviour they want without forcing a full change for every other client.

Benefits:

- simple default behaviour remains in place
- OTG can have the different rule they need
- fuel maintenance stays easier for the general case
- future clients can also use the override if required

## Outcome OTG should expect

If this change is implemented, OTG should be able to have:

- driver fuel based on driver base
- no driver fuel where no client fuel is charged
- a maintainable default setup for everyone else
- client-specific override behaviour where required
