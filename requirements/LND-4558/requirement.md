# LND-4558: Veefin Borrower Data Inquiry API Implementation

## Background

### Current Implementation

**FS-BRICK-SERVICE:**
- Location: `/Users/juvianto.chi/Desktop/code/fs-brick-service/`
- Java 11 with Spring Boot 2.6.3, PostgreSQL, Gradle
- Existing BNPL integration via `FsBnplPort` interface and `FsBnplAdapter` implementation
- Uses `los-auth-token` for BNPL service authentication
- AWS S3 SDK v1.12.225 available for S3 operations

**BNPL Service:**
- Location: `/Users/juvianto.chi/Desktop/code/bnpl/`
- Stores merchant enrichment data in `merchant_enrichment_data` table
- Schema: `merchant_data(ktp_number)` → `merchant_enrichment_data(s3_path, type, batch_id)`
- S3 download functionality already exists in BNPL

**Existing Data Flow (Enrichment Storage):**
```
BNPL (merchantOnboarding)
    ↓ (enrichment request)
FS-BRICK-SERVICE
    ↓ (upload file)
AWS S3 (storage)
    ↓ (return s3_path)
FS-BRICK-SERVICE
    ↓ (callback with s3_path via FsBnplAdapter.doLeadsEnrichmentCallback)
BNPL
    ↓ (store s3_path in merchant_enrichment_data table)
```

**NEW Data Flow (This Implementation - Enrichment Retrieval):**
```
Veefin Request
    ↓
FS-BRICK-SERVICE (/fs/brick/service/borrower/v1/inquiry-data)
    ↓ (query by ktpNumber + dataPoint)
BNPL (/merchant/enrichment-data/inquiry)
    ↓ (return s3_path from merchant_enrichment_data)
FS-BRICK-SERVICE
    ↓ (download file using s3_path)
AWS S3
    ↓ (file content)
FS-BRICK-SERVICE (build multipart response)
    ↓
Veefin Response (multipart/form-data)
```

**API Contract:**
- Documentation: `/Users/juvianto.chi/Desktop/code/documentation/apidocs/LND-4527.md`
- Authentication: HMAC-SHA256 signature (to be implemented in LND-4557)
- Response format: multipart/form-data with 3 parts (dataFile, dataPoint, dataId)

### Business Context

Veefin (partner) requires an API endpoint to query borrower merchant enrichment data using KTP numbers. The data includes credit reports (PEFINDO, FDC) and bank statements stored in AWS S3. This endpoint enables Veefin to perform credit assessments and risk evaluation.

**Current Challenge:**
- No direct API for Veefin to retrieve borrower enrichment data
- Data exists in BNPL database and AWS S3, but no integration endpoint
- Need secure, efficient way to serve S3 files as multipart responses

## Objective

Create API endpoint in FS-BRICK-SERVICE (`/fs/brick/service/borrower/v1/inquiry-data`) to handle borrower data inquiry requests from Veefin, integrating with BNPL service to retrieve S3 paths and returning enrichment data files as multipart responses with proper idempotency and error handling.

## Scope

### In Scope

1. **FS-BRICK-SERVICE: External API Endpoint**
   - POST `/fs/brick/service/borrower/v1/inquiry-data`
   - Request validation using existing `EnrichmentDataPoint` enum
   - Multipart response builder with 3 parts (dataFile, dataPoint, dataId)
   - Idempotency using PartnerEventAudit pattern (partner + service level)
   - Error handling for all failure scenarios (404, 400, 500)

2. **FS-BRICK-SERVICE: BNPL Integration**
   - Add new method to `FsBnplPort` interface for enrichment data inquiry
   - Implement method in `FsBnplAdapter` for BNPL communication
   - HTTP client configuration with `los-auth-token` authentication

3. **FS-BRICK-SERVICE: S3 Integration**
   - S3 file retrieval using existing AWS SDK
   - Stream S3 file content directly to multipart response
   - Error handling for S3 access failures

4. **BNPL Service: Internal Endpoint**
   - POST `/merchant/enrichment-data/inquiry` (POST used to avoid PII in URL logs)
   - Query `merchant_enrichment_data` by KTP number and data point type
   - Return S3 path, batch_id, and metadata
   - los-auth-token authentication validation
   - **Filter Adjustment**: Update `shouldNotFilter` configuration to allow FS-BRICK-SERVICE token
   - Status-based filtering (return 400 for in-progress/failed data)

5. **Testing Coverage**
   - Unit tests for all service and controller layers
   - Integration tests for BNPL communication
   - Error scenario testing (404, 400, 500 cases)
   - S3 retrieval failure testing

6. **Observability**
   - Structured logging with MDC (Mapped Diagnostic Context) for correlation IDs
   - **KTP Logging Policy**: Do NOT log KTP in application logs; MUST be stored in PartnerEventAudit payload only
   - PartnerEventAudit tracking for idempotency
   - Error logging with stack traces for S3 failures

### Out of Scope

1. **Authentication Implementation**
   - HMAC-SHA256 signature validation (handled in LND-4557)
   - Initial implementation uses placeholder/skip validation
   - 401 and authentication-related error handling (deferred to LND-4557)

2. **Advanced Resilience**
   - Circuit breaker implementation (no new Resilience4j usage)
   - Rate limiting

3. **Caching**
   - Application-level response caching
   - S3 file caching

4. **Data Processing**
   - File content validation or parsing
   - Credit score calculations or aggregations

5. **Batch Operations**
   - Bulk inquiry endpoints
   - Multiple KTP processing in single request

## Requirements

### Functional Requirements

#### FR-1: FS-BRICK External Endpoint
**Priority**: High
**Description**: Create REST endpoint to receive borrower data inquiry requests from Veefin.

**Acceptance Criteria:**
- Endpoint MUST be accessible at `POST /fs/brick/service/borrower/v1/inquiry-data`
- Request body MUST validate:
  - `ktpNumber`: exactly 16 digits, numeric only
  - `dataPoint`: EnrichmentDataPoint enum type (not string) for easier validation
- Controller MUST use `@Valid` annotation for request validation
- Controller MUST follow existing `PartnerController.java` pattern
- Endpoint MUST return multipart/form-data response on success

**Implementation Location:**
- File: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/controller/BorrowerDataInquiryController.java` (new)
- Base path: `/fs/brick/service/borrower/v1`
- **Note**: Use general naming (not partner-specific) to allow future extensibility

#### FR-2: Idempotency Implementation
**Priority**: High
**Description**: Implement idempotency using PartnerEventAudit pattern (2 levels: partner + service).

**Acceptance Criteria:**
- System MUST use `veefin-request-id` header as requestId for idempotency key
- System MUST check PartnerEventAudit before processing request
- For duplicate requests: System MUST refetch S3 path and return fresh file content
- System MUST save response in PartnerEventAudit as metadata containing:
  - `s3Path`: S3 location of the file
  - `dataId`: Batch ID from BNPL
- System MUST log both partner-level and service-level audit events
- Audit MUST record request payload, response metadata, and timestamps
- Pattern MUST follow existing implementation in `PartnerController.java`

**Implementation Location:**
- Utilize existing: `PartnerEventAuditEntity`, `EventAuditUtil`
- Pattern reference: `PartnerController.java:72-108`, `GetFdcDataCommand.java:30-72`, `GetPefindoDataCommand.java`

#### FR-3: BNPL Service Integration
**Priority**: High
**Description**: Integrate with BNPL service to query enrichment data by KTP number.

**Acceptance Criteria:**
- System MUST add new method to `FsBnplPort` interface:
  ```java
  EnrichmentDataResponse getEnrichmentDataByKtp(String ktpNumber, String dataPoint);
  ```
- `FsBnplAdapter` MUST implement HTTP POST to `/merchant/enrichment-data/inquiry`
- Adapter MUST send `los-auth-token` header with BNPL auth token
- Adapter MUST handle BNPL response codes (200, 400, 404, 500)
- Adapter MUST use existing `RestTemplateService.executeServiceCall()` pattern
- Request body: `{ "ktpNumber": "...", "dataPoint": "..." }`
- Response body: `{ "s3Path": "...", "batchId": "...", "type": "...", "status": "..." }`

**Implementation Location:**
- File: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/provider/FsBnplPort.java` (add method)
- File: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/provider/impl/FsBnplAdapter.java` (implement)

#### FR-4: S3 File Retrieval
**Priority**: High
**Description**: Retrieve enrichment data files from AWS S3 and stream to response.

**Acceptance Criteria:**
- System MUST use existing AWS S3 SDK (v1.12.225)
- System MUST use `@Value("${gt.bnplS3BucketName}")` for S3 bucket name configuration
- System MUST download file from S3 path returned by BNPL
- System MUST stream file content (not load entire file into memory)
- System MUST set content type as `application/octet-stream`
- System MUST use generic filename format: `data.json` or `data.xlsx` (based on file extension)
- System MUST handle S3 errors (file not found, access denied, timeout)

**Implementation Location:**
- File: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/service/S3FileService.java` (new or use existing)
- Verify existing S3 client configuration
- Reference: `PefindoServiceImpl.java` uses same bucket configuration pattern

#### FR-5: Multipart Response Builder
**Priority**: High
**Description**: Build multipart/form-data response with 3 parts as per API contract.

**Acceptance Criteria:**
- Response MUST have Content-Type: `multipart/form-data`
- Response MUST include 3 parts:
  1. **dataFile**: Binary file from S3 (application/octet-stream)
  2. **dataPoint**: Text field with data type (text/plain)
  3. **dataId**: Text field with batch_id (text/plain)
- dataFile part MUST have filename set (e.g., `data.json`)
- All parts MUST follow HTTP multipart specification
- Response builder MUST handle large files efficiently (streaming)

**Implementation Location:**
- File: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/service/impl/BorrowerDataInquiryServiceImpl.java` (new)
- Use Spring `MultipartBodyBuilder` for multipart construction

#### FR-6: BNPL Internal Endpoint
**Priority**: High
**Description**: Create internal BNPL endpoint to query enrichment data by KTP.

**Acceptance Criteria:**
- Endpoint MUST be accessible at `POST /merchant/enrichment-data/inquiry`
- Endpoint MUST validate `los-auth-token` header
- Endpoint MUST query `merchant_enrichment_data` table via:
  - JOIN with `merchant_data` on `merchant_id`
  - Filter by `merchant_data.ktp_number = :ktpNumber`
  - Filter by `merchant_enrichment_data.type = :dataPoint`
  - ORDER BY `merchant_enrichment_data.created_at DESC`
  - LIMIT 1 (return latest record)
- Endpoint MUST return 404 if no record found
- Endpoint MUST return 400 if record status is "IN_PROGRESS" or "FAILED"
- Endpoint MUST return 200 with S3 path and metadata if status is "COMPLETE"

**Implementation Location:**
- Controller: `bnpl/src/main/java/com/bukuwarung/fsbnplservice/controller/MerchantEnrichmentController.java` (new)
- Service: `bnpl/src/main/java/com/bukuwarung/fsbnplservice/service/MerchantEnrichmentService.java` (new method)
- Repository: `bnpl/src/main/java/com/bukuwarung/fsbnplservice/repository/MerchantEnrichmentDataRepository.java` (add query method)

#### FR-7: Error Handling
**Priority**: High
**Description**: Comprehensive error handling for all failure scenarios.

**Acceptance Criteria:**
- System MUST return 404 when no enrichment data found for KTP + dataPoint
  - Error body: `{ "code": 404, "message": "No enrichment data found for the provided KTP and data point" }`
- System MUST return 400 when enrichment data is in progress or failed
  - Error body: `{ "code": 400, "message": "Enrichment data is not ready. Status: {status}" }`
- System MUST return 500 when S3 file retrieval fails
  - Error body: `{ "code": 500, "message": "Failed to retrieve enrichment data file" }`
  - System MUST log S3 error with full stack trace and S3 path
- System MUST return 400 for invalid request format (validation errors)
- All error responses MUST follow consistent JSON structure
- **Note**: 401 and authentication-related errors are out of scope (handled in LND-4557)

**Implementation Location:**
- Global error handler: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/config/RestErrorControllerAdvice.java` (existing)
- Controller try-catch blocks in `BorrowerDataInquiryController`

### Non-Functional Requirements

#### NFR-1: Performance
**Priority**: Medium
**Description**: Endpoint must handle file retrieval and response within acceptable latency.

**Acceptance Criteria:**
- BNPL query response time MUST be < 500ms (p95)
- S3 file download MUST use streaming (not load full file into memory)
- FS-BRICK total response time MUST be < 5 seconds for 10MB files (p95)
- System MUST support concurrent requests (stateless design)

#### NFR-2: Backward Compatibility
**Priority**: High
**Description**: Implementation must not break existing functionality.

**Acceptance Criteria:**
- Existing `FsBnplPort` methods MUST remain unchanged
- Existing `PartnerController` patterns MUST be followed
- No changes to existing database schema
- No changes to existing S3 bucket structure or permissions

#### NFR-3: Maintainability
**Priority**: Medium
**Description**: Code must be clean, testable, and follow existing patterns.

**Acceptance Criteria:**
- Code MUST pass Google Java Format (Spotless)
- Code MUST follow existing controller/service/repository layering
- DTOs MUST use Lombok annotations (@Getter, @Setter, @Builder)
- Services MUST use constructor injection
- Logging MUST use @Slf4j with structured messages

#### NFR-4: Observability
**Priority**: Medium
**Description**: Adequate logging and audit trails for debugging and monitoring.

**Acceptance Criteria:**
- Controller MUST log request received with requestId (without KTP in logs)
- MUST use MDC (Mapped Diagnostic Context) to track requestId across all log statements
- BNPL adapter MUST log request/response to BNPL service (without KTP in logs)
- S3 retrieval MUST log S3 path and file size
- Errors MUST log with full stack trace and context
- PartnerEventAudit MUST record all requests with FULL payload including KTP for idempotency tracking
- **KTP Logging Policy**: Do NOT log KTP in application logs (slf4j); MUST store in PartnerEventAudit only
- Follow existing logging patterns from Pefindo and FDC implementations

#### NFR-5: Security
**Priority**: High
**Description**: Secure communication and data handling.

**Acceptance Criteria:**
- BNPL communication MUST use `los-auth-token` authentication
- los-auth-token MUST be configured via environment variable
- S3 access MUST use IAM role credentials (not hardcoded keys)
- Authentication headers MUST NOT be logged
- **PII (KTP) Handling**: MUST NOT be logged in application logs; MUST be stored in PartnerEventAudit for audit trail

### Technical Requirements

#### TR-1: FS-BRICK-SERVICE Configuration
**Required Configuration:**
```yaml
# application.yml
veefin:
  borrower-inquiry:
    enabled: true

fs:
  bnpl:
    merchant:
      baseUrl: ${BNPL_MERCHANT_BASE_URL}  # EXISTING (already configured)
    token: ${BNPL_MERCHANT_TOKEN}         # EXISTING (already configured)

gt:
  bnplS3BucketName: ${BNPL_S3_BUCKET_NAME:janus-buku-dev}  # EXISTING

aws:
  accessKeyId: ${AWS_ACCESS_KEY_ID}  # EXISTING
  secretKey: ${AWS_SECRET_KEY}       # EXISTING
  region: ${AWS_REGION}              # EXISTING
```

**Rationale:**
- Feature toggle for deployment control
- Environment-specific BNPL URL configuration
- S3 bucket name configuration (follows existing pattern in PefindoServiceImpl.java)
- S3 region configuration for AWS SDK

#### TR-2: BNPL Configuration
**Required Configuration:**
```yaml
# application.yml
merchant:
  enrichment:
    auth-token: ${LOS_AUTH_TOKEN:default-los-token}
```

**Rationale:**
- los-auth-token validation for internal endpoint security

#### TR-3: FS-BRICK DTOs
**File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/model/borrower/BorrowerInquiryRequest.java`
**Note**: Use general package naming (not partner-specific)

```java
package com.bukuwarung.fsbrickservice.model.borrower;

import com.bukuwarung.fsbrickservice.model.EnrichmentDataPoint;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;
import javax.validation.constraints.Pattern;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class BorrowerInquiryRequest {

  @NotBlank(message = "KTP number is required")
  @Pattern(regexp = "^\\d{16}$", message = "KTP must be exactly 16 digits")
  private String ktpNumber;

  @NotNull(message = "Data point is required")
  private EnrichmentDataPoint dataPoint; // Use enum type directly for easier validation
}
```

**File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/model/bnpl/EnrichmentDataResponse.java`

```java
package com.bukuwarung.fsbrickservice.model.bnpl;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class EnrichmentDataResponse {
  private String s3Path;
  private String batchId;
  private String type;
  private String status;
}
```

#### TR-4: BNPL DTOs
**File**: `bnpl/src/main/java/com/bukuwarung/fsbnplservice/dto/EnrichmentDataRequest.java`

```java
package com.bukuwarung.fsbnplservice.dto;

import javax.validation.constraints.NotBlank;
import javax.validation.constraints.Pattern;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class EnrichmentDataRequest {

  @NotBlank(message = "KTP number is required")
  @Pattern(regexp = "^\\d{16}$", message = "KTP must be exactly 16 digits")
  private String ktpNumber;

  @NotBlank(message = "Data point is required")
  private String dataPoint;
}
```

**File**: `bnpl/src/main/java/com/bukuwarung/fsbnplservice/dto/EnrichmentDataResponse.java`

```java
package com.bukuwarung.fsbnplservice.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class EnrichmentDataResponse {
  private String s3Path;
  private String batchId;
  private String type;
  private String status;
}
```

#### TR-5: BNPL Repository Query Method
**File**: `bnpl/src/main/java/com/bukuwarung/fsbnplservice/repository/MerchantEnrichmentDataRepository.java`

**Add Method:**
```java
@Query("SELECT med FROM MerchantEnrichmentDataEntity med " +
       "JOIN med.merchantData md " +
       "WHERE md.ktpNumber = :ktpNumber " +
       "AND med.type = :dataPoint " +
       "ORDER BY med.createdAt DESC")
Optional<MerchantEnrichmentDataEntity> findLatestByKtpAndType(
    @Param("ktpNumber") String ktpNumber,
    @Param("dataPoint") String dataPoint);
```

**Rationale:**
- Efficient query with JOIN
- Returns latest record by created_at
- Uses Optional for null safety

#### TR-6: BNPL Database Indexes
**Required**: Create database migration to add indexes for query performance

**Migration Script** (Flyway format):
```sql
-- V<next_version>__add_merchant_enrichment_data_indexes.sql
CREATE INDEX idx_merchant_enrichment_data_created_at
  ON merchant_enrichment_data(created_at DESC);

CREATE INDEX idx_merchant_enrichment_data_type
  ON merchant_enrichment_data(type);

-- Composite index for common query pattern
CREATE INDEX idx_merchant_enrichment_data_merchant_type
  ON merchant_enrichment_data(merchant_id, type, created_at DESC);
```

**Rationale:**
- Optimize query performance for `findLatestByKtpAndType` method
- Support efficient filtering by type and ordering by created_at
- Composite index covers JOIN and ORDER BY operations

### Test Requirements

#### Unit Tests (Required)

**FS-BRICK-SERVICE Tests:**

**File**: `fs-brick-service/src/test/java/com/bukuwarung/fsbrickservice/controller/BorrowerDataInquiryControllerTest.java`

**Test Cases:**
1. `testInquiryBorrower_Success()` - Valid request returns 200 with multipart response
2. `testInquiryBorrower_InvalidKtp()` - Returns 400 for invalid KTP format
3. `testInquiryBorrower_InvalidDataPoint()` - Returns 400 for invalid enum value
4. `testInquiryBorrower_NotFound()` - Returns 404 when no enrichment data
5. `testInquiryBorrower_DataInProgress()` - Returns 400 when status is IN_PROGRESS
6. `testInquiryBorrower_DataFailed()` - Returns 400 when status is FAILED
7. `testInquiryBorrower_S3Failure()` - Returns 500 when S3 download fails
8. `testInquiryBorrower_Idempotency()` - Returns cached response for duplicate requestId

**File**: `fs-brick-service/src/test/java/com/bukuwarung/fsbrickservice/provider/impl/FsBnplAdapterTest.java`

**Test Cases:**
1. `testGetEnrichmentData_Success()` - Successful BNPL call returns data
2. `testGetEnrichmentData_Unauthorized()` - Handles 401 from BNPL
3. `testGetEnrichmentData_NotFound()` - Handles 404 from BNPL
4. `testGetEnrichmentData_Timeout()` - Handles connection timeout
5. `testGetEnrichmentData_InvalidResponse()` - Handles malformed JSON

**BNPL Tests:**

**File**: `bnpl/src/test/java/com/bukuwarung/fsbnplservice/controller/MerchantEnrichmentControllerTest.java`

**Test Cases:**
1. `testGetEnrichmentData_Success()` - Returns 200 with S3 path
2. `testGetEnrichmentData_InvalidAuth()` - Returns 401 for invalid token
3. `testGetEnrichmentData_NotFound()` - Returns 404 when no record
4. `testGetEnrichmentData_InvalidKtp()` - Returns 400 for invalid KTP
5. `testGetEnrichmentData_StatusInProgress()` - Returns 400 for IN_PROGRESS status
6. `testGetEnrichmentData_StatusFailed()` - Returns 400 for FAILED status

**File**: `bnpl/src/test/java/com/bukuwarung/fsbnplservice/repository/MerchantEnrichmentDataRepositoryTest.java`

**Test Cases:**
1. `testFindLatestByKtpAndType_Found()` - Returns latest record
2. `testFindLatestByKtpAndType_MultipleRecords()` - Returns most recent
3. `testFindLatestByKtpAndType_NotFound()` - Returns empty Optional

**Test Data Requirements:**
- Sample merchant_data record with valid KTP
- Sample merchant_enrichment_data records with various statuses
- Mock S3 files (JSON for PEFINDO/FDC, XLSX for BANK_STATEMENT)
- Test resource location: `src/test/resources/fixtures/`

#### Integration Tests

**FS-BRICK End-to-End Test:**
1. Start both FS-BRICK and BNPL services locally
2. Populate test data in BNPL database
3. Use existing test S3 bucket with test files
4. Call `/fs/brick/service/borrower/v1/inquiry-data` endpoint
5. Verify multipart response with correct file content
6. Verify idempotency behavior with duplicate requests

**Test Infrastructure:**
- Use real database (no TestContainers)
- Use real S3 test bucket (no LocalStack)
- Follow existing test patterns from Pefindo and FDC implementations

**Manual Testing Checklist:**
- [ ] Valid request with PEFINDO data returns 200
- [ ] Valid request with FDC data returns 200
- [ ] Valid request with BANK_STATEMENT data returns 200
- [ ] Request with non-existent KTP returns 404
- [ ] Request with IN_PROGRESS status returns 400
- [ ] Request with FAILED status returns 400
- [ ] S3 file not found returns 500
- [ ] Duplicate requestId returns cached response
- [ ] Invalid KTP format returns 400
- [ ] Invalid dataPoint returns 400
- [ ] Unauthorized request returns 401 (when auth implemented)

## Implementation Plan

### Phase 1: BNPL Internal Endpoint (Day 1-2)

**Tasks:**
1. Create `EnrichmentDataRequest` and `EnrichmentDataResponse` DTOs
2. Add query method to `MerchantEnrichmentDataRepository`
3. Create `MerchantEnrichmentService` with business logic
4. Create `MerchantEnrichmentController` with POST endpoint
5. Add los-auth-token validation
6. Implement error handling (404, 400 for status)
7. Write unit tests for repository, service, controller
8. Test locally with curl

**Deliverables:**
- BNPL endpoint functional at `/merchant/enrichment-data/inquiry`
- Unit tests passing
- Manual test successful

### Phase 2: FS-BRICK BNPL Integration (Day 3)

**Tasks:**
1. Add method to `FsBnplPort` interface
2. Implement method in `FsBnplAdapter`
3. Create `EnrichmentDataResponse` DTO in FS-BRICK
4. Configure `${fs.bnpl.token}` in application.yml
5. Write adapter unit tests with mock RestTemplateService
6. Integration test with actual BNPL service

**Deliverables:**
- FsBnplAdapter can call BNPL endpoint successfully
- Adapter tests passing
- Integration verified

### Phase 3: FS-BRICK S3 Integration (Day 4)

**Tasks:**
1. Verify existing S3 client configuration
2. Create S3FileService for file download
3. Implement streaming download (not full load)
4. Handle S3 errors (not found, access denied)
5. Write unit tests with mock S3 client
6. Test with real S3 bucket

**Deliverables:**
- S3FileService functional
- Streaming download working
- Error handling verified

### Phase 4: FS-BRICK Controller & Multipart Response (Day 5-6)

**Tasks:**
1. Create `BorrowerInquiryRequest` DTO
2. Create `VeefinBorrowerController`
3. Create `VeefinBorrowerService` for orchestration
4. Implement multipart response builder
5. Add PartnerEventAudit integration for idempotency
6. Implement error handling (404, 400, 500)
7. Write controller and service unit tests
8. End-to-end integration test

**Deliverables:**
- External endpoint functional at `/fs/brick/service/borrower/v1/inquiry-data`
- Multipart response working
- Idempotency working
- All unit tests passing

### Phase 5: Testing & Documentation (Day 7)

**Tasks:**
1. Run full test suite (unit + integration)
2. Manual testing all scenarios (see checklist)
3. Fix any bugs found
4. Code review and Spotless formatting
5. Update README files (FS-BRICK and BNPL)
6. Create operational runbook
7. Prepare deployment plan

**Deliverables:**
- All tests passing
- Code formatted and reviewed
- Documentation complete
- Ready for deployment

## References

1. **Jira Ticket**: [LND-4558](https://bukuwarung.atlassian.net/browse/LND-4558)
2. **API Documentation**: `/Users/juvianto.chi/Desktop/code/documentation/apidocs/LND-4527.md`
3. **Authentication Ticket**: [LND-4557](https://bukuwarung.atlassian.net/browse/LND-4557) - HMAC-SHA256 signature
4. **Task File**: `/Users/juvianto.chi/Desktop/code/requirements/LND-4558/task.md`
5. **Clarification Document**: `/Users/juvianto.chi/Desktop/code/requirements/LND-4558/clarification.md`

**Codebase Locations:**
- FS-BRICK-SERVICE: `/Users/juvianto.chi/Desktop/code/fs-brick-service/`
- BNPL: `/Users/juvianto.chi/Desktop/code/bnpl/`

**External Documentation:**
- AWS S3 SDK: https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/examples-s3.html
- Spring Multipart: https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-multipart

## Approval

**Prepared By**: Claude Code
**Date**: 2025-11-24
**Status**: Ready for Implementation

**Implementation Notes:**
- Authentication validation is placeholder (LND-4557 dependency)
- 401 and auth-related errors are out of scope (handled in LND-4557)
- Do not implement new Resilience4j circuit breaker usage
- Do not implement rate limiting or caching mechanisms
- Use `EnrichmentDataPoint` enum type (not string) for dataPoint field validation
- Use `@Value("${gt.bnplS3BucketName}")` for S3 bucket configuration
- Use `veefin-request-id` header as the requestId in PartnerEventAudit
- For idempotency: refetch S3 path on duplicate requests, save s3Path and dataId in metadata
- **KTP Logging**: Do NOT log KTP in application logs; store in PartnerEventAudit payload only
- Use MDC (Mapped Diagnostic Context) for requestId correlation across log statements
- S3 download functionality already exists in BNPL (can be referenced)
- Follow existing PartnerEventAudit pattern for idempotency (2 levels)
- Use general class naming (not partner-specific) for extensibility
- BNPL `shouldNotFilter`: Update filter configuration to allow FS-BRICK-SERVICE token
- Database indexes required: Create migration script for merchant_enrichment_data indexes
- Test infrastructure: Use real database and S3 (no LocalStack/TestContainers)
- Follow existing patterns from Pefindo and FDC implementations for logging and structure
