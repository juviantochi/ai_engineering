# PDF Validation Enhancement for File Upload Endpoint

## Background

The `/upload/file` endpoint in the BNPL service (`UploadFileController.java`) currently accepts multiple file types (JPEG, PNG, PDF) with basic validation limited to MIME type detection via Apache Tika and file extension checking. There is no validation to ensure that uploaded PDF files are structurally valid, uncorrupted, and can be opened.

**Current Implementation:**
- **Location**: `UploadFileController.java` → `UploadFileServiceImpl.java`
- **Validation**: MIME type detection (Apache Tika 3.0.0) + file extension check (`CommonUtility.java:283-320`)
- **Accepted Types**: JPEG, PNG, PDF (defined in `CommonConstants.java:79-85`)
- **File Size Limit**: 5MB (configured in `application.properties:45-46`)
- **Storage**: AWS S3 bucket
- **Error Handling**: Generic `INVALID_FILE_TYPE` error with `HttpStatus.BAD_REQUEST`

**Business Context:**
Users upload PDF documents (primarily scanned PDFs) to the BNPL merchant onboarding system. Invalid or corrupted PDFs cause downstream processing issues and poor user experience. This enhancement ensures only valid, openable PDF files are accepted.

## Objective

Enhance the `/upload/file` endpoint to validate PDF structural integrity and enforce PDF-only uploads **specifically for bank statement uploads** (`type = "bank_statement"`), while maintaining backward compatibility for other upload types.

## Scope

### In Scope

1. **PDF-Only Upload Restriction for Bank Statements** ⭐ **KEY REQUIREMENT**
   - When `type = "bank_statement"`: ONLY accept PDF files, reject JPEG/PNG
   - When `type ≠ "bank_statement"`: Maintain existing behavior (accept JPEG, PNG, PDF)
   - **No breaking change** - other upload types unaffected

2. **PDF Structure Validation**
   - Validate PDF file structure integrity
   - Verify PDF can be opened and is not corrupted
   - Reject password-protected PDFs
   - Reject encrypted PDFs

3. **Error Handling Enhancement**
   - Return existing `INVALID_FILE_TYPE` error for validation failures
   - Log validation failures at WARN level using existing `@Slf4j`
   - Maintain existing `HttpStatus.BAD_REQUEST` response pattern

4. **Performance Requirements**
   - Synchronous validation within request lifecycle
   - Target validation latency < 2 seconds
   - Maintain existing 5MB file size limit

5. **Unit Test Coverage**
   - Valid PDF upload success scenarios
   - Corrupted PDF rejection
   - Empty PDF file rejection
   - Very large PDF handling (at 5MB limit)
   - Password-protected PDF rejection
   - Encrypted PDF rejection
   - Different PDF versions (1.4, 1.7, 2.0)

6. **Implementation Details**
   - Add Apache PDFBox dependency for PDF validation
   - Integrate validation into existing `UploadFileServiceImpl.uploadFile()` method
   - Maintain existing AWS S3 upload flow for valid PDFs

### Out of Scope

1. **Content-Level Validation**
   - PDF content validation (malicious scripts, embedded content)
   - OCR or text extraction from PDFs
   - Image-only PDF specific validation
   - PDF content structure analysis

2. **File Processing**
   - PDF conversion or optimization
   - PDF version conversion
   - PDF repair or healing of corrupted files

3. **Architectural Changes**
   - Asynchronous validation implementation
   - File size limit modifications (keeping 5MB)
   - Minimum file size enforcement
   - Multi-page vs single-page restrictions

4. **Testing Scope**
   - Concurrent upload scenarios
   - Rate limiting tests
   - Load/performance testing

5. **PDF Feature Support**
   - Multi-page PDF restrictions
   - Specific PDF version requirements (accept all valid versions)
   - Specific content requirements

## Requirements

### Functional Requirements

#### FR-1: PDF Validation
**Priority**: High
**Description**: Validate PDF structural integrity before accepting upload.

**Acceptance Criteria:**
- System MUST validate PDF file structure using Apache PDFBox
- System MUST verify PDF can be opened without errors
- System MUST reject corrupted or malformed PDF files
- System MUST reject password-protected PDF files
- System MUST reject encrypted PDF files
- System MUST accept all standard PDF versions (1.4, 1.7, 2.0, etc.)
- Validation MUST occur before AWS S3 upload

#### FR-2: Error Response Consistency
**Priority**: High
**Description**: Maintain consistent error handling with existing implementation.

**Acceptance Criteria:**
- System MUST return `HttpStatus.BAD_REQUEST` for validation failures
- System MUST use existing `ErrorConstants.INVALID_FILE_TYPE` error message
- System MUST log validation failures at WARN level
- Error response format MUST match existing error response structure

#### FR-3: Conditional File Type Handling
**Priority**: High
**Description**: Enforce PDF-only restriction for bank statements while maintaining backward compatibility.

**Acceptance Criteria:**
- When `type = "bank_statement"`:
  - System MUST accept ONLY PDF files
  - System MUST reject JPEG and PNG files with `INVALID_FILE_TYPE` error
  - System MUST validate PDF structure using Apache PDFBox
- When `type ≠ "bank_statement"`:
  - System MUST continue to support JPEG uploads
  - System MUST continue to support PNG uploads
  - System MUST continue to support PDF uploads
  - PDF validation is OPTIONAL (can be applied to all PDFs or just bank statements)
- System MUST maintain existing MIME type detection via Apache Tika

#### FR-4: Performance Requirements
**Priority**: High
**Description**: Ensure validation does not significantly impact response time.

**Acceptance Criteria:**
- PDF validation MUST complete synchronously within request
- Total endpoint response time MUST be < 2 seconds
- Validation MUST not block other upload requests
- System MUST maintain existing 5MB file size limit

### Non-Functional Requirements

#### NFR-1: Backward Compatibility
**Priority**: High
- Implementation MUST NOT break existing JPEG/PNG upload functionality
- API contract MUST remain unchanged (request/response format)
- Existing error codes and HTTP status codes MUST be maintained

#### NFR-2: Maintainability
**Priority**: Medium
- PDF validation logic MUST be separated into reusable utility method
- Code MUST follow existing project patterns (Google Java Format, Spotless)
- Validation logic MUST be easily testable in isolation

#### NFR-3: Observability
**Priority**: Medium
- Validation failures MUST be logged with sufficient context for debugging
- Log messages MUST include file name, MIME type, and failure reason
- Logs MUST follow existing structured logging pattern (logstash-logback-encoder)

### Technical Requirements

#### TR-1: Dependency Management
**Required Dependency:**
```gradle
implementation 'org.apache.pdfbox:pdfbox:2.0.27'
```

**Rationale:**
- Apache PDFBox is the standard Java library for PDF manipulation and validation
- Compatible with Java 11 and Spring Boot 2.6.3
- Actively maintained and widely used in enterprise applications
- Provides robust PDF structure validation APIs

#### TR-2: Implementation Location
**File**: `src/main/java/com/bukuwarung/fsbnplservice/service/impl/UploadFileServiceImpl.java`

**Method to Modify**: `uploadFile(MultipartFile file, String type, String uploadFileName)`

**Implementation Logic**:
```java
// Conditional logic based on type parameter
if ("bank_statement".equalsIgnoreCase(type)) {
  // Enforce PDF-only for bank statements
  if (!CommonConstants.PDF_MIME_TYPE.equals(mimeType)) {
    throw new ResponseStatusException(HttpStatus.BAD_REQUEST, ErrorConstants.INVALID_FILE_TYPE);
  }
  // Validate PDF structure
  CommonUtility.validatePdfStructure(file);
} else {
  // Existing behavior for other types (accept JPEG, PNG, PDF)
  // Optionally validate PDF structure for all PDFs
  if (CommonConstants.PDF_MIME_TYPE.equals(mimeType)) {
    CommonUtility.validatePdfStructure(file);
  }
}
```

**New Utility Method**: Add to `CommonUtility.java`
```java
public static void validatePdfStructure(MultipartFile file) throws IOException
```

#### TR-3: Error Constant
**File**: `src/main/java/com/bukuwarung/fsbnplservice/constants/ErrorConstants.java`

**Utilize Existing**: `INVALID_FILE_TYPE` (no new constants needed)

### Test Requirements

#### Unit Tests (Required)
**File**: `src/test/java/com/bukuwarung/fsbnplservice/service/impl/UploadFileServiceImplTest.java`

**Test Cases:**

**Bank Statement Upload Tests (type = "bank_statement")**:
1. **testUploadValidPdf_BankStatement()** - Verify successful PDF upload for bank statement
2. **testUploadCorruptedPdf_BankStatement()** - Verify rejection of corrupted PDF
3. **testUploadPasswordProtectedPdf_BankStatement()** - Verify rejection of password-protected PDF
4. **testUploadEncryptedPdf_BankStatement()** - Verify rejection of encrypted PDF
5. **testUploadJpegFile_BankStatement_Rejected()** - Verify JPEG rejected for bank statement ⚠️
6. **testUploadPngFile_BankStatement_Rejected()** - Verify PNG rejected for bank statement ⚠️

**Other Upload Type Tests (type ≠ "bank_statement")**:
7. **testUploadJpegFile_OtherType_Success()** - Verify JPEG upload works for other types ✅
8. **testUploadPngFile_OtherType_Success()** - Verify PNG upload works for other types ✅
9. **testUploadPdfFile_OtherType_Success()** - Verify PDF upload works for other types ✅

**PDF Validation Tests (All PDF uploads)**:
10. **testUploadPdfVersion14()** - Verify acceptance of PDF 1.4
11. **testUploadPdfVersion17()** - Verify acceptance of PDF 1.7
12. **testUploadPdfVersion20()** - Verify acceptance of PDF 2.0
13. **testUploadLargePdf()** - Verify handling of PDF at 5MB limit
14. **testUploadEmptyPdf()** - Verify rejection of empty/invalid PDF file

**Test Data Requirements:**
- Valid PDF samples (different versions: 1.4, 1.7, 2.0)
- Corrupted PDF samples (truncated, malformed structure)
- Empty file (0 bytes)
- Password-protected PDF sample
- Encrypted PDF sample
- Large PDF sample (close to 5MB)
- Valid JPEG sample
- Valid PNG sample

**Test Location**: `src/test/resources/test-files/`

## Implementation Plan

### Phase 1: Setup (Day 1)
1. Add Apache PDFBox dependency to `build.gradle`
2. Run `./gradlew build` to verify dependency resolution
3. Create test data directory and gather test PDF samples

### Phase 2: Implementation (Day 2-3)
1. Create `validatePdfStructure()` method in `CommonUtility.java`
2. Integrate validation into `UploadFileServiceImpl.uploadFile()`
3. Add logging for validation failures
4. Update error handling to use existing error constants

### Phase 3: Testing (Day 3-4)
1. Create comprehensive unit tests with all edge cases
2. Run tests and verify coverage
3. Manual testing with various PDF samples
4. Regression testing for JPEG/PNG uploads

### Phase 4: Code Review & Documentation (Day 4-5)
1. Self-review code against requirements
2. Run Spotless formatter and SonarQube analysis
3. Update inline documentation
4. Submit for code review

## References

1. **Jira Ticket**: [LND-4561](https://bukuwarung.atlassian.net/browse/LND-4561)
2. **Codebase Location**: `/Users/juvianto.chi/Desktop/code/bnpl`
3. **Controller**: `src/main/java/com/bukuwarung/fsbnplservice/controller/UploadFileController.java`
4. **Service Implementation**: `src/main/java/com/bukuwarung/fsbnplservice/service/impl/UploadFileServiceImpl.java`
5. **Utility Class**: `src/main/java/com/bukuwarung/fsbnplservice/util/CommonUtility.java`
6. **Constants**: `src/main/java/com/bukuwarung/fsbnplservice/constants/CommonConstants.java`
7. **Build Configuration**: `build.gradle`
8. **Application Properties**: `src/main/resources/application.properties`
9. **Apache PDFBox Documentation**: https://pdfbox.apache.org/
10. **Clarification Document**: `clarification.md` (same directory)

## Approval

**Prepared By**: Claude Code (AI Assistant)
**Date**: 2025-01-18
**Status**: Draft - Pending User Review

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-18 | Claude Code | Initial requirement document based on clarification session |
