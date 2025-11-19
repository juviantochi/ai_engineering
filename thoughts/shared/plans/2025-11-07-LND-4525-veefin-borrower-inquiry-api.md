# Veefin Borrower Data Inquiry API Implementation Plan

**Parent Ticket:** [LND-4525](https://bukuwarung.atlassian.net/browse/LND-4525)  
**Subtasks:** 
- [LND-4539](https://bukuwarung.atlassian.net/browse/LND-4539) - Credit Summary Storage (PREREQUISITE - Data Layer)
- [LND-4526](https://bukuwarung.atlassian.net/browse/LND-4526) - Veefin Token Filter & Capability Provider (PREREQUISITE - Auth Layer)
- [LND-4527](https://bukuwarung.atlassian.net/browse/LND-4527) - Related subtask

## Overview

Implement API layer for Veefin Borrower Data Inquiry that allows Veefin to query borrower credit information using KTP numbers. This plan covers the API endpoints and integration layer, building on top of the credit summary storage and authentication/capability provider infrastructure.

**Important**: This plan assumes **LND-4539 AND LND-4526 are completed first**, which implement:

**LND-4539 provides:**
- `merchant_credit_summary` table with credit metrics (scalar fields per Veefin requirements)
- Extraction logic from PEFINDO and FDC S3 JSONs
- Integration with enrichment callback to populate summaries

**LND-4526 provides:**
- `VeefinTokenFilter` for authentication (`veefin-x-token` header validation)
- `VeefinBorrowerInquiryProvider` extending `AbstractCapabilityProvider`
- Health monitoring integration (DB connectivity, BNPL `/ping`, data freshness)
- Feature flags, audit logging, circuit breaker patterns
- Veefin-specific controller and DTOs

## Relationship to RFC and Other Documents

**RFC Documents** (describe desired end state for LND-4525 parent):
- `bnpl/RFC/LND-4525-bnpl.md` - BNPL internal endpoint specification
- `fs-brick-service/RFC/LND-4525-fs-brick-service.md` - FS Brick external endpoint specification
- `documentation/apidocs/borrowers-data-inquiry-api-docs.md` - API contract

**Implementation Plans**:
- **LND-4539 Plan** (`thoughts/shared/plans/2025-11-07-LND-4539-merchant-credit-summary-storage.md`) - Data layer (prerequisite)
- **LND-4526 Plan** (`documentation/request/LND-4526/plan.md`) - Auth & Capability Provider layer (prerequisite)
- **This Plan (LND-4525)** - BNPL endpoint & FS Brick integration (final assembly)

## Current State Analysis

**Existing Infrastructure:**
- ✅ **Database**: Latest migration V59, PostgreSQL with proper indexes
- ✅ **Enrichment pipeline**: PEFINDO/FDC enrichment generates S3 JSONs and triggers callbacks
- ✅ **Enrichment callback**: `MerchantOnboardingServiceImpl#processMerchantEnrichmentDataCallback` updates `merchant_enrichment_data` table
- ✅ **Mutual TLS**: Infrastructure available for inter-service communication
- ✅ **Async processing**: `@EnableAsync` configured in both services
- ✅ **BNPL Health Check**: `/merchant/on-boarding/ping` endpoint exists

**What LND-4539 Provides (PREREQUISITE #1):**
- ✅ **`merchant_credit_summary` table**: Stores pre-computed credit metrics (V60 migration)
- ✅ **`MerchantCreditSummaryEntity`**: JPA entity for credit summary
- ✅ **`MerchantCreditSummaryRepository`**: Data access layer
- ✅ **Credit extractors**: PEFINDO and FDC metric extraction logic
- ✅ **Integration with callback**: Extraction triggered after enrichment callbacks complete

**What LND-4526 Provides (PREREQUISITE #2):**
- ✅ **`VeefinTokenFilter`**: Authentication filter for `veefin-x-token` header validation
- ✅ **`VeefinBorrowerInquiryProvider`**: Capability Provider extending `AbstractCapabilityProvider`
- ✅ **Health monitoring**: DB connectivity, BNPL `/ping` check, data freshness monitoring
- ✅ **Feature flags**: Per-borrower enable/disable capability
- ✅ **Circuit breaker**: Protection for BNPL service calls
- ✅ **Audit logging**: Request/response tracking
- ✅ **Veefin Controller & DTOs**: HTTP layer with request validation

**What's Missing (This Plan - LND-4525):**
- ❌ **BNPL internal endpoint**: No API to query credit summaries by KTP
- ❌ **FS Brick → BNPL integration**: Provider needs BNPL service adapter/port implementation
- ❌ **End-to-end wiring**: Connect all components built in LND-4526 with BNPL backend

**Key Discoveries:**
- BNPL service structure: `src/main/java/com/bukuwarung/bnpl/` (not `fsbnplservice`)
- FS Brick has hexagonal architecture with ports/adapters pattern
- Both services use Spotless with Google Java Style formatting
- Latest migration is V59, LND-4539 adds V60 (credit summary table)
- BNPL has existing `/merchant/on-boarding/ping` for health checks (will be used by Capability Provider)

## Desired End State

After **LND-4539** is completed (prerequisite #1):
- ✅ `merchant_credit_summary` table exists with credit metrics
- ✅ Enrichment pipeline populates summaries after callbacks complete
- ✅ Data is ready to be queried by API

After **LND-4526** is completed (prerequisite #2):
- ✅ `VeefinTokenFilter` validates `veefin-x-token` header
- ✅ `VeefinBorrowerInquiryProvider` (Capability Provider) is implemented with health checks
- ✅ Veefin controller `/fs/brick/service/veefin/v1/inquiry-borrowers` exists (calls Provider)
- ✅ Feature flags, circuit breaker, audit logging configured

After **this plan (LND-4525)** is completed:
1. **BNPL exposes** internal endpoint: `POST /v1/internal/veefin/borrower-summary`
   - Returns pre-computed credit metrics from `merchant_credit_summary`
   - Validates LOS auth token
   - Returns 404 if no data found
   
2. **FS Brick Capability Provider** integrates with BNPL:
   - `VeefinBorrowerInquiryProvider` calls BNPL via port/adapter
   - Health monitoring verifies BNPL `/ping` endpoint
   - Circuit breaker protects BNPL from overload
   - Audit logging captures all requests

3. **Integration configured**:
   - Mutual TLS between FS Brick and BNPL
   - Tokens stored in AWS Secrets Manager
   - Observability metrics and structured logging

### Verification:
- **Prerequisite check**: `merchant_credit_summary` table exists and has data
- **Prerequisite check**: `VeefinTokenFilter` and `VeefinBorrowerInquiryProvider` exist from LND-4526
- Call Veefin endpoint with valid KTP → returns 200 with credit data
- Call with invalid token → returns 401 (filter rejects)
- Call with non-existent KTP → returns 404
- BNPL endpoint p95 ≤ 500ms
- FS Brick endpoint p95 ≤ 1.0s
- Health check shows HEALTHY when all systems operational

## What We're NOT Doing

- ❌ **Data layer implementation** (that's LND-4539 - prerequisite)
- ❌ **Database migrations** (LND-4539 handles V60 migration)
- ❌ **Credit metric extraction** (LND-4539 implements extractors)
- ❌ **Enrichment callback integration** (LND-4539 implements extraction after callback)
- ❌ **Authentication filter** (LND-4526 implements `VeefinTokenFilter`)
- ❌ **Capability Provider pattern** (LND-4526 implements `VeefinBorrowerInquiryProvider`)
- ❌ **Health monitoring logic** (LND-4526 implements health checks in Provider)
- ❌ **Feature flags & circuit breaker** (LND-4526 implements these patterns)
- ❌ **Veefin controller & DTOs** (LND-4526 implements HTTP layer)
- ❌ Real-time S3 parsing (using pre-computed summaries from LND-4539)
- ❌ Aggregating PEFINDO and FDC metrics (returning raw provider data)
- ❌ Backfilling existing enrichment data
- ❌ Rate limiting (to be added later if needed)
- ❌ Caching responses (using database as source of truth)
- ❌ Webhook notifications to Veefin
- ❌ Batch inquiry endpoints (single KTP per request)

## Implementation Approach

**Strategy:** Since LND-4526 already implements FS Brick components (filter, provider, controller), this plan focuses on:
1. BNPL internal endpoint implementation
2. FS Brick port/adapter for BNPL integration
3. Wiring Provider to call BNPL service

**Flow:**
```
Veefin Platform
     │  HTTPS + veefin-x-token
     ▼
FS Brick Service
   ├─ VeefinTokenFilter (from LND-4526)
   │    └─ Validate token
   ├─ VeefinController (from LND-4526)
   │    └─ Validate payload & delegate to Provider
   ├─ VeefinBorrowerInquiryProvider (from LND-4526)
   │    └─ Health check, feature flags, audit logging
   │    └─ Circuit breaker protection
   │    └─ Business logic orchestration
   ├─ FsBnplPort (interface) - NEW IN THIS PLAN
   └─ FsBnplAdapter (HTTP client) - NEW IN THIS PLAN
        │  Mutual TLS + los-auth-token
        ▼
BNPL Service
   ├─ VeefinCreditSummaryController (internal) - NEW IN THIS PLAN
   │    └─ Validate LOS token
   ├─ CreditSummaryService - NEW IN THIS PLAN
   │    └─ Query merchant_credit_summary (from LND-4539)
   └─ Returns credit metrics
```

**Key Insight:** Most FS Brick components exist from LND-4526. We only need to:
- Implement BNPL endpoint
- Create FS Brick adapter to call BNPL
- Wire Provider to use the adapter

---

## Phase 1: BNPL Internal Endpoint - Domain Layer

### Overview
Create service layer to query credit summaries from the database. 

**Note**: This phase assumes `MerchantCreditSummaryEntity` and `MerchantCreditSummaryRepository` already exist from LND-4539. We only add the **service layer** and **DTOs** specific to the API.

**Prerequisites:**
- ✅ LND-4539 completed: `merchant_credit_summary` table exists
- ✅ LND-4539 completed: `MerchantCreditSummaryRepository` exists with query methods

### Changes Required:

#### 1. DTO for Credit Summary Response
**File**: `src/main/java/com/bukuwarung/bnpl/dto/BorrowerCreditSummaryDto.java`
**Changes**: New file

```java
package com.bukuwarung.bnpl.dto;

import java.math.BigDecimal;
import java.time.ZonedDateTime;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class BorrowerCreditSummaryDto {
  private String ktpNumber;
  private Integer totalFacilities;
  private Integer worstOverdueDays;
  private BigDecimal totalOutstandingBalance;
  private ZonedDateTime dataTimestamp;
  private String sourceProvider;
  private String batchId;
}
```

#### 2. Service Interface
**File**: `src/main/java/com/bukuwarung/bnpl/service/veefin/CreditSummaryService.java`
**Changes**: New file

```java
package com.bukuwarung.bnpl.service.veefin;

import com.bukuwarung.bnpl.dto.BorrowerCreditSummaryDto;

public interface CreditSummaryService {
  
  BorrowerCreditSummaryDto getCreditSummaryByKtp(String ktpNumber);
}
```

#### 3. Service Implementation
**File**: `src/main/java/com/bukuwarung/bnpl/service/veefin/impl/CreditSummaryServiceImpl.java`
**Changes**: New file

**Note**: This service uses `MerchantCreditSummaryRepository` from LND-4539. We need to add a query method `findLatestByMerchantIdAndStatus(UUID merchantId, String status)` or similar to fetch READY summaries.

```java
package com.bukuwarung.bnpl.service.veefin.impl;

import com.bukuwarung.bnpl.dto.BorrowerCreditSummaryDto;
import com.bukuwarung.bnpl.entity.MerchantCreditSummary;
import com.bukuwarung.bnpl.exception.CreditSummaryNotFoundException;
import com.bukuwarung.bnpl.repository.MerchantCreditSummaryRepository;
import com.bukuwarung.bnpl.service.veefin.CreditSummaryService;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
@Slf4j
public class CreditSummaryServiceImpl implements CreditSummaryService {

  @Autowired private MerchantCreditSummaryRepository repository;

  @Override
  public BorrowerCreditSummaryDto getCreditSummaryByKtp(String ktpNumber) {
    log.info("Fetching credit summary for KTP: {}", maskKtp(ktpNumber));
    
    MerchantCreditSummary summary = repository
        .findLatestByKtpNumber(ktpNumber)
        .orElseThrow(() -> new CreditSummaryNotFoundException(
            "No credit summary found for KTP: " + maskKtp(ktpNumber)));
    
    // For now, return PEFINDO data if available, else FDC
    // TODO: Implement provider selection logic if needed
    return BorrowerCreditSummaryDto.builder()
        .ktpNumber(summary.getKtpNumber())
        .totalFacilities(summary.getPefindoTotalFacilities() != null 
            ? summary.getPefindoTotalFacilities() 
            : summary.getFdcTotalFacilities())
        .worstOverdueDays(summary.getPefindoWorstOverdueDays() != null
            ? summary.getPefindoWorstOverdueDays()
            : summary.getFdcWorstOverdueDays())
        .totalOutstandingBalance(summary.getPefindoTotalOutstandingBalance() != null
            ? summary.getPefindoTotalOutstandingBalance()
            : summary.getFdcTotalOutstandingBalance())
        .dataTimestamp(summary.getPefindoDataTimestamp() != null
            ? summary.getPefindoDataTimestamp()
            : summary.getFdcDataTimestamp())
        .sourceProvider(summary.getPefindoTotalFacilities() != null 
            ? "PEFINDO" : "FDC")
        .batchId(summary.getBatchId())
        .build();
  }
  
  private String maskKtp(String ktp) {
    if (ktp == null || ktp.length() < 8) return "***";
    return ktp.substring(0, 4) + "********" + ktp.substring(ktp.length() - 4);
  }
}
```

#### 4. Custom Exception
**File**: `src/main/java/com/bukuwarung/bnpl/exception/CreditSummaryNotFoundException.java`
**Changes**: New file

```java
package com.bukuwarung.bnpl.exception;

public class CreditSummaryNotFoundException extends RuntimeException {
  
  public CreditSummaryNotFoundException(String message) {
    super(message);
  }
}
```

### Success Criteria:

#### Automated Verification:
- [ ] **Prerequisites verified**: LND-4539 completed, `merchant_credit_summary` table exists, repository available
- [ ] Code compiles: `cd /Users/juvianto.chi/Desktop/code/bnpl && make clean-build`
- [ ] Spotless passes: `./gradlew spotlessCheck`
- [ ] Service bean loads: `./gradlew bootRun` starts without errors

#### Manual Verification:
- [ ] Repository is autowirable in service
- [ ] Service can be called from controller (next phase)
- [ ] Exception is properly thrown when KTP not found
- [ ] Verify `MerchantCreditSummaryRepository` exists from LND-4539 (may need to add query method)

---

## Phase 2: BNPL Internal Endpoint - Controller Layer

### Overview
Create REST controller with LOS authentication for internal endpoint.

### Changes Required:

#### 1. Request DTO
**File**: `src/main/java/com/bukuwarung/bnpl/dto/BorrowerSummaryRequest.java`
**Changes**: New file

```java
package com.bukuwarung.bnpl.dto;

import javax.validation.constraints.NotBlank;
import javax.validation.constraints.Pattern;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class BorrowerSummaryRequest {
  
  @NotBlank(message = "KTP number is required")
  @Pattern(regexp = "^\\d{16}$", message = "KTP must be exactly 16 digits")
  private String ktpNumber;
}
```

#### 2. Controller
**File**: `src/main/java/com/bukuwarung/bnpl/controller/internal/veefin/VeefinCreditSummaryController.java`
**Changes**: New file

```java
package com.bukuwarung.bnpl.controller.internal.veefin;

import com.bukuwarung.bnpl.dto.BorrowerCreditSummaryDto;
import com.bukuwarung.bnpl.dto.BorrowerSummaryRequest;
import com.bukuwarung.bnpl.exception.CreditSummaryNotFoundException;
import com.bukuwarung.bnpl.service.veefin.CreditSummaryService;
import javax.validation.Valid;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestHeader;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/v1/internal/veefin")
@Slf4j
public class VeefinCreditSummaryController {

  @Autowired private CreditSummaryService creditSummaryService;
  
  @Value("${veefin.credit-summary.auth-token}")
  private String expectedAuthToken;

  @PostMapping("/borrower-summary")
  public ResponseEntity<?> getBorrowerSummary(
      @RequestHeader("los-auth-token") String authToken,
      @Valid @RequestBody BorrowerSummaryRequest request) {
    
    log.info("Received borrower summary request for KTP: {}", 
        maskKtp(request.getKtpNumber()));
    
    // Validate auth token
    if (!expectedAuthToken.equals(authToken)) {
      log.warn("Invalid auth token received");
      return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
          .body(Map.of("error", "Unauthorized"));
    }
    
    try {
      BorrowerCreditSummaryDto summary = 
          creditSummaryService.getCreditSummaryByKtp(request.getKtpNumber());
      
      log.info("Successfully retrieved credit summary for KTP: {}",
          maskKtp(request.getKtpNumber()));
      
      return ResponseEntity.ok(summary);
      
    } catch (CreditSummaryNotFoundException e) {
      log.info("Credit summary not found: {}", e.getMessage());
      return ResponseEntity.status(HttpStatus.NOT_FOUND)
          .body(Map.of(
              "code", 404,
              "message", "No credit data found for the provided KTP number"
          ));
      
    } catch (Exception e) {
      log.error("Error processing borrower summary request", e);
      return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
          .body(Map.of(
              "code", 500,
              "message", "Internal server error"
          ));
    }
  }
  
  private String maskKtp(String ktp) {
    if (ktp == null || ktp.length() < 8) return "***";
    return ktp.substring(0, 4) + "********" + ktp.substring(ktp.length() - 4);
  }
}
```

#### 3. Configuration
**File**: `src/main/resources/application.yml`
**Changes**: Add configuration section

```yaml
veefin:
  credit-summary:
    enabled: true
    auth-token: ${LOS_TOKEN:default-token-for-local}
```

### Success Criteria:

#### Automated Verification:
- [ ] Code compiles: `make clean-build`
- [ ] Spotless passes: `./gradlew spotlessCheck`
- [ ] Service starts: `./gradlew bootRun`
- [ ] Health check passes: `curl localhost:8080/actuator/health`

#### Manual Verification:
- [ ] Endpoint returns 401 with invalid token
- [ ] Endpoint returns 400 with invalid KTP format
- [ ] Endpoint returns 404 when KTP not in database
- [ ] Endpoint returns 200 with valid request and data exists
- [ ] Test with curl:
```bash
curl -X POST http://localhost:8080/v1/internal/veefin/borrower-summary \
  -H "Content-Type: application/json" \
  -H "los-auth-token: ${LOS_TOKEN}" \
  -d '{"ktpNumber": "1234567890123456"}'
```

---

## Phase 3: FS Brick Integration - BNPL Adapter

### Overview
Create adapter in FS Brick to call BNPL internal endpoint with mutual TLS. This adapter will be called by `VeefinBorrowerInquiryProvider` (from LND-4526).

**Note**: `VeefinBorrowerInquiryProvider` already exists from LND-4526. We're adding the BNPL port/adapter that the Provider will use.

### Changes Required:

#### 1. DTO Classes
**File**: `src/main/java/com/bukuwarung/fsbrick/dto/BnplBorrowerSummaryRequest.java`
**Changes**: New file

```java
package com.bukuwarung.fsbrick.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class BnplBorrowerSummaryRequest {
  private String ktpNumber;
}
```

**File**: `src/main/java/com/bukuwarung/fsbrick/dto/BnplBorrowerSummaryResponse.java`
**Changes**: New file

```java
package com.bukuwarung.fsbrick.dto;

import java.math.BigDecimal;
import java.time.ZonedDateTime;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class BnplBorrowerSummaryResponse {
  private String ktpNumber;
  private Integer totalFacilities;
  private Integer worstOverdueDays;
  private BigDecimal totalOutstandingBalance;
  private ZonedDateTime dataTimestamp;
  private String sourceProvider;
  private String batchId;
}
```

#### 2. Port Interface
**File**: `src/main/java/com/bukuwarung/fsbrick/port/FsBnplPort.java`
**Changes**: Add new method to existing interface

```java
// Add to existing interface
BnplBorrowerSummaryResponse getBorrowerCreditSummary(String ktpNumber) 
    throws BnplIntegrationException;
```

#### 3. Adapter Implementation
**File**: `src/main/java/com/bukuwarung/fsbrick/adapter/FsBnplAdapter.java`
**Changes**: Add method implementation

```java
// Add to existing class
@Value("${bnpl.base-url}")
private String bnplBaseUrl;

@Value("${bnpl.auth-token}")
private String bnplAuthToken;

@Autowired
private RestTemplate bnplRestTemplate; // Assume configured with mutual TLS

@Override
public BnplBorrowerSummaryResponse getBorrowerCreditSummary(String ktpNumber) 
    throws BnplIntegrationException {
  
  String url = bnplBaseUrl + "/v1/internal/veefin/borrower-summary";
  
  HttpHeaders headers = new HttpHeaders();
  headers.setContentType(MediaType.APPLICATION_JSON);
  headers.set("los-auth-token", bnplAuthToken);
  
  BnplBorrowerSummaryRequest request = BnplBorrowerSummaryRequest.builder()
      .ktpNumber(ktpNumber)
      .build();
  
  HttpEntity<BnplBorrowerSummaryRequest> entity = 
      new HttpEntity<>(request, headers);
  
  try {
    log.info("Calling BNPL borrower summary endpoint for KTP: {}", 
        maskKtp(ktpNumber));
    
    ResponseEntity<BnplBorrowerSummaryResponse> response = 
        bnplRestTemplate.exchange(
            url,
            HttpMethod.POST,
            entity,
            BnplBorrowerSummaryResponse.class
        );
    
    if (response.getStatusCode() == HttpStatus.OK) {
      log.info("Successfully retrieved borrower summary from BNPL");
      return response.getBody();
    } else {
      throw new BnplIntegrationException(
          "Unexpected response from BNPL: " + response.getStatusCode());
    }
    
  } catch (HttpClientErrorException.NotFound e) {
    log.info("BNPL returned 404: No credit data found");
    throw new BnplIntegrationException("No credit data found", 404);
    
  } catch (HttpClientErrorException | HttpServerErrorException e) {
    log.error("BNPL integration error: {}", e.getMessage());
    throw new BnplIntegrationException(
        "BNPL service error: " + e.getStatusCode(), 
        e.getStatusCode().value());
    
  } catch (Exception e) {
    log.error("Failed to call BNPL service", e);
    throw new BnplIntegrationException("Failed to communicate with BNPL service");
  }
}

private String maskKtp(String ktp) {
  if (ktp == null || ktp.length() < 8) return "***";
  return ktp.substring(0, 4) + "********" + ktp.substring(ktp.length() - 4);
}
```

#### 4. Exception Class
**File**: `src/main/java/com/bukuwarung/fsbrick/exception/BnplIntegrationException.java`
**Changes**: New file (if not exists) or extend

```java
package com.bukuwarung.fsbrick.exception;

import lombok.Getter;

@Getter
public class BnplIntegrationException extends RuntimeException {
  
  private final int statusCode;
  
  public BnplIntegrationException(String message) {
    super(message);
    this.statusCode = 500;
  }
  
  public BnplIntegrationException(String message, int statusCode) {
    super(message);
    this.statusCode = statusCode;
  }
}
```

#### 5. Configuration
**File**: `src/main/resources/application.yml`
**Changes**: Add BNPL configuration

```yaml
bnpl:
  base-url: ${BNPL_BASE_URL:http://localhost:8081}
  auth-token: ${BNPL_TOKEN:default-token}
  timeout:
    connect: 2000
    read: 5000
  mutual-tls:
    enabled: true
    keystore: ${BNPL_KEYSTORE_PATH:/path/to/keystore.jks}
    keystore-password: ${BNPL_KEYSTORE_PASSWORD}
    truststore: ${BNPL_TRUSTSTORE_PATH:/path/to/truststore.jks}
    truststore-password: ${BNPL_TRUSTSTORE_PASSWORD}
```

### Success Criteria:

#### Automated Verification:
- [ ] Code compiles: `cd /Users/juvianto.chi/Desktop/code/fs-brick-service && make clean-build`
- [ ] Spotless passes: `./gradlew spotlessCheck`
- [ ] Service starts: `./gradlew bootRun`

#### Manual Verification:
- [ ] Adapter can call BNPL endpoint successfully
- [ ] Proper error handling for 404, 500 responses
- [ ] Timeout configuration works (test with slow endpoint)
- [ ] Auth token is sent in header
- [ ] Test integration:
```bash
# Start both services
# FS Brick on 8080, BNPL on 8081
# Call adapter method via test or actuator
```

---

## Phase 4: Wire Provider to BNPL Adapter

### Overview
Update `VeefinBorrowerInquiryProvider` (from LND-4526) to use the newly created BNPL adapter. The Provider already has health monitoring, feature flags, circuit breaker, and audit logging. We just need to wire it to call BNPL.

### Changes Required:

#### 1. Update VeefinBorrowerInquiryProvider
**File**: `src/main/java/com/bukuwarung/fsbrick/provider/veefin/VeefinBorrowerInquiryProvider.java`
**Changes**: Inject and use BnplPort

```java
// Add to existing VeefinBorrowerInquiryProvider from LND-4526
@Autowired
private FsBnplPort fsBnplPort;

// Update the inquiry method to call BNPL
public VeefinBorrowerInquiryResponse inquireBorrower(
    VeefinBorrowerInquiryRequest request) {
  
  // Existing health check, feature flag logic from LND-4526...
  
  try {
    // NEW: Call BNPL service via adapter
    BnplBorrowerSummaryResponse bnplResponse = 
        fsBnplPort.getBorrowerCreditSummary(request.getKtpNumber());
    
    // Map to Veefin response format
    DataDto data = DataDto.builder()
        .jmlFasilitas(bnplResponse.getTotalFacilities())
        .jmlHariTunggakanTerburuk(bnplResponse.getWorstOverdueDays())
        .jmlSaldoTerutang(bnplResponse.getTotalOutstandingBalance())
        .build();
    
    return VeefinBorrowerInquiryResponse.builder()
        .ktpNumber(request.getKtpNumber())
        .requestId(request.getRequestId())
        .code(200)
        .description("success")
        .data(data)
        .build();
    
  } catch (BnplIntegrationException e) {
    // Handle BNPL errors (404, 500, etc.)
    return buildErrorResponse(request, e);
  }
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Code compiles: `make clean-build`
- [ ] Spotless passes: `./gradlew spotlessCheck`
- [ ] Provider autowires BnplPort successfully
- [ ] Unit tests pass: Test Provider with mocked BnplPort

#### Manual Verification:
- [ ] Provider successfully calls BNPL adapter
- [ ] Health check still works (BNPL `/ping` check from LND-4526)
- [ ] Circuit breaker protects BNPL calls
- [ ] Audit logging captures all requests
- [ ] Feature flags control per-borrower access

---

## Phase 5: End-to-End Testing & Documentation

### Overview
Test the complete flow from Veefin → FS Brick → BNPL → Database and ensure comprehensive documentation.

**Note**: Most components already exist from LND-4526 (filter, provider, controller, DTOs). This phase focuses on integration testing and documentation.

### Testing Strategy

#### Unit Tests (Already in LND-4526):
- ✅ **VeefinTokenFilter**: Token validation logic
- ✅ **VeefinBorrowerInquiryProvider**: Health checks, feature flags, circuit breaker
- ✅ **VeefinController**: Request validation, error handling
- **NEW: BnplCreditSummaryService**: Query logic (this plan)
- **NEW: BnplController**: LOS token validation (this plan)
- **NEW: FsBnplAdapter**: Integration error mapping (this plan)

#### Integration Tests:
1. **BNPL Endpoint**:
```bash
# Test BNPL internal endpoint
curl -X POST http://localhost:8081/v1/internal/veefin/borrower-summary \
  -H "Content-Type: application/json" \
  -H "los-auth-token: ${LOS_TOKEN}" \
  -d '{"ktpNumber": "1234567890123456"}'
```

2. **FS Brick End-to-End**:
```bash
# Test complete flow (requires both services running)
curl -X POST http://localhost:8080/fs/brick/service/veefin/v1/inquiry-borrowers \
  -H "Content-Type: application/json" \
  -H "veefin-x-token: ${VEEFIN_TOKEN}" \
  -d '{
    "ktpNumber": "1234567890123456",
    "requestId": "550e8400-e29b-41d4-a716-446655440000"
  }'
```

3. **Health Monitoring** (from LND-4526):
```bash
# Check provider health
curl http://localhost:8080/actuator/health
# Should show BNPL ping check status
```

### Documentation Updates

#### Files to Create/Update:

1. **BNPL README**
**File**: `bnpl/README.md`
**Add**: Document new internal endpoint
```markdown
## Internal Endpoints

### Veefin Borrower Summary
**Endpoint**: `POST /v1/internal/veefin/borrower-summary`
**Auth**: `los-auth-token` header
**Purpose**: Query credit summary by KTP for Veefin integration
**Response**: Credit metrics (total facilities, worst overdue days, outstanding balance)
```

2. **FS Brick README**
**File**: `fs-brick-service/README.md`
**Add**: Document Veefin integration
```markdown
## Veefin Integration

### Borrower Inquiry API
**Endpoint**: `POST /fs/brick/service/veefin/v1/inquiry-borrowers`
**Auth**: `veefin-x-token` header
**Components**:
- VeefinTokenFilter (authentication)
- VeefinBorrowerInquiryProvider (capability provider with health checks)
- BnplAdapter (internal BNPL service integration)
```

3. **Operational Runbook**
**File**: `documentation/runbooks/veefin-borrower-inquiry.md`
**Content**: Complete runbook with troubleshooting guide

### Success Criteria:

#### Automated Verification:
- [ ] All unit tests pass: `make test`
- [ ] Integration tests pass (both services running)
- [ ] Spotless passes: `make clean-build`
- [ ] Health check shows HEALTHY status

#### Manual Verification:
- [ ] Happy path: Valid KTP with data → 200 response
- [ ] Not found: KTP without data → 404
- [ ] Invalid auth: Wrong token → 401 (filter rejects)
- [ ] Invalid format: Malformed KTP → 400
- [ ] BNPL down: Simulate failure → circuit breaker triggers
- [ ] Health monitoring: BNPL `/ping` check works
- [ ] Feature flags: Can enable/disable per borrower
- [ ] Audit logging: All requests logged

---

## Testing Strategy

### Unit Tests (Completed in Prerequisites):
- ✅ **VeefinTokenFilter**: Token validation (LND-4526)
- ✅ **VeefinBorrowerInquiryProvider**: Health, flags, circuit breaker (LND-4526)
- ✅ **VeefinController**: Request validation (LND-4526)
- **NEW: BNPL Service**: Test service layer, repository queries
- **NEW: BNPL Controller**: Test validation, error handling, auth
- **NEW: FS Brick Adapter**: Test BNPL integration, error mapping

### Integration Tests:
- **BNPL Internal Endpoint**: Valid/invalid requests, error codes
- **FS Brick External Endpoint**: End-to-end flow with BNPL
- **Mutual TLS**: Certificate validation
- **Health Monitoring**: BNPL `/ping` check integration
- **Circuit Breaker**: Failure threshold and recovery
- **Error Scenarios**: 404, 500, timeout handling

### Performance Tests:
- **BNPL p95 latency**: Target ≤ 500ms
- **FS Brick p95 latency**: Target ≤ 1.0s
- **Load test**: 100 RPS sustained for 5 minutes
- **Database queries**: Verify index usage with EXPLAIN

---

## Performance Considerations

**Database Queries:**
- Indexes on `merchant_credit_summary(ktp_number)` ensure fast lookups
- Expected query time: <50ms for KTP lookup
- Repository uses `LIMIT 1` to prevent full table scans

**Inter-service Communication:**
- Mutual TLS handshake: ~100-200ms
- BNPL processing: ~100-300ms
- Total FS Brick → BNPL round trip: ~300-500ms

**Caching Strategy:**
- No application-level caching (database is source of truth)
- Health check results cached for 60 seconds (configured in LND-4526 Provider)
- Consider Redis cache if load increases

**Scaling:**
- Both services are stateless, scale horizontally
- Database connection pooling configured
- Circuit breaker protects BNPL from overload (configured in LND-4526)

---

## Rollout & Migration Plan

### Prerequisites Checklist
Before starting LND-4525 implementation, verify:
- ✅ **LND-4539 completed**: `merchant_credit_summary` table exists with data
- ✅ **LND-4526 completed**: `VeefinTokenFilter`, `VeefinBorrowerInquiryProvider`, Controller exist
- ✅ **BNPL health endpoint**: `/merchant/on-boarding/ping` is operational

### Phase 1: Development & Testing (Week 1)
- Implement BNPL internal endpoint (service + controller)
- Implement FS Brick BNPL adapter (port + adapter)
- Wire Provider (from LND-4526) to use adapter
- Write unit and integration tests
- Deploy to development environment

### Phase 2: Staging Deployment (Week 2)
- Deploy BNPL service to staging
- Deploy FS Brick service to staging (with LND-4526 components)
- Configure secrets in AWS Secrets Manager
- Mutual TLS certificate setup
- Integration testing with Veefin team (if available)
- Performance testing

### Phase 3: Production Rollout (Week 3)
- Deploy BNPL service with feature flag off
- Deploy FS Brick service (includes LND-4526 components) with feature flag off
- Configure production secrets
- Verify monitoring and alerts
- Enable feature flags for internal testing
- Coordinate with Veefin for production cutover
- Monitor metrics for 24 hours

### Rollback Plan
- Feature flags allow instant disable: `capability.providers.veefin-borrower-inquiry.enabled=false` (from LND-4526)
- BNPL endpoint isolated (only affects Veefin flow)
- No database migrations required (using existing tables from LND-4539)
- Services can rollback independently

### Phase 3: Production Rollout (Week 3)
- Deploy BNPL service with feature flag off
- Deploy FS Brick service (includes LND-4526 components) with feature flag off
- Configure production secrets
- Verify monitoring and alerts
- Enable feature flags for internal testing
- Coordinate with Veefin for production cutover
- Monitor metrics for 24 hours

### Rollback Plan
- Feature flags allow instant disable: `capability.providers.veefin-borrower-inquiry.enabled=false` (from LND-4526)
- BNPL endpoint isolated (only affects Veefin flow)
- No database migrations required (using existing tables from LND-4539)
- Services can rollback independently

---

## Monitoring & Alerts

### Metrics to Track (Configured in LND-4526):
- **Request rates**: Requests per second by endpoint
- **Success rates**: 2xx vs 4xx vs 5xx
- **Latency**: p50, p95, p99 for both services
- **BNPL integration**: Round-trip time, error rate
- **Health status**: Provider health check results (DB, BNPL ping, data freshness)
- **Circuit breaker**: Open/closed state, failure count
- **Database**: Query time, connection pool usage

### Alerts:
- **BNPL endpoint**: p95 > 750ms for 5 minutes
- **FS Brick endpoint**: p95 > 1.5s for 5 minutes
- **Error rate**: > 1% for 5 minutes
- **BNPL integration failures**: > 5% for 5 minutes
- **Health check failures**: UNHEALTHY status for 2 minutes
- **Circuit breaker**: Open state for > 1 minute
- **Unauthorized requests**: Spike detection
- **Database slow queries**: > 1s execution time

### Dashboards:
- Veefin API health dashboard (includes Provider health metrics from LND-4526)
- BNPL internal endpoint metrics
- Inter-service communication health
- Circuit breaker status

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| LND-4539 not completed | **CRITICAL** | Block LND-4525 start until credit_summary table exists; verify data population |
| LND-4526 not completed | **CRITICAL** | Block LND-4525 start until filter, provider, controller exist |
| Credit summaries not populated | High | Monitor enrichment pipeline; alert if `merchant_credit_summary` empty after enrichment |
| BNPL service unavailable | High | Circuit breaker configured in Provider (LND-4526); return 503 with retry hints |
| Health check fails | High | Provider health monitoring (LND-4526) checks DB, BNPL `/ping`, data freshness |
| Mutual TLS certificate expiry | High | Certificate monitoring alerts 30 days before expiry; documented rotation procedure |
| Token compromise | Medium | Regular token rotation schedule; IP whitelisting; suspicious activity monitoring |
| Database performance issues | Medium | Proper indexing; query optimization; connection pooling; read replicas if needed |
| Feature flag misconfiguration | Medium | Provider-level flags from LND-4526; test in staging first |
| Veefin integration issues | Low | Comprehensive API documentation; sandbox environment for testing; support runbook |

---

## Dependencies

### Critical Prerequisites (MUST BE COMPLETED FIRST)
- **LND-4539 - Credit Summary Storage** ✅
  - `merchant_credit_summary` table created (V60 migration)
  - `MerchantCreditSummaryEntity` and repository exist
  - Credit metric extractors implemented (PEFINDO and FDC)
  - Integration with enrichment callback to populate summaries
  - Test data exists for integration testing

- **LND-4526 - Veefin Token Filter & Capability Provider** ✅
  - `VeefinTokenFilter` for `veefin-x-token` authentication
  - `VeefinBorrowerInquiryProvider` extending `AbstractCapabilityProvider`
  - Health monitoring (DB, BNPL `/ping`, data freshness)
  - Feature flags, audit logging, circuit breaker
  - Veefin controller and DTOs

### Other Dependencies
- **Enrichment Pipeline**: Must populate `merchant_credit_summary` table (from LND-4539)
- **AWS Secrets Manager**: Token configuration for both services
- **Mutual TLS Infrastructure**: Certificates provisioned and configured
- **BNPL Health Endpoint**: `/merchant/on-boarding/ping` available (existing)
- **Veefin Team**: Token exchange, integration testing, production cutover coordination

---

## References

- **Parent Ticket**: [LND-4525](https://bukuwarung.atlassian.net/browse/LND-4525) - Veefin Integration (Parent)
- **Prerequisite Subtasks**: 
  - [LND-4539](https://bukuwarung.atlassian.net/browse/LND-4539) - Credit Summary Storage (data layer - must complete first)
  - [LND-4526](https://bukuwarung.atlassian.net/browse/LND-4526) - Veefin Token Filter & Capability Provider (auth/provider layer - must complete first)
- **Related Subtask**: [LND-4527](https://bukuwarung.atlassian.net/browse/LND-4527) - Borrower Data Inquiry Requirements
- **RFC Documents**:
  - BNPL RFC: `/Users/juvianto.chi/Desktop/code/bnpl/RFC/LND-4525-bnpl.md`
  - FS Brick RFC: `/Users/juvianto.chi/Desktop/code/fs-brick-service/RFC/LND-4525-fs-brick-service.md`
- **API Documentation**: `/Users/juvianto.chi/Desktop/code/documentation/apidocs/borrowers-data-inquiry-api-docs.md`
- **Implementation Plans**:
  - LND-4539 Plan: `/Users/juvianto.chi/Desktop/code/thoughts/shared/plans/2025-11-07-LND-4539-merchant-credit-summary-storage.md`
  - LND-4526 Plan: `/Users/juvianto.chi/Desktop/code/documentation/request/LND-4526/plan.md`
  - This Plan (LND-4525): `/Users/juvianto.chi/Desktop/code/thoughts/shared/plans/2025-11-07-LND-4525-veefin-borrower-inquiry-api.md`
- **Architecture Patterns**:
  - Capability Provider Pattern: `/Users/juvianto.chi/Desktop/code/thoughts/capability-provider-pattern.md`
