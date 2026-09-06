---
name: equipmentwatch-value-an-asset
description: >-
  Resolve a piece of heavy equipment from a manufacturer and model name to an EquipmentWatch
  modelRdbId, then retrieve its Fair Market Value, Forced Liquidation Value and Orderly
  Liquidation Value for a given year, condition and region.
api: equipmentwatch:values
generated: '2026-09-06'
method: generated
source: >-
  Grounded in the published OpenAPI at https://docs.equipmentwatchapi.com/openapi.yaml.
  The source spec declares no operationIds, so every step below names the real METHOD and
  PATH rather than an invented operationId.
operations:
  - GET /taxonomy/manufacturers
  - GET /taxonomy/models
  - GET /values/condition
  - GET /values/region
  - GET /values/value
  - GET /values/trending
---

# Value a piece of heavy equipment

Base URL: `https://equipmentwatchapi.com/v1` (sandbox: `https://sandbox.equipmentwatchapi.com/v1`)
Every request needs `x-api-key: <key>`.

## 1. Resolve the manufacturer

```
GET /taxonomy/manufacturers?manufacturer=Caterpillar&limit=50
```

Take `manufacturerRdbId` from the match. Names are aliased, so a near-miss spelling may still
resolve — but confirm the returned name before continuing rather than trusting the first row.

## 2. Resolve the model

```
GET /taxonomy/models?manufacturerId=<manufacturerRdbId>&model=120M&limit=50
```

Take `modelRdbId`. Models carry a `modelAliases` array; several rows may look like the same
machine because they are different instances or discontinued variants (`modelInstanceName`
often says so, e.g. `"CA121PDB (disc. 2008)"`). Pick deliberately — the wrong instance produces
a valuation for a different machine.

`limit` is capped at 50 and the response carries no total count. If 50 rows come back, page with
`offset` until a short page returns before deciding there is no better match.

## 3. Read the allowed condition and region values

```
GET /values/condition
GET /values/region
```

Do not hard-code these. The `condition` query parameter is an enum in the spec —
`Excellent`, `Very Good`, `Good`, `Fair`, `Poor` — but the region set is data, not a spec enum.

## 4. Fetch the valuation

```
GET /values/value?modelId=<modelRdbId>&year=2011&condition=Very%20Good
```

`modelId` and `year` are the required inputs on this operation. Optional refinements:
`configurationSequence` for a specific build, and `olvBasis` to control the Orderly Liquidation
Value basis.

The published examples return `adjustedFmv`, `adjustedFlv`, `unadjustedFmv`, `unadjustedFlv`,
`original` and a `revisionDate`, alongside the resolved taxonomy. Report the `revisionDate` with
any number you surface — a valuation without its revision date is not auditable.

## 5. Optionally, trend it

```
GET /values/trending?modelId=<modelRdbId>&year=2011
```

## Rules

- **Currency is never declared.** The API returns bare numbers. USD is implied by the North
  American scope but is not stated anywhere in the contract. Do not label a figure as USD to a
  user without saying that the currency is an inference.
- **No response schema is published.** `components/schemas` in the OpenAPI holds a single stub.
  Field names above come from the provider's published response examples. Treat any field as
  optional and handle its absence.
- **401 means either missing OR invalid key.** The envelope is
  `{"errorCode":10,"errorMessage":"Invalid API Key","errorDescription":"API Key Provided is invalid"}`.
  EquipmentWatch does not distinguish the two, so do not report "credential expired" — report
  "key rejected".
- **A 404 is not JSON.** Unmatched routes return an HTML error page. Guard your parser.
- **No rate-limit headers exist.** Nothing tells you when to back off. Throttle yourself and
  retry with exponential backoff on 5xx.
- **This is a read-only flow.** No step here changes anything at EquipmentWatch, so there is
  nothing to reverse and no idempotency key to send.
