# LND-4558: Veefin Borrower Data Inquiry - Investigation & Clarification

**Date:** 2025-11-21
**Status:** Investigation Complete - Awaiting User Clarification

---

## ✅ Section 1: Findings from Current Implementation

### 🏗️ Repository Structure

**FS-BRICK-SERVICE:**
- **Location**: `/Users/juvianto.chi/Desktop/code/fs-brick-service/`
- **Language**: Java 11
- **Framework**: Spring Boot 2.6.3
- **Build Tool**: Gradle
- **Code Style**: Google Java Format (enforced via Spotless)

**BNPL:**
- **Location**: `/Users/juvianto.chi/Desktop/code/bnpl/`
- **Language**: Java (Spring Boot application)
- **Database**: PostgreSQL with JPA/Hibernate

### 📊 Current Tech Stack

**FS-BRICK-SERVICE Dependencies (build.gradle):**
```
- Spring Boot: 2.6.3
- Spring Data JPA
- Spring Web
- Lombok: 1.18.22
- AWS SDK S3: 1.12.225
- MapStruct: 1.4.2.Final
- Gson: 2.9.0
- Apache HttpClient: 4.5.13
- Resilience4j (circuit breaker support) -> dont implement new usage for this project.
```

**Key Libraries Available:**
- ✅ AWS S3 SDK already available
- ✅ RestTemplate for HTTP calls
- ✅ @Slf4j logging via Lombok
- ✅ @Valid for request validation
- ✅ ObjectMapper (Gson) for JSON serialization

### 🔌 Current BNPL Connection Pattern

**FsBnplPort Interface** (`FsBnplPort.java`):
```java
public interface FsBnplPort {
  @Nullable
  MerchantIdentityDto getBnplMerchantOnboardingInfo(@NonNull String userId, LoanUserType productType);
  void doLeadsEnrichmentCallback(String batchId, EnrichmentDataPoint type, String s3Path);
}
```

**FsBnplAdapter Implementation** (`FsBnplAdapter.java`):
- **Base URL**: Configured via `${fs.bnpl.merchant.baseUrl}`
- **Authentication**: Uses header `los-auth-token` with value from `${fs.bnpl.token}`
- **HTTP Client**: `RestTemplateService.executeServiceCall()`
- **Pattern**: Standard REST client with GET/PATCH methods
- **Error Handling**: RuntimeException on failures with error logging

**Example Existing Call:**
```java
String requestUrl = UriComponentsBuilder.fromUriString(baseUrl)
    .path("/merchant/identity")
    .queryParam("userId", userId)
    .encode()
    .toUriString();

MerchantIdentityDto response = restTemplateService.executeServiceCall(
    requestUrl, HttpMethod.GET, "los-auth-token", fsBnplToken,
    null, "bnplGetBnplMerchantOnboardingInfo",
    MerchantIdentityDto.class, null);
```

### 🗄️ BNPL Database Schema

**merchant_data table:**
- Primary Key: `merchant_id` (UUID)
- **KTP Field**: `ktp_number` (varchar(50), NOT NULL)
- Relationship: Parent table for merchant information

**merchant_enrichment_data table (V59 migration):**
- Primary Key: `id` (UUID)
- Foreign Key: `merchant_id` → `merchant_data.merchant_id`
- **Key Fields**:
  - `batch_id` (VARCHAR(60))
  - `type` (VARCHAR(15)) - e.g., "PEFINDO", "FDC", "BANK_STATEMENT"
  - `s3_path` (VARCHAR(200)) - **Location of data file in S3**
  - `status` (VARCHAR(20)) - LeadsEnrichmentStatus enum
  - `created_at`, `updated_at`, `created_by`, `updated_by`

**Entity Relationship:**
```
merchant_data (ktp_number)
    ↓ (merchant_id)
merchant_enrichment_data (s3_path, type)
```

### 🔍 API Documentation Review

**Endpoint Specification (LND-4527.md):**
- **Path**: `POST /fs/brick/service/borrower/v1/inquiry-data`
- **Authentication**: HMAC-SHA256 Signature
- **Request Headers**:
  - `veefin-x-partner-id` (string, required)
  - `veefin-x-request-id` (string, required, 16 chars alphanumeric)
  - `veefin-x-timestamp` (string, required, Unix epoch within 5 min)
  - `veefin-x-signature` (string, required, Base64 HMAC-SHA256)

**Request Body:**
```json
{
  "ktpNumber": "1234567890123456",  // 16 digits
  "dataPoint": "PEFINDO"           // Enum: PEFINDO, FDC, BANK_STATEMENT. existing EnrichmentDataPoint enum 
}
```

**Response Format:**
- **Content-Type**: `multipart/form-data`
- **Parts**:
  1. `dataFile` (application/octet-stream) - File from S3
  2. `dataPoint` (text/plain) - Type of data
  3. `dataId` (text/plain) - Unique identifier

**Success Response (200):**
```http
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary...

------WebKitFormBoundary...
Content-Disposition: form-data; name="dataFile"; filename="data.json"
Content-Type: application/octet-stream

[Binary file content from S3]
------WebKitFormBoundary...
Content-Disposition: form-data; name="dataPoint"

PEFINDO
------WebKitFormBoundary...
Content-Disposition: form-data; name="dataId"

DATA123456789
------WebKitFormBoundary...--
```

### 🎯 Existing Controller Pattern

**Reference**: `PartnerController.java`
- **Base Path**: `/fs/brick/service/partner`
- **Logging**: `@Slf4j` with structured logging
- **Validation**: `@Valid` on request bodies
- **Error Handling**: Try-catch with custom error responses
- **Response Format**: `ResponseEntity<Object>` with proper HTTP status codes
- **Dependencies Injection**: Constructor-based injection

**Common Pattern:**
```java
@PostMapping(value = "/endpoint")
public ResponseEntity<Object> methodName(@RequestBody @Valid RequestDto request) {
  log.info("Request received...");
  try {
    // Business logic
    ResponseDto result = service.process(request);
    return ResponseEntity.ok().contentType(MediaType.APPLICATION_JSON).body(result);
  } catch (Exception e) {
    log.error("Error: ", e);
    return ResponseEntity.internalServerError().body(errorObject);
  }
}
```

### 🔐 Authentication Context

**From Task Context:**
- Authentication implementation is pending in LND-4557 (separate ticket)
- Current task (LND-4558) focuses on endpoint implementation
- HMAC-SHA256 signature authentication will be added later

**Existing Auth in FS-BRICK-SERVICE:**
- Token-based validation exists (seen in FsBnplAdapter using `los-auth-token`)
- Pattern: Custom headers with secret validation
- Need to configure in fsBnpl for bnpl to use los-auth-token

### 📦 AWS S3 Integration

**Available in FS-BRICK-SERVICE:**
- ✅ AWS Java SDK S3: 1.12.225 (already in dependencies)
- Need to verify S3 client configuration patterns in existing code -> there's already one to download from S3 in BNPL

### 🔄 Data Flow Understanding

**Based on task description:**
```
Veefin Request
    ↓
FS-BRICK-SERVICE (/fs/brick/service/borrower/v1/inquiry-data)
    ↓ (Query by ktpNumber)
BNPL Service (new endpoint needed)
    ↓ (Query merchant_enrichment_data via merchant_id)
BNPL Database
    ↓ (Get s3_path)
AWS S3 (retrieve file)
    ↓
FS-BRICK-SERVICE (build multipart response)
    ↓
Veefin Response
```

---

## ❓ Section 2: Questions Requiring User Clarification

### 🔴 Critical Questions

#### Q1: BNPL Service Endpoint Implementation
**Context**: Task mentions creating endpoint in FS-BRICK-SERVICE to call BNPL, but doesn't specify BNPL endpoint details.

**Question**: Should we create a NEW internal endpoint in BNPL service, or does one already exist?

**Options**:
- **A) Create new BNPL endpoint** (Recommended based on investigation):
  - Path: `/merchant/enrichment-data/inquiry` or similar
  - Method: POST
  - Request: `{ "ktpNumber": "...", "dataPoint": "PEFINDO" }`
  - Response: `{ "s3Path": "...", "dataId": "...", "merchantId": "..." }`
  - Authentication: `los-auth-token` (consistent with existing pattern in FsBnplAdapter)

- **B) Use existing endpoint** (if available):
  - Please specify the endpoint path and contract

**Your Answer**: new endpoint, option A.

#### Q2: S3 File Retrieval Responsibility
**Context**: File is stored in S3, need to retrieve and return as multipart.

**Question**: Which service should retrieve the file from S3?

**Options**:
- **A) FS-BRICK-SERVICE retrieves from S3** (Recommended):
  - BNPL returns only `s3_path` and metadata
  - FS-BRICK-SERVICE downloads file and builds multipart response
  - **Pros**: Separation of concerns, BNPL stays lightweight
  - **Cons**: FS-BRICK needs S3 read permissions

- **B) BNPL retrieves and returns file**:
  - BNPL downloads file and returns it to FS-BRICK-SERVICE
  - FS-BRICK-SERVICE forwards to Veefin
  - **Pros**: Centralized S3 access in BNPL
  - **Cons**: Larger payload between services

**Your Answer**: A.

#### Q3: Missing Data Scenario
**Context**: What if ktpNumber exists but no enrichment data for requested dataPoint?

**Question**: What should the response be when ktpNumber exists but no PEFINDO/FDC data available?

**Options**:
- **A) Return 404 Not Found** (Recommended based on API docs):
  - Error code: 404
  - Message: "No enrichment data found for the provided KTP and data point"

- **B) Return 200 with empty data**:
  - Include placeholder or null values

**Your Answer**: A. if there's no data in merchant_enrichment_data. return 400 if data is on progress or failed.

#### Q4: Multiple Enrichment Records
**Context**: A merchant might have multiple enrichment records for the same type.

**Question**: If multiple records exist for same ktpNumber and dataPoint (e.g., multiple PEFINDO entries), which one to return?

**Options**:
- **A) Return latest by created_at DESC** (Recommended):
  - Query: `ORDER BY created_at DESC LIMIT 1`
  - Most recent enrichment data

- **B) Return latest by status = 'READY'**:
  - Filter by specific status first

- **C) Return based on batch_id priority**:
  - If specific batch logic exists

**Your Answer**: A. fetch the latest.

### 🟡 Important Questions

#### Q5: BNPL Endpoint Path Convention
**Context**: Need to determine endpoint path in BNPL service.

**Question**: What should the BNPL internal endpoint path be?

**Suggested Options**:
- **A)** `/merchant/enrichment-data` (matches existing `/merchant/identity` pattern)
- **B)** `/v1/internal/veefin/borrower-data` (explicit internal/partner designation)
- **C)** `/merchant/inquiry-data` (matches FS-BRICK external path pattern)
- **D)** Other: _____________________

**Recommended**: **Option A** (consistent with existing FsBnplAdapter usage of `/merchant/identity`)

**Your Answer**: A 

#### Q6: dataId Field Source
**Context**: API requires `dataId` in multipart response.

**Question**: What should be used as `dataId` in the response?

**Options**:
- **A) batch_id from merchant_enrichment_data** (Recommended):
  - Already exists in database
  - Unique identifier for enrichment batch

- **B) UUID id from merchant_enrichment_data**:
  - Primary key of the record

- **C) Generate new UUID**:
  - Fresh identifier per request

**Your Answer**: A
 
#### Q7: Error Handling for S3 Retrieval Failure
**Context**: S3 file might be deleted or inaccessible.

**Question**: What should happen if S3 file doesn't exist or download fails?

**Options**:
- **A) Return 500 Internal Server Error** (Recommended):
  - Error code: 500
  - Message: "Failed to retrieve enrichment data file"
  - Log error with S3 path for investigation

- **B) Return 404 Not Found**:
  - Treat as "data not available"

**Your Answer**: A

### 🟢 Optional Questions

#### Q8: Idempotency Handling
**Context**: API uses `veefin-x-request-id` for idempotency.

**Question**: Should we implement idempotency caching?

**Options**:
- **A) Implement later** (Recommended for MVP):
  - Focus on core functionality first
  - Add caching in follow-up ticket

- **B) Implement now with partner audit pattern**:
  - Similar to existing PartnerController patterns

**Default**: **Option A** (implement later - out of scope for initial implementation)

**Your Answer**: Implement now using PartnerEventAudit pattern. look for the pattern. it has 2 level 1 is partner level and 1 is service level.

#### Q9: Logging Sensitivity
**Context**: KTP is PII (Personally Identifiable Information).

**Question**: Should we mask KTP in logs?

**Options**:
- **A) Mask KTP in logs** (Recommended for production):
  - Log format: `1234****`***`3456` (first 4 and last 4 digits only)
  - Example utility method: `maskKtp(String ktp)`

- **B) Log full KTP for debugging**:
  - Only in development environments

**Recommended**: **Option A** (consistent with data privacy practices)

**Your Answer**: B

#### Q10: Response Filename Convention
**Context**: Multipart response includes filename for dataFile.

**Question**: What should the filename format be?

**Suggested Options**:
- **A)** `{dataPoint}_{ktpNumber}_{timestamp}.{ext}` (e.g., `PEFINDO_1234567890123456_1700000000.json`)
- **B)** `{dataPoint}_data.{ext}` (e.g., `PEFINDO_data.json`)
- **C)** Use original S3 file key as filename
- **D)** `data.{ext}` (generic, based on API docs example)

**Recommended**: **Option B** (simple, descriptive, avoids exposing full KTP)

**Your Answer**: D

---

## 📝 Section 3: Additional Discovered Information

### Related Implementation Plan
- **LND-4525 Plan**: Comprehensive Veefin Borrower Data Inquiry implementation plan exists
- **Location**: `/Users/juvianto.chi/Desktop/code/fs-brick-service/thoughts/shared/plans/2025-11-07-LND-4525-veefin-borrower-inquiry-api.md`
- **Note**: LND-4525 plan describes slightly different approach (credit summary from merchant_credit_summary table) -> irrelevant.
- **Clarification Needed**: Confirm if LND-4558 follows different approach (direct S3 retrieval) vs LND-4525 (pre-computed summaries) -> current plan is irrelevant. go with direct S3 retrieval.

### Authentication Ticket Reference
- **Ticket**: LND-4557 (HMAC-SHA256 signature authentication)
- **Status**: Not yet implemented (separate from this task)
- **Impact**: Initial implementation can use placeholder auth or skip validation

### Out of Scope (Confirmed from Task)
- ❌ Circuit breaker implementation
- ❌ Rate limiter implementation
- ❌ Authentication implementation (handled in LND-4557)

---

## 📋 Instructions for User

**Please answer the questions above by filling in "Your Answer" fields.**

For questions where investigation already provides strong recommendation, you can:
- Confirm the recommended option, OR
- Provide alternative with reasoning

**Priority:**
1. **Critical Questions (Q1-Q4)**: Must be answered to proceed
2. **Important Questions (Q5-Q7)**: Recommended to clarify for complete implementation
3. **Optional Questions (Q8-Q10)**: Can use recommended defaults if not specified

**Once you provide answers**, I will:
1. Update this clarification.md with your responses
2. Generate comprehensive `requirement.md`
3. Copy requirements to both FS-BRICK-SERVICE and BNPL repositories

**If any question is unclear or you need more context**, please ask and I will provide additional investigation.

---

**Next Steps After Clarification:**
1. Update clarification.md with answers
2. Generate requirement.md with complete specifications
3. Copy to repositories:
   - `/Users/juvianto.chi/Desktop/code/fs-brick-service/requirements/LND-4558/`
   - `/Users/juvianto.chi/Desktop/code/bnpl/requirements/LND-4558/`
4. Mark task.md as completed with summary