# Clarification Questions for LND-4561: PDF Validation

## 🔍 **Current Endpoint Behavior**
1. **Where is the `/upload/file` endpoint currently implemented?** UploadFileController.java (Answered)
2. **What file types are currently accepted?** JPEG, PNG, PDF - validated via Apache Tika MIME type detection + file extension validation (Answered - see CommonUtility.java:283-320)
3. **What happens to uploaded files after validation?** Stored in AWS S3 bucket (configured via `bnpl.merchant.s3.bucket.name`) (Answered - see UploadFileServiceImpl.java:38)
4. **Is there a current file size limit?** Yes, 5MB default (configured in application.properties:45-46 as `spring.servlet.multipart.max-file-size`) (Answered) 

## 📄 **PDF Validation Requirements**
5. **What constitutes a "corrupted PDF" for your use case?**
   - Should we check PDF structure validity? if possible.
   - Should we verify PDF can be parsed? no. this will be scanned pdf.
   - Should we attempt to extract content/metadata? no.

6. **What should the system do when validation fails?**
   - Return specific error codes? 
   - Log the failure?
   - Any retry mechanism? NO

## 📏 **Boundaries & Edge Cases**
7. **What is "very large PDF" for your context?**
   - What's the maximum file size we should accept? (e.g., 10MB, 50MB, 100MB?)
   - Should we enforce minimum size to prevent empty files? 

8. **Should we validate PDF version compatibility?** (PDF 1.4, 1.7, 2.0, etc.)

9. **Should we check for:**
   - Password-protected PDFs? Yes
   - PDFs with encryption? Yes
   - Multi-page vs single-page restrictions? NO
   - Specific content requirements? NO

## 🔧 **Implementation Details**
10. **Which PDF validation library should we use?** Apache PDFBox recommended for Java (need confirmation - existing project uses Apache Tika 3.0.0, Apache POI 5.2.0)
11. **What's your backend tech stack?** Java 11, Spring Boot 2.6.3 (Answered - see build.gradle)
12. **Performance requirements?**
    - Should validation be synchronous or asynchronous? sync (Answered)
    - Acceptable latency for validation? under 2s. read ticket description and comments. (Need Jira access)

## 🎯 **Out of Scope Clarification**
13. **Should we handle:**
    - PDF content validation (e.g., no malicious scripts)? No
    - OCR or text extraction? No
    - Image-only PDF validation? No
    - PDF conversion or optimization? No

## 🧪 **Testing Scope**
14. **For unit tests, should we also cover:**
    - Valid PDF upload success cases? yes
    - Different PDF versions? yes
    - Concurrent upload scenarios? no
    - Rate limiting? no

---

## 📊 **Investigation Summary**

### ✅ **Findings from Current Implementation:**
- **Location**: UploadFileController.java → UploadFileService.java → UploadFileServiceImpl.java
- **Current Tech Stack**: Java 11, Spring Boot 2.6.3, AWS S3, Apache Tika 3.0.0
- **Current Validation**: MIME type detection (Apache Tika) + file extension check (CommonUtility.java)
- **Accepted File Types**: JPEG, PNG, PDF (CommonConstants.java:79-85)
- **File Size Limit**: 5MB default (application.properties:45-46)
- **Storage**: AWS S3 (bnpl.merchant.s3.bucket.name)
- **Error Handling**: Throws ResponseStatusException with HttpStatus.BAD_REQUEST and ErrorConstants.INVALID_FILE_TYPE
- **Available Libraries**: Apache Tika 3.0.0, Apache POI 5.2.0 (could add Apache PDFBox for PDF validation)

### ✅ **Final Decisions:**

**Based on User Input & Existing Patterns:**
1. **Error Codes & Messages**: Use existing `INVALID_FILE_TYPE` error with `HttpStatus.BAD_REQUEST` (following current pattern)
2. **Logging**: Use existing `@Slf4j` pattern - log validation failures at WARN level
3. **Maximum PDF Size**: Keep existing 5MB limit (configured in `application.properties`)
4. **Minimum File Size**: No minimum enforced (following current implementation)
5. **PDF Version Support**: Accept all valid PDF versions (no restrictions)
6. **PDF Library**: Apache PDFBox (suitable for Java, compatible with Java 11 Spring Boot 2.6.3)
7. **Performance**: Synchronous validation, target < 2s latency (from Jira ticket LND-4561)

---

## 📝 **Ready for Requirement Document**
All clarifications complete. Proceeding to create requirement.md.

---
---

# 🔧 IMPLEMENTATION PLANNING PHASE QUESTIONS

**Date**: 2025-11-19
**Phase**: Creating implementation plan from RFC
**Source**: Analysis of RFC document `/rfc/LND-4561-pdf-validation.md`

---

## ❓ Clarifying Questions Before Implementation

### 1. **PDF Validation Scope for Non-Bank-Statement Uploads**
The RFC mentions two options for PDF validation:
- **Option A**: Validate ALL PDF uploads (regardless of type parameter)
- **Option B**: Validate ONLY when `type = "bank_statement"`

**Question**: Should we validate PDF structure for ALL PDF uploads, or ONLY for bank statement uploads?

**Recommendation**: I suggest **Option A** (validate all PDFs) for consistency and to catch corrupted PDFs early regardless of upload type.

**Answer**: _______________

---

### 2. **Page Count Limit for Security**
The RFC mentions an optional safeguard against PDF bombs:

```java
if (document.getNumberOfPages() > 1000) {
  throw new ResponseStatusException(...);
}
```

**Question**: Should we implement a maximum page count limit? If yes, what limit?

**Recommendation**: Start without a hard limit, monitor in production, and add if needed based on actual data.

**Answer**: _______________

---

### 3. **Test Data Generation Strategy**
For creating test PDFs (encrypted, password-protected, corrupted), the RFC proposes:
- **Option A**: Generate programmatically using PDFBox in test setup
- **Option B**: Use pre-made PDF samples from external sources

**Question**: Should we generate test PDFs programmatically or use pre-made samples?

**Recommendation**: **Option A** (programmatic generation) for reproducibility and version control, but we can also include a few real-world PDF samples for regression testing.

**Answer**: _______________

---

### 4. **Bank Statement Type Value**
The RFC uses `"bank_statement"` (lowercase with underscore) for the type parameter.

**Question**: Is this the exact value used in production? Should we use:
- `"bank_statement"` (lowercase with underscore)
- `"BANK_STATEMENT"` (uppercase with underscore)
- `"bankStatement"` (camelCase)
- Or some other value?

**Current Implementation**: The code uses `.equalsIgnoreCase()`, so it's case-insensitive, but I want to confirm the canonical value for logging and documentation.

**Answer**: _______________

---

### 5. **Logging Context for PDF-Only Rejection**
When rejecting non-PDF files for bank statements, should the log message differentiate between:
- **General rejection**: "Non-PDF file upload rejected for bank statement: fileName={}, mimeType={}"
- **Type-specific**: "JPEG file upload rejected for bank statement (PDF-only required): fileName={}"

**Question**: Should we use generic or type-specific log messages?

**Recommendation**: Generic message is simpler and covers all non-PDF types (JPEG, PNG, etc.).

**Answer**: _______________

---

## 📋 Next Steps

Once these questions are answered, I will:
1. Create comprehensive implementation plan in `plans/LND-4561-pdf-validation.md`
2. Include specific file paths, code snippets, and test cases
3. Provide step-by-step implementation checklist
4. Document all external dependencies and integration points

---

**Status**: ⏳ Waiting for answers before creating implementation plan