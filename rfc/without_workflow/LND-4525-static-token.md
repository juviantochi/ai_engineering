# RFC – Veefin Borrower Data Inquiry API

| Status | Draft |
| --- | --- |
| **RFC #** | LND-VEF-001 |
| **Author(s)** | Lending Backend Team |
| **Sponsor** | Lending Product |
| **Updated** | 04-Nov-2025 |
| **Jira Ticket** | [LND-0000](https://bukuwarung.atlassian.net/browse/LND-0000) |
| **Repositories Impacted** | `fs-brick-service`, `bnpl` |

---

## Objective

**Expected Outcome**: Deliver a resilient borrower inquiry capability for Veefin via FS Brick Service that returns pre-derived credit metrics (facility count, worst days past due, outstanding balance) within contractual SLA.

**Success Criteria**:
- Average p95 response time ≤ 1.0 s for `/veefin/v1/inquiry-borrowers`.
- ≥ 99.5% success rate (HTTP 2xx) for valid requests.
- Idempotent responses for identical `{requestId, ktpNumber}` within 24 hours.
- No direct dependency on live S3 reads during request handling.

---

## Security Notice

- All secrets (e.g., Veefin static token, LOS auth token, database credentials) are referenced as placeholders such as `${VEEFIN_TOKEN}` and sourced from AWS Secrets Manager.
- Tokens are injected via environment variables in deployment pipelines; never store or log real credentials in repositories or documents.
- Ensure API logs exclude sensitive identifiers beyond hashed/truncated values where feasible.

---

## Motivation

### Why Are We Doing This?
- Enable Veefin to generate merchant credit limits using their custom BRE without any manual intervention from BukuWarung.

### What Use Cases Does It Support?
1. **Veefin limit generation** – Veefin requests borrower metrics as part of their automated limit-calculation workflow.
2. *(Additional use cases TBD – confirm with stakeholders.)*

### Expected Outcome
- **Veefin** can compute merchant limits automatically via BRE inputs from borrower inquiry data.
- *(Additional outcomes TBD – confirm with stakeholders.)*

---

## User Benefit

### For Borrowers
- Accelerated loan decisioning through partner channels.
- Reduced friction with consistent credit evaluation data.

### For Veefin
- Single endpoint to fetch credit metrics with strong idempotency guarantees.
- Predictable performance not tied to S3 availability.

### For BukuWarung / Gandeng Tangan
- Operational efficiency via automated partner integration.
- Better observability over borrower data usage and partner demand.

---

## Design Proposal

### High-Level Architecture

```
Veefin Platform
    │  HTTPS (x-brick-token)
    ▼
FS Brick Service
  ├─ VeefinBorrowerInquiryController
  ├─ BorrowerInquiryService
  │    └─ FsBnplPort.getBorrowerCreditSummary()
  │
  ▼  Mutual TLS + los-auth-token
BNPL Service
  ├─ VeefinCreditSummaryController (internal)
  ├─ CreditSummaryService
  │    ├─ MerchantDataRepository (merchant_data)
  │    ├─ MerchantCreditSummaryRepository (merchant_credit_summary)
  │    └─ CreditSummaryMapper
  └─ Credit summaries are pre-populated during enrichment callback handling; request path never touches S3.
```

**Out of scope**: Processes that derive and persist `merchant_credit_summary` after FDC/PEFINDO enrichment complete (triggered before Veefin lead creation).

### Component Details

#### 1. **VeefinBorrowerInquiryController** (FS Brick)
- **Purpose**: Expose `POST /fs/brick/service/veefin/v1/inquiry-borrowers`.
- **Responsibilities**:
  - Validate request payloads (UUID requestId, 16-digit KTP).
  - Authenticate requests using `FsBrickTokenService`.
  - Apply partner idempotency rules via the existing audit mechanism.
  - Translate domain errors into the API response contract.
- **Location**: `fs-brick-service/src/main/java/.../controller/veefin`.

#### 2. **BorrowerInquiryService** (FS Brick)
- **Purpose**: Orchestrate borrower inquiry handling.
- **Responsibilities**:
  - Normalize input after controller checks pass.
  - Call BNPL through `FsBnplPort`.
  - Map BNPL DTO to API response.
- **Observability**: Structured logs.

#### 3. **FsBnplPort / FsBnplAdapter** (FS Brick)
- **Enhancement**: Add `BorrowerCreditSummaryDto getBorrowerCreditSummary(String ktpNumber)`
- **Security**: Establish mutual TLS to BNPL and send LOS auth token header (`los-auth-token: ${BNPL_TOKEN}`).

#### 4. **VeefinCreditSummaryController** (BNPL)
- **Purpose**: Internal REST endpoint (`/v1/internal/veefin/borrower-summary`).
- **Responsibilities**:
  - Enforce LOS authentication.
  - Delegate to `CreditSummaryService`.
  - Convert domain errors to HTTP codes (404 not found, 410 stale, 500 failure).

#### 5. **CreditSummaryService** (BNPL)
- **Purpose**: Fetch borrower credit summary.
- **Responsibilities**:
  - Resolve merchant by KTP via `MerchantDataRepository`.
  - Fetch the latest entry from `merchant_credit_summary` return DTO.
- **Assumption**: Enrichment pipeline has already populated the summary in the on boarding process; service never reads S3 synchronously.

#### 6. **MerchantCreditSummaryRepository / Entity** (BNPL)
- **Schema (per enrichment batch)**:
  - `id` UUID primary key (generated inside BNPL)
  - `merchant_enrichment_data_id` UUID (FK → `merchant_enrichment_data`, unique)
  - `merchant_id` UUID (FK → `merchant_data`)
  - `batch_id` VARCHAR(60) and `enrichment_type` VARCHAR(15) with unique constraint on the pair
  - `ktp_number` VARCHAR(50) (indexed for lookup)
  - `source_provider` VARCHAR(15), `provider_status` VARCHAR(32), `reference_id` VARCHAR(128)
  - `data_timestamp` TIMESTAMP WITH TIME ZONE
  - `total_facilities` INT, `worst_overdue_days` INT, `total_outstanding_balance` NUMERIC(20,2)
  - Audit fields: `created_at` (timestamp default now), `created_by` (default `MERCHANT_ONBOARDING`)
- **Purpose**: Persist enrichment-derived borrower summaries so BNPL can respond without re-reading S3.

#### 7. **CreditSummaryMapper** (BNPL)
- **Purpose**: Map DB entity to DTO consumed by FS Brick.

#### 8. **Observability**
- **Health Checks**: BNPL exposes readiness/liveness including.
- **Logging**: Include `ktpHash`, `requestId`, `batchId` (if available), no raw PII.

---

## Data Flow

1. **Inbound request**: Veefin calls `POST /fs/brick/service/veefin/v1/inquiry-borrowers` with `ktpNumber` and `requestId`; FS Brick authenticates header `x-brick-token: ${VEEFIN_TOKEN}`.
2. **Validation & idempotency**: Controller validates the payload and leverages the standard FS Brick partner event audit mechanism.
   - Matching duplicate → return the previously recorded outcome.
   - Mismatched duplicate (different KTP) → return 400.
   - New request → continue processing and log metadata.
3. **BNPL lookup**: FS Brick invokes `FsBnplPort.getBorrowerCreditSummary` over mutual TLS, including the LOS auth token.
4. **Credit summary retrieval**: `VeefinCreditSummaryController` calls `CreditSummaryService`, which:
   - Confirms merchant exists.
   - Reads `merchant_credit_summary`.
5. **Response assembly**: BNPL returns `BorrowerCreditSummaryDto`; FS Brick maps to the API contract, logs the outcome via existing audit hooks, emits metrics, and responds 200. No live S3 read occurs in this path.
6. **Error propagation**:
   - `404` from BNPL → FS Brick returns 404 (data not found).
   - Unexpected errors → FS Brick returns 500.

---

## Rollout & Migration Plan

- **Pre-requisites**:
  - BNPL schema migration to add `merchant_credit_summary` table and indexes.
  - Deployment of summary materialization pipeline (handled by separate workstream).
- **Fallback**: revert FS Brick endpoint if severe issues arise; BNPL endpoint can remain idle.
- **Data backfill**: Out of scope.

---

## Testing Strategy

- **Unit Tests**:
  - Payload validation (invalid KTP/requestId).
  - Controller-level idempotency duplicate detection and conflict handling.
  - DTO mapping for BNPL response.
- **Integration Tests**:
  - FS Brick ↔ BNPL contract tests with WireMock.
  - Database integration tests for `merchant_credit_summary` repository.
- **Performance Tests**:
  - Load test 100 RPS sustained; ensure FS Brick p95 ≤ 1 s and BNPL CPU within limits.
- **Security Tests**:
  - Verify static token enforcement.

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Summary not pre-populated before inquiry | High | Monitor enrichment callbacks; alert when `merchant_enrichment_data` remains `IN_PROGRESS` beyond SLA and trigger replay/backfill before enabling partner traffic. |
| Idempotency logging failures | Medium | Controller retries audit writes with alerting; provide reconciliation tooling for partner audit records. |

---
## Appendix

- **Sample Request**:
```json
{
  "ktpNumber": "1234567890123456",
  "requestId": "550e8400-e29b-41d4-a716-446655440000"
}
```

- **Sample Response**:
```json
{
  "ktpNumber": "1234567890123456",
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "code": 200,
  "description": "success",
  "data": {
    "jmlFasilitas": 750,
    "jmlHariTunggakanTerburuk": 12,
    "jmlSaldoTerutang": 1233.12
  }
}
```

- **Related Documents**:
  - `documentation/request/veefin-data-point/borrowers-data-inquiry-api-req.md`
  - `documentation/request/veefin-data-point/borrowers-data-inquiry-sys-req.md`
