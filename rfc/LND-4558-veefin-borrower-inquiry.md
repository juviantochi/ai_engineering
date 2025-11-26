# RFC – Veefin Borrower Data Inquiry API

| Status | **Draft** |
| --- | --- |
| **RFC #** | LND-4558 |
| **Author(s)** | @JuviantoChi |
| **Sponsor** | Product Team |
| **Updated** | 2025-11-24 |
| **Jira Ticket** | [LND-4558](https://bukuwarung.atlassian.net/browse/LND-4558) |

---

## Objective

**Expected Outcome**: Enable Veefin (external partner) to query borrower merchant enrichment data (PEFINDO credit reports, FDC credit reports, bank statements) via a secure, idempotent API endpoint that serves files from AWS S3 as multipart/form-data responses.

**Success Criteria**:
- ✅ Veefin can successfully retrieve enrichment data files by KTP number and data point type
- ✅ API returns multipart response with file content, data point type, and batch ID
- ✅ Idempotency ensures duplicate requests return fresh file content
- ✅ Performance: < 5 seconds p95 response time for 10MB files
- ✅ Error handling for all scenarios (404, 400, 500)
- ✅ Comprehensive audit trail via PartnerEventAudit
- ✅ Zero impact on existing BNPL and fs-brick-service functionality
- ✅ Production deployment successful with monitoring in place

---

## Security Notice

🔒 **All sensitive parameters in this document are masked with placeholders**.

**Real credentials must be obtained from**:
- **BNPL authentication**: `${BNPL_MERCHANT_TOKEN}` via AWS Secrets Manager or environment variable
- **S3 bucket name**: `${BNPL_S3_BUCKET_NAME}` via environment variable
- **AWS credentials**: IAM role-based (not hardcoded keys)
- **Veefin authentication**: HMAC-SHA256 signature (LND-4557 - separate ticket)

**Never commit real credentials to version control.**

---

## Motivation

### Why Are We Doing This?

**Business Problem**: Veefin, a strategic partner, requires programmatic access to borrower merchant enrichment data to perform credit assessments and risk evaluations for their lending products. Currently, this data exists in BukuWarung's systems (BNPL database and AWS S3) but lacks an external API for partner consumption.

**Market Opportunity**: Enabling Veefin's lending operations through data sharing creates:
- Partnership revenue opportunities
- Merchant ecosystem growth
- Cross-selling potential for BukuWarung financial products

**Technical Gap**:
- Data pipeline exists: Veefin → FS-BRICK-SERVICE → BNPL → AWS S3 (for enrichment requests)
- No reverse pipeline exists for data retrieval
- No external API to serve enrichment data files to partners

### What Use Cases Does It Support?

1. **Veefin Credit Assessment Workflow**:
   - Partner requests borrower data by KTP number
   - Specifies data point type (PEFINDO, FDC, or BANK_STATEMENT)
   - Receives latest enrichment file for credit scoring
   - Performs risk evaluation and lending decisions

2. **Idempotent Data Access**:
   - Duplicate requests (network retries, client errors) return fresh file content
   - Complete audit trail for compliance and troubleshooting
   - Request ID tracking for end-to-end observability

3. **Multi-Format Support**:
   - PEFINDO credit reports (JSON format)
   - FDC credit reports (JSON format)
   - Bank statements (XLSX format)

4. **Secure Partner Integration**:
   - Authentication via HMAC-SHA256 signatures (LND-4557)
   - Authorized data access only
   - PII handling compliance

### Expected Outcome

**For Veefin**:
- Reliable, fast API to access borrower enrichment data
- Multipart response format with file + metadata
- Idempotent operations for safe retries
- Clear error messages for troubleshooting

**For BukuWarung**:
- Secure, auditable partner data distribution
- Revenue opportunity through partnership
- Reusable pattern for future partner integrations
- Maintained data governance and compliance

**For Engineering**:
- Clean separation of concerns (BNPL owns data, fs-brick-service serves to partners)
- Scalable architecture for additional partners
- Comprehensive observability and monitoring

---

## User Benefit

### For Veefin (External Partner)
- **Fast Credit Decisions**: Instant access to borrower enrichment data reduces turnaround time
- **Reliable API**: Idempotency and error handling ensure robust integration
- **Complete Data**: Multipart response includes file + metadata (type, batch ID)
- **Multiple Data Sources**: Support for PEFINDO, FDC, and bank statements in single API

### For BukuWarung Product Team
- **Partnership Enablement**: Technical foundation for Veefin partnership revenue
- **Scalable Platform**: Architecture supports additional partners with minimal changes
- **Data Monetization**: Existing enrichment data becomes shareable asset
- **Compliance Ready**: Audit trail and access controls built-in

### For BukuWarung Engineering
- **Reusable Patterns**: PartnerEventAudit pattern, multipart response builder
- **Clean Architecture**: Clear service boundaries, minimal coupling
- **Operational Excellence**: Comprehensive logging, monitoring, error handling
- **Backward Compatibility**: Zero impact on existing functionality

---

## Design Proposal

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Veefin (External Partner)                        │
│  - Credit assessment system                                          │
│  - Risk evaluation engine                                            │
└───────────────────────┬─────────────────────────────────────────────┘
                        │
                        │ POST /fs/brick/service/borrower/v1/inquiry-data
                        │ Headers: veefin-request-id, veefin-signature
                        │ Body: { ktpNumber, dataPoint }
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        FS-BRICK-SERVICE                              │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ VeefinSignatureAuthenticationFilter (LND-4557 - Future)       │ │
│  │  - HMAC-SHA256 signature validation                           │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              ▼                                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ VeefinBorrowerController                                       │ │
│  │  - Feature flag check (FeatureFlagConfiguration)               │ │
│  │  - Request validation (@Valid)                                 │ │
│  │  - Idempotency check (PartnerEventAudit)                       │ │
│  └───────────────────┬────────────────────────────────────────────┘ │
│                      ▼                                               │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ VeefinBorrowerServiceImpl                                      │ │
│  │  - Orchestrate: BNPL query → S3 download → Multipart response │ │
│  └──────────┬───────────────────────────────────┬─────────────────┘ │
│             │                                   │                    │
│             ▼                                   ▼                    │
│  ┌──────────────────────┐         ┌──────────────────────────────┐ │
│  │ FsBnplAdapter        │         │ S3FileService                │ │
│  │  - HTTP POST         │         │  - Stream from S3            │ │
│  │  - los-auth-token    │         │  - Bucket: bnplS3BucketName  │ │
│  └──────────┬───────────┘         └──────────────────────────────┘ │
└─────────────┼──────────────────────────────────────────────────────┘
              │
              │ HTTP POST /merchant/enrichment-data/inquiry
              │ Header: los-auth-token
              │ Body: { ktpNumber, dataPoint }
              │
              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         BNPL Service                                 │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ MerchantEnrichmentController                                   │ │
│  │  - Validate los-auth-token                                     │ │
│  │  - Request validation (@Valid)                                 │ │
│  └───────────────────┬────────────────────────────────────────────┘ │
│                      ▼                                               │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ MerchantEnrichmentService                                      │ │
│  │  - Query repository: latest record by KTP + dataPoint          │ │
│  │  - Validate status (COMPLETE only, reject IN_PROGRESS/FAILED)   │ │
│  └───────────────────┬────────────────────────────────────────────┘ │
│                      ▼                                               │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ MerchantEnrichmentDataRepository                               │ │
│  │  - JOIN merchant_data ON merchant_id                           │ │
│  │  - WHERE ktp_number AND type                                   │ │
│  │  - ORDER BY created_at DESC LIMIT 1                            │ │
│  └────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        PostgreSQL Database                           │
│  merchant_data:                                                      │
│    - id, ktp_number, name, created_at                                │
│                                                                      │
│  merchant_enrichment_data:                                           │
│    - id, merchant_id, s3_path, type, batch_id, status, created_at   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                            AWS S3                                    │
│  Bucket: ${BNPL_S3_BUCKET_NAME}                                      │
│  Files: Path stored in merchant_enrichment_data.s3_path             │
│    - No assumed structure, use s3_path from database as-is          │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Flow

#### Successful Request Flow

```
1. Veefin → FS-BRICK
   POST /fs/brick/service/borrower/v1/inquiry-data
   Headers: veefin-request-id: req-123, veefin-signature: hmac-sha256-sig
   Body: { "ktpNumber": "1234567890123456", "dataPoint": "PEFINDO" }

2. FS-BRICK: Feature Flag Check
   - Check featureFlagConfig.isEnabled("veefin-borrower-inquiry", ktpNumber)
   - If disabled: Return 503 Service Unavailable
   - If enabled: Continue to next step

3. FS-BRICK: Authentication (LND-4557 - Future)
   - Validate HMAC-SHA256 signature
   - Verify timestamp freshness

4. FS-BRICK: Start Service Event Audit (PEA Step 1)
   - Call eventAuditUtil.startServiceEventAudit(requestId, "veefin", "borrower-inquiry", requestPayload)
   - Method handles idempotency internally (checks if audit exists)
   - If status = SUCCESS: Returns existing audit entity with metadata (duplicate request)
   - If not exists: Creates new audit entity with status = IN_PROGRESS
   - Note: Use existing EventAuditUtil pattern, do not create custom implementation

5. FS-BRICK: Update Partner Level Audit - IN_PROGRESS (PEA Step 2)
   - Call eventAuditUtil.updatePartnerLevelEventAudit(auditEntity.getId(), "IN_PROGRESS", bnplRequestJson, null)
   - Tracks that BNPL call is starting
   - Stores BNPL request payload in partnerMessageDetail

6. FS-BRICK → BNPL
   POST /merchant/enrichment-data/inquiry
   Header: los-auth-token: ${BNPL_MERCHANT_TOKEN}
   Body: { "ktpNumber": "1234567890123456", "dataPoint": "PEFINDO" }
   Note: POST used to avoid PII (KTP) in URL logs

7. BNPL: Query Database
   SELECT med.s3_path, med.batch_id, med.type, med.status, med.created_at
   FROM merchant_enrichment_data med
   JOIN merchant_data md ON med.merchant_id = md.id
   WHERE md.ktp_number = '1234567890123456'
     AND med.type = 'PEFINDO'
   ORDER BY med.created_at DESC
   LIMIT 1

8. BNPL → FS-BRICK
   200 OK
   Body: {
     "s3Path": "example/path/from/database.json",  // Actual path from merchant_enrichment_data.s3_path
     "batchId": "batch-456",
     "type": "PEFINDO",
     "status": "COMPLETE"
   }

9. FS-BRICK: Update Partner Level Audit - SUCCESS (PEA Step 3)
   - Call eventAuditUtil.updatePartnerLevelEventAudit(auditEntity.getId(), "SUCCESS", bnplRequestJson, bnplResponseJson)
   - Tracks successful BNPL call completion
   - Stores BNPL response in partnerMessageDetail

10. FS-BRICK: Download from S3
   - S3 GetObject: bucket=${BNPL_S3_BUCKET_NAME}, key=<s3Path from BNPL response>
   - Stream InputStream (no memory load)
   - Note: s3Path is used exactly as returned by BNPL, no assumptions about structure

11. FS-BRICK: Build Multipart Response
    Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

    ------WebKitFormBoundary
    Content-Disposition: form-data; name="dataFile"; filename="data.json"
    Content-Type: application/octet-stream

    { ... file content stream ... }
    ------WebKitFormBoundary
    Content-Disposition: form-data; name="dataPoint"
    Content-Type: text/plain

    PEFINDO
    ------WebKitFormBoundary
    Content-Disposition: form-data; name="dataId"
    Content-Type: text/plain

    batch-456
    ------WebKitFormBoundary--

12. FS-BRICK: Success Service Event Audit (PEA Step 4)
    - Build s3MetadataJson containing:
      * s3Path: "example/path/from/database.json"
      * dataId: "batch-456"
      * dataPoint: "PEFINDO"
      * retrievalDate: "2024-11-24T10:30:00Z"
    - Call eventAuditUtil.successServiceEventAudit(auditEntity.getId(), s3MetadataJson)
    - Marks service-level status as SUCCESS
    - Stores S3 metadata in messageDetail field for idempotency
    - No new PartnerEventAudit entity is created; this updates the record from Step 4
    - Final audit record:
      * requestId: "req-123"
      * partnerName: "veefin"
      * serviceName: "borrower-inquiry"
      * status: SUCCESS
      * requestPayload: { ktpNumber, dataPoint }
      * messageDetail: { s3Path, dataId, dataPoint, retrievalDate }
      * partnerMessageDetail: { s3Path, batchId, type, status }

13. FS-BRICK → Veefin
    200 OK
    Content-Type: multipart/form-data
    Body: [multipart content]
```

#### Error Scenarios

**Scenario 1: No Enrichment Data Found (404)**

```java
public ResponseEntity<?> processInquiry(BorrowerInquiryRequest request, String requestId) {
    // 1. PEA Step 1 - Start service audit (idempotency check)
    PartnerEventAuditEntity audit = eventAuditUtil.startServiceEventAudit(...);

    // 2. PEA Step 2 - Update partner audit IN_PROGRESS
    eventAuditUtil.updatePartnerLevelEventAudit(..., IN_PROGRESS, bnplRequestJson, null);

    try {
        // 3. Call BNPL - throws HttpClientErrorException.NotFound (404)
        EnrichmentDataResponse response = bnplPort.getEnrichmentDataByKtp(...);

    } catch (HttpClientErrorException.NotFound e) {
        // 4. PEA Step 3 - Update partner audit FAILED (404 from BNPL)
        eventAuditUtil.updatePartnerLevelEventAudit(..., FAILED, bnplRequestJson, e.getResponseBodyAsString());

        // 5. PEA Step 4 - Mark service as FAILED
        eventAuditUtil.failedServiceEventAudit(audit.getId(), "No enrichment data found for KTP");

        // 6. Return 404 to Veefin
        // PEA Final State: status=FAILED, partnerStatus=FAILED
        // messageDetail: "No enrichment data found for KTP"
        // partnerMessageDetail: BNPL 404 error response
        return ResponseEntity.status(404).body(errorResponse);
    }
}
```

**Scenario 2: Data Not Ready (IN_PROGRESS/FAILED) - 400**

```java
public ResponseEntity<?> processInquiry(BorrowerInquiryRequest request, String requestId) {
    // 1. PEA Step 1 - Start service audit (idempotency check)
    PartnerEventAuditEntity audit = eventAuditUtil.startServiceEventAudit(...);

    // 2. PEA Step 2 - Update partner audit IN_PROGRESS
    eventAuditUtil.updatePartnerLevelEventAudit(..., IN_PROGRESS, bnplRequestJson, null);

    try {
        // 3. Call BNPL - throws HttpClientErrorException.BadRequest (400)
        // BNPL returns 400 when record found but status != COMPLETE
        EnrichmentDataResponse response = bnplPort.getEnrichmentDataByKtp(...);

    } catch (HttpClientErrorException.BadRequest e) {
        // 4. PEA Step 3 - Update partner audit SUCCESS (HTTP call succeeded)
        // Critical: partnerStatus=SUCCESS because HTTP call succeeded (200/400 both valid HTTP responses)
        eventAuditUtil.updatePartnerLevelEventAudit(..., SUCCESS, bnplRequestJson, e.getResponseBodyAsString());

        // 5. PEA Step 4 - Mark service as FAILED (business requirement cannot be fulfilled)
        // status=FAILED because we cannot serve the file (data not ready)
        eventAuditUtil.failedServiceEventAudit(audit.getId(), "Data not ready: IN_PROGRESS");

        // 6. Return 400 to Veefin
        // PEA Final State: status=FAILED (cannot serve file), partnerStatus=SUCCESS (HTTP call succeeded)
        // messageDetail: "Data not ready: {status}"
        // partnerMessageDetail: BNPL 400 response with status field
        return ResponseEntity.status(400).body(errorResponse);
    }
}
```

**Scenario 3: S3 File Not Found (500)**

```java
public ResponseEntity<?> processInquiry(BorrowerInquiryRequest request, String requestId) {
    // 1-3. PEA Steps 1-3 completed successfully (BNPL returned 200 OK)
    EnrichmentDataResponse bnplResponse = bnplPort.getEnrichmentDataByKtp(...);
    eventAuditUtil.updatePartnerLevelEventAudit(..., SUCCESS, bnplRequestJson, bnplResponseJson);

    try {
        // 4. Download from S3 using s3Path from BNPL response
        InputStream fileStream = s3FileService.downloadFile(bnplResponse.getS3Path());

    } catch (AmazonS3Exception e) {
        // 5. S3 throws NoSuchKey exception - file not found
        // PEA Step 4 - Mark service as FAILED (cannot serve file)
        String errorDetail = String.format("S3 file not found: path=%s, error=%s",
                                           bnplResponse.getS3Path(), e.getErrorCode());
        eventAuditUtil.failedServiceEventAudit(audit.getId(), errorDetail);

        // 6. Return 500 to Veefin
        // PEA Final State: status=FAILED, partnerStatus=SUCCESS
        // messageDetail: S3 error details (path, errorType)
        // partnerMessageDetail: BNPL 200 response
        // Critical: Reveals data inconsistency - requires investigation
        return ResponseEntity.status(500).body(errorResponse);
    }
}
```

**Scenario 4: BNPL Timeout/Network Error (500)**

```java
public ResponseEntity<?> processInquiry(BorrowerInquiryRequest request, String requestId) {
    // 1. PEA Step 1 - Start service audit (idempotency check)
    PartnerEventAuditEntity audit = eventAuditUtil.startServiceEventAudit(...);

    // 2. PEA Step 2 - Update partner audit IN_PROGRESS
    eventAuditUtil.updatePartnerLevelEventAudit(..., IN_PROGRESS, bnplRequestJson, null);

    try {
        // 3. Call BNPL - throws RestClientException (timeout, connection error)
        EnrichmentDataResponse response = bnplPort.getEnrichmentDataByKtp(...);

    } catch (RestClientException e) {
        // 4. PEA Step 3 - Update partner audit FAILED (network/timeout error)
        String errorDetail = String.format("BNPL call failed: %s", e.getMessage());
        eventAuditUtil.updatePartnerLevelEventAudit(..., FAILED, bnplRequestJson, errorDetail);

        // 5. PEA Step 4 - Mark service as FAILED
        eventAuditUtil.failedServiceEventAudit(audit.getId(), "BNPL service unavailable");

        // 6. Return 500 to Veefin
        // PEA Final State: status=FAILED, partnerStatus=FAILED
        // messageDetail: "BNPL service unavailable"
        // partnerMessageDetail: Error details (timeout, connection error)
        return ResponseEntity.status(500).body(errorResponse);
    }
}
```

**Scenario 5: Duplicate Request (Idempotency)**

```java
public ResponseEntity<?> processInquiry(BorrowerInquiryRequest request, String requestId) {
    // 1. PEA Step 1 - Start service audit (idempotency check)
    PartnerEventAuditEntity audit = eventAuditUtil.startServiceEventAudit(
        requestId, "veefin", "borrower-inquiry", requestPayloadJson
    );

    // 2. Check if duplicate request (null-safe constant-first pattern)
    if ("SUCCESS".equals(audit.getStatus())) {
        log.info("Duplicate request detected. RequestId: {}", requestId);

        // 3. Extract S3 metadata from stored audit record
        // messageDetail contains: { "s3Path": "...", "dataId": "..." }
        String messageDetail = audit.getMessageDetail();
        String s3Path = extractS3Path(messageDetail);
        String batchId = extractBatchId(messageDetail);
        String dataPoint = request.getDataPoint();

        // 4. Refetch fresh S3 file (S3 is source of truth, not cached)
        // File content may have been updated since first request
        InputStream fileStream = s3FileService.downloadFile(s3Path);

        // 5. Build multipart response with fresh S3 content
        MultiValueMap<String, Object> body = buildMultipartResponse(
            fileStream, dataPoint, batchId
        );

        // 6. Return response (no PEA updates - use existing audit record)
        // Why refetch? S3 is source of truth, file may have been updated
        return ResponseEntity.ok()
            .contentType(MediaType.MULTIPART_FORM_DATA)
            .body(body);
    }

    // 7. Not duplicate - continue with normal flow
    // ... (proceed with BNPL call, S3 download, etc.)
}
```

---

## API Specification

### Endpoint

**URL**: `POST /fs/brick/service/borrower/v1/inquiry-data`

**Headers**:

| Header                | Required      | Description                                  | Example               |
|-----------------------|---------------|----------------------------------------------|-----------------------|
| `veefin-request-id`   | Yes           | Unique request identifier for idempotency    | `req-20241124-001`    |
| `veefin-signature`    | Yes (Future)  | HMAC-SHA256 signature (LND-4557)             | `sha256=abc123...`    |
| `veefin-timestamp`    | Yes (Future)  | Request timestamp (LND-4557)                 | `1700000000`          |
| `Content-Type`        | Yes           | Request content type                         | `application/json`    |

**Request Body**:
```json
{
  "ktpNumber": "1234567890123456",
  "dataPoint": "PEFINDO"
}
```

**Field Validation**:
- `ktpNumber`: String, exactly 16 digits, numeric only, required
- `dataPoint`: Enum (`PEFINDO`, `FDC`, `BANK_STATEMENT`), required

**Response (Success - 200 OK)**:
```
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="dataFile"; filename="data.json"
Content-Type: application/octet-stream

{ ... PEFINDO credit report JSON ... }
------WebKitFormBoundary
Content-Disposition: form-data; name="dataPoint"
Content-Type: text/plain

PEFINDO
------WebKitFormBoundary
Content-Disposition: form-data; name="dataId"
Content-Type: text/plain

batch-456
------WebKitFormBoundary--
```

**Response Parts**:
1. **dataFile**: Binary file content (application/octet-stream)
   - PEFINDO/FDC: JSON format
   - BANK_STATEMENT: XLSX format
2. **dataPoint**: Text field with data type (text/plain)
3. **dataId**: Text field with batch identifier (text/plain)

**Error Responses**:

| Status | Scenario              | Response Body                                                                                      |
|--------|-----------------------|----------------------------------------------------------------------------------------------------|
| 400    | Invalid KTP format    | `{ "code": 400, "message": "KTP must be exactly 16 digits" }`                                     |
| 400    | Invalid dataPoint     | `{ "code": 400, "message": "Data point is required" }`                                            |
| 400    | Data not ready        | `{ "code": 400, "message": "Enrichment data is not ready. Status: IN_PROGRESS" }`                |
| 401    | Auth failure (Future) | `{ "code": 401, "message": "Authentication failed" }`                                             |
| 404    | No data found         | `{ "code": 404, "message": "No enrichment data found for the provided KTP and data point" }`     |
| 500    | S3 retrieval failure  | `{ "code": 500, "message": "Failed to retrieve enrichment data file" }`                           |
| 500    | Internal error        | `{ "code": 500, "message": "Internal server error" }`                                             |
| 503    | Service unavailable   | `{ "code": 503, "message": "Veefin borrower inquiry service is currently unavailable" }`          |

---

## Component Breakdown

### fs-brick-service Components

| Component                    | Type       | Purpose                     | Status   |
|------------------------------|------------|-----------------------------|----------|
| `VeefinBorrowerController`   | Controller | External API endpoint       | New      |
| `VeefinBorrowerServiceImpl`  | Service    | Orchestration logic         | New      |
| `BorrowerInquiryRequest`     | DTO        | Request validation          | New      |
| `FsBnplPort` (enhanced)      | Interface  | BNPL integration contract   | Enhanced |
| `FsBnplAdapter` (enhanced)   | Adapter    | BNPL HTTP client            | Enhanced |
| `EnrichmentDataResponse`     | DTO        | BNPL response               | New      |
| `S3FileService`              | Service    | S3 file streaming (wrapper around AWS SDK AmazonS3) | EXISTING |
| `PartnerEventAuditEntity`    | Entity     | Idempotency tracking        | Existing |
| `EventAuditUtil`             | Utility    | Audit operations            | Existing |

### bnpl Components

| Component                                       | Type       | Purpose               | Status   |
|-------------------------------------------------|------------|-----------------------|----------|
| `MerchantEnrichmentController`                  | Controller | Internal API endpoint | New      |
| `MerchantEnrichmentService`                     | Service    | Business logic        | New      |
| `EnrichmentDataRequest`                         | DTO        | Request validation    | New      |
| `EnrichmentDataResponse`                        | DTO        | Response payload      | New      |
| `MerchantEnrichmentDataRepository` (enhanced)   | Repository | Database query        | Enhanced |
| `MerchantEnrichmentDataEntity`                  | Entity     | Domain model          | Existing |
| `MerchantDataEntity`                            | Entity     | Domain model          | Existing |

---

## Configuration

### Environment Variables

**fs-brick-service**:
```bash
# BNPL Integration (EXISTING - already configured)
BNPL_MERCHANT_BASE_URL=https://bnpl.internal.bukuwarung.com  # EXISTING
BNPL_MERCHANT_TOKEN=${secret:los-auth-token}                 # EXISTING

# S3 Configuration (EXISTING - already configured)
BNPL_S3_BUCKET_NAME=bw-merchant-enrichment-prod             # EXISTING
AWS_REGION=ap-southeast-1                                    # EXISTING
AWS_ACCESS_KEY_ID=${secret:aws-access-key}                   # EXISTING
AWS_SECRET_KEY=${secret:aws-secret-key}                      # EXISTING

# Feature Toggle (NEW - to be added)
VEEFIN_BORROWER_INQUIRY_ENABLED=true                         # NEW
```

**bnpl**:
```bash
# Authentication (EXISTING - already configured)
LOS_AUTH_TOKEN=${secret:los-auth-token}  # EXISTING - already used for internal authentication
```

**Note**: Most configuration already exists. Only `VEEFIN_BORROWER_INQUIRY_ENABLED` needs to be added.

### application.yml Updates

**fs-brick-service**:
```yaml
# NEW - Feature toggle for Veefin borrower inquiry
veefin:
  borrower-inquiry:
    enabled: ${VEEFIN_BORROWER_INQUIRY_ENABLED:true}  # NEW - Controls service availability
    # When false: Returns 503 Service Unavailable
    # When true: Processes inquiry requests normally
    # Implementation: Checked at controller entry point (see FS-BRICK RFC)

# EXISTING - BNPL integration (no changes needed)
fs:
  bnpl:
    merchant:
      baseUrl: ${BNPL_MERCHANT_BASE_URL}  # EXISTING (already configured)
    token: ${BNPL_MERCHANT_TOKEN}         # EXISTING (already configured)

# EXISTING - S3 configuration (no changes needed)
gt:
  bnplS3BucketName: ${BNPL_S3_BUCKET_NAME:janus-buku-dev}  # EXISTING

# EXISTING - AWS configuration (no changes needed)
aws:
  accessKeyId: ${AWS_ACCESS_KEY_ID}  # EXISTING
  secretKey: ${AWS_SECRET_KEY}       # EXISTING
  region: ${AWS_REGION}              # EXISTING
```

**bnpl**:
```yaml
# EXISTING - Authentication already configured
los-auth-token: ${LOS_AUTH_TOKEN:test}  # EXISTING - already present
```

**Summary of Changes**:
- ✅ **Reuses existing configuration**: `fs.bnpl`, `gt.bnplS3BucketName`, `aws`, `LOS_AUTH_TOKEN` (no changes)
- ➕ **New configuration**: `veefin.borrower-inquiry.enabled` only

**Token Naming Note**: `BNPL_MERCHANT_TOKEN` (fs-brick-service) and `LOS_AUTH_TOKEN` (bnpl) reference the same los-auth-token secret. Deployment scripts must map the shared secret to both environment variables so fs-brick-service can forward the header BNPL expects.

---

## Idempotency Strategy

### Idempotency Flow

**Using Existing EventAuditUtil Pattern**:

**First Request**:
1. Extract `veefin-request-id` header as requestId
2. Call `eventAuditUtil.startServiceEventAudit(requestId, "veefin", "borrower-inquiry", requestPayload)`
3. Method checks if audit exists:
   - Not found → Creates new audit entity, returns it
4. Process request normally
5. Update audit entity with response metadata:
   ```json
   {
     "s3Path": "example/path/from/database.json",
     "dataId": "batch-456"
   }
   ```
   Note: s3Path is exact value from merchant_enrichment_data.s3_path column

**Duplicate Request**:
1. Extract `veefin-request-id` header as requestId
2. Call `eventAuditUtil.startServiceEventAudit(requestId, "veefin", "borrower-inquiry", requestPayload)`
   - Method finds existing audit entity with status = SUCCESS
   - Returns existing entity immediately (idempotency handled internally)
3. Check `"SUCCESS".equals(auditEntity.getStatus())` → Duplicate detected (null-safe constant-first pattern)
4. Extract metadata from audit entity:
   - Parse `messageDetail` field (contains S3 metadata JSON from Step 4)
   - Parse `partnerMessageDetail` field (contains BNPL response JSON from Step 3)
5. Extract s3Path and batchId from parsed metadata
6. Refetch S3 file using s3Path (fresh content, not cached)
7. Build multipart response with fresh S3 content
8. Return response (no update to existing audit record)

**IMPORTANT**: Use existing `EventAuditUtil` methods following the 4-step pattern. Do NOT create custom idempotency implementation.

### Complete PEA Flow (4 Steps - MANDATORY)

**Step 1: startServiceEventAudit**
```java
PartnerEventAuditEntity auditEntity = eventAuditUtil.startServiceEventAudit(
    requestId,           // from veefin-request-id header
    "veefin",           // partner name
    "borrower-inquiry", // service name
    requestPayloadJson  // request body as JSON string
);

// Idempotency check - CRITICAL (null-safe constant-first pattern)
if ("SUCCESS".equals(auditEntity.getStatus())) {
    log.info("Duplicate request detected. RequestId: {}", requestId);
    return processInquiryFromAudit(auditEntity);
}
```
- Creates audit entity with status = IN_PROGRESS
- If existing entity with SUCCESS → returns it immediately (idempotency)
- If existing entity with FAILED → allows retry with incremented retryCount
- If existing entity with IN_PROGRESS → throws IllegalStateException

**Step 2: updatePartnerLevelEventAudit IN_PROGRESS with request**
```java
String bnplRequestJson = objectMapper.writeValueAsString(bnplRequest);
eventAuditUtil.updatePartnerLevelEventAudit(
    "borrower-inquiry",            // service name
    "veefin",                      // partner name
    requestId,                     // request ID
    EventAuditLevelStatus.IN_PROGRESS,
    bnplRequestJson,               // BNPL request payload
    null                           // No response yet
);
```
- Tracks that BNPL call is starting
- Stores BNPL request payload in `partnerRequestPayload` for debugging
- Sets `partnerStatus` = IN_PROGRESS

**Step 3: updatePartnerLevelEventAudit SUCCESS/FAILED with response**
```java
// On Success:
String bnplResponseJson = objectMapper.writeValueAsString(bnplResponse);
eventAuditUtil.updatePartnerLevelEventAudit(
    "borrower-inquiry",
    "veefin",
    requestId,
    EventAuditLevelStatus.SUCCESS,
    bnplRequestJson,
    bnplResponseJson  // BNPL response with s3Path and batchId
);

// On Failure:
eventAuditUtil.updatePartnerLevelEventAudit(
    "borrower-inquiry",
    "veefin",
    requestId,
    EventAuditLevelStatus.FAILED,
    bnplRequestJson,
    e.getMessage()    // Error message
);
```
- Tracks BNPL call result
- Stores response or error in `partnerMessageDetail`
- Sets `partnerStatus` = SUCCESS or FAILED

**Step 4: successServiceEventAudit / failedServiceEventAudit**
```java
// On Success:
String s3Metadata = String.format(
    "{\"s3Path\":\"%s\",\"batchId\":\"%s\"}",
    bnplResponse.getS3Path(),
    bnplResponse.getBatchId()
);
eventAuditUtil.successServiceEventAudit(auditEntity.getId(), s3Metadata);

// On Failure:
eventAuditUtil.failedServiceEventAudit(auditEntity.getId(), e.getMessage());
```
- Marks service-level status as SUCCESS or FAILED
- Stores S3 metadata in `messageDetail` for idempotency retrieval
- Final state: `status` = SUCCESS/FAILED, `partnerStatus` = SUCCESS/FAILED

**Pattern Reference**: See `GetFdcDataCommand.java` (lines 30-72) for existing implementation example.

**Why Refetch S3?**
- File content may have been updated
- Ensures latest data is returned
- S3 is source of truth, not audit metadata

---

## Performance & Scalability

### Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| BNPL query latency | < 500ms | p95 |
| S3 download time | Variable (depends on file size) | Streaming |
| Total API response time | < 5 seconds for 10MB files | p95 |
| Concurrent requests | 100+ RPS | Horizontal scaling |
| Memory usage | Constant (streaming) | No file buffering |

### Scalability Considerations

**Horizontal Scaling**:
- Stateless design (no local caching)
- Load balancing across multiple fs-brick-service instances
- Database connection pooling in BNPL

**Database Performance**:
- Indexes on `merchant_data.ktp_number`
- Indexes on `merchant_enrichment_data.type` and `created_at`
- Composite index for optimal JOIN performance

**S3 Performance**:
- Streaming download (no memory load)
- S3 Transfer Acceleration (if needed)
- CloudFront CDN (future enhancement)

**Bottleneck Mitigation**:
- BNPL query: Database indexing
- S3 download: Streaming, network bandwidth
- File serialization: Direct streaming to HTTP response

---

## Monitoring & Alerting

### Logging

**Error Logging**:
- Log all errors with full context (KTP, requestId, error message, stack trace)
- Log levels: INFO for success, WARN for business errors (404, 400), ERROR for system failures (500)
- Include correlation IDs for request tracing
- PII handling: Log KTP as permitted, do not log file content

**Key Log Events**:
- Request received: KTP, dataPoint, requestId
- BNPL call: Request/response status, duration
- S3 download: File path, size, duration
- Idempotency hits: Duplicate detection with requestId
- Errors: Full error context with stack trace

## Security & Compliance

### Authentication & Authorization

**Current (LND-4558)**:
- BNPL internal auth: `los-auth-token` header
- S3 access: IAM role-based
- No external partner authentication (placeholder)

**Future (LND-4557)**:
- Veefin authentication: HMAC-SHA256 signature
- Headers: `veefin-signature`, `veefin-timestamp`
- Request body hashing and signature verification

### Data Privacy

**PII Handling**:
- KTP logging: Full KTP logged (permitted per compliance)
- File content: Not logged (only metadata)
- Audit trail: Complete request/response metadata

**Compliance**:
- GDPR/PII guidelines followed
- Data access logged for audit
- Partner data sharing agreement required

### Secrets Management

**DO**:
- ✅ Use AWS Secrets Manager or environment variables
- ✅ Rotate secrets regularly
- ✅ Use IAM roles for S3 access
- ✅ Encrypt secrets at rest and in transit

**DON'T**:
- ❌ Commit secrets to version control
- ❌ Hardcode credentials in code
- ❌ Log authentication tokens
- ❌ Share secrets via insecure channels

---

## Rollout & Migration Plan

### Phase 1: Development & Testing

**BNPL Implementation**
- Create internal endpoint `/merchant/enrichment-data`
- Implement repository query method
- Write unit tests
- Local testing with curl

**FS-BRICK BNPL Integration**
- Add FsBnplPort method
- Implement FsBnplAdapter
- Integration test with BNPL
- Unit tests for adapter

**FS-BRICK S3 Integration**
- Use existing S3FileService for streaming download
- Test integration with real S3 bucket
- Add unit tests for borrower inquiry S3 flow

**FS-BRICK Controller & Multipart**
- Create VeefinBorrowerController
- Create VeefinBorrowerServiceImpl
- Implement multipart response builder
- Add PartnerEventAudit integration
- Unit tests for controller and service

**End-to-End Testing**
- Full integration test (fs-brick-service + BNPL + S3)
- Test all error scenarios
- Performance testing with various file sizes
- Code review and Spotless formatting

### Phase 2: Staging Deployment

**BNPL Staging**
- Deploy BNPL to staging
- Configure LOS_AUTH_TOKEN
- Verify database indexes
- Smoke test with curl

**FS-BRICK Staging**
- Deploy fs-brick-service to staging
- Configure environment variables (BNPL_MERCHANT_BASE_URL, BNPL_MERCHANT_TOKEN, BNPL_S3_BUCKET_NAME)
- End-to-end integration test
- Performance test with real data

**UAT with Veefin**
- Provide Veefin staging endpoint
- Support integration testing
- Collect feedback and fix issues
- Documentation review

**Staging Validation**
- Run full test suite
- Performance benchmarking
- Security review
- Deployment runbook review

### Phase 3: Production Deployment

**Production Deployment**
- Deploy BNPL to production (off-hours)
- Deploy fs-brick-service to production
- Feature flag: `{"veefin-borrower-inquiry": {"status": false, "allow": "test-ktp-1,test-ktp-2", "block": ""}}`
- Smoke test with allowed KTP numbers

**Gradual Rollout**
- Enable for all: `{"veefin-borrower-inquiry": {"status": true, "allow": "", "block": ""}}`
- Monitor logs and metrics
- Test with Veefin production credentials
- Monitor error rates and latency

**Production Monitoring**
- Monitor for initial period
- Address any issues immediately
- Collect performance metrics
- Veefin production integration

### Rollback Plan

**If Critical Issues Detected**:
1. Disable feature flag: `{"veefin-borrower-inquiry": {"status": false, "allow": "", "block": ""}}`
2. Notify Veefin of temporary unavailability
3. Investigate and fix issues
4. Deploy hotfix to staging
5. Test thoroughly
6. Re-enable in production

**Full Rollback**:
1. Revert fs-brick-service deployment
2. Revert BNPL deployment
3. Verify existing functionality intact
4. Schedule post-mortem

---

## Risks & Mitigations

| Risk | Impact | Probability | Mitigation | Owner |
|------|--------|-------------|------------|-------|
| S3 file not found | API returns 500 | Medium | BNPL validates S3 path exists before returning; log errors for investigation | Engineering |
| Large file OOM | Service crash | Low | Use streaming (InputStream) not byte array; load test with large files | Engineering |
| BNPL timeout | API timeout | Low | Configure RestTemplate timeout; implement retry logic; monitor BNPL performance | Engineering |
| Database query slow | High latency | Medium | Create database indexes; query optimization; connection pool tuning | Engineering |
| Invalid S3 permissions | 500 error | Low | Test IAM role in staging; verify S3 bucket policy; document setup | DevOps |
| PartnerEventAudit deadlock | Request failure | Very Low | Existing pattern proven in PartnerController; tested under load | Engineering |
| Authentication bypass (no LND-4557) | Security risk | Medium | Deploy LND-4557 quickly; document temporary placeholder; restrict access | Security/Engineering |
| High idempotency rate | Performance impact | Low | Refetch is fast; monitor idempotency rate; investigate client retry logic | Engineering |
| Data availability (IN_PROGRESS) | High 400 rate | Medium | Expected behavior; document for Veefin; improve enrichment pipeline SLA | Product/Engineering |

---

## Open Questions

**Resolved**:
- ✅ **Authentication approach**: Placeholder in LND-4558, HMAC-SHA256 in LND-4557
- ✅ **Idempotency strategy**: Refetch S3 on duplicate requests
- ✅ **S3 bucket configuration**: Use `@Value("${gt.bnplS3BucketName}")`
- ✅ **Request header name**: `veefin-request-id`
- ✅ **DataPoint type**: EnrichmentDataPoint enum
- ✅ **KTP logging**: Full KTP permitted (no masking)
- ✅ **Error handling scope**: 401 errors deferred to LND-4557

**Pending**:
- None

---

## Dependencies

### Internal Dependencies

| Dependency | Type | Status | Risk |
|------------|------|--------|------|
| LND-4557 (Veefin Authentication) | Feature | In Progress | Medium - Temporary placeholder acceptable |
| merchant_enrichment_data table | Database | Exists | Low - Stable schema |
| AWS S3 bucket (BNPL) | Infrastructure | Exists | Low - Already in use |
| PartnerEventAudit pattern | Code | Exists | Low - Proven in PartnerController |

### External Dependencies

| Dependency | Type | Status | Risk |
|------------|------|--------|------|
| Veefin integration | Partnership | Pending | Low - API design agreed |
| AWS S3 service | Infrastructure | Stable | Very Low - Managed service |
| PostgreSQL database | Infrastructure | Stable | Very Low - Production ready |

---

## Testing Strategy

### Unit Tests

**Coverage Target**: > 80%

**fs-brick-service**:
- Controller: 8 test cases
- Service: 10 test cases
- Adapter: 5 test cases
- S3 Service: 3 test cases

**bnpl**:
- Controller: 8 test cases
- Service: 3 test cases
- Repository: 5 test cases

### Integration Tests

**FS-BRICK → BNPL**:
- Mock BNPL with WireMock
- Test all response codes (200, 400, 404, 500)
- Verify los-auth-token sent

**FS-BRICK → S3**:
- Use LocalStack or mock S3
- Test streaming download
- Test error scenarios (not found, access denied)

**End-to-End**:
- Real database (TestContainers PostgreSQL)
- Real S3 (LocalStack)
- Full request flow with assertions

### Performance Tests

**Load Test**:
- Measure p95, p99 latency
- Monitor memory and CPU usage

**Stress Test**:
- Identify breaking point
- Verify graceful degradation

**File Size Tests**:
- Small files (< 1MB): < 1 second
- Medium files (1-5MB): < 3 seconds
- Large files (5-10MB): < 5 seconds

### Manual Test Checklist

- [ ] Valid PEFINDO request returns 200 with JSON file
- [ ] Valid FDC request returns 200 with JSON file
- [ ] Valid BANK_STATEMENT request returns 200 with XLSX file
- [ ] Non-existent KTP returns 404
- [ ] IN_PROGRESS status returns 400
- [ ] FAILED status returns 400
- [ ] S3 file not found returns 500
- [ ] Duplicate requestId returns fresh file (idempotency)
- [ ] Invalid KTP format returns 400
- [ ] Invalid dataPoint returns 400
- [ ] Invalid los-auth-token (BNPL) returns 401
- [ ] Missing los-auth-token (BNPL) returns 401

---

## Success Metrics

### Technical Metrics

**Post-Launch (Early Phase)**:
- Error rate < 1%
- p95 latency < 5 seconds
- Availability > 99.9%
- Zero critical incidents

**Post-Launch (Stable Phase)**:
- Error rate < 0.5%
- p95 latency < 3 seconds
- Idempotency hit rate < 10%

### Business Metrics

**Short Term**:
- Veefin integration successful
- API calls processed reliably
- Partner satisfaction: positive feedback

**Long Term**:
- Additional partners onboarded (if applicable)
- API usage growth
- Zero security incidents
- Positive ROI on partnership

---

## Repository-Specific RFCs

For detailed implementation specifications, see:

**fs-brick-service**:
- **Location**: `fs-brick-service/RFC/LND-4558-fs-brick-service.md`
- **Scope**: External API, BNPL integration, S3 integration, multipart response
- **Components**: VeefinBorrowerController, VeefinBorrowerServiceImpl, FsBnplAdapter, S3FileService

**bnpl**:
- **Location**: `bnpl/RFC/LND-4558-bnpl.md`
- **Scope**: Internal API, database query, authentication
- **Components**: MerchantEnrichmentController, MerchantEnrichmentService, Repository query method

---

## References

1. **Jira Ticket**: [LND-4558](https://bukuwarung.atlassian.net/browse/LND-4558)
2. **Requirement Document**: `/Users/juvianto.chi/Desktop/code/requirements/LND-4558/requirement.md`
3. **API Documentation**: `/Users/juvianto.chi/Desktop/code/documentation/apidocs/LND-4527.md`
4. **Authentication RFC**: [LND-4557](https://bukuwarung.atlassian.net/browse/LND-4557) - HMAC-SHA256 signature
5. **Task File**: `/Users/juvianto.chi/Desktop/code/requirements/LND-4558/task.md`
6. **Clarification Document**: `/Users/juvianto.chi/Desktop/code/requirements/LND-4558/clarification.md`
7. **AWS S3 Documentation**: https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/examples-s3.html
8. **Spring Multipart Documentation**: https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-multipart

---

## Approval

**Status**: 🚧 Draft
**Date**: 2025-11-24
**Approved By**: Pending Review

**Key Decisions**:
- Authentication placeholder acceptable for LND-4558 (full implementation in LND-4557)
- Idempotency via refetch S3 (not cached response)
- EnrichmentDataPoint enum for type safety
- Full KTP logging permitted per compliance
- Repository-specific RFCs created for detailed implementation

**Next Steps**:
1. Begin Phase 1 development
2. Schedule staging deployment
3. Coordinate with Veefin for UAT
4. Production deployment
5. Monitor and iterate based on metrics
