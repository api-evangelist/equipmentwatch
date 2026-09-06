---
name: equipmentwatch-cost-recovery-rate
description: >-
  Retrieve Rental Rate Blue Book ownership and operating cost recovery rates (and the FHWA
  rate) for a machine, pick the correct build configuration, and compare against retail rental
  rates to decide own-versus-rent.
api: equipmentwatch:costs
generated: '2026-09-06'
method: generated
source: >-
  Grounded in the published OpenAPI at https://docs.equipmentwatchapi.com/openapi.yaml and the
  response example on https://equipmentwatch.com/api/costs/. The source spec declares no
  operationIds, so steps name the real METHOD and PATH.
operations:
  - GET /taxonomy/models
  - GET /cost/configurations
  - GET /cost/cost-recovery
  - GET /cost/icr
  - GET /rental/rentalrates
---

# Get a cost recovery rate

Base URL: `https://equipmentwatchapi.com/v1`. Header: `x-api-key: <key>`.

## 1. Resolve the model

```
GET /taxonomy/models?manufacturer=Caterpillar&model=120M&limit=50
```

Keep `modelRdbId`.

## 2. Pick the configuration

```
GET /cost/configurations?modelId=<modelRdbId>&year=2010
```

Cost operations are configuration-sensitive: `configurationSequence` is a required parameter on
the cost surface. Do not default it to `1` because the example on the marketing page shows a `1`
— fetch the real configurations and choose one, and tell the user which one you chose.

## 3. Fetch the cost recovery rate

```
GET /cost/cost-recovery?modelId=<modelRdbId>&year=2010&configurationSequence=<seq>
```

Optional: `date` (YYYY-MM-DD) pins a specific Blue Book revision. Omit it and you get the most
recent revision — which means the same call can return a different number next month. **If the
number is going into a bid, an invoice or a reimbursement claim, pin the `date`.** That is the
difference between a reproducible figure and one that silently drifts.

The published example returns `ownershipCost`, `hourlyOwnershipCost`, `hourlyOperatingCost` and
`fhwaRate` alongside the resolved taxonomy.

## 4. Internal charge rates

```
GET /cost/icr?modelId=<modelRdbId>&year=2010&configurationSequence=<seq>
```

The ICR surface is the customisable form — it applies an organisation's own cost factors on top
of the Blue Book basis. Use it when the caller wants *their* charge rate, not the industry
benchmark.

## 5. Compare against rental

```
GET /rental/rentalrates?modelId=<modelRdbId>
```

Ownership cost per hour against daily / weekly / monthly retail rental rates is the own-versus-rent
comparison. Note the two figures have different bases — hourly operating and ownership cost
versus a calendar rental rate — so convert explicitly and show your working rather than
presenting a single derived verdict.

## Rules

- **Pin the revision date for anything financial.** Unpinned calls track the latest revision.
- **Currency is not declared by the API.** Do not label figures USD without flagging it as an
  inference from the North American scope.
- **`fhwaRate` is a regulatory reimbursement rate**, not the same thing as an ownership cost.
  Do not substitute one for the other.
- **No response schema is published**, so treat every field as optional and handle absence.
- **Read-only.** No writes, no idempotency key, nothing to reverse.
