# RFC: LND-4561 - PDF Validation Enhancement for File Upload Endpoint

**Ticket**: [LND-4561](https://bukuwarung.atlassian.net/browse/LND-4561)
**Repository**: fs-bnpl-service
**Author**: Juvianto Chi
**Date**: 2025-01-18
**Last Updated**: 2025-01-18 (v1.2 - Conditional PDF-Only for Bank Statements)

## Status
- [x] Draft
- [ ] Proposed
- [ ] Accepted
- [ ] Rejected
- [ ] Superseded

---

## ✅ NO BREAKING CHANGE - Backward Compatible

This RFC enforces PDF-only uploads **specifically for bank statements** (`type = "bank_statement"`) while maintaining full backward compatibility for other upload types. JPEG and PNG uploads remain supported for non-bank-statement types.

---

## Context

The `/upload/file` endpoint in the BNPL service currently accepts multiple file types (JPEG, PNG, PDF) with basic validation limited to MIME type detection via Apache Tika and file extension checking. There is no validation to ensure that uploaded PDF files are structurally valid, uncorrupted, and can be opened.

**Current Implementation:**
- **Controller**: `UploadFileController.java` (line 27-39)
- **Service**: `UploadFileServiceImpl.java` (line 29-40)
- **Validation**: MIME type detection (Apache Tika 3.0.0) + file extension check
- **Accepted Types**: JPEG, PNG, PDF (all types supported)
- **Storage**: AWS S3 bucket
- **File Size Limit**: 5MB (configured in `application.properties:45-46`)
- **Type Parameter**: Used to determine upload category (e.g., "bank_statement", "invoice", etc.)

**Business Problem:**
Users upload PDF documents (primarily scanned PDFs) to the BNPL merchant onboarding system, particularly bank statements. Invalid or corrupted PDFs cause downstream processing issues and poor user experience. Bank statements specifically should only be in PDF format for regulatory and processing consistency.

**Pain Points:**
- Corrupted PDFs might be discovered only after S3 upload
- Password-protected PDFs fail silently in downstream processing
- Generic error messages provide no actionable feedback
- Wasted storage and bandwidth on invalid files
- No enforcement of PDF-only requirement for documents requiring structured parsing

---

## Proposal

**Add conditional PDF-only enforcement and PDF structural validation** to the `/upload/file` endpoint using Apache PDFBox library. The changes will:

1. **When `type = "bank_statement"`**:
   - Enforce PDF-only uploads (reject JPEG, PNG, and other formats)
   - Validate PDF file structure is valid and openable
   - Reject corrupted, password-protected, and encrypted PDFs

2. **When `type ≠ "bank_statement"`**:
   - Maintain existing behavior (accept JPEG, PNG, PDF)
   - Optionally validate PDF structure for all PDFs (recommended)
   - **Full backward compatibility** - no changes to existing flows

3. **Common improvements**:
   - Return consistent error responses using existing error constants
   - Structured logging for all validation failures

**Impact**:
- ✅ **NO BREAKING CHANGE**: Existing JPEG/PNG uploads for non-bank-statement types unaffected
- ⚠️ **Targeted restriction**: Bank statement uploads (type="bank_statement") now PDF-only
- Synchronous validation with <2 second target latency
- No API contract changes (same endpoint, same parameters, same error format)

---

## Goals

### In Scope
- ✅ **Conditional PDF-ONLY restriction**: Reject non-PDF files ONLY when `type = "bank_statement"`
- ✅ **Backward compatibility**: Maintain JPEG/PNG support for other upload types
- ✅ PDF structural validation using Apache PDFBox
- ✅ Rejection of corrupted, encrypted, and password-protected PDFs
- ✅ Support for all standard PDF versions (1.4, 1.7, 2.0)
- ✅ Comprehensive unit test coverage (14 test cases - bank statement + other types)
- ✅ Performance target: <2 seconds total response time
- ✅ Synchronous validation within request lifecycle

### Non-Goals (Out of Scope)
- ❌ PDF content validation (malicious scripts, embedded content)
- ❌ OCR or text extraction from PDFs
- ❌ PDF conversion, optimization, or repair
- ❌ Asynchronous validation implementation
- ❌ File size limit modifications (keeping 5MB)
- ❌ Multi-page vs single-page restrictions
- ❌ Specific PDF version requirements (accept all valid versions)
- ❌ Breaking changes to existing JPEG/PNG upload flows ✅ **Backward compatible**
- ❌ Creating a separate endpoint for bank statement uploads (reuse existing endpoint)

---

## Design

### Architecture Overview

```
┌─────────────────┐
│  Client Request │
│  POST /upload   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│  UploadFileController.java:27               │
│  - Accepts: MultipartFile, type, fileName   │
│  - Returns: ResponseDto<String>             │
└────────┬────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│  UploadFileServiceImpl.uploadFile():29      │
│  1. MIME detection (Tika) - line 31-33      │
│  2. Filter file type - line 32              │
│  3. Extension validation - line 34-35       │
│  4. [NEW] PDF validation - after line 35    │ ◄── New Component
│  5. S3 upload - line 38                     │
└────────┬────────────────────────────────────┘
         │
         ├──────────────────┬─────────────────┐
         │                  │                 │
         ▼                  ▼                 ▼
┌────────────────┐  ┌──────────────┐  ┌──────────────────┐
│ Apache Tika    │  │ CommonUtility│  │ [NEW] CommonUtil │
│ MIME Detection │  │ Extension    │  │ PDF Validation   │
│ (existing)     │  │ Validation   │  │ (new method)     │
└────────────────┘  └──────────────┘  └──────────────────┘
```

### Key Components

#### 1. New Validation Method: `CommonUtility.validatePdfStructure()`

**Location**: `src/main/java/com/bukuwarung/fsbnplservice/util/CommonUtility.java`

**Purpose**: Validate PDF structural integrity using Apache PDFBox

**Design**:
```java
/**
 * Validates PDF file structure and ensures the file is openable.
 *
 * @param file The MultipartFile to validate
 * @throws ResponseStatusException with BAD_REQUEST if PDF is invalid
 * @throws IOException if file reading fails
 */
public static void validatePdfStructure(MultipartFile file) throws IOException {
  try (PDDocument document = PDDocument.load(file.getInputStream())) {

    // Check if PDF is encrypted
    if (document.isEncrypted()) {
      log.warn("Encrypted PDF upload rejected: fileName={}", file.getOriginalFilename());
      throw new ResponseStatusException(
          HttpStatus.BAD_REQUEST,
          ErrorConstants.INVALID_FILE_TYPE
      );
    }

    // Check if document can be accessed (password-protected check)
    // PDFBox will throw IOException if password is required

    // Verify document has at least basic structure
    if (document.getNumberOfPages() == 0) {
      log.warn("Empty PDF upload rejected: fileName={}", file.getOriginalFilename());
      throw new ResponseStatusException(
          HttpStatus.BAD_REQUEST,
          ErrorConstants.INVALID_FILE_TYPE
      );
    }

    // If we reach here, PDF is valid and openable
    log.debug("PDF validation successful: fileName={}, pages={}",
        file.getOriginalFilename(),
        document.getNumberOfPages()
    );

  } catch (InvalidPasswordException e) {
    log.warn("Password-protected PDF upload rejected: fileName={}",
        file.getOriginalFilename());
    throw new ResponseStatusException(
        HttpStatus.BAD_REQUEST,
        ErrorConstants.INVALID_FILE_TYPE
    );
  } catch (IOException e) {
    log.warn("Corrupted PDF upload rejected: fileName={}, error={}",
        file.getOriginalFilename(),
        e.getMessage()
    );
    throw new ResponseStatusException(
        HttpStatus.BAD_REQUEST,
        ErrorConstants.INVALID_FILE_TYPE
    );
  }
}
```

**Key Features**:
- ✅ Validates PDF can be loaded and parsed
- ✅ Detects encrypted PDFs via `document.isEncrypted()`
- ✅ Detects password-protected PDFs via `InvalidPasswordException`
- ✅ Detects corrupted PDFs via `IOException` during load
- ✅ Logs all validation failures at WARN level with context
- ✅ Uses existing `ErrorConstants.INVALID_FILE_TYPE` for consistency
- ✅ Properly closes PDDocument via try-with-resources

**Exception Handling**:
| Exception Type | Cause | Response |
|----------------|-------|----------|
| `InvalidPasswordException` | Password-protected PDF | `HttpStatus.BAD_REQUEST` + `INVALID_FILE_TYPE` |
| `IOException` (load failure) | Corrupted/malformed PDF | `HttpStatus.BAD_REQUEST` + `INVALID_FILE_TYPE` |
| Encrypted check = true | Encrypted PDF | `HttpStatus.BAD_REQUEST` + `INVALID_FILE_TYPE` |
| Pages = 0 | Empty/invalid PDF | `HttpStatus.BAD_REQUEST` + `INVALID_FILE_TYPE` |

---

#### 2. Modified Service Method: `UploadFileServiceImpl.uploadFile()`

**Location**: `src/main/java/com/bukuwarung/fsbnplservice/service/impl/UploadFileServiceImpl.java:29-40`

**Current Implementation**:
```java
@Override
public String uploadFile(MultipartFile file, String type, String uploadFileName) throws IOException {
  String mimeType = CommonUtility.filterUploadFileType(
      tika.detect(file.getBytes(), file.getOriginalFilename()));
  filterUploadFileNameExtension(mimeType, uploadFileName);
  filterUploadFileNameExtension(mimeType, Objects.requireNonNull(file.getOriginalFilename()));
  String fileName = System.currentTimeMillis() + "_" + uploadFileName;
  String key = MessageConstants.Folder_Name + type + MessageConstants.FILE_PATH + fileName;
  s3Client.putObject(bucketName, key, file.getInputStream(), null);
  return fileName;
}
```

**Modified Implementation** (changes highlighted):
```java
@Override
public String uploadFile(MultipartFile file, String type, String uploadFileName) throws IOException {
  // Existing MIME type detection
  String mimeType = CommonUtility.filterUploadFileType(
      tika.detect(file.getBytes(), file.getOriginalFilename()));

  // Existing extension validation
  filterUploadFileNameExtension(mimeType, uploadFileName);
  filterUploadFileNameExtension(mimeType, Objects.requireNonNull(file.getOriginalFilename()));

  // [NEW] Conditional logic based on upload type
  if ("bank_statement".equalsIgnoreCase(type)) {
    // [NEW] Enforce PDF-only for bank statements
    if (!CommonConstants.PDF_MIME_TYPE.equals(mimeType)) {
      log.warn("Non-PDF file upload rejected for bank statement: fileName={}, mimeType={}",
          file.getOriginalFilename(), mimeType);
      throw new ResponseStatusException(
          HttpStatus.BAD_REQUEST,
          ErrorConstants.INVALID_FILE_TYPE
      );
    }
    // [NEW] Validate PDF structure for bank statements
    CommonUtility.validatePdfStructure(file);
  } else {
    // [OPTIONAL] Validate PDF structure for all PDFs (recommended)
    if (CommonConstants.PDF_MIME_TYPE.equals(mimeType)) {
      CommonUtility.validatePdfStructure(file);
    }
  }

  // Existing S3 upload flow
  String fileName = System.currentTimeMillis() + "_" + uploadFileName;
  String key = MessageConstants.Folder_Name + type + MessageConstants.FILE_PATH + fileName;
  s3Client.putObject(bucketName, key, file.getInputStream(), null);
  return fileName;
}
```

**Key Changes**:
1. ✅ **Added conditional logic** based on `type` parameter
2. ⚠️ **PDF-only enforcement** when `type = "bank_statement"`
3. ✅ **Backward compatibility** for other types (JPEG, PNG, PDF all accepted)
4. ✅ **Optional PDF validation** for all PDFs (recommended for consistency)
5. ✅ **Clear logging** differentiates bank statement rejections from general validation

**Integration Point**: Conditional check executes **after** MIME type and extension validation, **before** S3 upload.

**Backward Compatibility**: ✅ **JPEG and PNG uploads work normally** for non-bank-statement types. Only bank statement uploads are restricted to PDF-only.

---

### Data Flow

#### Current Flow (Accepts JPEG/PNG/PDF - All Types)
```
1. Client uploads file → UploadFileController
2. MIME detection via Tika
3. filterUploadFileType() → accepts PDF, JPEG, PNG
4. Extension validation → filterUploadFileNameExtension()
5. S3 upload → return fileName
```

#### New Flow (Conditional - Bank Statement PDF-Only)
```
1. Client uploads file → UploadFileController (with type parameter)
2. MIME detection via Tika
3. filterUploadFileType() → accepts PDF, JPEG, PNG
4. Extension validation → filterUploadFileNameExtension()
5. [NEW] Conditional check based on type:

   IF type = "bank_statement":
     a. Check if MIME type is PDF
        - If NOT PDF → log.warn() → throw 400 Bad Request → REJECT
     b. Validate PDF structure:
        - Load PDF with PDFBox
        - Check encryption status
        - Check password protection
        - Verify page count > 0
        - Close PDDocument
        - If validation fails → throw 400 Bad Request
     c. Proceed to S3 upload

   ELSE (type ≠ "bank_statement"):
     a. [OPTIONAL] If MIME type is PDF → validate PDF structure
     b. Proceed to S3 upload (accepts JPEG, PNG, PDF)

6. S3 upload → return fileName
```

**Key Differences**:
- ✅ **Backward compatible** - Non-bank-statement types unchanged
- ⚠️ **Bank statements restricted to PDF-only** (new requirement)
- ✅ **JPEG and PNG uploads still work** for other types
- ✅ **PDF validation** for all PDFs (recommended) or just bank statements

#### Error Response Flow
```
PDF Validation Failure
  ↓
CommonUtility.validatePdfStructure() throws ResponseStatusException
  ↓
UploadFileServiceImpl.uploadFile() propagates exception
  ↓
UploadFileController catches exception (Spring auto-handling)
  ↓
Response: {
  "status": 400,
  "error": "Bad Request",
  "message": "Upload file type is invalid"
}
```

**Note**: Error response format follows existing Spring Boot exception handling pattern.

---

### Health Monitoring & Failover Logic

**Not Applicable**: This is a synchronous validation enhancement with no distributed components or failover requirements.

**Performance Monitoring**:
- Monitor endpoint response time (target: <2 seconds)
- Track PDF validation failure rate
- Alert if validation latency exceeds 1.5 seconds

---

### Override Mechanism

**Not Applicable**: No manual override or routing logic in this enhancement.

---

### Auditing & Logging

**Log Events**:

| Event | Level | Message Template | Context Variables |
|-------|-------|------------------|-------------------|
| Encrypted PDF rejected | WARN | `Encrypted PDF upload rejected: fileName={}` | `file.getOriginalFilename()` |
| Password-protected rejected | WARN | `Password-protected PDF upload rejected: fileName={}` | `file.getOriginalFilename()` |
| Corrupted PDF rejected | WARN | `Corrupted PDF upload rejected: fileName={}, error={}` | `file.getOriginalFilename()`, `e.getMessage()` |
| Empty PDF rejected | WARN | `Empty PDF upload rejected: fileName={}` | `file.getOriginalFilename()` |
| Valid PDF accepted | DEBUG | `PDF validation successful: fileName={}, pages={}` | `file.getOriginalFilename()`, `document.getNumberOfPages()` |

**Log Format**: Existing Logstash logback encoder (structured JSON logging)

**Log Destination**: Application logs (existing infrastructure)

**Retention**: Follow existing log retention policy (not modified)

**No PII**: File names are logged, but no file content or user data is logged.

---

## Implementation Plan

### Phase 1: Dependency Setup (Day 1)
**Duration**: 1-2 hours

1. ✅ Add Apache PDFBox dependency to `build.gradle`:
   ```gradle
   implementation 'org.apache.pdfbox:pdfbox:2.0.27'
   ```

2. ✅ Run `./gradlew build --refresh-dependencies` to verify:
   - Dependency resolution succeeds
   - No version conflicts with existing dependencies
   - Build passes with new dependency

3. ✅ Verify compatibility:
   - Java 11 compatibility ✓
   - Spring Boot 2.6.3 compatibility ✓
   - No conflicts with Apache Tika 3.0.0 ✓

### Phase 2: Test Data Preparation (Day 1)
**Duration**: 2-3 hours

1. ✅ Create test resources directory:
   ```bash
   mkdir -p src/test/resources/test-files
   ```

2. ✅ Gather/generate test PDF samples:
   - `valid-pdf-1.4.pdf` (PDF version 1.4)
   - `valid-pdf-1.7.pdf` (PDF version 1.7)
   - `valid-pdf-2.0.pdf` (PDF version 2.0)
   - `large-pdf-4.9mb.pdf` (close to 5MB limit)
   - `corrupted-pdf.pdf` (truncated/malformed)
   - `empty-pdf.pdf` (0 bytes or invalid structure)
   - `password-protected.pdf` (password-protected)
   - `encrypted.pdf` (encrypted)
   - `valid-image.jpg` (regression test)
   - `valid-image.png` (regression test)

3. ✅ Document test data sources in `test-files/README.md`

### Phase 3: Core Implementation (Day 2)
**Duration**: 4-6 hours

1. ✅ Create `validatePdfStructure()` method in `CommonUtility.java`:
   - Add imports: `org.apache.pdfbox.pdmodel.PDDocument`, `org.apache.pdfbox.pdmodel.encryption.InvalidPasswordException`
   - Implement validation logic per design specification
   - Add structured logging with context

2. ✅ Integrate validation into `UploadFileServiceImpl.uploadFile()`:
   - Add conditional check: `if (PDF_MIME_TYPE.equals(mimeType))`
   - Call `CommonUtility.validatePdfStructure(file)`
   - Maintain existing S3 upload flow

3. ✅ Run formatter and linter:
   ```bash
   ./gradlew spotlessApply
   ./gradlew checkstyleMain
   ```

4. ✅ Compile and verify:
   ```bash
   ./gradlew clean build -x test
   ```

### Phase 4: Testing (Day 3)
**Duration**: 6-8 hours

1. ✅ Create comprehensive unit tests in `UploadFileServiceTest.java`:

**Test Cases** (14 total):

```java
// ===== Bank Statement Upload Tests (type = "bank_statement") =====

@Test
void testUploadValidPdf_BankStatement() {
  /* PDF upload successful for bank statement */
}

@Test
void testUploadCorruptedPdf_BankStatement() {
  /* Corrupted PDF rejected for bank statement */
}

@Test
void testUploadPasswordProtectedPdf_BankStatement() {
  /* Password-protected PDF rejected for bank statement */
}

@Test
void testUploadEncryptedPdf_BankStatement() {
  /* Encrypted PDF rejected for bank statement */
}

@Test
void testUploadJpegFile_BankStatement_Rejected() {
  /* ⚠️ JPEG rejected for bank statement (PDF-only enforcement) */
}

@Test
void testUploadPngFile_BankStatement_Rejected() {
  /* ⚠️ PNG rejected for bank statement (PDF-only enforcement) */
}

// ===== Other Upload Type Tests (type ≠ "bank_statement") =====

@Test
void testUploadJpegFile_OtherType_Success() {
  /* ✅ JPEG upload works for other types (backward compatible) */
}

@Test
void testUploadPngFile_OtherType_Success() {
  /* ✅ PNG upload works for other types (backward compatible) */
}

@Test
void testUploadPdfFile_OtherType_Success() {
  /* ✅ PDF upload works for other types */
}

// ===== PDF Validation Tests (All PDF uploads) =====

@Test
void testUploadPdfVersion14() {
  /* PDF 1.4 accepted */
}

@Test
void testUploadPdfVersion17() {
  /* PDF 1.7 accepted */
}

@Test
void testUploadPdfVersion20() {
  /* PDF 2.0 accepted */
}

@Test
void testUploadLargePdf() {
  /* 4.9MB PDF accepted */
}

@Test
void testUploadEmptyPdf() {
  /* Empty/invalid PDF rejected */
}
```

**Note**: Test count 14 to cover both bank statement (PDF-only) and other upload types (backward compatible).

2. ✅ Run test suite:
   ```bash
   ./gradlew test --tests UploadFileServiceTest
   ```

3. ✅ Verify test coverage:
   ```bash
   ./gradlew jacocoTestReport
   # Check coverage report: build/reports/jacoco/test/html/index.html
   # Target: >80% coverage for new code
   ```

4. ✅ Manual testing:
   - Start local service
   - Upload each test file via Postman/curl
   - Verify responses match expectations
   - Check logs for proper WARN/DEBUG messages

### Phase 5: Code Review & Documentation (Day 4)
**Duration**: 3-4 hours

1. ✅ Self-review checklist:
   - [ ] Code follows Google Java Format
   - [ ] All test cases pass
   - [ ] No SonarQube critical/blocker issues
   - [ ] Javadoc complete for public methods
   - [ ] Error messages user-friendly
   - [ ] Logging follows existing patterns

2. ✅ Run quality gates:
   ```bash
   ./gradlew sonarqube
   # Check: 0 critical/blocker issues
   ```

3. ✅ Update inline documentation:
   - Add Javadoc to `validatePdfStructure()`
   - Update method comments in `uploadFile()`

4. ✅ Submit for PR review with checklist:
   - [x] All tests pass
   - [x] Backward compatibility verified
   - [x] No new dependencies beyond PDFBox
   - [x] Error handling consistent
   - [x] Logging structured and appropriate

---

## Rollback Plan

**Rollback Complexity**: ✅ **LOW** (backward compatible change, minimal risk)

### Immediate Rollback (if production issues detected)

**Option 1: Disable PDF-Only Enforcement for Bank Statements**
```java
// In UploadFileServiceImpl.uploadFile()
// Comment out or remove the bank_statement conditional block
if ("bank_statement".equalsIgnoreCase(type)) {
  // Temporarily disabled - rollback to accept all file types
  // if (!CommonConstants.PDF_MIME_TYPE.equals(mimeType)) {
  //   throw new ResponseStatusException(...);
  // }
}
```
- **Advantage**: Quick fix, restores previous behavior for bank statements
- **Duration**: ~5-10 minutes hotfix deployment
- **Risk**: Very low (only affects bank statement uploads)

**Option 2: Code Revert + Redeploy**
```bash
git revert <commit-hash>
./gradlew build
# Deploy previous version
```
- **Duration**: ~15-30 minutes
- **Risk**: Very low (backward compatible change)

### Partial Rollback (if PDF validation has issues)

If PDF validation logic causes problems:
1. Keep PDF-only enforcement for bank statements
2. Temporarily disable PDF structure validation
3. Investigate and fix validation logic
4. Re-enable validation after fix

### Rollback Triggers

Monitor for:
- ❌ Bank statement upload failure rate >20% (expected: <5%)
- ❌ Endpoint latency >2 seconds (p95) for bank statement uploads
- ❌ Spike in support tickets about bank statement upload failures
- ⚠️ Non-bank-statement uploads affected (should never happen - indicates bug)

**Rollback Decision Owner**: Engineering Lead + Product Manager

**Note**: Since this is backward compatible for non-bank-statement types, rollback risk is minimal and only affects bank statement uploads.

---

## Testing Strategy

### Unit Tests (Primary Coverage)
**Location**: `src/test/java/com/bukuwarung/fsbnplservice/service/UploadFileServiceTest.java`

**Mock Strategy**:
- Mock S3Client (existing)
- Mock Tika (existing)
- Use real MultipartFile with test PDF resources
- Use real PDFBox (no mocking of PDF validation logic)

**Test Matrix**:

| Test Case | Test File | Type Parameter | Expected Result | Validates |
|-----------|-----------|----------------|-----------------|-----------|
| **Bank Statement Tests** | | | | |
| Valid PDF (bank stmt) | `valid-pdf-1.7.pdf` | `bank_statement` | Success (200) | PDF-only enforcement |
| Corrupted PDF (bank stmt) | `corrupted-pdf.pdf` | `bank_statement` | Exception (400) | Corruption detection |
| Password PDF (bank stmt) | `password-protected.pdf` | `bank_statement` | Exception (400) | Password detection |
| Encrypted PDF (bank stmt) | `encrypted.pdf` | `bank_statement` | Exception (400) | Encryption detection |
| **⚠️ JPEG (bank stmt)** | `valid-image.jpg` | `bank_statement` | **Exception (400)** | **PDF-only: JPEG rejected** |
| **⚠️ PNG (bank stmt)** | `valid-image.png` | `bank_statement` | **Exception (400)** | **PDF-only: PNG rejected** |
| **Other Type Tests** | | | | |
| **✅ JPEG (other type)** | `valid-image.jpg` | `invoice` | **Success (200)** | **Backward compat: JPEG works** |
| **✅ PNG (other type)** | `valid-image.png` | `invoice` | **Success (200)** | **Backward compat: PNG works** |
| **✅ PDF (other type)** | `valid-pdf-1.7.pdf` | `invoice` | **Success (200)** | **PDF works for all types** |
| **PDF Validation Tests** | | | | |
| PDF 1.4 | `valid-pdf-1.4.pdf` | `any` | Success (200) | PDF 1.4 support |
| PDF 1.7 | `valid-pdf-1.7.pdf` | `any` | Success (200) | PDF 1.7 support |
| PDF 2.0 | `valid-pdf-2.0.pdf` | `any` | Success (200) | PDF 2.0 support |
| Large PDF (4.9MB) | `large-pdf-4.9mb.pdf` | `any` | Success (200) | Size limit handling |
| Empty PDF | `empty-pdf.pdf` | `any` | Exception (400) | Empty file handling |

**Coverage Target**: >85% line coverage for new code (conditional logic increases coverage needs)

---

### Integration Tests (Optional)
**Scope**: End-to-end upload flow with real S3

**Not Prioritized**: Unit tests provide sufficient coverage for initial release. Integration tests can be added post-MVP if needed.

---

### Performance Testing
**Tool**: JMeter or Gatling

**Scenarios**:
1. **Baseline**: Current endpoint performance without PDF validation
2. **With Validation**: Same test with PDF validation enabled
3. **Target**: <2 seconds p95 latency for 5MB PDF upload

**Test Plan**:
```
- Concurrent users: 10
- Ramp-up: 30 seconds
- Duration: 5 minutes
- File: 3MB valid PDF
- Expected p95: <2 seconds
- Expected p99: <3 seconds
```

**Acceptance Criteria**: p95 latency increase <500ms compared to baseline

---

### Manual Testing Checklist

**Bank Statement Uploads (type = "bank_statement")**:

*Happy Path*:
- [  ] Upload valid PDF → Success (200)
- [  ] Upload large PDF (4.9MB) → Success (200)

*Error Scenarios*:
- [  ] Upload corrupted PDF → 400 error with `INVALID_FILE_TYPE`
- [  ] Upload password-protected PDF → 400 error with `INVALID_FILE_TYPE`
- [  ] Upload encrypted PDF → 400 error with `INVALID_FILE_TYPE`
- [  ] Upload empty PDF → 400 error with `INVALID_FILE_TYPE`
- [  ] **Upload JPEG file → 400 error** (PDF-only enforcement)
- [  ] **Upload PNG file → 400 error** (PDF-only enforcement)
- [  ] Upload 6MB PDF → 413 Payload Too Large (existing behavior)

**Other Upload Types (type ≠ "bank_statement", e.g., "invoice")**:

*Backward Compatibility Verification*:
- [  ] **Upload JPEG file → Success (200)** ✅ Backward compatible
- [  ] **Upload PNG file → Success (200)** ✅ Backward compatible
- [  ] Upload valid PDF → Success (200) ✅ PDF works for all types
- [  ] Upload 6MB JPEG → 413 Payload Too Large (existing behavior)

**Log Verification**:
- [  ] WARN log for encrypted PDF rejection
- [  ] WARN log for password-protected PDF rejection
- [  ] WARN log for corrupted PDF rejection
- [  ] **WARN log for non-PDF bank statement rejection** (specific message)
- [  ] DEBUG log for valid PDF acceptance
- [  ] **No WARN logs for JPEG/PNG uploads when type ≠ "bank_statement"**

---

## Open Questions

### 1. Test Data Acquisition Strategy
**Question**: How to obtain encrypted and password-protected PDF samples for testing?

**Options**:
- **Option A**: Generate programmatically using PDFBox in test setup - !!SELECTED OPTION
- **Option B**: Download from external sources (e.g., PDF test corpus)
- **Option C**: Create manually using PDF tools (Adobe Acrobat, LibreOffice)

**Recommendation**: **Option A** (programmatic generation) for reproducibility and automation.

Example test setup:
```java
@BeforeAll
static void generateTestPdfs() throws IOException {
  // Generate password-protected PDF
  PDDocument doc = new PDDocument();
  doc.addPage(new PDPage());
  StandardProtectionPolicy policy = new StandardProtectionPolicy("password", "", new AccessPermission());
  doc.protect(policy);
  doc.save("src/test/resources/test-files/password-protected.pdf");
  doc.close();
}
```

**Status**: Answered


## Prior Art / Alternatives Considered

### Alternative 1: Apache Tika PDF Validation
**Description**: Use Apache Tika's PDF parser for validation instead of PDFBox

**Pros**:
- Already a project dependency (no new dependency)
- Tika can detect corrupted PDFs

**Cons**:
- ❌ Tika doesn't provide explicit encryption/password detection APIs
- ❌ Less control over validation logic
- ❌ Error messages less specific

**Rejection Reason**: Insufficient API for encryption and password detection requirements

---

### Alternative 2: iText Library
**Description**: Use iText for PDF validation

**Pros**:
- Comprehensive PDF manipulation library
- Strong validation capabilities

**Cons**:
- ❌ Dual licensing (AGPL or commercial license)
- ❌ Licensing costs for commercial use
- ❌ Heavier dependency than PDFBox

**Rejection Reason**: Licensing complexity and cost. PDFBox (Apache License 2.0) is open-source friendly.

---

### Alternative 3: Asynchronous Validation
**Description**: Validate PDFs asynchronously after S3 upload

**Pros**:
- Faster initial response to user
- Non-blocking upload flow

**Cons**:
- ❌ Increases complexity (queue, workers, notifications)
- ❌ User uploads invalid file, discovers later (poor UX)
- ❌ Wasted S3 storage for invalid files
- ❌ Requires new infrastructure (SQS, workers)

**Rejection Reason**: Complexity not justified for <2 second synchronous validation. Better UX to fail fast.

---

### Alternative 4: Client-Side Validation
**Description**: Validate PDFs in frontend before upload

**Pros**:
- Reduces backend load
- Instant feedback to user

**Cons**:
- ❌ Client-side validation easily bypassed
- ❌ Not enforceable for API integrations
- ❌ Requires JavaScript PDF libraries (large bundle size)

**Rejection Reason**: Cannot replace server-side validation. Can be added as complementary UX improvement later.

---

## Security & Compliance

### Security Considerations

#### 1. PDF Bomb Protection
**Risk**: Large PDFs with excessive compression ratios (PDF bombs) could cause memory exhaustion

**Mitigation**:
- ✅ Existing 5MB file size limit provides primary protection
- ✅ PDFBox 2.0.27 includes PDF bomb protections
- ✅ JVM heap size limits prevent out-of-memory

**Additional Safeguard** (optional):
```java
// In validatePdfStructure()
if (document.getNumberOfPages() > 1000) {
  throw new ResponseStatusException(
    HttpStatus.BAD_REQUEST,
    "PDF page count exceeds maximum allowed"
  );
}
```

**Recommendation**: Monitor memory usage post-deployment. Add page limit if issues observed.

---

#### 2. Path Traversal in File Names
**Risk**: Malicious file names with path traversal characters (`../`, etc.)

**Current Protection**: Existing `filterUploadFileNameExtension()` validates extension only, doesn't sanitize path

**Status**: ✅ **Out of scope for this RFC** - Separate security issue that should be addressed independently

**Recommendation**: File separate ticket for file name sanitization

---

#### 3. Encrypted PDF Handling
**Risk**: Encrypted PDFs could contain malicious content hidden from validation

**Mitigation**: ✅ **Explicitly rejected** - Encrypted PDFs are blocked by validation

---

#### 4. Denial of Service via Corrupted PDFs
**Risk**: Malicious users could flood endpoint with corrupted PDFs to consume resources

**Mitigation**:
- ✅ Fast-fail validation (<500ms for corrupted PDFs)
- ✅ Existing rate limiting (if configured at API gateway level)
- ✅ Error logging doesn't leak system information

**Recommendation**: Ensure API gateway rate limiting is configured (separate infrastructure concern)

---

### Compliance Considerations

#### GDPR / Data Privacy
- ✅ **No PII logged**: Only file names logged, no file content
- ✅ **No data retention change**: Files stored same as before (S3)
- ✅ **No new data processing**: Validation reads but doesn't persist PDF content

**Impact**: ✅ **No GDPR implications**

---

#### Audit Trail
- ✅ All PDF rejections logged at WARN level
- ✅ Logs include: timestamp, filename, rejection reason
- ✅ Logs structured (JSON) for easy querying

**Compliance**: ✅ Meets existing audit requirements

---

## Performance Impact

### Latency Analysis

#### Current Baseline (Estimated)
| Operation | Latency |
|-----------|---------|
| MIME detection (Tika) | ~50-100ms |
| Extension validation | ~5ms |
| S3 upload (3MB file) | ~800-1200ms |
| **Total (current)** | **~1-1.5 seconds** |

#### With PDF Validation (Projected)
| Operation | Latency |
|-----------|---------|
| MIME detection (Tika) | ~50-100ms |
| Extension validation | ~5ms |
| **PDF validation (new)** | **~200-500ms** |
| S3 upload (3MB file) | ~800-1200ms |
| **Total (with validation)** | **~1.2-2 seconds** |

**Worst Case**: 2 seconds (5MB PDF with complex structure)
**Target**: <2 seconds (p95)
**Expected**: 1.5-1.8 seconds (p95)

---

### Throughput Impact

**Assumptions**:
- Single-threaded synchronous validation
- No connection pooling changes
- Existing thread pool configuration

**Analysis**:
- **Current throughput**: Limited by S3 upload, not CPU
- **With validation**: +200-500ms per PDF upload
- **Impact on non-PDF uploads**: ✅ **Zero** (validation bypassed)

**Bottleneck**: S3 upload remains the bottleneck, not PDF validation

**Conclusion**: ✅ Minimal throughput impact (<10% for PDF uploads only)

---

### Scalability Implications

#### Memory Usage
**PDFBox Memory Profile**:
- Small PDFs (<1MB): ~10-20MB heap per document
- Large PDFs (5MB): ~50-80MB heap per document
- Document closed after validation (memory released)

**Concurrent Uploads**:
- 10 concurrent 5MB PDF uploads: ~500-800MB heap usage
- JVM heap size: TBD (check `application.properties` or deployment config)

**Recommendation**:
- Monitor heap usage post-deployment
- Ensure JVM `-Xmx` configured for peak load
- Typical recommendation: `-Xmx2G` for moderate load

---

#### CPU Usage
**PDF Validation CPU Profile**:
- Mostly I/O-bound (reading file structure)
- Minimal CPU for structure parsing
- No compression/decompression (just validation)

**Expected CPU Impact**: <10% increase during PDF uploads

---

### Performance Testing Plan

**Pre-Deployment**:
1. Baseline test: 100 uploads without validation
2. With validation test: 100 uploads with validation
3. Compare p50, p95, p99 latencies

**Success Criteria**:
- ✅ p95 latency <2 seconds
- ✅ p99 latency <3 seconds
- ✅ No memory leaks (heap stable over 10-minute test)
- ✅ No CPU spikes >80%

---

## Monitoring & Observability

### Metrics to Track

#### Application Metrics (via Micrometer/Prometheus)

**Endpoint Metrics**:
```java
// Counter: PDF validation attempts
pdf.validation.attempts{status="success|failure", reason="encrypted|password|corrupted|empty"}

// Timer: PDF validation duration
pdf.validation.duration{percentile="p50|p95|p99"}

// Counter: Upload endpoint responses
upload.file.responses{status="200|400|413|500", file_type="pdf|jpeg|png"}
```

**Business Metrics**:
```java
// Gauge: PDF validation failure rate
pdf.validation.failure.rate

// Counter: File uploads by type
file.uploads.total{file_type="pdf|jpeg|png"}
```

---

### Logging Strategy

**Log Levels**:

| Level | Event | Purpose |
|-------|-------|---------|
| **DEBUG** | Valid PDF accepted | Development/troubleshooting |
| **WARN** | PDF validation failure (encrypted, password, corrupted) | Operational monitoring |
| **ERROR** | Unexpected exceptions in validation logic | Critical errors requiring attention |

**Structured Log Format** (JSON via Logstash):
```json
{
  "timestamp": "2025-01-18T10:15:30.123Z",
  "level": "WARN",
  "logger": "com.bukuwarung.fsbnplservice.util.CommonUtility",
  "message": "Encrypted PDF upload rejected",
  "context": {
    "fileName": "document_12345.pdf",
    "mimeType": "application/pdf",
    "fileSize": 2048000,
    "rejectionReason": "encrypted"
  }
}
```

---

### Alerting Rules

**Critical Alerts** (PagerDuty/Opsgenie):
- ❌ PDF validation failure rate >20% for 5 minutes
- ❌ Endpoint p95 latency >3 seconds for 5 minutes
- ❌ Upload endpoint error rate >10% for 5 minutes

**Warning Alerts** (Slack/Email):
- ⚠️ PDF validation failure rate >10% for 10 minutes
- ⚠️ Endpoint p95 latency >2 seconds for 10 minutes
- ⚠️ Spike in password-protected PDF rejections (>5/min)

**Informational**:
- ℹ️ Daily summary: Total uploads, validation success/failure breakdown

---

### Dashboard Widgets

**Grafana Dashboard: "File Upload Service"**

**Panel 1: Upload Volume**
- Line chart: Uploads per minute by file type (PDF, JPEG, PNG)

**Panel 2: PDF Validation Results**
- Pie chart: Success vs Failure breakdown
- Bar chart: Failure reasons (encrypted, password, corrupted, empty)

**Panel 3: Performance**
- Line chart: Endpoint latency (p50, p95, p99)
- Line chart: PDF validation duration (p50, p95, p99)

**Panel 4: Error Rate**
- Line chart: 4xx/5xx error rate over time
- Single stat: Current error rate (last 5 min)

---

### Runbook: High PDF Validation Failure Rate

**Scenario**: Alert fires for >20% PDF validation failure rate

**Investigation Steps**:
1. Check dashboard for failure reason breakdown
2. Query logs for recent WARN messages:
   ```
   level:WARN AND logger:CommonUtility AND message:*PDF*
   ```
3. Identify pattern:
   - **If mostly "encrypted"**: Possible user confusion, update docs
   - **If mostly "password-protected"**: Same as encrypted
   - **If mostly "corrupted"**: Investigate file source (scanner issue?)
   - **If spike in volume**: Potential abuse, check API logs

**Remediation**:
- High false-positive rate: Review validation logic
- User education needed: Update API docs, add user-friendly error messages
- Abuse suspected: Implement rate limiting

---

## Appendix

### A. Dependency Configuration

**File**: `build.gradle`

**Change**:
```gradle
dependencies {
    // Existing dependencies
    implementation 'org.apache.tika:tika-core:3.0.0'
    implementation 'org.apache.tika:tika-parsers:3.0.0'

    // [NEW] PDF validation
    implementation 'org.apache.pdfbox:pdfbox:2.0.27'

    // Other dependencies...
}
```

**Compatibility**:
- **Java**: 11+ ✅
- **Spring Boot**: 2.6.3+ ✅
- **Apache Tika**: 3.0.0 ✅ (no conflicts)

---

### B. Constants Reference

**File**: `src/main/java/com/bukuwarung/fsbnplservice/constants/CommonConstants.java:79-85`

```java
public static final String JPEG_MIME_TYPE = "image/jpeg";     // Line 79
public static final String PNG_MIME_TYPE = "image/png";       // Line 80
public static final String PDF_MIME_TYPE = "application/pdf"; // Line 81
public static final String JPG_EXT = "jpg";                   // Line 82
public static final String JPEG_EXT = "jpeg";                 // Line 83
public static final String PNG_EXT = "png";                   // Line 84
public static final String PDF_EXT = "pdf";                   // Line 85
```

**File**: `src/main/java/com/bukuwarung/fsbnplservice/constants/ErrorConstants.java:9`

```java
public static final String INVALID_FILE_TYPE = "Upload file type is invalid"; // Line 9
```

---

### C. Configuration Reference

**File**: `src/main/resources/application.properties:45-46`

```properties
spring.servlet.multipart.max-file-size = ${MULTIPART_FILE_SIZE:5MB}     # Line 45
spring.servlet.multipart.max-request-size = ${MULTIPART_REQUEST_SIZE:5MB} # Line 46
```

**No changes required** - Existing 5MB limit maintained

---

### D. Test Data Generation Script

**Optional**: Script to generate test PDFs programmatically

**File**: `src/test/java/com/bukuwarung/fsbnplservice/util/TestPdfGenerator.java`

```java
public class TestPdfGenerator {

  public static void generatePasswordProtectedPdf() throws IOException {
    PDDocument doc = new PDDocument();
    doc.addPage(new PDPage());

    StandardProtectionPolicy policy = new StandardProtectionPolicy(
        "password123", "", new AccessPermission());
    doc.protect(policy);

    doc.save("src/test/resources/test-files/password-protected.pdf");
    doc.close();
  }

  public static void generateEncryptedPdf() throws IOException {
    PDDocument doc = new PDDocument();
    doc.addPage(new PDPage());

    StandardProtectionPolicy policy = new StandardProtectionPolicy(
        "", "", new AccessPermission());
    policy.setEncryptionKeyLength(128);
    doc.protect(policy);

    doc.save("src/test/resources/test-files/encrypted.pdf");
    doc.close();
  }

  public static void generateCorruptedPdf() throws IOException {
    // Write incomplete PDF structure
    FileOutputStream fos = new FileOutputStream(
        "src/test/resources/test-files/corrupted.pdf");
    fos.write("%PDF-1.4\n".getBytes());
    fos.write("incomplete structure".getBytes());
    fos.close();
  }
}
```

---

### E. Example API Request/Response

**Request**:
```bash
curl -X POST http://localhost:8080/upload/file \
  -H "Authorization: Bearer ${FIREBASE_TOKEN}" \
  -F "type=INVOICE" \
  -F "uploadfileName=merchant_agreement.pdf" \
  -F "file=@document.pdf"
```

**Response (Success)**:
```json
{
  "status": "success",
  "message": "file uploaded successfully",
  "data": "1705578930123_merchant_agreement.pdf"
}
```

**Response (Validation Failure)**:
```json
{
  "timestamp": "2025-01-18T10:15:30.123+00:00",
  "status": 400,
  "error": "Bad Request",
  "message": "Upload file type is invalid",
  "path": "/upload/file"
}
```

---

### F. Codebase File References

| Component | File Path | Lines |
|-----------|-----------|-------|
| Controller | `src/main/java/com/bukuwarung/fsbnplservice/controller/UploadFileController.java` | 27-39 |
| Service | `src/main/java/com/bukuwarung/fsbnplservice/service/impl/UploadFileServiceImpl.java` | 29-40 |
| Utility | `src/main/java/com/bukuwarung/fsbnplservice/util/CommonUtility.java` | 283-320 |
| Constants | `src/main/java/com/bukuwarung/fsbnplservice/constants/CommonConstants.java` | 79-85 |
| Error Constants | `src/main/java/com/bukuwarung/fsbnplservice/constants/ErrorConstants.java` | 9 |
| Build Config | `build.gradle` | - |
| Properties | `src/main/resources/application.properties` | 45-46 |
| Tests | `src/test/java/com/bukuwarung/fsbnplservice/service/UploadFileServiceTest.java` | - |

---

### G. References

1. **Jira Ticket**: [LND-4561](https://bukuwarung.atlassian.net/browse/LND-4561)
2. **Requirement Document**: `/Users/juvianto.chi/Desktop/code/requirements/LND-4561/requirement.md`
3. **Clarification Document**: `/Users/juvianto.chi/Desktop/code/requirements/LND-4561/clarification.md`
4. **Repository**: `/Users/juvianto.chi/Desktop/code/bnpl`
5. **Apache PDFBox Documentation**: https://pdfbox.apache.org/
6. **Apache PDFBox GitHub**: https://github.com/apache/pdfbox
7. **RFC Generation Guide**: `/Users/juvianto.chi/Desktop/code/documentation/rfc-generation-guide.md`

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-18 | Juvianto Chi | Initial RFC draft based on requirement document and codebase verification |
| 1.1 | 2025-01-18 | Juvianto Chi | **BREAKING CHANGE**: Updated to enforce PDF-only restriction. Added Migration Plan section. Updated implementation to reject all non-PDF files (JPEG, PNG). Updated test cases to verify non-PDF rejection. |

---

**End of RFC**
