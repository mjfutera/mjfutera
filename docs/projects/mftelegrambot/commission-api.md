# MF Telegram Bot - Commission API

## Overview
This document describes how to integrate with the commission endpoints.

Machine-readable specification:
- ./commission-api.openapi.yaml

Base URL example:
- https://tgbotapi.michalfutera.pro

Authentication:
- Header: Authorization: Bearer <API_SECRET>
- Content-Type: application/json

Currency and amounts:
- Input amount is always in cents.
- Stored and returned amount is in USD decimal (cents / 100).

---

## Endpoints

### 1) Add Commission Batch
- Method: POST
- Path: /api/commission

Use this endpoint to create a new commission batch for a unique source_ref.

Required fields:
- source_project: string (must exist in projects.slug)
- source_ref: string (must be unique for commission creation)
- commissions: array (non-empty)

Commission item:
- user_id: number (must be an existing partner)
- amount: number (in cents, > 0)
- message: string (optional)

Request example:
```json
{
  "source_project": "sample-project",
  "source_ref": "order_2026_0001",
  "commissions": [
    { "user_id": 123456789, "amount": 2500, "message": "Sale #0001" },
    { "user_id": 100200300, "amount": 900 }
  ]
}
```

Success response example (200):
```json
{
  "status": "ok",
  "processed": 2,
  "failed": 0,
  "results": [
    { "user_id": 123456789, "transaction_id": 101, "amount": 25, "new_balance": 4250.12 },
    { "user_id": 100200300, "transaction_id": 102, "amount": 9, "new_balance": 168.55 }
  ],
  "errors": []
}
```

Partial response example (200):
```json
{
  "status": "partial",
  "processed": 1,
  "failed": 1,
  "results": [
    { "user_id": 123456789, "transaction_id": 103, "amount": 25, "new_balance": 4275.12 }
  ],
  "errors": [
    { "user_id": 999999999, "error": "User is not a partner" }
  ]
}
```

---

### 2) Edit Commission Batch
- Method: POST
- Path: /api/commission/edit

Use this endpoint to update an existing commission batch by replacing it with a new commissions array.

Important behavior:
- If an existing entry matches exactly (same user_id and same amount), it is left unchanged.
- If an existing entry is removed or changed, the old transaction is marked as voided and balance is reverted.
- If a new entry appears, a new active transaction is created and balance is increased.
- This design supports repeated edits safely.

Required fields:
- source_project: string
- source_ref: string
- commissions: array (non-empty)

Optional fields:
- message: string (reason for update, used in change logs/notifications)

Request example:
```json
{
  "source_project": "sample-project",
  "source_ref": "order_2026_0001",
  "message": "Order line items corrected",
  "commissions": [
    { "user_id": 123456789, "amount": 2500 },
    { "user_id": 556677889, "amount": 1100, "message": "Added missing partner" }
  ]
}
```

Success response example (200):
```json
{
  "status": "ok",
  "source_project": "sample-project",
  "source_ref": "order_2026_0001",
  "unchanged": 1,
  "voided": 1,
  "added": 1,
  "details": {
    "voided": [
      { "user_id": 100200300, "amount": 9, "transaction_id": 102 }
    ],
    "added": [
      { "user_id": 556677889, "amount": 11, "transaction_id": 120 }
    ]
  }
}
```

---

### 3) Delete Commission Batch
- Method: POST
- Path: /api/commission/delete

Use this endpoint to void all active commissions in a batch.

Required fields:
- source_project: string
- source_ref: string

Optional fields:
- message: string (reason for cancellation)

Request example:
```json
{
  "source_project": "sample-project",
  "source_ref": "order_2026_0001",
  "message": "Order refunded"
}
```

Success response example (200):
```json
{
  "status": "ok",
  "source_project": "sample-project",
  "source_ref": "order_2026_0001",
  "voided": 2
}
```

---

## Error Format
Validation and business errors use a consistent schema:

```json
{
  "status": "error",
  "code": "ERROR_CODE",
  "message": "Human readable message",
  "details": {
    "optional": "context"
  }
}
```

Common error codes:
- UNAUTHORIZED (403)
- INVALID_JSON (400)
- MISSING_SOURCE_PROJECT (400)
- MISSING_SOURCE_REF (400)
- SOURCE_PROJECT_NOT_FOUND (404)
- DUPLICATE_SOURCE_REF (409) - create only
- INVALID_COMMISSIONS (400)
- USER_NOT_PARTNER (400)
- COMMISSION_BATCH_NOT_FOUND (404) - edit/delete
- INVALID_MODE (400)

---

## Integration Notes
- source_ref should be your stable external reference (for example order ID).
- Do not generate a new source_ref for edits of the same business event.
- Reuse the same source_ref when calling edit/delete on an existing batch.
- Use integer cents to avoid floating-point issues.
- Retry logic: if you receive 409 DUPLICATE_SOURCE_REF on create, check whether the batch was already created.

---

## Minimal cURL Examples

Create:
```bash
curl -X POST "https://tgbotapi.michalfutera.pro/api/commission" \
  -H "Authorization: Bearer YOUR_API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source_project": "sample-project",
    "source_ref": "order_2026_0001",
    "commissions": [
      { "user_id": 123456789, "amount": 2500 }
    ]
  }'
```

Edit:
```bash
curl -X POST "https://tgbotapi.michalfutera.pro/api/commission/edit" \
  -H "Authorization: Bearer YOUR_API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source_project": "sample-project",
    "source_ref": "order_2026_0001",
    "message": "Correction",
    "commissions": [
      { "user_id": 123456789, "amount": 2600 }
    ]
  }'
```

Delete:
```bash
curl -X POST "https://tgbotapi.michalfutera.pro/api/commission/delete" \
  -H "Authorization: Bearer YOUR_API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source_project": "sample-project",
    "source_ref": "order_2026_0001",
    "message": "Refund"
  }'
```
