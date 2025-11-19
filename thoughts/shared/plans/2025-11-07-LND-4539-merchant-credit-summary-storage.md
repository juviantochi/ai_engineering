# Merchant Credit Summary Storage Implementation Plan

**Parent Ticket:** [LND-4525](https://bukuwarung.atlassian.net/browse/LND-4525) - Veefin Borrower Data Inquiry API  
**This Ticket:** [LND-4539](https://bukuwarung.atlassian.net/browse/LND-4539) - Merchant Credit Summary Storage (PREREQUISITE for LND-4525)

## Overview

Implement storage of scalar credit summary fields extracted from enrichment S3 payloads into a new `merchant_credit_summary` table. This provides fast lookup for Veefin's borrower inquiry API without parsing S3 JSONs on every request.

**Scope**: Data persistence layer only. API endpoint exposure is handled separately in parent ticket LND-4525.

## Relationship to Other Tickets and Documents

**Ticket Hierarchy:**
- **Parent**: LND-4525 - Veefin Borrower Data Inquiry API (overall feature)
- **This Ticket (LND-4539)**: Data Storage Layer - **MUST BE COMPLETED FIRST**
- **Sibling**: LND-4527 - Related to borrower data inquiry requirements

**Related Documents:**
- **Task specification**: `documentation/request/LND-4539/task.md`
- **Sample data**: `documentation/request/LND-4527/sample_PEFINDO.json`, `sample_FDC.json`
- **Parent RFC**: `bnpl/RFC/LND-4525-bnpl.md` (describes complete vision including this component)

**Important**: LND-4525 (parent ticket) builds on this foundation. This plan must complete first before API endpoints can be implemented.

## Current State Analysis

**Existing Infrastructure:**
- ✅ **Enrichment flow**: BNPL triggers Kafka event → fs-brick processes enrichment → generates S3 JSON → callbacks BNPL
- ✅ **Enrichment callback**: `MerchantOnboardingServiceImpl#processMerchantEnrichmentDataCallback` at line ~2891-2911
- ✅ **Database table**: `merchant_enrichment_data` captures enrichment attempts with S3 paths
- ✅ **Current migration**: V59 (before this task)
- ✅ **S3 infrastructure**: `S3Provider` utility exists with `AmazonS3` client
- ✅ **Async config**: `@EnableAsync` configured in `AsyncConfiguration.java` (thread pool: core=10, max=15)
- ✅ **JSON parsing**: `ObjectMapper` pattern used in services (e.g., `NotificationServiceImpl`)

**What's Missing (This Plan - LND-4539):**
- ❌ **`merchant_credit_summary` table**: Doesn't exist yet (will create V60 migration)
- ❌ **Credit metric extraction logic**: No code to parse S3 JSON and extract Veefin fields
- ❌ **Async processing trigger**: Callback doesn't trigger summary extraction after enrichment completes
- ❌ **Entity and repository**: No JPA entity or repository for credit summary

### Key Discoveries:
- Current migration is V59 (merchant_enrichment_data table)
- Service class: `src/main/java/com/bukuwarung/fsbnplservice/service/impl/MerchantOnboardingServiceImpl.java`
- Enrichment providers: PEFINDO and FDC (enum: `EnrichmentDataPoint`)
- Sample JSONs available showing exact data structure to parse

## Desired End State

After **this plan (LND-4539)** is completed:
1. **New table** `merchant_credit_summary` stores extracted credit metrics
2. **Scalar fields only** (no JSONB) with columns matching Veefin requirements:
   - `total_fasilitas` (INT)
   - `jml_hari_tunggakan_terburuk` (INT)
   - `jml_saldo_terutang` (NUMERIC(20,2))
3. **Insert-once semantics**: Each enrichment callback inserts one summary row (no updates)
4. **S3 path stays** in `merchant_enrichment_data` (no duplication)
5. **Ready for API consumption** by LND-4525

### Verification:
- Database migration V60 exists and applies cleanly
- After PEFINDO enrichment callback → `merchant_credit_summary` has row with PEFINDO data
- After FDC enrichment callback → `merchant_credit_summary` has row with FDC data
- Query by `merchant_id` + `batch_id` returns correct summary
- S3 path remains only in `merchant_enrichment_data`

## What We're NOT Doing

- ❌ **API endpoint exposure** (that's LND-4525 parent ticket)
- ❌ **Aggregating PEFINDO and FDC** (storing separately per enrichment)
- ❌ **Updating existing rows** (insert-once per batch_id)
- ❌ **JSONB columns** (scalar fields only per task requirements)
- ❌ **Backfilling existing enrichment data** (can be done later via script)
- ❌ **Real-time S3 monitoring** (only process on enrichment callback)
- ❌ **Complex idempotency** (simple: don't insert if batch_id already exists)

## Implementation Approach

**Strategy:** Extend enrichment callback to extract and store credit metrics after S3 path is saved.

**Flow:**
```
Enrichment Callback (existing)
    ↓
Validate & Update merchant_enrichment_data (existing)
    ↓
[NEW] If status = COMPLETED:
    ↓
    Download S3 JSON
    ↓
    Parse JSON based on enrichment type (PEFINDO vs FDC)
    ↓
    Extract Veefin fields (total_fasilitas, jml_hari_tunggakan_terburuk, jml_saldo_terutang)
    ↓
    Insert into merchant_credit_summary (if batch_id doesn't exist)
```

**Key Design Decisions:**
1. One row per enrichment batch (identified by `batch_id`)
2. Insert-once semantics (skip if `batch_id` already exists)
3. Different column sources for PEFINDO vs FDC
4. Synchronous processing in callback (simple, no async complexity for MVP)

---

## Phase 1: Database Schema

### Overview
Create the `merchant_credit_summary` table with Veefin-required fields.

### Changes Required:

#### 1. Flyway Migration
**File**: `src/main/resources/schema/V60__Create_Merchant_Credit_Summary_Table.sql`
**Changes**: New file

```sql
CREATE TABLE merchant_credit_summary (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    merchant_id UUID NOT NULL,
    pefindo_batch_id VARCHAR(60),
    fdc_batch_id VARCHAR(60),
    enrichment_type VARCHAR(15) NOT NULL,
    status VARCHAR(32) NOT NULL,
    total_fasilitas INT,
    jml_hari_tunggakan_terburuk INT,
    jml_saldo_terutang NUMERIC(20,2),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    created_by VARCHAR(50) NOT NULL DEFAULT 'SYSTEM',
    
    CONSTRAINT fk_merchant_credit_summary_merchant 
        FOREIGN KEY(merchant_id) REFERENCES merchant_data(merchant_id),
    CONSTRAINT uq_merchant_credit_summary_batch 
        UNIQUE (pefindo_batch_id, fdc_batch_id, enrichment_type)
);

CREATE INDEX idx_merchant_credit_summary_merchant ON merchant_credit_summary(merchant_id);
CREATE INDEX idx_merchant_credit_summary_batch_pefindo ON merchant_credit_summary(pefindo_batch_id);
CREATE INDEX idx_merchant_credit_summary_batch_fdc ON merchant_credit_summary(fdc_batch_id);

COMMENT ON TABLE merchant_credit_summary IS 'Stores extracted credit metrics from PEFINDO and FDC enrichment for Veefin API';
COMMENT ON COLUMN merchant_credit_summary.enrichment_type IS 'PEFINDO or FDC';
COMMENT ON COLUMN merchant_credit_summary.status IS 'READY or NOT_READY - only READY should be returned to Veefin';
COMMENT ON COLUMN merchant_credit_summary.total_fasilitas IS 'Total facilities count (Veefin jmlFasilitas)';
COMMENT ON COLUMN merchant_credit_summary.jml_hari_tunggakan_terburuk IS 'Worst overdue days (Veefin jmlHariTunggakanTerburuk)';
COMMENT ON COLUMN merchant_credit_summary.jml_saldo_terutang IS 'Total outstanding balance (Veefin jmlSaldoTerutang)';
```

**Rationale:**
- `pefindo_batch_id` and `fdc_batch_id` separate columns per task.md requirements
- `enrichment_type` distinguishes PEFINDO vs FDC records
- `status` field tracks if data is READY (per task.md: "only valid if status is READY")
- Unique constraint prevents duplicate inserts for same batch
- Indexes optimize queries by merchant_id and batch_id

### Success Criteria:

#### Automated Verification:
- [ ] Migration file exists: `V60__Create_Merchant_Credit_Summary_Table.sql`
- [ ] Migration applies cleanly: `cd /Users/juvianto.chi/Desktop/code/bnpl && ./gradlew flywayMigrate`
- [ ] Table structure verified: Connect to DB and run `\d merchant_credit_summary`
- [ ] Indexes exist: `\di merchant_credit_summary*`

#### Manual Verification:
- [ ] Foreign key to `merchant_data` enforced (try inserting with invalid merchant_id → fails)
- [ ] Unique constraint works (try duplicate batch_id → fails)
- [ ] Default values work (insert without created_at → uses NOW())

---

## Phase 2: Domain Model

### Overview
Create JPA entity and repository for the new table.

### Changes Required:

#### 1. Entity Class
**File**: `src/main/java/com/bukuwarung/fsbnplservice/entity/MerchantCreditSummaryEntity.java`
**Changes**: New file

```java
package com.bukuwarung.fsbnplservice.entity;

import java.math.BigDecimal;
import java.time.OffsetDateTime;
import java.util.UUID;
import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.JoinColumn;
import javax.persistence.ManyToOne;
import javax.persistence.Table;
import lombok.Getter;
import lombok.Setter;
import org.hibernate.annotations.CreationTimestamp;

@Entity
@Table(name = "merchant_credit_summary")
@Getter
@Setter
public class MerchantCreditSummaryEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.AUTO)
  @Column(name = "id")
  private UUID id;

  @ManyToOne
  @JoinColumn(name = "merchant_id", referencedColumnName = "merchant_id", nullable = false)
  private MerchantDataEntity merchantData;

  @Column(name = "pefindo_batch_id", length = 60)
  private String pefindoBatchId;

  @Column(name = "fdc_batch_id", length = 60)
  private String fdcBatchId;

  @Column(name = "enrichment_type", length = 15, nullable = false)
  private String enrichmentType;

  @Column(name = "status", length = 32, nullable = false)
  private String status;

  @Column(name = "total_fasilitas")
  private Integer totalFasilitas;

  @Column(name = "jml_hari_tunggakan_terburuk")
  private Integer jmlHariTunggakanTerburuk;

  @Column(name = "jml_saldo_terutang", precision = 20, scale = 2)
  private BigDecimal jmlSaldoTerutang;

  @Column(name = "created_at", nullable = false, updatable = false)
  @CreationTimestamp
  private OffsetDateTime createdAt;

  @Column(name = "created_by", length = 50, nullable = false)
  private String createdBy = "SYSTEM";
}
```

#### 2. Repository Interface
**File**: `src/main/java/com/bukuwarung/fsbnplservice/repository/MerchantCreditSummaryRepository.java`
**Changes**: New file

```java
package com.bukuwarung.fsbnplservice.repository;

import com.bukuwarung.fsbnplservice.entity.MerchantCreditSummaryEntity;
import java.util.Optional;
import java.util.UUID;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface MerchantCreditSummaryRepository
    extends JpaRepository<MerchantCreditSummaryEntity, UUID> {

  Optional<MerchantCreditSummaryEntity> findByPefindoBatchIdAndEnrichmentType(
      String pefindoBatchId, String enrichmentType);

  Optional<MerchantCreditSummaryEntity> findByFdcBatchIdAndEnrichmentType(
      String fdcBatchId, String enrichmentType);

  boolean existsByPefindoBatchIdAndEnrichmentType(String pefindoBatchId, String enrichmentType);

  boolean existsByFdcBatchIdAndEnrichmentType(String fdcBatchId, String enrichmentType);
}
```

**Rationale:**
- Query methods support checking if batch already processed (idempotency)
- Separate methods for PEFINDO vs FDC batch IDs per task requirements

### Success Criteria:

#### Automated Verification:
- [ ] Code compiles: `cd /Users/juvianto.chi/Desktop/code/bnpl && make clean-build`
- [ ] Spotless passes: `./gradlew spotlessCheck`
- [ ] Service starts: `./gradlew bootRun` (entity loads without errors)

#### Manual Verification:
- [ ] Repository is autowirable in service classes
- [ ] JPA mappings correct (check Hibernate logs on startup)
- [ ] Can insert and query test data via repository

---

## Phase 3: Credit Metric Extraction Logic

### Overview
Implement parser logic to extract Veefin fields from PEFINDO and FDC JSON payloads.

### Changes Required:

#### 1. DTO for Extracted Data
**File**: `src/main/java/com/bukuwarung/fsbnplservice/dto/CreditMetricsDto.java`
**Changes**: New file

```java
package com.bukuwarung.fsbnplservice.dto;

import java.math.BigDecimal;
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
public class CreditMetricsDto {
  private Integer totalFasilitas;
  private Integer jmlHariTunggakanTerburuk;
  private BigDecimal jmlSaldoTerutang;
  private String status;
}
```

#### 2. PEFINDO Extractor
**File**: `src/main/java/com/bukuwarung/fsbnplservice/service/extractor/PefindoCreditMetricsExtractor.java`
**Changes**: New file

```java
package com.bukuwarung.fsbnplservice.service.extractor;

import com.bukuwarung.fsbnplservice.dto.CreditMetricsDto;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.math.BigDecimal;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Component
@Slf4j
public class PefindoCreditMetricsExtractor {

  private final ObjectMapper objectMapper = new ObjectMapper();

  public CreditMetricsDto extract(String jsonContent) throws Exception {
    JsonNode root = objectMapper.readTree(jsonContent);
    
    // Extract status
    String status = root.path("status").asText("NOT_READY");
    
    JsonNode debitur = root.path("report").path("debitur");

    // total_fasilitas = jml_aktif_fasilitas + jml_tutup_fasilitas
    Integer totalFasilitas =
        debitur.path("jml_aktif_fasilitas").asInt(0)
            + debitur.path("jml_tutup_fasilitas").asInt(0);

    // jml_hari_tunggakan_terburuk
    Integer jmlHariTunggakanTerburuk = debitur.path("jml_hari_tunggakan_terburuk").asInt(0);

    // jml_saldo_terutang
    BigDecimal jmlSaldoTerutang =
        BigDecimal.valueOf(debitur.path("jml_saldo_terutang").asDouble(0.0));

    log.info("Extracted PEFINDO metrics: totalFasilitas={}, jmlHariTunggakanTerburuk={}, jmlSaldoTerutang={}, status={}",
        totalFasilitas, jmlHariTunggakanTerburuk, jmlSaldoTerutang, status);

    return CreditMetricsDto.builder()
        .totalFasilitas(totalFasilitas)
        .jmlHariTunggakanTerburuk(jmlHariTunggakanTerburuk)
        .jmlSaldoTerutang(jmlSaldoTerutang)
        .status(status.equals("Success") ? "READY" : "NOT_READY")
        .build();
  }
}
```

#### 3. FDC Extractor
**File**: `src/main/java/com/bukuwarung/fsbnplservice/service/extractor/FdcCreditMetricsExtractor.java`
**Changes**: New file

```java
package com.bukuwarung.fsbnplservice.service.extractor;

import com.bukuwarung.fsbnplservice.dto.CreditMetricsDto;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.math.BigDecimal;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Component
@Slf4j
public class FdcCreditMetricsExtractor {

  private final ObjectMapper objectMapper = new ObjectMapper();

  public CreditMetricsDto extract(String jsonContent) throws Exception {
    JsonNode root = objectMapper.readTree(jsonContent);
    
    // Extract status
    String status = root.path("status").asText("NOT_READY");

    // total_fasilitas = count of pinjaman array
    JsonNode pinjamanArray = root.path("pinjaman");
    Integer totalFasilitas = pinjamanArray.size();

    // jml_hari_tunggakan_terburuk = max of dpd_max across all pinjaman
    int maxDpd = 0;
    for (JsonNode pinjaman : pinjamanArray) {
      int dpdMax = pinjaman.path("dpd_max").asInt(0);
      if (dpdMax > maxDpd) {
        maxDpd = dpdMax;
      }
    }
    Integer jmlHariTunggakanTerburuk = maxDpd;

    // jml_saldo_terutang = sum of sisa_pinjaman_berjalan
    double totalOutstanding = 0.0;
    for (JsonNode pinjaman : pinjamanArray) {
      totalOutstanding += pinjaman.path("sisa_pinjaman_berjalan").asDouble(0.0);
    }
    BigDecimal jmlSaldoTerutang = BigDecimal.valueOf(totalOutstanding);

    log.info("Extracted FDC metrics: totalFasilitas={}, jmlHariTunggakanTerburuk={}, jmlSaldoTerutang={}, status={}",
        totalFasilitas, jmlHariTunggakanTerburuk, jmlSaldoTerutang, status);

    return CreditMetricsDto.builder()
        .totalFasilitas(totalFasilitas)
        .jmlHariTunggakanTerburuk(jmlHariTunggakanTerburuk)
        .jmlSaldoTerutang(jmlSaldoTerutang)
        .status(status.equals("found") ? "READY" : "NOT_READY")
        .build();
  }
}
```

**Rationale:**
- Extractors follow exact mapping rules from task.md
- PEFINDO: adds active + closed facilities
- FDC: counts pinjaman array, finds max dpd_max, sums outstanding balances
- Status mapping: PEFINDO "Success" → READY, FDC "found" → READY

### Success Criteria:

#### Automated Verification:
- [ ] Code compiles: `make clean-build`
- [ ] Spotless passes: `./gradlew spotlessCheck`
- [ ] Unit tests pass (create tests with sample JSONs)

#### Manual Verification:
- [ ] PEFINDO extractor returns correct values from `sample_PEFINDO.json`
- [ ] FDC extractor returns correct values from `sample_FDC.json`
- [ ] Extractors handle missing fields gracefully (no NPE)

**Test Template** (create in `src/test/java/com/bukuwarung/fsbnplservice/service/extractor/`):

```java
@Test
void testPefindoExtraction() throws Exception {
  String json = Files.readString(Paths.get("src/test/resources/fixtures/sample_PEFINDO.json"));
  CreditMetricsDto result = pefindoExtractor.extract(json);
  
  assertThat(result.getTotalFasilitas()).isEqualTo(1); // 0 active + 1 closed
  assertThat(result.getJmlHariTunggakanTerburuk()).isEqualTo(0);
  assertThat(result.getJmlSaldoTerutang()).isEqualByComparingTo(BigDecimal.ZERO);
  assertThat(result.getStatus()).isEqualTo("READY");
}
```

---

## Phase 4: Integration with Enrichment Callback

### Overview
Wire the extraction and storage logic into the existing enrichment callback handler.

### Changes Required:

#### 1. Update Callback Handler
**File**: `src/main/java/com/bukuwarung/fsbnplservice/service/impl/MerchantOnboardingServiceImpl.java`
**Changes**: Modify existing method around line 2891-2911

**Add dependencies:**
```java
@Autowired private AmazonS3 s3Client;
@Autowired private MerchantCreditSummaryRepository merchantCreditSummaryRepository;
@Autowired private PefindoCreditMetricsExtractor pefindoExtractor;
@Autowired private FdcCreditMetricsExtractor fdcExtractor;
```

**Current code (around line 2891-2911):**
```java
public void processMerchantEnrichmentDataCallback(String batchId, String type, String s3Path) {
  /* validate that the entity exists */
  Optional<MerchantEnrichmentDataEntity> optMerchantEnrichmentData =
      merchantEnrichmentDataRepository.findOneByBatchIdAndType(batchId, type);
  if (optMerchantEnrichmentData.isEmpty()) {
    String message =
        String.format("No enrichment data found for batchId %s and type %s", batchId, type);
    log.error(message);
    throw new IllegalArgumentException(message);
  }
  /* validate that the status is IN_PROGRESS */
  MerchantEnrichmentDataEntity entity = optMerchantEnrichmentData.get();
  if (!(entity.getStatus().equals(LeadsEnrichmentStatus.IN_PROGRESS)
      || entity.getStatus().equals(LeadsEnrichmentStatus.FAILED))) {
    String message =
        String.format(
            "Enrichment data status for batchId %s and type %s is not in IN_PROGRESS nor FAILED",
            batchId, type);
    log.error(message);
    throw new IllegalArgumentException(message);
  }
  /* do the S3 path and status update */
  LeadsEnrichmentStatus status =
      s3Path == null || s3Path.isBlank()
          ? LeadsEnrichmentStatus.FAILED
          : LeadsEnrichmentStatus.COMPLETED;
  entity.setS3Path(s3Path);
  entity.setStatus(status);
  merchantEnrichmentDataRepository.save(entity);
}
```

**New code (add after save):**
```java
public void processMerchantEnrichmentDataCallback(String batchId, String type, String s3Path) {
  /* validate that the entity exists */
  Optional<MerchantEnrichmentDataEntity> optMerchantEnrichmentData =
      merchantEnrichmentDataRepository.findOneByBatchIdAndType(batchId, type);
  if (optMerchantEnrichmentData.isEmpty()) {
    String message =
        String.format("No enrichment data found for batchId %s and type %s", batchId, type);
    log.error(message);
    throw new IllegalArgumentException(message);
  }
  /* validate that the status is IN_PROGRESS */
  MerchantEnrichmentDataEntity entity = optMerchantEnrichmentData.get();
  if (!(entity.getStatus().equals(LeadsEnrichmentStatus.IN_PROGRESS)
      || entity.getStatus().equals(LeadsEnrichmentStatus.FAILED))) {
    String message =
        String.format(
            "Enrichment data status for batchId %s and type %s is not in IN_PROGRESS nor FAILED",
            batchId, type);
    log.error(message);
    throw new IllegalArgumentException(message);
  }
  /* do the S3 path and status update */
  LeadsEnrichmentStatus status =
      s3Path == null || s3Path.isBlank()
          ? LeadsEnrichmentStatus.FAILED
          : LeadsEnrichmentStatus.COMPLETED;
  entity.setS3Path(s3Path);
  entity.setStatus(status);
  merchantEnrichmentDataRepository.save(entity);

  /* NEW: Extract and store credit summary if completed */
  if (status == LeadsEnrichmentStatus.COMPLETED) {
    try {
      extractAndStoreCreditSummary(entity, batchId, type, s3Path);
    } catch (Exception e) {
      log.error("Failed to extract credit summary for batchId: {}, type: {}", 
          batchId, type, e);
      // Don't throw - enrichment callback should succeed even if summary extraction fails
    }
  }
}

private void extractAndStoreCreditSummary(
    MerchantEnrichmentDataEntity enrichmentEntity, 
    String batchId, 
    String type, 
    String s3Path) {
  
  log.info("Extracting credit summary for batchId: {}, type: {}", batchId, type);
  
  // Check idempotency: skip if already processed
  boolean exists = false;
  if (EnrichmentDataPoint.PEFINDO.name().equals(type)) {
    exists = merchantCreditSummaryRepository.existsByPefindoBatchIdAndEnrichmentType(batchId, type);
  } else if (EnrichmentDataPoint.FDC.name().equals(type)) {
    exists = merchantCreditSummaryRepository.existsByFdcBatchIdAndEnrichmentType(batchId, type);
  }
  
  if (exists) {
    log.info("Credit summary already exists for batchId: {}, type: {}, skipping", batchId, type);
    return;
  }
  
  // Download JSON from S3
  String jsonContent = downloadJsonFromS3(s3Path);
  
  // Extract metrics based on type
  CreditMetricsDto metrics;
  if (EnrichmentDataPoint.PEFINDO.name().equals(type)) {
    metrics = pefindoExtractor.extract(jsonContent);
  } else if (EnrichmentDataPoint.FDC.name().equals(type)) {
    metrics = fdcExtractor.extract(jsonContent);
  } else {
    log.warn("Unknown enrichment type: {}, skipping credit summary extraction", type);
    return;
  }
  
  // Store in database
  MerchantCreditSummaryEntity summary = new MerchantCreditSummaryEntity();
  summary.setMerchantData(enrichmentEntity.getMerchantData());
  summary.setEnrichmentType(type);
  summary.setStatus(metrics.getStatus());
  summary.setTotalFasilitas(metrics.getTotalFasilitas());
  summary.setJmlHariTunggakanTerburuk(metrics.getJmlHariTunggakanTerburuk());
  summary.setJmlSaldoTerutang(metrics.getJmlSaldoTerutang());
  
  // Set appropriate batch_id column
  if (EnrichmentDataPoint.PEFINDO.name().equals(type)) {
    summary.setPefindoBatchId(batchId);
  } else if (EnrichmentDataPoint.FDC.name().equals(type)) {
    summary.setFdcBatchId(batchId);
  }
  
  merchantCreditSummaryRepository.save(summary);
  
  log.info("Successfully stored credit summary for batchId: {}, type: {}", batchId, type);
}

private String downloadJsonFromS3(String s3Path) {
  // Parse S3 URI: s3://bucket-name/path/to/file.json
  URI uri = URI.create(s3Path);
  String bucket = uri.getHost();
  String key = uri.getPath().substring(1); // Remove leading '/'

  log.debug("Downloading S3 object: bucket={}, key={}", bucket, key);

  try {
    S3Object s3Object = s3Client.getObject(bucket, key);
    try (BufferedReader reader =
        new BufferedReader(
            new InputStreamReader(s3Object.getObjectContent(), StandardCharsets.UTF_8))) {
      return reader.lines().collect(Collectors.joining("\n"));
    }
  } catch (Exception e) {
    log.error("S3 download failed: bucket={}, key={}", bucket, key, e);
    throw new RuntimeException("Failed to download S3 object: " + s3Path, e);
  }
}
```

**Rationale:**
- Extraction happens synchronously after enrichment completes (simple for MVP)
- Failures in extraction don't break enrichment callback (logged but not thrown)
- Idempotency check prevents duplicate inserts
- Batch ID stored in appropriate column (pefindo_batch_id or fdc_batch_id)

### Success Criteria:

#### Automated Verification:
- [ ] Code compiles: `make clean-build`
- [ ] Spotless passes: `./gradlew spotlessApply`
- [ ] Unit tests pass: Create test for callback with mocked S3 and repository
- [ ] Integration test: Mock full callback → verify summary inserted

#### Manual Verification:
- [ ] Trigger PEFINDO enrichment callback → check `merchant_credit_summary` has row
- [ ] Trigger FDC enrichment callback → check `merchant_credit_summary` has row
- [ ] Trigger duplicate callback → verify no duplicate insert (logged "skipping")
- [ ] Failed enrichment (no S3 path) → no summary inserted
- [ ] S3 download failure → logged error but callback succeeds

---

## Phase 5: Testing & Validation

### Overview
Comprehensive testing to ensure correctness and robustness.

### Changes Required:

#### 1. Copy Sample JSONs to Test Resources
```bash
mkdir -p src/test/resources/fixtures
cp /Users/juvianto.chi/Desktop/code/documentation/request/LND-4527/sample_PEFINDO.json src/test/resources/fixtures/
cp /Users/juvianto.chi/Desktop/code/documentation/request/LND-4527/sample_FDC.json src/test/resources/fixtures/
```

#### 2. Extractor Unit Tests
**File**: `src/test/java/com/bukuwarung/fsbnplservice/service/extractor/PefindoCreditMetricsExtractorTest.java`

```java
package com.bukuwarung.fsbnplservice.service.extractor;

import static org.assertj.core.api.Assertions.assertThat;

import com.bukuwarung.fsbnplservice.dto.CreditMetricsDto;
import java.math.BigDecimal;
import java.nio.file.Files;
import java.nio.file.Paths;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class PefindoCreditMetricsExtractorTest {

  private PefindoCreditMetricsExtractor extractor;

  @BeforeEach
  void setUp() {
    extractor = new PefindoCreditMetricsExtractor();
  }

  @Test
  void testExtractFromSampleJson() throws Exception {
    String json =
        Files.readString(Paths.get("src/test/resources/fixtures/sample_PEFINDO.json"));

    CreditMetricsDto result = extractor.extract(json);

    assertThat(result.getTotalFasilitas()).isEqualTo(1); // 0 active + 1 closed
    assertThat(result.getJmlHariTunggakanTerburuk()).isEqualTo(0);
    assertThat(result.getJmlSaldoTerutang()).isEqualByComparingTo(BigDecimal.ZERO);
    assertThat(result.getStatus()).isEqualTo("READY"); // Success maps to READY
  }

  @Test
  void testExtractHandlesMissingFields() throws Exception {
    String json = "{\"status\":\"Success\",\"report\":{\"debitur\":{}}}";

    CreditMetricsDto result = extractor.extract(json);

    assertThat(result.getTotalFasilitas()).isEqualTo(0);
    assertThat(result.getJmlHariTunggakanTerburuk()).isEqualTo(0);
    assertThat(result.getStatus()).isEqualTo("READY");
  }
}
```

**File**: `src/test/java/com/bukuwarung/fsbnplservice/service/extractor/FdcCreditMetricsExtractorTest.java`

```java
package com.bukuwarung.fsbnplservice.service.extractor;

import static org.assertj.core.api.Assertions.assertThat;

import com.bukuwarung.fsbnplservice.dto.CreditMetricsDto;
import java.math.BigDecimal;
import java.nio.file.Files;
import java.nio.file.Paths;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class FdcCreditMetricsExtractorTest {

  private FdcCreditMetricsExtractor extractor;

  @BeforeEach
  void setUp() {
    extractor = new FdcCreditMetricsExtractor();
  }

  @Test
  void testExtractFromSampleJson() throws Exception {
    String json =
        Files.readString(Paths.get("src/test/resources/fixtures/sample_FDC.json"));

    CreditMetricsDto result = extractor.extract(json);

    assertThat(result.getTotalFasilitas()).isEqualTo(6);
    assertThat(result.getJmlHariTunggakanTerburuk()).isEqualTo(91); // max dpd_max
    assertThat(result.getJmlSaldoTerutang())
        .isEqualByComparingTo(BigDecimal.valueOf(5000000.0));
    assertThat(result.getStatus()).isEqualTo("READY"); // found maps to READY
  }

  @Test
  void testExtractHandlesEmptyPinjamanArray() throws Exception {
    String json = "{\"status\":\"found\",\"pinjaman\":[]}";

    CreditMetricsDto result = extractor.extract(json);

    assertThat(result.getTotalFasilitas()).isEqualTo(0);
    assertThat(result.getJmlHariTunggakanTerburuk()).isEqualTo(0);
    assertThat(result.getStatus()).isEqualTo("READY");
  }
}
```

#### 3. Integration Test
**File**: Add test to `src/test/java/com/bukuwarung/fsbnplservice/service/impl/MerchantOnboardingServiceImplTest.java`

```java
@Test
void testEnrichmentCallbackExtractsCreditSummary() {
  // Setup test merchant and enrichment data
  MerchantDataEntity merchant = createTestMerchant();
  String batchId = "test-batch-123";
  String s3Path = "s3://test-bucket/enrichments/pefindo_test.json";
  
  // Mock S3 response with sample JSON
  // Mock repository to verify save was called
  
  // Execute callback
  service.processMerchantEnrichmentDataCallback(batchId, "PEFINDO", s3Path);
  
  // Verify credit summary was saved
  verify(merchantCreditSummaryRepository, times(1)).save(any());
}
```

### Success Criteria:

#### Automated Verification:
- [ ] All unit tests pass: `./gradlew test`
- [ ] Extractor tests pass with sample JSONs
- [ ] Integration tests pass
- [ ] Spotless passes: `./gradlew spotlessCheck`
- [ ] Build succeeds: `make clean-build`

#### Manual Verification:
- [ ] Run tests with `-i` flag to see detailed output
- [ ] Verify test fixtures load correctly
- [ ] Check code coverage includes new code

---

## Testing Strategy

### Unit Tests:
- **Extractors**: Test with real sample JSONs, verify extracted values match expected
- **Edge cases**: Missing fields, empty arrays, null values, malformed JSON
- **Status mapping**: Verify PEFINDO "Success" → READY, FDC "found" → READY

### Integration Tests:
- **End-to-end callback**: Mock full callback → S3 download → extraction → database insert
- **Idempotency**: Call twice with same batchId, verify second call skips
- **Different enrichment types**: Process PEFINDO then FDC, verify both in database
- **Error scenarios**: S3 download failure, parsing error, database error

### Manual Testing Steps:
1. **Setup test merchant**:
   - Insert test merchant in `merchant_data`
   - Insert enrichment record in `merchant_enrichment_data` with status IN_PROGRESS

2. **Trigger PEFINDO callback**:
   ```bash
   curl -X PATCH http://localhost:8080/merchant/leads-enrichment/callback \
     -H "Content-Type: application/json" \
     -d '{
       "batchId": "test-pefindo-batch-123",
       "type": "PEFINDO",
       "s3Path": "s3://test-bucket/enrichments/pefindo_test.json"
     }'
   ```

3. **Verify extraction**:
   - Check logs for "Extracting credit summary"
   - Query: `SELECT * FROM merchant_credit_summary WHERE pefindo_batch_id = 'test-pefindo-batch-123'`
   - Verify: `enrichment_type = 'PEFINDO'`, fields populated correctly

4. **Trigger FDC callback**:
   ```bash
   curl -X PATCH http://localhost:8080/merchant/leads-enrichment/callback \
     -H "Content-Type: application/json" \
     -d '{
       "batchId": "test-fdc-batch-456",
       "type": "FDC",
       "s3Path": "s3://test-bucket/enrichments/fdc_test.json"
     }'
   ```

5. **Verify FDC extraction**:
   - Query: `SELECT * FROM merchant_credit_summary WHERE fdc_batch_id = 'test-fdc-batch-456'`
   - Verify: `enrichment_type = 'FDC'`, fields populated correctly

6. **Test idempotency**:
   - Repeat step 2 (PEFINDO callback with same batchId)
   - Verify logs show "already exists, skipping"
   - Verify database has only 1 row (no duplicate)

---

## Performance Considerations

**S3 Download:**
- JSON size: ~50-100KB (based on samples)
- Download time: <1 second on average
- Synchronous processing acceptable for MVP (happens once per enrichment)

**Database Inserts:**
- Single row insert per enrichment (minimal overhead)
- Indexes ensure fast lookups later
- No N+1 queries (single insert operation)

**Future Optimizations** (if needed):
- Move to async processing with @Async annotation
- Add retry logic for transient S3 failures
- Batch processing if multiple enrichments complete simultaneously

---

## Migration Notes

**Flyway Migration:**
- V60 creates new table, no data migration needed
- Safe to rollback: Drop table if needed
- No impact on existing enrichment flow (additive change)

**Backward Compatibility:**
- Enrichment callback continues to work without changes
- Credit summary extraction is optional (graceful degradation if extraction fails)
- Old enrichments (before V60) won't have summaries (acceptable)

**Forward Compatibility:**
- Table designed for LND-4525 API requirements
- Can add more columns later if Veefin needs additional fields
- Insert-once semantics prevents conflicts with future updates

---

## Rollout & Migration Plan

### Phase 1: Development & Testing (Week 1)
- Implement migration, entity, repository
- Implement extractors and add unit tests
- Integrate with callback handler
- Test locally with sample JSONs

### Phase 2: Staging Deployment (Week 2)
- Deploy migration to staging database
- Deploy service to staging
- Test with real S3 data
- Coordinate with enrichment pipeline team

### Phase 3: Production Rollout (Week 3)
- Deploy migration to production during low-traffic window
- Deploy service to production
- Monitor enrichment callbacks
- Verify summaries being created
- Monitor for 48 hours before declaring success

### Rollback Plan
- If extraction causes issues, comment out the new code in callback handler
- Table can remain (doesn't break anything)
- Re-deploy service without extraction logic
- Fix issues and redeploy

---

## Monitoring & Alerts

### Metrics to Track:
- **Enrichment callback success rate**: Should remain ≥99.9%
- **Credit summary insertion rate**: Track how many summaries created per hour
- **S3 download failures**: Alert if >1% failure rate
- **Extraction errors**: Alert if >1% of extractions fail

### Logs to Monitor:
- "Extracting credit summary" - indicates extraction started
- "Successfully stored credit summary" - indicates success
- "Failed to extract credit summary" - indicates error (should investigate)
- "already exists, skipping" - indicates idempotency working

### Alerts:
- Enrichment callback failure rate > 0.5% for 5 minutes
- No credit summaries inserted in last hour (during business hours)
- S3 download errors > 1% for 5 minutes

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| S3 download failures | Medium | Log and continue; enrichment callback succeeds anyway; can reprocess later |
| JSON parsing errors | Medium | Defensive programming; handle missing fields gracefully; log and continue |
| Database insert failures | Low | Transaction isolation; unique constraint prevents duplicates; retry can work |
| Performance degradation | Low | Extraction is lightweight; only runs once per enrichment; monitor callback latency |
| Incorrect field extraction | High | Thorough testing with real sample JSONs; unit tests with assertions |

---

## Dependencies

### Critical Prerequisites
- ✅ **Enrichment flow working**: BNPL ↔ fs-brick enrichment callbacks operational
- ✅ **S3 infrastructure**: S3 bucket and access configured
- ✅ **Sample data**: Real JSON samples from fs-brick available for testing

### Other Dependencies
- **LND-4525 (Parent)**: Will consume this data via API (happens AFTER this ticket)
- **Database**: PostgreSQL with migration capability
- **fs-brick team**: Coordination for testing end-to-end flow

---

## References

- **Parent Ticket**: [LND-4525](https://bukuwarung.atlassian.net/browse/LND-4525) - Veefin Borrower Data Inquiry API
- **This Ticket**: [LND-4539](https://bukuwarung.atlassian.net/browse/LND-4539) - Merchant Credit Summary Storage
- **Task Specification**: `/Users/juvianto.chi/Desktop/code/documentation/request/LND-4539/task.md`
- **Sample PEFINDO JSON**: `/Users/juvianto.chi/Desktop/code/documentation/request/LND-4527/sample_PEFINDO.json`
- **Sample FDC JSON**: `/Users/juvianto.chi/Desktop/code/documentation/request/LND-4527/sample_FDC.json`
- **Parent RFC**: `/Users/juvianto.chi/Desktop/code/bnpl/RFC/LND-4525-bnpl.md`
- **Enrichment Callback**: `src/main/java/com/bukuwarung/fsbnplservice/service/impl/MerchantOnboardingServiceImpl.java:L2891-2911`
