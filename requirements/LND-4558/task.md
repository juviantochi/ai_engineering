scope repository: @bnpl; @fs-brick-service

You are task to help user brainstorm and clarify the request bellow.
Make sure everything is clear to be output into a requirement.
Every clarified things should update this task.md file.

background:
- We already have API Documentation created for veefin to inquiry data for specific borrower.
- The service that will be directly connected to veefin is FS-BRICK-SERVICE.
- The actual data file is stored in AWS S3, which S3 path is stored in BNPL database.


The task is:
- Create API Endpoint in FS-BRICK-SERVICE to handle the inquiry request from Veefin. (`fs/brick/borrower/v1/inquiry-data`)
- Get the data from BNPL in `merchant_enrichment_data` using ktpNumber passed from Veefin via fs-brick-service.
- Once fs-brick-service already get the S3 path from BNPL, it should get the file from AWS S3 then response to veefin as multiparts along with other part as presented in the API Document.

context:
- BNPL currently store the S3 path in `merchant_enrichment_data` connected to `merchant_data` via ` merchant_id`
- direct connection from fs-brick-service to BNPL is possible via the bnplPort need to add another service for data inquiry.
- authentication method still yet to be implemented in https://bukuwarung.atlassian.net/browse/LND-4557

what we are not doing:
- Implement circuit breaker or rate limiter in this task.

Make sure what will be out of scope if not sure. 
DO NOT ASUME anything. ALWAYS ASK FOR USER CLARIFICATION IF NEEDED.

References:
- API Documentation: `/Users/juvianto.chi/Desktop/code/documentation/apidocs/LND-4527.md`
- LND-4557: Authentication ticket (HMAC-SHA256 signature)

**OUTPUT**
Decent requirement and put it in requirement.md file under the same folder.
requirement should have these sections:
- Title
- Background
- Objective
- Scope (Whats in and out of scope)
- Requirements
- References

---

## ✅ TASK COMPLETED

**Status**: Requirements document generated successfully
**Date**: 2025-11-24

### Investigation Process Summary

**Investigation Completed:**
- ✅ FS-BRICK-SERVICE codebase structure analyzed
  - Java 11, Spring Boot 2.6.3, existing BNPL integration pattern
  - AWS S3 SDK v1.12.225 available
  - PartnerEventAudit pattern identified for idempotency
- ✅ BNPL database schema analyzed
  - merchant_data → merchant_enrichment_data relationship
  - KTP number field confirmed in merchant_data
  - S3 path storage in merchant_enrichment_data
- ✅ API documentation reviewed (LND-4527.md)
  - Multipart response format with 3 parts
  - HMAC-SHA256 authentication (separate ticket LND-4557)
  - EnrichmentDataPoint enum usage confirmed

**User Clarifications Received:**
- **BNPL Endpoint**: Create new endpoint at `/merchant/enrichment-data` (Option A)
- **S3 Retrieval**: FS-BRICK-SERVICE retrieves from S3 (Option A)
- **Missing Data**: Return 404 if no data, 400 if in progress/failed
- **Multiple Records**: Fetch latest by created_at DESC (Option A)
- **BNPL Path**: `/merchant/enrichment-data` (Option A)
- **dataId Source**: Use batch_id (Option A)
- **S3 Failure**: Return 500 Internal Server Error (Option A)
- **Idempotency**: Implement using PartnerEventAudit pattern (2 levels)
- **Logging**: Log full KTP (no masking - Option B)
- **Filename Format**: Generic `data.{ext}` (Option D)

**Key Decisions:**
- Do not implement new Resilience4j circuit breaker usage
- Use existing EnrichmentDataPoint enum
- Configure los-auth-token for BNPL authentication
- S3 download functionality already exists in BNPL (can reference)
- LND-4525 plan is irrelevant, use direct S3 retrieval approach

### Deliverables

**✅ Files Created:**
1. `clarification.md` - Investigation findings and clarification questions with answers
2. `requirement.md` - Comprehensive requirements specification

**✅ Requirements Structure:**
- Title: LND-4558: Veefin Borrower Data Inquiry API Implementation
- Background: Current implementation and business context
- Objective: Clear single-statement goal
- Scope: In-scope (6 items) and out-of-scope (5 items)
- Requirements:
  - 7 Functional Requirements (FR-1 to FR-7)
  - 5 Non-Functional Requirements (NFR-1 to NFR-5)
  - 5 Technical Requirements (TR-1 to TR-5) with DTOs and configurations
  - Test Requirements with specific test cases
- Implementation Plan: 5 phases over 7 days
- References: Jira ticket, API docs, authentication ticket

**✅ Repository Integration:**
Requirements successfully copied to impacted repositories:
- `/Users/juvianto.chi/Desktop/code/fs-brick-service/requirements/LND-4558/`
  - clarification.md
  - requirement.md
  - task.md
- `/Users/juvianto.chi/Desktop/code/bnpl/requirements/LND-4558/`
  - clarification.md
  - requirement.md
  - task.md

### Implementation Readiness

**Ready for Development:**
- ✅ All critical questions answered
- ✅ Technical specifications complete
- ✅ Database queries specified
- ✅ DTOs defined with validation
- ✅ Error handling scenarios documented
- ✅ Test cases identified
- ✅ Implementation plan with phases
- ✅ Configuration requirements specified

**Key Implementation Files Identified:**
- FS-BRICK: `VeefinBorrowerController.java`, `FsBnplAdapter.java`, `S3FileService.java`
- BNPL: `MerchantEnrichmentController.java`, `MerchantEnrichmentService.java`
- DTOs: Request/Response models for both services
- Configuration: application.yml updates for both services

**Next Steps for Implementation:**
1. Start with Phase 1: BNPL Internal Endpoint (Day 1-2)
2. Follow implementation plan sequentially
3. Reference existing patterns (PartnerController, FsBnplAdapter)
4. Use PartnerEventAudit for idempotency (2-level pattern)
5. Write unit tests alongside implementation
6. Conduct integration testing between services

### Quality Assurance

**Standards Met:**
- ✅ Investigation-first approach (comprehensive codebase analysis)
- ✅ Pattern-based defaults (existing controller/adapter patterns referenced)
- ✅ Concrete specificity (file paths, method signatures, exact queries)
- ✅ Implementation-ready (immediate development possible)
- ✅ All requirements traceable to business need
- ✅ Error scenarios comprehensively covered
- ✅ Test coverage specified

**Documentation Quality:**
- Detailed acceptance criteria for all requirements
- Specific file locations and code snippets
- Configuration with environment variables
- Rationale provided for technical decisions
- Clear priority levels (High/Medium)
- References to existing patterns and code

---

**Completion Confirmed**: All deliverables created and distributed to impacted repositories. Requirements are implementation-ready with comprehensive specifications, test cases, and phased implementation plan.
