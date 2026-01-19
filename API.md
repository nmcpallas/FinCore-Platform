# Ledger API Contract (Phase 1)

This document describes the HTTP API for the Phase 1 ledger core:
- double-entry journal (JournalEntry + Postings)
- concurrency-safe commands (transfer, holds)
- idempotent request processing (Operation / Idempotency-Key)
- balance snapshots (available/held)

## Navigation
- [Guarantees](#guarantees)
- [Conventions](#conventions)
- [Error model](#error-model-problem-details)
- [Endpoints](#endpoints)
  - [Accounts](#accounts)
  - [Balances](#balances)
  - [Holds / Reservations](#holds--reservations)
  - [Ledger / Audit](#ledger--audit)


## Guarantees

- **Double-entry correctness:** every successful command persists a **balanced** journal entry where `sum(debits) == sum(credits)`.
- **Atomicity:** each command writes `journal_entry + postings (+ balance snapshot + operation status)` in a **single DB transaction**.
- **Idempotency:** repeating the same command with the same `Idempotency-Key` returns the **same result** without duplicating postings.
- **Auditability:** ledger facts are **append-only** (no updates/deletes); corrections use new entries (reversal pattern).

---

## Conventions

### Base URL
`/api/v1`

### Money format
- Amounts in requests and responses are **strings**, never JSON numbers:
  - ✅ `"10.00"`
  - ❌ `10.00`
- Currency is an uppercase ISO code, e.g. `"USD"`.
- Internally the system stores money as minor units (e.g. cents) or strict decimal with fixed scale.

### Idempotency
All **command** endpoints MUST support idempotency.

**Request header**
- `Idempotency-Key: <uuid>`

**Rules**
- If the same `Idempotency-Key` is used with the **same request meaning** (same `request_hash`), the API must return the **same response** as the first successful call.
- If the same `Idempotency-Key` is reused with a **different request** (`request_hash` mismatch), the API returns:
  - `409 Conflict` with error code `IDEMPOTENCY_KEY_REUSED`

### Pagination
List endpoints use cursor-like pagination:
- `limit` (default 50, max 200)
- `cursor` (opaque string)

### Timestamps
All timestamps are ISO-8601 UTC:
- `2026-01-19T12:34:56Z`

---

## Error model (Problem Details)

Errors follow RFC 7807-like shape (`application/problem+json`):

```json
{
  "type": "about:blank",
  "title": "Insufficient funds",
  "status": 422,
  "code": "INSUFFICIENT_FUNDS",
  "detail": "Account A has available 5.00 USD, required 10.00 USD",
  "instance": "/api/v1/transfers",
  "traceId": "..."
}
```

## Common error codes
- VALIDATION_ERROR
- CURRENCY_MISMATCH
- INSUFFICIENT_FUNDS
- ACCOUNT_NOT_FOUND
- HOLD_NOT_FOUND
- HOLD_NOT_ACTIVE
- IDEMPOTENCY_KEY_REUSED
- CONCURRENCY_RETRY_EXHAUSTED

## Endpoints

### Accounts

#### *POST /accounts*
Create account

Request
```json
{
  "ownerId": "12345",
  "currency": "USD",
  "type": "USER"
}
```
Response (201)
```json
{
  "accountId": "acc_01H...",
  "currency": "USD",
  "type": "USER",
  "createdAt": "2026-01-19T12:00:00Z"
}
```
Errors
- 400 VALIDATION_ERROR

#### *GET /accounts/{accountId}*

Get account

Response (200)
```json
{
  "accountId": "acc_01H...",
  "ownerId": "12345",
  "currency": "USD",
  "type": "USER",
  "status": "ACTIVE",
  "createdAt": "2026-01-19T12:00:00Z"
}
```
Errors
- 404 ACCOUNT_NOT_FOUND

### Balances

#### *GET /accounts/{accountId}/balance*

Returns the current balance snapshot.

Response (200)
```json
{
  "accountId": "acc_01H...",
  "currency": "USD",
  "available": "100.00",
  "held": "25.00",
  "total": "125.00",
  "asOf": "2026-01-19T12:34:56Z"
}
```
Errors
- 404 ACCOUNT_NOT_FOUND

### Transfers

#### *POST /transfers*

Creates a balanced journal entry that moves funds from one account to another.

Headers
```text
Idempotency-Key: <uuid>
```
Request
```json
{
  "fromAccountId": "acc_A",
  "toAccountId": "acc_B",
  "amount": "10.00",
  "currency": "USD",
  "note": "rent"
}
```
Response (201)
```json
{
  "operationId": "op_01H...",
  "status": "SUCCEEDED",
  "journalEntryId": "je_01H..."
}
```
**Idempotency behavior**
Same Idempotency-Key + same request meaning → return the same journalEntryId and response.
Same Idempotency-Key + different request meaning → 409 IDEMPOTENCY_KEY_REUSED.

Errors
- 404 ACCOUNT_NOT_FOUND
- 400 CURRENCY_MISMATCH
- 422 INSUFFICIENT_FUNDS
- 409 IDEMPOTENCY_KEY_REUSED

### Holds / Reservations

#### *POST /holds*

Reserves funds on an account: available → held.

Headers
```text
Idempotency-Key: <uuid>
```

Request
```json
{
  "accountId": "acc_A",
  "amount": "25.00",
  "currency": "USD",
  "reason": "booking"
}
```
Response (201)
```json
{
  "operationId": "op_01H...",
  "status": "ACTIVE",
  "holdId": "hold_01H...",
  "journalEntryId": "je_01H..."
}
```
Errors
- 404 ACCOUNT_NOT_FOUND
- 400 CURRENCY_MISMATCH
- 422 INSUFFICIENT_FUNDS
- 409 IDEMPOTENCY_KEY_REUSED

#### *POST /holds/{holdId}/release*

Releases reserved funds: held → available.

Headers
```text
Idempotency-Key: <uuid>
```
Response (200)
```json
{
  "operationId": "op_01H...",
  "status": "RELEASED",
  "holdId": "hold_01H...",
  "journalEntryId": "je_01H..."
}
```
Errors
- 404 HOLD_NOT_FOUND
- 409 HOLD_NOT_ACTIVE
- 409 IDEMPOTENCY_KEY_REUSED

#### *POST /holds/{holdId}/capture*

Captures reserved funds and transfers them to a target account: held → target.

Headers
```text
Idempotency-Key: <uuid>
```
Request
```json
{
  "toAccountId": "acc_MERCHANT",
  "amount": "25.00",
  "currency": "USD"
}
```
Response (200)
```json
{
  "operationId": "op_01H...",
  "status": "CAPTURED",
  "holdId": "hold_01H...",
  "journalEntryId": "je_01H..."
}
```
Errors
- 404 HOLD_NOT_FOUND
- 404 ACCOUNT_NOT_FOUND
- 409 HOLD_NOT_ACTIVE
- 400 CURRENCY_MISMATCH
- 422 INSUFFICIENT_HELD_FUNDS
- 409 IDEMPOTENCY_KEY_REUSED

### Ledger / Audit

#### *GET /journal-entries/{journalEntryId}*

Returns the immutable journal entry and its postings (audit view).

Response (200)
```json
{
  "journalEntryId": "je_01H...",
  "operationId": "op_01H...",
  "type": "TRANSFER",
  "createdAt": "2026-01-19T12:34:56Z",
  "metadata": {
    "note": "rent"
  },
  "postings": [
    {
      "postingId": "p_01H...",
      "accountId": "acc_A",
      "direction": "DEBIT",
      "amount": "10.00",
      "currency": "USD"
    },
    {
      "postingId": "p_01H...",
      "accountId": "acc_B",
      "direction": "CREDIT",
      "amount": "10.00",
      "currency": "USD"
    }
  ]
}
```
Errors
- 404 JOURNAL_ENTRY_NOT_FOUND

#### *GET /accounts/{accountId}/postings?limit=50&cursor=...*

Returns postings affecting the account, ordered newest-first (or oldest-first; pick one and keep it consistent).

Response (200)
```json
{
  "accountId": "acc_A",
  "items": [
    {
      "postingId": "p_01H...",
      "journalEntryId": "je_01H...",
      "operationId": "op_01H...",
      "direction": "DEBIT",
      "amount": "10.00",
      "currency": "USD",
      "createdAt": "2026-01-19T12:34:56Z"
    }
  ],
  "nextCursor": "..."
}
```
Errors
- 404 ACCOUNT_NOT_FOUND

### Operations (Idempotency)

#### *GET /operations/{operationId}*

Returns the operation lifecycle info and the linked journal entry (if any).

Response (200)
```json
{
  "operationId": "op_01H...",
  "type": "TRANSFER",
  "status": "SUCCEEDED",
  "requestHash": "sha256:...",
  "journalEntryId": "je_01H...",
  "createdAt": "2026-01-19T12:34:00Z",
  "updatedAt": "2026-01-19T12:34:56Z"
}
```
Errors
- 404 OPERATION_NOT_FOUND
