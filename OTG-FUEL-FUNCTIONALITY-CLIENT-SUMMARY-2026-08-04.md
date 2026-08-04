# OTG fuel functionality summary

## What this change will achieve

We will update the fuel logic so OTG can have fuel behaviour that better matches how you want to pay drivers and charge clients.

## The functionality OTG will get

### 1. Driver fuel can be based on driver pay instead of client charge

At the moment, driver fuel is tied too closely to the client-side fuel calculation.

After this change, OTG will be able to use a setup where:

- driver fuel is calculated from the **driver base amount**
- instead of only being calculated from the **client charge amount**

This gives OTG a more accurate way to calculate fuel reimbursement for drivers.

### 2. If fuel is not charged to the client, driver fuel will not be paid

A key requirement from OTG is that driver fuel should not continue to be paid when client fuel is switched off.

After this change:

- if the client is not charged fuel for that service or job component
- the driver fuel for that same part will be **zero**

This keeps the logic commercially aligned.

### 3. Client-specific fuel settings

OTG will be able to have its **own fuel rules** without forcing the same behaviour on every other client.

That means OTG can have:

- its own driver fuel basis
- its own driver fuel percentage where needed
- its own override behaviour where appropriate

## How the setup will work

### Default behaviour

The system will continue to support a generic/default fuel setup for normal use.

### OTG override behaviour

OTG can be configured so that:

- driver fuel is based on **driver base**
- a client-specific driver fuel percentage can be used if required
- if no client-specific driver fuel percentage is entered, the system falls back to the standard default percentage

This gives flexibility without making the whole system harder to manage.

## What users will be able to see and maintain

### In client configuration

The client configuration area will show the fuel settings relevant to OTG's available services, including:

- whether fuel applies
- which basis is being used for driver fuel
- whether OTG is using a default percentage or an override

### In admin fuel settings

The existing fuel maintenance area will be extended so the fuel settings can be updated properly, including:

- default fuel settings
- default driver fuel percentage behaviour
- client-specific override behaviour where needed

## Result for OTG

Once implemented, OTG will have:

- driver fuel based on the right commercial basis
- no driver fuel where no client fuel is charged
- client-specific control where needed
- clearer visibility of fuel settings in the system
- a cleaner and more maintainable long-term setup
