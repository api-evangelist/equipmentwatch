---
name: equipmentwatch-verify-serial-number
description: >-
  Verify the year of manufacture of a machine from its serial number, and reconcile the
  answer against the model the counterparty claims — the underwriting and financing check
  EquipmentWatch's Verification API exists to serve.
api: equipmentwatch:verification
generated: '2026-09-06'
method: generated
source: >-
  Grounded in the published OpenAPI at https://docs.equipmentwatchapi.com/openapi.yaml and the
  response example on https://equipmentwatch.com/api/verification/. The source spec declares no
  operationIds, so steps name the real METHOD and PATH.
operations:
  - GET /verification/serialnumberverification
  - GET /taxonomy/models
  - GET /values/value
---

# Verify a serial number

Base URL: `https://equipmentwatchapi.com/v1`. Header: `x-api-key: <key>`.

## 1. Verify

```
GET /verification/serialnumberverification?serialNumber=60311270
```

Narrow with `manufacturerId`, `modelId` or `categoryId` when you already know them — serial
number formats are not globally unique across manufacturers, so an unnarrowed lookup can return
candidates from several OEMs.

The published example returns the queried `serialNumber` plus a `models` array, each entry
carrying `modelRdbId`, `modelName`, `modelAliases`, `modelInstanceName`, the resolved taxonomy,
a `year`, and the `lowSerialNumber` / `highSerialNumber` bounds of the range the serial fell in.

## 2. Read the answer correctly

The `year` is derived from a **serial range**, not from a per-unit record. A serial sitting at
the boundary of a range, or a machine whose serial was reissued, will resolve to the range's
year. Always surface `lowSerialNumber` and `highSerialNumber` alongside the year so a human can
see how wide the inference is.

If `models` has more than one entry, you have an **ambiguous** result, not a verified one. Say
so. Do not pick the first row.

## 3. Reconcile against the claim

Compare the returned `modelRdbId` / `modelName` and `year` against what the counterparty
declared. A mismatch is the finding this whole flow exists to produce:

- Year mismatch → the asset may be older than represented; the valuation and the loan-to-value
  or premium built on it are wrong.
- Model mismatch → the serial does not belong to the machine being described.

## 4. Optionally, re-value on the verified facts

```
GET /values/value?modelId=<verified modelRdbId>&year=<verified year>&condition=<condition>
```

Re-running the valuation on the *verified* year rather than the *claimed* year is the point of
the exercise.

## Rules

- **Never assert a year the API did not return.** If the serial does not resolve, report that it
  did not resolve. An invented year in an underwriting or financing workflow is a real financial
  error, not a formatting slip.
- **Report ambiguity as ambiguity.** Multiple candidate models means unverified.
- **Handle the error shapes.** 401 is `{"errorCode":10,...}` JSON; an unmatched path returns
  HTML, not JSON.
- **Read-only.** Nothing here writes to EquipmentWatch; no idempotency key applies and there is
  nothing to reverse.
