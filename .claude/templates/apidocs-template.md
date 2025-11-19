# API Documentation Template

*Purpose*: Capture only the details partners need to integrate with this API. Replace bracketed placeholders and delete helper notes when publishing a final version.

## Overview

| Item | Details |
| --- | --- |
| Service Name | `[Service / Module Name]` |
| Owner | Lending Backend |
| Intended Audience | `[External Partner / Internal Service / etc.]` |
| Base URL | see `environment details`. |
| Version | `v[major.minor.patch]` |
| Last Updated | [YYYY-MM-DD] |

---

## Authentication & Authorization

| Item | Guidance |
| --- | --- |
| Scheme | `[Bearer / Basic / API Key / Custom Header]` |
| Required Headers | `Authorization: Bearer <token>` / `[X-Partner-Token: <value>]` |
| IP Whitelisting | `[List partner IP ranges or reference onboarding document; mention process to request updates.]` |

---

## Environment Details

| Environment | Base URL | Authentication Notes | 
| --- | --- | --- |
| Staging | `https://[staging-host]/[base-path]` | `[e.g., shared credentials via vault]` |
| Production | `https://[prod-host]/[base-path]` | `[e.g., rotating client credentials]` |

---

## Endpoint Summary

| Endpoint | Method | Purpose | Auth Required |
| --- | --- | --- | --- |
| `/fs/brick/service/partner/create-loan` | `POST` | `[Submit new loan proposal to partner]` | `Yes` |
| `[/{resource}]` | `[GET/POST/etc.]` | `[Short description]` | `[Yes/No]` |

---

## Endpoint Details

### `[Endpoint Friendly Name]`

* **Path & Method:** `POST /fs/brick/service/partner/create-loan`
* **Business Description:** `[Explain what the client achieves and downstream effects (e.g., triggers partner GT submission).]`
* **Prerequisites:** `[Required prior steps, such as borrower registration or idempotency key management.]`

<details>
<summary><strong>Request</strong></summary>

* **Description:** `[Clarify required fields, validation rules, idempotency expectations, and typical payload size.]`

```json
{
  "partnerType": "GT",
  "loan": {
    "loanId": "string",
    "userId": "string",
    "loanAmount": 0,
    "tenure": 0
  },
  "metadata": {
    "channel": "LOS",
    "requestId": "uuid"
  }
}
```

| Field | Type | Required | Description | Constraints |
| --- | --- | --- | --- | --- |
| `partnerType` | `string` | Yes | `[Enum: GT, KW, ...]` | `[Max length, case sensitivity]` |
| `loan.loanId` | `string` | Yes | `[Internal loan reference]` | `[Pattern or format]` |
| `loan.loanAmount` | `number` | Yes | `[Amount in IDR]` | `[Min, max, decimals]` |
| `metadata.requestId` | `string` | Recommended | `[Idempotency key]` | `[UUID format]` |

</details>

<details>
<summary><strong>Response</strong></summary>

* **Description:** `[Summarize success behavior, async follow-ups, and key identifiers returned.]`

```json
{
  "code": 200,
  "message": "Proposal submitted",
  "partnerOrderId": "GT-123456",
  "errors": []
}
```

| Field | Type | Description |
| --- | --- | --- |
| `code` | `integer` | `[Matches HTTP status]` |
| `message` | `string` | `[Human-readable outcome]` |
| `partnerOrderId` | `string` | `[Identifier used in partner callbacks/status checks]` |
| `errors` | `array<object>` | `[Optional validation or partner-side warnings]` |

</details>

> Repeat the **Endpoint Details** section for each API route, replacing examples and tables accordingly.

---

## Error Handling & Response Codes

### Standard Error Body

```json
{
  "code": 401,
  "message": "Unauthorized",
  "details": {
    "reason": "MISSING_TOKEN",
    "hint": "Provide X-Partner-Token header"
  },
  "traceId": "7f4c1e0b4c224a63"
}
```

| Field | Type | Description |
| --- | --- | --- |
| `code` | `integer` | `[HTTP status mirrored in body]` |
| `message` | `string` | `[Short summary for humans]` |
| `details` | `object` | `[Structured diagnostics partners can parse]` |
| `traceId` | `string` | `[Use when contacting support; aligns with logs/traces]` |

### Error Catalogue

| HTTP Code | Error Code | Scenario | Client Guidance |
| --- | --- | --- | --- |
| `400` | `INVALID_PAYLOAD` | `[Failed validation rule]` | `[Correct input and retry]` |
| `401` | `MISSING_TOKEN` | `[No Authorization header]` | `[Obtain/refresh token]` |
| `403` | `FORBIDDEN_SCOPE` | `[Token lacks role/scope]` | `[Request new scope]` |
| `404` | `NOT_FOUND` | `[Unknown loanId or resource]` | `[Verify identifiers]` |
| `409` | `STATE_CONFLICT` | `[Duplicate submission detected]` | `[Use latest status or new idempotency key]` |
| `429` | `RATE_LIMIT` | `[Quota exceeded]` | `[Backoff for Retry-After interval]` |
| `500` | `INTERNAL_ERROR` | `[Unexpected failure]` | `[Retry with exponential backoff; include traceId when escalating]` |

---

## Change Log *(Optional)*

| Version | Date | Change Summary | Notes |
| --- | --- | --- | --- |
| `v1.0.0` | `[YYYY-MM-DD]` | `[Initial release of partner endpoints]` | `[Backward compatible]` |
| `v1.1.0` | `[YYYY-MM-DD]` | `[Added bulk status endpoint]` | `[New scope required]` |

if anything the user provide is unclear, ask for clarification.
put the final apidocs to documentation/apidocs/[service-name]-api-docs.md