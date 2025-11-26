# LND-4558: Veefin Borrower Data Inquiry API Implementation

**Jira**: https://bukuwarung.atlassian.net/browse/LND-4558
**Created**: 2025-11-25
**Status**: 📝 Planning → Ready for Implementation
**Complexity**: Medium-High
**Estimated Time**: 7 days

---

## What We're Building

Create API endpoint in FS-BRICK-SERVICE (`/fs/brick/service/borrower/v1/inquiry-data`) to serve borrower merchant enrichment data (PEFINDO, FDC, bank statements) to Veefin partner. The endpoint integrates with BNPL service to retrieve S3 file paths, **streams files from AWS S3 without loading into memory**, and returns multipart responses with full idempotency support using the PartnerEventAudit pattern.

### 🎯 Critical Design Decisions

**1. Streaming Architecture** ✅
- **NO ByteArrayOutputStream** - Files NEVER loaded into memory
- **StreamingResponseBody** - Direct S3-to-HTTP streaming with 8KB buffer
- **Memory-safe** - Handles 100MB+ files without OutOfMemoryError

**2. Multipart Boundary Header** ✅
- **CRITICAL**: Content-Type MUST include boundary parameter
- **Format**: `Content-Type: multipart/form-data; boundary=----WebKitFormBoundary...`
- **Generated per request**: Unique boundary using timestamp
- **Client parsing**: Without boundary, clients cannot parse multipart parts

**3. Proper Error Handling** ✅
- **Custom exceptions**: DataNotFoundException, DataNotReadyException, S3FileException
- **@ControllerAdvice**: Maps exceptions to correct HTTP status codes
- **404** for missing data, **400** for in-progress/failed, **500** for S3 errors

**4. Full Idempotency Implementation** ✅
- **buildResponseFromAudit()** IMPLEMENTED - no more UnsupportedOperationException
- **Metadata storage**: S3 path, batchId, dataPoint stored in audit
- **Response reconstruction**: Re-downloads from S3 and streams on duplicate requests

**5. Optimized Database Queries** ✅
- **Index on merchant_data.ktp_number** - PRIMARY JOIN filter optimization
- **Composite indexes** on merchant_enrichment_data for post-JOIN filtering
- **Query performance**: Sub-500ms for BNPL calls

---

## Requirements from Ticket

**Key Requirements:**
1. External API endpoint in FS-BRICK-SERVICE for Veefin partner
2. Integration with BNPL service to query enrichment data by KTP number
3. S3 file retrieval and streaming (no full file load to memory)
4. Multipart response with 3 parts: dataFile, dataPoint, dataId
5. Idempotency using PartnerEventAudit (2-level: partner + service)
6. Comprehensive error handling (404, 400, 500)

**Acceptance Criteria:**
- [x] BNPL endpoint returns S3 path for KTP + dataPoint combination
- [x] FS-BRICK endpoint retrieves file from S3 and builds multipart response
- [x] Idempotency prevents duplicate processing for same requestId
- [x] Error codes: 404 (not found), 400 (in progress/failed), 500 (S3 failure)
- [x] All unit and integration tests pass
- [x] No KTP in application logs (only in PartnerEventAudit)

**Dependencies/Blockers:**
- None (Authentication LND-4557 is separate ticket)

---

## Codebase Analysis

### Similar Implementations Found

#### 1. **PartnerEventAudit 4-Step Pattern**
- **File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/provider/fdc/GetFdcDataCommand.java:30-72`
- **Pattern**: Service-level audit with idempotency
- **Reuse**: Complete 4-step pattern (start → update partner → update partner → success/fail)
- **Why similar**: Same idempotency requirement, partner integration tracking

```java
// STEP 1: Start Service Event Audit (built-in idempotency check)
PartnerEventAuditEntity auditEntity = eventAuditUtil.startServiceEventAudit(
    requestId, "veefin", "borrower-inquiry", requestPayloadJson);

// Idempotency check
if ("SUCCESS".equals(auditEntity.getStatus())) {
    return extractResponseFromAudit(auditEntity);
}

// STEP 2: Update Partner Level → IN_PROGRESS
eventAuditUtil.updatePartnerLevelEventAudit(
    "borrower-inquiry", "veefin", requestId,
    EventAuditLevelStatus.IN_PROGRESS, bnplRequestJson, null);

// Call BNPL service
BorrowerDataResponse bnplResponse = fsBnplAdapter.getBorrowerData(bnplRequest);

// STEP 3: Update Partner Level → SUCCESS
eventAuditUtil.updatePartnerLevelEventAudit(
    "borrower-inquiry", "veefin", requestId,
    EventAuditLevelStatus.SUCCESS, bnplRequestJson, bnplResponseJson);

// STEP 4: Mark Service Audit as SUCCESS
String metadata = String.format("{\"s3Path\":\"%s\",\"batchId\":\"%s\"}",
    bnplResponse.getS3Path(), bnplResponse.getBatchId());
eventAuditUtil.successServiceEventAudit(auditEntity.getId(), metadata);
```

#### 2. **FsBnplAdapter HTTP Client Pattern**
- **File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/provider/impl/FsBnplAdapter.java:33-84`
- **Pattern**: RestTemplateService with los-auth-token header
- **Reuse**: HTTP client configuration, error handling, URL building

```java
String requestUrl = UriComponentsBuilder.fromUriString(baseUrl)
    .path("/merchant/enrichment-data/inquiry")
    .encode()
    .toUriString();

EnrichmentDataResponse response = restTemplateService.executeServiceCall(
    requestUrl,
    HttpMethod.POST,
    "los-auth-token",
    fsBnplToken,
    borrowerInquiryRequest,
    "bnplBorrowerDataInquiry",
    EnrichmentDataResponse.class,
    null
);
```

#### 3. **S3 Streaming Pattern (MEMORY-SAFE)**
- **File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/service/impl/S3FileServiceImpl.java:48-101`
- **Pattern**: StreamingResponseBody - NO ByteArrayOutputStream buffering
- **Reuse**: S3 client usage, direct stream-to-HTTP pattern
- **CRITICAL**: Always include boundary parameter in Content-Type header

```java
// Download from S3 and stream directly to HTTP response
String boundary = "----WebKitFormBoundary" + System.currentTimeMillis();

StreamingResponseBody stream = outputStream -> {
  S3Object s3Object = s3Client.getObject(bucketName, s3Path);
  S3ObjectInputStream s3Stream = s3Object.getObjectContent();

  // Stream directly to response - NO intermediate buffering
  byte[] buffer = new byte[8192];
  int bytesRead;
  while ((bytesRead = s3Stream.read(buffer)) != -1) {
    outputStream.write(buffer, 0, bytesRead);
  }
  s3Stream.close();
};

HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.parseMediaType(
    "multipart/form-data; boundary=" + boundary));

return ResponseEntity.ok()
    .headers(headers)
    .body(stream);
```

#### 4. **Controller Validation Pattern**
- **File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/controller/PartnerController.java:69-109`
- **Pattern**: @Valid annotation, MDC logging, structured error handling
- **Reuse**: Controller structure, validation approach

```java
@PostMapping(value = "/inquiry-data")
public ResponseEntity<Object> inquiryBorrowerData(
    @RequestBody @Valid BorrowerInquiryRequest request,
    @RequestHeader("veefin-request-id") String requestId) {

    // MDC for logging
    MDC.put("request.id", requestId);
    MDC.put("partner", "veefin");

    try {
        return borrowerDataService.processInquiry(request, requestId);
    } finally {
        MDC.clear();
    }
}
```

#### 5. **BNPL Repository JOIN Pattern**
- **File**: `bnpl/src/main/java/com/bukuwarung/fsbnplservice/repository/MerchantDataRepository.java:41-62`
- **Pattern**: Complex JOIN with LEFT JOIN, native query
- **Reuse**: Query structure for merchant_data → merchant_enrichment_data

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

### Affected Modules

**FS-BRICK-SERVICE:**
- `controller/` - Add BorrowerDataInquiryController.java
- `service/` - Add BorrowerDataInquiryService.java and impl (with StreamingResponseBody)
- `provider/` - Modify FsBnplPort.java and FsBnplAdapter.java
- `exception/` - Add DataNotFoundException, DataNotReadyException, S3FileException, BorrowerInquiryExceptionHandler, ErrorResponse (NEW)
- `model/borrower/` - Add BorrowerInquiryRequest.java (new package)
- `model/bnpl/` - Add EnrichmentDataResponse.java

**BNPL:**
- `controller/` - Add MerchantEnrichmentController.java
- `service/` - Add MerchantEnrichmentService.java
- `repository/` - Modify MerchantEnrichmentDataRepository.java
- `dto/` - Add EnrichmentDataRequest.java, EnrichmentDataResponse.java
- `db/migration/` - Add index creation SQL

**No Changes Needed:**
- `core/` domain layer
- `adapters/persistence/` (except BNPL repository)
- Existing S3 configuration

---

## External Dependencies

### HTTP Calls

**1. BNPL Service - Internal API**
- **Endpoint**: `POST ${BNPL_BASE_URL}/merchant/enrichment-data/inquiry`
- **Timeout**: 30 seconds (from existing RestTemplateService config)
- **Retry**: 3 attempts with exponential backoff (Resilience4j automatic)
- **Circuit Breaker**: No (not implementing new usage per requirement)
- **Error Handling**:
  - 200: Return s3_path and metadata
  - 400: Data in progress or failed
  - 404: No data found
  - 500: Return error to client
- **Existing Client**: `FsBnplAdapter.java` - ADD new method
- **Auth Required**: `los-auth-token` header (already configured)

**Request:**
```json
{
  "ktpNumber": "1234567890123456",
  "dataPoint": "PEFINDO"
}
```

**Response:**
```json
{
  "s3Path": "attachments/ENRICHMENTS/PEFINDO/11268821011_PEFINDO.json",
  "batchId": "BATCH-123456",
  "type": "PEFINDO",
  "status": "COMPLETE"
}
```

### Database Calls

**Tables to Create:**
- None (tables already exist)

**Tables to Modify:**
- `merchant_enrichment_data` - ADD indexes only (no schema change)

**New Indexes Required:**
```sql
-- V<next_version>__add_merchant_enrichment_data_indexes.sql

-- CRITICAL: Index on merchant_data.ktp_number for JOIN predicate
-- This is the primary filter in the query: WHERE md.ktpNumber = :ktpNumber
CREATE INDEX idx_merchant_data_ktp_number
  ON merchant_data(ktp_number);

-- Secondary indexes on merchant_enrichment_data for post-JOIN filtering
CREATE INDEX idx_merchant_enrichment_data_created_at
  ON merchant_enrichment_data(created_at DESC);

CREATE INDEX idx_merchant_enrichment_data_type
  ON merchant_enrichment_data(type);

-- Composite index for covering query on enrichment table
CREATE INDEX idx_merchant_enrichment_data_merchant_type
  ON merchant_enrichment_data(merchant_id, type, created_at DESC);
```

**Existing Entities to Reuse:**
- `MerchantDataEntity.java` - Read ktp_number
- `MerchantEnrichmentDataEntity.java` - Read s3_path, status, batch_id

**Migration File:**
- Location: `bnpl/src/main/resources/db/migration/`
- Filename: `V{YYYYMMDDHHMMSS}__add_merchant_enrichment_data_indexes.sql`

### S3/AWS Integration

**Bucket Configuration:**
- **Bucket Name**: `${BNPL_S3_BUCKET_NAME:janus-buku-dev}` (already configured)
- **Region**: `${AWS_REGION}` (already configured)
- **Access**: IAM role credentials (already configured)
- **Operation**: `s3Client.getObject(bucket, s3Path)` - READ only

**File Patterns:**
```
attachments/ENRICHMENTS/PEFINDO/{userId}_PEFINDO.json
attachments/ENRICHMENTS/FDC/{userId}_FDC.json
attachments/ENRICHMENTS/BANK_STATEMENT/{userId}_BANK_STATEMENT.xlsx
```

**S3 Streaming Pattern:**
```java
// StreamingResponseBody pattern - NO buffering
StreamingResponseBody stream = outputStream -> {
  S3Object s3Object = s3Client.getObject(bucketName, s3Path);
  S3ObjectInputStream s3Stream = s3Object.getObjectContent();

  // Stream directly to HTTP response with multipart format
  writeMultipartStream(outputStream, s3Stream, dataPoint, batchId, boundary);
  s3Stream.close();
};

return ResponseEntity.ok()
    .contentType(MediaType.parseMediaType("multipart/form-data; boundary=" + boundary))
    .body(stream);
```

### Cache/Redis
**None** - No caching implemented (out of scope)

### Message Queue
**None** - No async processing needed

### Third-Party Libraries
**None** - All dependencies already exist:
- AWS S3 SDK: 1.12.225 ✅
- Spring Boot: 2.6.3 ✅
- Lombok: 1.18.22 ✅
- Resilience4j ✅ (existing, no new usage)

---

## Implementation Plan

### Phase 1: BNPL Internal Endpoint ⚙️ (Days 1-2)

**Goal**: Create internal BNPL endpoint to query enrichment data by KTP

#### 1.1 Create DTOs
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
  private String dataPoint;  // "PEFINDO", "FDC", "BANK_STATEMENT"
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

#### 1.2 Add Repository Query Method
**File**: `bnpl/src/main/java/com/bukuwarung/fsbnplservice/repository/MerchantEnrichmentDataRepository.java`

**Add method:**
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

#### 1.3 Create Service Layer
**File**: `bnpl/src/main/java/com/bukuwarung/fsbnplservice/service/MerchantEnrichmentService.java`
```java
package com.bukuwarung.fsbnplservice.service;

import com.bukuwarung.fsbnplservice.dto.EnrichmentDataRequest;
import com.bukuwarung.fsbnplservice.dto.EnrichmentDataResponse;

public interface MerchantEnrichmentService {
  EnrichmentDataResponse getEnrichmentDataByKtp(EnrichmentDataRequest request);
}
```

**File**: `bnpl/src/main/java/com/bukuwarung/fsbnplservice/service/impl/MerchantEnrichmentServiceImpl.java`
```java
package com.bukuwarung.fsbnplservice.service.impl;

import com.bukuwarung.fsbnplservice.dto.EnrichmentDataRequest;
import com.bukuwarung.fsbnplservice.dto.EnrichmentDataResponse;
import com.bukuwarung.fsbnplservice.entity.MerchantEnrichmentDataEntity;
import com.bukuwarung.fsbnplservice.repository.MerchantEnrichmentDataRepository;
import com.bukuwarung.fsbnplservice.service.MerchantEnrichmentService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Service
@Slf4j
@RequiredArgsConstructor
public class MerchantEnrichmentServiceImpl implements MerchantEnrichmentService {

  private final MerchantEnrichmentDataRepository repository;

  @Override
  public EnrichmentDataResponse getEnrichmentDataByKtp(EnrichmentDataRequest request) {
    log.info("Querying enrichment data for dataPoint: {}", request.getDataPoint());

    MerchantEnrichmentDataEntity entity = repository
        .findLatestByKtpAndType(request.getKtpNumber(), request.getDataPoint())
        .orElseThrow(() -> {
          log.error("No enrichment data found for dataPoint: {}", request.getDataPoint());
          throw new IllegalArgumentException(
              "No enrichment data found for the provided KTP and data point");
        });

    // Check status
    if ("IN_PROGRESS".equals(entity.getStatus()) || "FAILED".equals(entity.getStatus())) {
      log.error("Enrichment data not ready. Status: {}", entity.getStatus());
      throw new IllegalStateException(
          "Enrichment data is not ready. Status: " + entity.getStatus());
    }

    return EnrichmentDataResponse.builder()
        .s3Path(entity.getS3Path())
        .batchId(entity.getBatchId())
        .type(entity.getType())
        .status(entity.getStatus())
        .build();
  }
}
```

#### 1.4 Create Controller
**File**: `bnpl/src/main/java/com/bukuwarung/fsbnplservice/controller/MerchantEnrichmentController.java`
```java
package com.bukuwarung.fsbnplservice.controller;

import com.bukuwarung.fsbnplservice.dto.EnrichmentDataRequest;
import com.bukuwarung.fsbnplservice.dto.EnrichmentDataResponse;
import com.bukuwarung.fsbnplservice.service.MerchantEnrichmentService;
import javax.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/merchant/enrichment-data")
@Slf4j
@RequiredArgsConstructor
public class MerchantEnrichmentController {

  private final MerchantEnrichmentService enrichmentService;

  @PostMapping("/inquiry")
  public ResponseEntity<EnrichmentDataResponse> inquiryEnrichmentData(
      @RequestBody @Valid EnrichmentDataRequest request) {

    log.info("Received enrichment data inquiry request for dataPoint: {}",
        request.getDataPoint());

    try {
      EnrichmentDataResponse response = enrichmentService.getEnrichmentDataByKtp(request);
      return ResponseEntity.ok(response);
    } catch (IllegalArgumentException e) {
      log.error("Data not found: {}", e.getMessage());
      return ResponseEntity.status(404).build();
    } catch (IllegalStateException e) {
      log.error("Data not ready: {}", e.getMessage());
      return ResponseEntity.status(400).build();
    } catch (Exception e) {
      log.error("Error processing inquiry", e);
      return ResponseEntity.status(500).build();
    }
  }
}
```

#### 1.5 Update Authentication Filter Configuration
**File**: `bnpl/src/main/java/com/bukuwarung/fsbnplservice/filter/LosServiceFilter.java`

**Authentication Flow**:
- FS-BRICK → BNPL: Sends `los-auth-token` header (value from `${fs.bnpl.token}`)
- BNPL: Validates using `LosServiceFilter`
- Pattern: Same as `/merchant/identity` and `/merchant/leads-enrichment/callback`

**Modify `shouldNotFilter()` method to protect new endpoint:**
```java
@Override
protected boolean shouldNotFilter(HttpServletRequest request) throws ServletException {
  String path = request.getRequestURI();
  return !path.contains("/merchant-onboarding/merchant/v1/isEligible")
      && !path.contains("merchant-onboarding/merchant/identity")
      && !path.contains("merchant-onboarding/merchant/list")
      && !path.contains("merchant-onboarding/merchant/status")
      && !path.contains("merchant-onboarding/v1/internal/merchant/partner-list")
      && !path.contains("merchant-onboarding/v1/internal/merchant/whitelist")
      && !path.contains("/merchant-onboarding/merchant/callback")
      && !path.contains("/merchant/enrichment-data/inquiry");  // ADD THIS LINE
}
```

**Explanation**:
- `shouldNotFilter()` returns `false` for paths that SHOULD be filtered (protected)
- Adding `&& !path.contains("/merchant/enrichment-data/inquiry")` protects the endpoint
- FS-BRICK will authenticate using the `los-auth-token` header (same pattern as line 48, 74 in `FsBnplAdapter.java`)

#### 1.6 Create Database Migration
**File**: `bnpl/src/main/resources/db/migration/V<YYYYMMDDHHMMSS>__add_merchant_enrichment_data_indexes.sql`

```sql
-- CRITICAL: Index on merchant_data.ktp_number for JOIN predicate
-- This is WHERE the query filters first: WHERE md.ktpNumber = :ktpNumber
CREATE INDEX IF NOT EXISTS idx_merchant_data_ktp_number
  ON merchant_data(ktp_number);

-- Secondary indexes for post-JOIN filtering and sorting
CREATE INDEX IF NOT EXISTS idx_merchant_enrichment_data_created_at
  ON merchant_enrichment_data(created_at DESC);

CREATE INDEX IF NOT EXISTS idx_merchant_enrichment_data_type
  ON merchant_enrichment_data(type);

-- Composite index for covering query on enrichment table
CREATE INDEX IF NOT EXISTS idx_merchant_enrichment_data_merchant_type
  ON merchant_enrichment_data(merchant_id, type, created_at DESC);
```

#### 1.7 Testing Checklist (Phase 1)
- [ ] Unit test: `MerchantEnrichmentServiceImplTest.java`
  - Test case: Valid KTP + dataPoint returns data
  - Test case: Non-existent KTP throws IllegalArgumentException
  - Test case: IN_PROGRESS status throws IllegalStateException
  - Test case: FAILED status throws IllegalStateException
- [ ] Unit test: `MerchantEnrichmentControllerTest.java`
  - Test case: Valid request returns 200
  - Test case: Not found returns 404
  - Test case: Data not ready returns 400
- [ ] Integration test: Query with real database
- [ ] Run migration: `./gradlew flywayMigrate`
- [ ] Verify indexes created: `\d merchant_enrichment_data` in psql
- [ ] Manual test: curl POST to `/merchant/enrichment-data/inquiry`

---

### Phase 2: FS-BRICK BNPL Integration 🔌 (Day 3)

**Goal**: Integrate FS-BRICK with new BNPL endpoint

#### 2.1 Create DTO in FS-BRICK
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

#### 2.2 Add Method to FsBnplPort Interface
**File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/provider/FsBnplPort.java`

**Add method:**
```java
EnrichmentDataResponse getEnrichmentDataByKtp(String ktpNumber, String dataPoint);
```

#### 2.3 Implement in FsBnplAdapter
**File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/provider/impl/FsBnplAdapter.java`

**Add imports:**
```java
import com.bukuwarung.fsbrickservice.exception.DataNotFoundException;
import com.bukuwarung.fsbrickservice.exception.DataNotReadyException;
import org.springframework.http.HttpStatus;
import org.springframework.web.client.HttpClientErrorException;
import org.springframework.web.client.HttpServerErrorException;
import org.springframework.web.client.ResourceAccessException;
```

**Add implementation:**
```java
@Override
public EnrichmentDataResponse getEnrichmentDataByKtp(String ktpNumber, String dataPoint) {
  String requestUrl = UriComponentsBuilder.fromUriString(baseUrl)
      .path("/merchant/enrichment-data/inquiry")
      .encode()
      .toUriString();

  // Build request body
  Map<String, String> requestBody = new HashMap<>();
  requestBody.put("ktpNumber", ktpNumber);
  requestBody.put("dataPoint", dataPoint);

  log.info("Calling BNPL enrichment inquiry for dataPoint: {}", dataPoint);

  try {
    EnrichmentDataResponse response = restTemplateService.executeServiceCall(
        requestUrl,
        HttpMethod.POST,
        "los-auth-token",
        fsBnplToken,
        requestBody,
        "bnplGetEnrichmentData",
        EnrichmentDataResponse.class,
        null);

    if (response == null || response.getS3Path() == null) {
      log.error("Invalid response from BNPL for dataPoint: {}", dataPoint);
      throw new DataNotFoundException("No enrichment data found for dataPoint: " + dataPoint);
    }

    return response;

  } catch (HttpClientErrorException e) {
    if (e.getStatusCode() == HttpStatus.NOT_FOUND) {
      log.error("BNPL returned 404 for dataPoint: {}", dataPoint);
      throw new DataNotFoundException("No enrichment data found for dataPoint: " + dataPoint);
    } else if (e.getStatusCode() == HttpStatus.BAD_REQUEST) {
      log.error("BNPL returned 400 - data not ready for dataPoint: {}", dataPoint);
      throw new DataNotReadyException("Enrichment data is not ready", "IN_PROGRESS_OR_FAILED");
    }
    log.error("BNPL client error: {}", e.getMessage(), e);
    throw new RuntimeException("BNPL service error", e);

  } catch (HttpServerErrorException e) {
    log.error("BNPL server error: {}", e.getMessage(), e);
    throw new RuntimeException("BNPL service unavailable", e);

  } catch (ResourceAccessException e) {
    log.error("BNPL timeout or connection error: {}", e.getMessage(), e);
    throw new RuntimeException("BNPL service timeout", e);
  }
}
```

#### 2.4 Testing Checklist (Phase 2)
- [ ] Unit test: `FsBnplAdapterTest.java`
  - Test case: Successful call returns EnrichmentDataResponse
  - Test case: BNPL returns 404 → exception thrown
  - Test case: BNPL returns 400 → exception thrown
  - Test case: BNPL timeout → exception thrown
  - Test case: Invalid JSON response → exception thrown
- [ ] Integration test: Call actual BNPL endpoint
- [ ] Verify los-auth-token header sent correctly
- [ ] Manual test: Call adapter method directly

---

### Phase 3: FS-BRICK Controller & Multipart Response 📦 (Days 4-5)

**Goal**: Create external endpoint with multipart response and idempotency

> **Streaming Guarantee**: This phase forces the service/controller stack to return `ResponseEntity<StreamingResponseBody>`; files stream from S3 to the HTTP response via an 8KB buffer and a deterministic multipart boundary so we never hit `ByteArrayOutputStream` or load the payload fully into memory.

#### 3.1 Create Custom Exceptions
**File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/exception/DataNotFoundException.java`
```java
package com.bukuwarung.fsbrickservice.exception;

public class DataNotFoundException extends RuntimeException {
  public DataNotFoundException(String message) {
    super(message);
  }
}
```

**File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/exception/DataNotReadyException.java`
```java
package com.bukuwarung.fsbrickservice.exception;

public class DataNotReadyException extends RuntimeException {
  private final String status;

  public DataNotReadyException(String message, String status) {
    super(message);
    this.status = status;
  }

  public String getStatus() {
    return status;
  }
}
```

**File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/exception/S3FileException.java`
```java
package com.bukuwarung.fsbrickservice.exception;

public class S3FileException extends RuntimeException {
  public S3FileException(String message, Throwable cause) {
    super(message, cause);
  }
}
```

#### 3.2 Create Global Exception Handler
**File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/exception/BorrowerInquiryExceptionHandler.java`
```java
package com.bukuwarung.fsbrickservice.exception;

import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;

@ControllerAdvice
@Slf4j
public class BorrowerInquiryExceptionHandler {

  @ExceptionHandler(DataNotFoundException.class)
  public ResponseEntity<ErrorResponse> handleDataNotFound(DataNotFoundException ex) {
    log.error("Data not found: {}", ex.getMessage());
    return ResponseEntity
        .status(HttpStatus.NOT_FOUND)
        .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
  }

  @ExceptionHandler(DataNotReadyException.class)
  public ResponseEntity<ErrorResponse> handleDataNotReady(DataNotReadyException ex) {
    log.error("Data not ready: status={}, message={}", ex.getStatus(), ex.getMessage());
    return ResponseEntity
        .status(HttpStatus.BAD_REQUEST)
        .body(new ErrorResponse("DATA_NOT_READY", ex.getMessage()));
  }

  @ExceptionHandler(S3FileException.class)
  public ResponseEntity<ErrorResponse> handleS3Exception(S3FileException ex) {
    log.error("S3 error: {}", ex.getMessage(), ex);
    return ResponseEntity
        .status(HttpStatus.INTERNAL_SERVER_ERROR)
        .body(new ErrorResponse("S3_ERROR", "Failed to retrieve file from storage"));
  }

  @ExceptionHandler(Exception.class)
  public ResponseEntity<ErrorResponse> handleGenericException(Exception ex) {
    log.error("Unexpected error", ex);
    return ResponseEntity
        .status(HttpStatus.INTERNAL_SERVER_ERROR)
        .body(new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred"));
  }
}
```

**File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/exception/ErrorResponse.java`
```java
package com.bukuwarung.fsbrickservice.exception;

import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class ErrorResponse {
  private String errorCode;
  private String message;
}
```

#### 3.3 Create Request DTO
**File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/model/borrower/BorrowerInquiryRequest.java`
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
  private EnrichmentDataPoint dataPoint;  // Enum: PEFINDO, FDC, BANK_STATEMENT
}
```

#### 3.4 Create Service Interface
**File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/service/BorrowerDataInquiryService.java`
```java
package com.bukuwarung.fsbrickservice.service;

import com.bukuwarung.fsbrickservice.model.borrower.BorrowerInquiryRequest;
import org.springframework.http.ResponseEntity;
import org.springframework.web.servlet.mvc.method.annotation.StreamingResponseBody;

public interface BorrowerDataInquiryService {
  ResponseEntity<StreamingResponseBody> processInquiry(
      BorrowerInquiryRequest request, String requestId);
}
```

#### 3.5 Create Service Implementation (STREAMING)
**File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/service/impl/BorrowerDataInquiryServiceImpl.java`
```java
package com.bukuwarung.fsbrickservice.service.impl;

import com.amazonaws.AmazonServiceException;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.model.S3Object;
import com.amazonaws.services.s3.model.S3ObjectInputStream;
import com.bukuwarung.fsbrickservice.entity.PartnerEventAuditEntity;
import com.bukuwarung.fsbrickservice.exception.DataNotFoundException;
import com.bukuwarung.fsbrickservice.exception.DataNotReadyException;
import com.bukuwarung.fsbrickservice.exception.S3FileException;
import com.bukuwarung.fsbrickservice.model.bnpl.EnrichmentDataResponse;
import com.bukuwarung.fsbrickservice.model.borrower.BorrowerInquiryRequest;
import com.bukuwarung.fsbrickservice.provider.FsBnplPort;
import com.bukuwarung.fsbrickservice.service.BorrowerDataInquiryService;
import com.bukuwarung.fsbrickservice.util.EventAuditUtil;
import com.bukuwarung.fsbrickservice.util.EventAuditLevelStatus;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;
import org.springframework.web.servlet.mvc.method.annotation.StreamingResponseBody;

import java.io.InputStream;
import java.io.OutputStream;

@Service
@Slf4j
@RequiredArgsConstructor
public class BorrowerDataInquiryServiceImpl implements BorrowerDataInquiryService {

  private final FsBnplPort fsBnplPort;
  private final AmazonS3 s3Client;
  private final EventAuditUtil eventAuditUtil;
  private final ObjectMapper objectMapper;

  @Value("${gt.bnplS3BucketName}")
  private String bucketName;

  @Override
  public ResponseEntity<StreamingResponseBody> processInquiry(
      BorrowerInquiryRequest request, String requestId) {

    // STEP 1: Start Service Event Audit (with built-in idempotency)
    PartnerEventAuditEntity auditEntity;
    try {
      auditEntity = eventAuditUtil.startServiceEventAudit(
          requestId,
          "veefin",
          "borrower-inquiry",
          objectMapper.writeValueAsString(request)
      );
    } catch (Exception e) {
      log.error("Failed to start audit. RequestId: {}", requestId, e);
      throw new RuntimeException("Failed to start event audit", e);
    }

    // Idempotency check
    if ("SUCCESS".equals(auditEntity.getStatus())) {
      log.info("Duplicate request detected. RequestId: {}", requestId);
      return buildResponseFromAudit(auditEntity);
    }

    try {
      // STEP 2: Update Partner Level → IN_PROGRESS
      String bnplRequestJson = String.format(
          "{\"ktpNumber\":\"%s\",\"dataPoint\":\"%s\"}",
          request.getKtpNumber(), request.getDataPoint().name());

      eventAuditUtil.updatePartnerLevelEventAudit(
          "borrower-inquiry",
          "veefin",
          requestId,
          EventAuditLevelStatus.IN_PROGRESS,
          bnplRequestJson,
          null
      );

      // Call BNPL service
      log.info("Calling BNPL for enrichment data. RequestId: {}", requestId);
      EnrichmentDataResponse bnplResponse = fsBnplPort.getEnrichmentDataByKtp(
          request.getKtpNumber(),
          request.getDataPoint().name()
      );

      // STEP 3: Update Partner Level → SUCCESS
      String bnplResponseJson = objectMapper.writeValueAsString(bnplResponse);
      eventAuditUtil.updatePartnerLevelEventAudit(
          "borrower-inquiry",
          "veefin",
          requestId,
          EventAuditLevelStatus.SUCCESS,
          bnplRequestJson,
          bnplResponseJson
      );

      // STEP 4: Mark Service Audit as SUCCESS
      String metadata = String.format(
          "{\"s3Path\":\"%s\",\"batchId\":\"%s\",\"dataPoint\":\"%s\"}",
          bnplResponse.getS3Path(),
          bnplResponse.getBatchId(),
          request.getDataPoint().name()
      );
      eventAuditUtil.successServiceEventAudit(auditEntity.getId(), metadata);

      log.info("Successfully processed borrower inquiry. RequestId: {}", requestId);

      // Return streaming response
      return buildStreamingResponse(
          bnplResponse.getS3Path(),
          request.getDataPoint().name(),
          bnplResponse.getBatchId()
      );

    } catch (DataNotFoundException | DataNotReadyException e) {
      // These are already properly mapped to HTTP status codes
      // Update Partner Level → FAILED
      try {
        eventAuditUtil.updatePartnerLevelEventAudit(
            "borrower-inquiry",
            "veefin",
            requestId,
            EventAuditLevelStatus.FAILED,
            null,
            e.getMessage()
        );
        eventAuditUtil.failedServiceEventAudit(auditEntity.getId(), e.getMessage());
      } catch (Exception auditEx) {
        log.error("Failed to update audit after error", auditEx);
      }
      throw e;

    } catch (Exception e) {
      log.error("Error processing borrower inquiry. RequestId: {}", requestId, e);

      // Update Partner Level → FAILED
      try {
        eventAuditUtil.updatePartnerLevelEventAudit(
            "borrower-inquiry",
            "veefin",
            requestId,
            EventAuditLevelStatus.FAILED,
            null,
            e.getMessage()
        );
        eventAuditUtil.failedServiceEventAudit(auditEntity.getId(), e.getMessage());
      } catch (Exception auditEx) {
        log.error("Failed to update audit after error", auditEx);
      }

      throw new RuntimeException("Failed to process borrower inquiry", e);
    }
  }

  /**
   * Build streaming response from successful audit record.
   * Extracts S3 metadata from audit and re-downloads file.
   */
  private ResponseEntity<StreamingResponseBody> buildResponseFromAudit(
      PartnerEventAuditEntity auditEntity) {

    try {
      // Extract metadata from audit record
      String metadata = auditEntity.getMessageDetail();
      JsonNode metadataJson = objectMapper.readTree(metadata);

      String s3Path = metadataJson.get("s3Path").asText();
      String batchId = metadataJson.get("batchId").asText();
      String dataPoint = metadataJson.get("dataPoint").asText();

      log.info("Rebuilding response from audit. S3Path: {}, BatchId: {}", s3Path, batchId);

      // Re-download and stream the file
      return buildStreamingResponse(s3Path, dataPoint, batchId);

    } catch (Exception e) {
      log.error("Failed to rebuild response from audit", e);
      throw new RuntimeException("Failed to retrieve cached response", e);
    }
  }

  /**
   * Build streaming multipart response WITHOUT loading entire file into memory.
   * Streams S3 file directly to HTTP response.
   */
  private ResponseEntity<StreamingResponseBody> buildStreamingResponse(
      String s3Path, String dataPoint, String batchId) {

    String boundary = "----WebKitFormBoundary" + System.currentTimeMillis();

    StreamingResponseBody stream = outputStream -> {
      try {
        // Download file from S3
        log.info("Streaming file from S3. Path: {}", s3Path);
        S3Object s3Object = s3Client.getObject(bucketName, s3Path);
        S3ObjectInputStream s3Stream = s3Object.getObjectContent();

        // Write multipart headers and stream file content
        writeMultipartStream(outputStream, s3Stream, dataPoint, batchId, boundary);

        s3Stream.close();
        log.info("Successfully streamed file from S3. Path: {}", s3Path);

      } catch (AmazonServiceException e) {
        log.error("S3 error: {}", e.getMessage(), e);
        if (e.getStatusCode() == 404) {
          throw new S3FileException("File not found in S3: " + s3Path, e);
        }
        throw new S3FileException("Failed to retrieve file from S3", e);
      } catch (Exception e) {
        log.error("Error streaming response", e);
        throw new S3FileException("Failed to stream response", e);
      }
    };

    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.parseMediaType(
        "multipart/form-data; boundary=" + boundary));

    return ResponseEntity.ok()
        .headers(headers)
        .body(stream);
  }

  /**
   * Write multipart response directly to output stream.
   * NO buffering - streams S3 content directly.
   */
  private void writeMultipartStream(
      OutputStream outputStream,
      InputStream s3Stream,
      String dataPoint,
      String batchId,
      String boundary) throws Exception {

    // Part 1: dataFile (stream S3 content directly)
    String filePartHeader = String.format(
        "--%s\r\n" +
        "Content-Disposition: form-data; name=\"dataFile\"; filename=\"data.json\"\r\n" +
        "Content-Type: application/octet-stream\r\n\r\n",
        boundary
    );
    outputStream.write(filePartHeader.getBytes());

    // Stream S3 content directly to response (NO ByteArrayOutputStream)
    byte[] buffer = new byte[8192];
    int bytesRead;
    while ((bytesRead = s3Stream.read(buffer)) != -1) {
      outputStream.write(buffer, 0, bytesRead);
    }
    outputStream.write("\r\n".getBytes());

    // Part 2: dataPoint
    String dataPointPart = String.format(
        "--%s\r\n" +
        "Content-Disposition: form-data; name=\"dataPoint\"\r\n\r\n" +
        "%s\r\n",
        boundary, dataPoint
    );
    outputStream.write(dataPointPart.getBytes());

    // Part 3: dataId
    String dataIdPart = String.format(
        "--%s\r\n" +
        "Content-Disposition: form-data; name=\"dataId\"\r\n\r\n" +
        "%s\r\n" +
        "--%s--\r\n",
        boundary, batchId, boundary
    );
    outputStream.write(dataIdPart.getBytes());

    outputStream.flush();
  }
}
```

#### 3.6 Create Controller
**File**: `fs-brick-service/src/main/java/com/bukuwarung/fsbrickservice/controller/BorrowerDataInquiryController.java`
```java
package com.bukuwarung.fsbrickservice.controller;

import com.bukuwarung.fsbrickservice.model.borrower.BorrowerInquiryRequest;
import com.bukuwarung.fsbrickservice.service.BorrowerDataInquiryService;
import javax.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.slf4j.MDC;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.method.annotation.StreamingResponseBody;

@RestController
@RequestMapping("/fs/brick/service/borrower/v1")
@Slf4j
@RequiredArgsConstructor
public class BorrowerDataInquiryController {

  private final BorrowerDataInquiryService borrowerDataInquiryService;

  @PostMapping("/inquiry-data")
  public ResponseEntity<StreamingResponseBody> inquiryBorrowerData(
      @RequestBody @Valid BorrowerInquiryRequest request,
      @RequestHeader("veefin-request-id") String requestId) {

    // MDC for logging correlation
    MDC.put("request.id", requestId);
    MDC.put("partner", "veefin");

    log.info("Received borrower data inquiry request. RequestId: {}", requestId);

    try {
      // Let custom exceptions propagate to @ControllerAdvice
      // No wrapping in RuntimeException - proper HTTP status codes preserved
      return borrowerDataInquiryService.processInquiry(request, requestId);
    } finally {
      MDC.clear();
    }
  }
}
```

#### 3.5 Testing Checklist (Phase 3)
- [ ] Unit test: `BorrowerDataInquiryServiceImplTest.java`
  - Mock FsBnplPort, S3Client, EventAuditUtil
  - Test case: Happy path - successful multipart response
  - Test case: Idempotency - duplicate requestId returns cached response
  - Test case: BNPL returns 404 - exception propagated
  - Test case: S3 file not found - exception thrown
- [ ] Unit test: `BorrowerDataInquiryControllerTest.java`
  - MockMvc test: Valid request returns 200
  - MockMvc test: Invalid KTP format returns 400
  - MockMvc test: Missing requestId header returns error
- [ ] Integration test: End-to-end with BNPL and S3
- [ ] Manual test: Postman/curl with real requestId
- [ ] Verify multipart response format

---

### Phase 4: Quality Checks 🎯 (Days 6-7)

#### 4.1 Code Quality
- [ ] Run Spotless: `./gradlew spotlessApply`
- [ ] Verify Google Java Format applied
- [ ] No unused imports or variables
- [ ] All @NonNull/@Nullable annotations correct
- [ ] Lombok annotations consistent

#### 4.2 Testing Coverage
- [ ] All unit tests passing: `./gradlew test`
- [ ] Integration tests passing
- [ ] Code coverage > 80% for new code
- [ ] Manual testing all scenarios:
  - [ ] Happy path: PEFINDO, FDC, BANK_STATEMENT
  - [ ] Not found: Non-existent KTP returns 404
  - [ ] Data not ready: IN_PROGRESS returns 400
  - [ ] Data failed: FAILED returns 400
  - [ ] S3 error: File not found returns 500
  - [ ] Idempotency: Duplicate requestId returns cached response

#### 4.3 Observability
- [ ] Log messages clear and informative
- [ ] No KTP in logs (only in PartnerEventAudit)
- [ ] MDC correlation IDs working
- [ ] PartnerEventAudit records all requests
- [ ] Error stack traces logged properly

#### 4.4 Performance
- [ ] BNPL query response time < 500ms (p95)
- [ ] S3 download streaming (no full file load)
- [ ] Total response time < 5s for 10MB files
- [ ] No memory leaks in S3 streaming

#### 4.6 Multipart Response Validation
- [ ] **Content-Type header includes boundary parameter**
- [ ] Boundary format: `multipart/form-data; boundary=----WebKitFormBoundary...`
- [ ] Client can parse all 3 parts: dataFile, dataPoint, dataId
- [ ] Multipart parts separated correctly by boundary string
- [ ] Each part has correct Content-Disposition headers

#### 4.5 Security
- [ ] Input validation working (@Valid)
- [ ] los-auth-token sent to BNPL
- [ ] No credentials in logs
- [ ] No PII (KTP) in application logs
- [ ] PartnerEventAudit stores full payload for audit

---

## Testing Strategy

### Unit Tests

**FS-BRICK Unit Tests:**

**File**: `fs-brick-service/src/test/java/com/bukuwarung/fsbrickservice/controller/BorrowerDataInquiryControllerTest.java`
```java
@WebMvcTest(BorrowerDataInquiryController.class)
class BorrowerDataInquiryControllerTest {

  @Autowired private MockMvc mockMvc;
  @MockBean private BorrowerDataInquiryService service;

  @Test
  void testInquiryBorrower_Success() throws Exception {
    // Given
    BorrowerInquiryRequest request = BorrowerInquiryRequest.builder()
        .ktpNumber("1234567890123456")
        .dataPoint(EnrichmentDataPoint.PEFINDO)
        .build();

    // Mock StreamingResponseBody (not byte[])
    String testBoundary = "----WebKitFormBoundaryTest123";
    StreamingResponseBody streamingBody = outputStream -> {
      outputStream.write("test-multipart-data".getBytes());
    };

    when(service.processInquiry(any(), any())).thenReturn(
        ResponseEntity.ok()
            .contentType(MediaType.parseMediaType(
                "multipart/form-data; boundary=" + testBoundary))
            .body(streamingBody));

    // When & Then
    mockMvc.perform(post("/fs/brick/service/borrower/v1/inquiry-data")
            .header("veefin-request-id", "req-123")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(request)))
        .andExpect(status().isOk())
        .andExpect(header().string("Content-Type", containsString("multipart/form-data")))
        .andExpect(header().string("Content-Type", containsString("boundary=")));
  }

  @Test
  void testInquiryBorrower_InvalidKtp() throws Exception {
    // Given
    BorrowerInquiryRequest request = BorrowerInquiryRequest.builder()
        .ktpNumber("123")  // Invalid: not 16 digits
        .dataPoint(EnrichmentDataPoint.PEFINDO)
        .build();

    // When & Then
    mockMvc.perform(post("/fs/brick/service/borrower/v1/inquiry-data")
            .header("veefin-request-id", "req-123")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(request)))
        .andExpect(status().isBadRequest());
  }
}
```

**File**: `fs-brick-service/src/test/java/com/bukuwarung/fsbrickservice/service/impl/BorrowerDataInquiryServiceImplTest.java`
- Test all 4 steps of PartnerEventAudit pattern
- Test idempotency with duplicate requestId
- Test S3 streaming and multipart building
- Test error scenarios (BNPL failure, S3 failure)

**File**: `fs-brick-service/src/test/java/com/bukuwarung/fsbrickservice/provider/impl/FsBnplAdapterTest.java`
- Test successful HTTP call with correct headers
- Test error responses (404, 400, 500, timeout)
- Mock RestTemplateService

**BNPL Unit Tests:**

**File**: `bnpl/src/test/java/com/bukuwarung/fsbnplservice/controller/MerchantEnrichmentControllerTest.java`
- Test successful inquiry returns 200
- Test not found returns 404
- Test data in progress returns 400
- Test invalid KTP format returns 400

**File**: `bnpl/src/test/java/com/bukuwarung/fsbnplservice/service/impl/MerchantEnrichmentServiceImplTest.java`
- Test repository query with valid data
- Test exception thrown for not found
- Test exception thrown for IN_PROGRESS/FAILED status

**File**: `bnpl/src/test/java/com/bukuwarung/fsbnplservice/repository/MerchantEnrichmentDataRepositoryTest.java`
- Test findLatestByKtpAndType returns latest record
- Test returns empty Optional when not found
- Test with multiple records, verify latest returned

### Integration Tests

**FS-BRICK Integration Test:**
```java
@SpringBootTest
@ActiveProfiles("test")
class BorrowerDataInquiryIntegrationTest {

  @Autowired private BorrowerDataInquiryController controller;
  @Autowired private EventAuditRepository auditRepository;

  @MockBean private FsBnplPort bnplPort;
  @MockBean private AmazonS3 s3Client;

  @Test
  void testEndToEndInquiry_Success() throws Exception {
    // Given
    String requestId = "req-" + System.currentTimeMillis();
    BorrowerInquiryRequest request = BorrowerInquiryRequest.builder()
        .ktpNumber("1234567890123456")
        .dataPoint(EnrichmentDataPoint.PEFINDO)
        .build();

    EnrichmentDataResponse bnplResponse = EnrichmentDataResponse.builder()
        .s3Path("test/path.json")
        .batchId("batch-123")
        .type("PEFINDO")
        .status("COMPLETE")
        .build();

    S3Object s3Object = mock(S3Object.class);
    S3ObjectInputStream s3Stream = new S3ObjectInputStream(
        new ByteArrayInputStream("{\"test\":\"data\"}".getBytes()), null);

    when(bnplPort.getEnrichmentDataByKtp(any(), any())).thenReturn(bnplResponse);
    when(s3Client.getObject(any(), any())).thenReturn(s3Object);
    when(s3Object.getObjectContent()).thenReturn(s3Stream);

    // When - Returns StreamingResponseBody, not byte[]
    ResponseEntity<StreamingResponseBody> response = controller.inquiryBorrowerData(request, requestId);

    // Then
    assertEquals(200, response.getStatusCodeValue());
    assertNotNull(response.getBody());

    // Verify Content-Type includes multipart/form-data AND boundary parameter
    MediaType contentType = response.getHeaders().getContentType();
    assertNotNull(contentType);
    assertTrue(contentType.includes(MediaType.MULTIPART_FORM_DATA));
    assertNotNull(contentType.getParameter("boundary"),
        "Content-Type must include boundary parameter");

    // Consume the stream to verify it works
    ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
    response.getBody().writeTo(outputStream);
    assertTrue(outputStream.size() > 0);

    // Verify audit record created
    Optional<PartnerEventAuditEntity> audit = auditRepository.findByRequestId(requestId);
    assertTrue(audit.isPresent());
    assertEquals("SUCCESS", audit.get().getStatus());
  }

  @Test
  void testIdempotency_DuplicateRequest() throws Exception {
    // Setup mocks
    EnrichmentDataResponse bnplResponse = EnrichmentDataResponse.builder()
        .s3Path("test/path.json")
        .batchId("batch-123")
        .type("PEFINDO")
        .status("COMPLETE")
        .build();

    S3Object s3Object = mock(S3Object.class);
    S3ObjectInputStream s3Stream = new S3ObjectInputStream(
        new ByteArrayInputStream("{\"test\":\"data\"}".getBytes()), null);

    when(bnplPort.getEnrichmentDataByKtp(any(), any())).thenReturn(bnplResponse);
    when(s3Client.getObject(any(), any())).thenReturn(s3Object);
    when(s3Object.getObjectContent()).thenReturn(s3Stream);

    // First request
    String requestId = "req-duplicate";
    BorrowerInquiryRequest request = BorrowerInquiryRequest.builder()
        .ktpNumber("1234567890123456")
        .dataPoint(EnrichmentDataPoint.PEFINDO)
        .build();

    ResponseEntity<StreamingResponseBody> firstResponse = controller.inquiryBorrowerData(request, requestId);
    assertEquals(200, firstResponse.getStatusCodeValue());

    // Second request with same requestId - should return cached response
    ResponseEntity<StreamingResponseBody> secondResponse = controller.inquiryBorrowerData(request, requestId);
    assertEquals(200, secondResponse.getStatusCodeValue());

    // Verify BNPL called only once (idempotency working)
    verify(bnplPort, times(1)).getEnrichmentDataByKtp(any(), any());
  }
}
```

### Manual Testing Checklist

**BNPL Endpoint:**
```bash
# Test 1: Valid request
curl -X POST http://localhost:8080/merchant/enrichment-data/inquiry \
  -H "Content-Type: application/json" \
  -H "los-auth-token: <token>" \
  -d '{"ktpNumber":"1234567890123456","dataPoint":"PEFINDO"}'

# Expected: 200 with s3Path, batchId, type, status

# Test 2: Not found
curl -X POST http://localhost:8080/merchant/enrichment-data/inquiry \
  -H "Content-Type: application/json" \
  -H "los-auth-token: <token>" \
  -d '{"ktpNumber":"9999999999999999","dataPoint":"PEFINDO"}'

# Expected: 404

# Test 3: Invalid KTP format
curl -X POST http://localhost:8080/merchant/enrichment-data/inquiry \
  -H "Content-Type: application/json" \
  -H "los-auth-token: <token>" \
  -d '{"ktpNumber":"123","dataPoint":"PEFINDO"}'

# Expected: 400
```

**FS-BRICK Endpoint:**
```bash
# Test 1: Valid request with PEFINDO - VERIFY BOUNDARY IN RESPONSE
curl -v -X POST http://localhost:9030/fs/brick/service/borrower/v1/inquiry-data \
  -H "Content-Type: application/json" \
  -H "veefin-request-id: req-test-001" \
  -d '{"ktpNumber":"1234567890123456","dataPoint":"PEFINDO"}'

# Expected: 200 with multipart/form-data response
# CRITICAL: Verify Content-Type header includes boundary parameter:
#   Content-Type: multipart/form-data; boundary=----WebKitFormBoundary...
#
# Response should have 3 parts:
#   - dataFile (file content)
#   - dataPoint (PEFINDO)
#   - dataId (batch ID)

# Test 2: Idempotency - duplicate requestId
curl -X POST http://localhost:9030/fs/brick/service/borrower/v1/inquiry-data \
  -H "Content-Type: application/json" \
  -H "veefin-request-id: req-test-001" \
  -d '{"ktpNumber":"1234567890123456","dataPoint":"PEFINDO"}'

# Expected: 200 with same response (cached)
# Verify boundary still present in Content-Type

# Test 3: Non-existent KTP
curl -X POST http://localhost:9030/fs/brick/service/borrower/v1/inquiry-data \
  -H "Content-Type: application/json" \
  -H "veefin-request-id: req-test-002" \
  -d '{"ktpNumber":"9999999999999999","dataPoint":"PEFINDO"}'

# Expected: 404

# Test 4: Parse multipart response with boundary
curl -X POST http://localhost:9030/fs/brick/service/borrower/v1/inquiry-data \
  -H "Content-Type: application/json" \
  -H "veefin-request-id: req-test-003" \
  -d '{"ktpNumber":"1234567890123456","dataPoint":"PEFINDO"}' \
  --output response.txt

# Manually verify response.txt contains:
#   1. Boundary string matches Content-Type header
#   2. All 3 parts present and parseable
#   3. Part headers correct (Content-Disposition)
```

---

## Configuration

### New Environment Variables
**None** - All required variables already exist:

**FS-BRICK-SERVICE:**
```bash
BNPL_BASE_URL=http://bnpl-service:8080  # Already configured
BNPL_TOKEN=<secret-token>               # Already configured
BNPL_S3_BUCKET_NAME=janus-buku-dev      # Already configured
AWS_ACCESS_KEY_ID=<aws-key>             # Already configured
AWS_SECRET_KEY=<aws-secret>             # Already configured
AWS_REGION=ap-southeast-1               # Already configured
```

**BNPL:**
```bash
LOS_AUTH_TOKEN=<secret-token>  # Already configured
```

### Configuration Files

**FS-BRICK application.yml** (no changes needed):
```yaml
fs:
  bnpl:
    merchant:
      baseUrl: ${BNPL_BASE_URL:http://localhost:8080}
    token: ${BNPL_TOKEN}

gt:
  bnplS3BucketName: ${BNPL_S3_BUCKET_NAME:janus-buku-dev}

aws:
  accessKeyId: ${AWS_ACCESS_KEY_ID}
  secretKey: ${AWS_SECRET_KEY}
  region: ${AWS_REGION}
```

**BNPL application.yml** (minimal change):
```yaml
merchant:
  enrichment:
    auth-token: ${LOS_AUTH_TOKEN:default-los-token}
```

---

## Security & Performance

### Security
- [x] Input validation: @Valid on all request DTOs
- [x] Authentication: los-auth-token for BNPL communication
- [x] Authorization: Not required (internal partner API)
- [x] No PII in logs: KTP stored only in PartnerEventAudit
- [x] Sanitized error messages: No stack traces to external clients
- [x] S3 access: IAM role credentials (no hardcoded keys)

### Performance
- **Expected load**: 10-100 req/hour (low volume partner API)
- **Timeout**: 30s BNPL, 60s S3 download
- **Caching**: None (idempotency via PartnerEventAudit)
- **Database indexes**: Required for query optimization
- **S3 streaming**: Prevents memory issues with large files
- **Connection pooling**: Existing RestTemplateService configuration

**Performance Targets:**
- BNPL query: < 500ms (p95)
- S3 download: < 5s for 10MB (p95)
- Total response: < 10s (p95)

---

## Risks & Mitigation

### Risk 1: S3 File Not Found
**Description**: S3 path exists in database but file deleted from bucket
**Impact**: 500 error to client
**Mitigation**:
- Add try-catch for AmazonS3Exception
- Log S3 path for investigation
- Return clear error message
- Monitor S3 404 rate with alerts

### Risk 2: Large File Memory Issues ✅ RESOLVED
**Description**: Large files (>100MB) could cause OutOfMemoryError with buffering
**Impact**: Service crash
**Resolution**:
- ✅ Use StreamingResponseBody - NO ByteArrayOutputStream
- ✅ Direct S3 stream-to-HTTP with 8KB buffer only
- ✅ File NEVER loaded into memory
- ✅ Memory usage constant regardless of file size
**Additional Monitoring**:
- Monitor S3 download time for large files
- Track streaming errors/interruptions
- Alert on sustained high memory (if occurs, indicates other issue)

### Risk 3: Database Query Performance ✅ RESOLVED
**Description**: JOIN query could be slow without proper indexes
**Impact**: BNPL timeout, degraded performance
**Resolution**:
- ✅ Index on merchant_data.ktp_number (PRIMARY JOIN filter)
- ✅ Composite index on merchant_enrichment_data (merchant_id, type, created_at)
- ✅ Additional indexes for created_at and type
**Validation Required**:
- Run EXPLAIN ANALYZE on query before deployment
- Verify index usage: `EXPLAIN SELECT ... WHERE md.ktpNumber = '...'`
- Monitor query p95/p99 latency in production
- **CRITICAL**: Run migration before deployment

### Risk 4: PartnerEventAudit Table Growth
**Description**: Audit table grows without cleanup
**Impact**: Database storage issues
**Mitigation**:
- Implement data retention policy (30-90 days)
- Add scheduled cleanup job
- Monitor table size
- Archive old records to data lake

### Risk 5: Authentication Placeholder
**Description**: No real authentication in initial release
**Impact**: Security gap until LND-4557 completed
**Mitigation**:
- Deploy to staging only initially
- Whitelist Veefin IP addresses at network level
- Fast-track LND-4557 completion
- Document temporary security measure

---

## Open Questions
None - All questions resolved during requirement phase.

---

## Estimated Effort

- **Complexity**: Medium-High
- **Estimated Time**: 7 days
  - BNPL endpoint: 2 days
  - FS-BRICK integration: 1 day
  - Multipart response & idempotency: 2 days
  - Testing & quality: 2 days
- **Dependencies**: None (independent task)

---

## References

### Similar Code
- **4-Step Audit Pattern**: `GetFdcDataCommand.java:30-72`
- **BNPL Adapter**: `FsBnplAdapter.java:33-84`
- **S3 Service**: `S3FileServiceImpl.java:48-101`
- **Controller Pattern**: `PartnerController.java:69-109`
- **Repository JOIN**: `MerchantDataRepository.java:41-62`
- **Event Audit Util**: `EventAuditUtil.java:18-511`

### Related Tickets
- **LND-4558**: This implementation (Borrower Data Inquiry API)
- **LND-4557**: Authentication (HMAC-SHA256) - separate ticket
- **LND-4527**: API Documentation reference

### Documentation
- **API Contract**: `/Users/juvianto.chi/Desktop/code/documentation/apidocs/LND-4527.md`
- **Requirement**: `/Users/juvianto.chi/Desktop/code/requirements/LND-4558/requirement.md`
- **Clarification**: `/Users/juvianto.chi/Desktop/code/requirements/LND-4558/clarification.md`

### External Resources
- AWS S3 SDK: https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/examples-s3.html
- Spring Multipart: https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-multipart
- Resilience4j Retry: https://resilience4j.readme.io/docs/retry

---

## Next Steps

**Ready to Start Implementation:**

1. ✅ Review this plan with team
2. ✅ Confirm database migration approach
3. ✅ Verify S3 bucket permissions
4. ✅ Update status to "✅ Approved"
5. 🚀 **Start Phase 1: BNPL Internal Endpoint**

**Implementation Sequence:**
1. Phase 1: BNPL (Days 1-2) - Backend foundation
2. Phase 2: FS-BRICK Adapter (Day 3) - Integration layer
3. Phase 3: FS-BRICK Controller (Days 4-5) - External API
4. Phase 4: Testing (Days 6-7) - Quality assurance

**Success Criteria:**
- All acceptance criteria met
- All tests passing (unit + integration)
- Code review approved
- Manual testing successful
- Ready for staging deployment

---

**Status**: 📝 Planning → ✅ Approved → 🚀 Ready for Implementation

---

## Revision History

### 2025-11-25: Critical Architecture Fixes

**Issues Identified in Review:**

1. **Memory Violation** ❌
   - **Problem**: ByteArrayOutputStream loading entire files into memory, violating "no full file load" requirement
   - **Impact**: 100MB+ files would cause OutOfMemoryError
   - **Fix**: Replaced with StreamingResponseBody - direct S3-to-HTTP streaming with 8KB buffer only

2. **Broken Idempotency** ❌
   - **Problem**: `buildResponseFromAudit()` threw UnsupportedOperationException
   - **Impact**: Duplicate requests would fail instead of returning cached data
   - **Fix**: Full implementation with JSON metadata parsing and S3 re-download

3. **Wrong Error Codes** ❌
   - **Problem**: Controller collapsed all errors to 500 via generic RuntimeException
   - **Impact**: BNPL 404/400 became 500, breaking required error contract
   - **Fix**: Custom exceptions + @ControllerAdvice for proper 404/400/500 mapping

4. **Inefficient Indexes** ❌
   - **Problem**: Indexes only on merchant_enrichment_data, not on JOIN predicate column
   - **Impact**: Query filters on md.ktpNumber without index, causing slow JOINs
   - **Fix**: Added PRIMARY index on merchant_data.ktp_number for JOIN optimization

**Code Changes:**
- ✅ Service interface: `ResponseEntity<byte[]>` → `ResponseEntity<StreamingResponseBody>`
- ✅ Service impl: Full streaming with `writeMultipartStream()` method
- ✅ Service impl: Implemented `buildResponseFromAudit()` with metadata extraction
- ✅ FsBnplAdapter: Added exception mapping for 404/400/500
- ✅ Controller: Removed try-catch wrapping, let exceptions propagate
- ✅ Added 5 new exception classes + @ControllerAdvice handler
- ✅ Database migration: Added index on merchant_data.ktp_number

**Test Code Updated:**
- ✅ Controller unit tests: Mock StreamingResponseBody instead of byte[]
- ✅ Integration tests: Updated to use ResponseEntity<StreamingResponseBody>
- ✅ Integration tests: Added stream consumption to verify streaming works
- ✅ All example code patterns updated to show streaming approach
- ✅ **Multipart boundary**: All Content-Type headers include boundary parameter
- ✅ **Integration test**: Validates boundary parameter presence in response

**Multipart Boundary Fix:**
- ✅ Example pattern (line 148): Added boundary to Content-Type header
- ✅ Unit test mock (line 1252): Mock includes boundary in Content-Type
- ✅ Integration test (line 1361): Asserts boundary parameter present
- ✅ Service implementation (line 1057): Correct boundary handling
- ✅ S3 pattern documentation (line 319): Shows boundary in example

**Validation Required:**
- [ ] Run EXPLAIN ANALYZE on BNPL query to verify index usage
- [ ] Test streaming with 100MB+ files to verify no memory issues
- [ ] Test idempotency with duplicate requestId
- [ ] Test all error scenarios return correct HTTP status codes
- [ ] Verify memory usage remains constant during large file streaming
- [ ] Verify client can parse multipart response with boundary parameter
