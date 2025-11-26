# LND-4558 Implementation Brainstorm & Gap Analysis

**Date**: 2025-11-25
**Status**: Discovery Phase
**RFCs Analyzed**:
- Veefin Borrower Inquiry API (Main)
- BNPL RFC (LND-4558-bnpl.md)
- FS-BRICK-SERVICE RFC (LND-4558-fs-brick-service.md)

---

## Executive Summary

After analyzing the three RFCs and codebase knowledge, this document identifies critical gaps, open questions, and additional information needed before implementation can begin.

**Overall RFC Quality**: ⭐⭐⭐⭐ (4/5 - Very comprehensive with clear specifications)

**Key Strengths**:
- ✅ Clear architecture diagrams and data flow
- ✅ Comprehensive error handling scenarios
- ✅ Detailed PEA (PartnerEventAudit) 4-step pattern
- ✅ Performance targets defined
- ✅ Testing strategy outlined

**Critical Gaps Identified**: 7 categories requiring clarification

---

## 🚨 CRITICAL QUESTIONS (Must Resolve Before Implementation)

### 1. Database Schema & Existing Data

**Q1.1**: What is the actual current state of the database schema?
- Does `merchant_data` table exist with the exact structure specified? 
> yes
- Does `merchant_enrichment_data` table exist? 
> yes
- Are there existing indexes on `ktp_number`, `type`, `created_at`? 
> no
- **Action**: Need to run schema inspection queries or review migration scripts

**Q1.2**: What existing data exists in these tables?
- How many records currently in `merchant_enrichment_data`? 
> not applicable, new table not yet on production.
- What are the actual `s3_path` formats used? (RFC says "no assumed structure" but examples needed) 
> example string "attachments/ENRICHMENTS/PEFINDO/11268821011_PEFINDO.json"
- What are the distribution of `status` values (COMPLETE, IN_PROGRESS, FAILED)?
- **Action**: Need sample data queries from production/staging

**Q1.3**: Sample S3 path validation
- RFC states: "s3_path is used exactly as returned by BNPL, no assumptions about structure"
- Can you provide 5-10 real examples of `s3_path` values from the database?
- Are there any path patterns we should be aware of? (e.g., date-based folders, region prefixes)
- **Action**: Query `SELECT DISTINCT s3_path FROM merchant_enrichment_data LIMIT 20`
> For S3 path please expect strings only and assume you can just use it by attaching to bucket name.

---

### 2. S3 Bucket Configuration & Access

**Q2.1**: S3 bucket permissions verification
- Current IAM role for fs-brick-service - does it have GetObject permissions on `${BNPL_S3_BUCKET_NAME}`?
- Is there a specific IAM policy document we need to reference?
- Are there any bucket policies restricting access?
- **Action**: Need AWS IAM policy review

**Q2.2**: S3 bucket naming across environments
- What is the actual bucket name in dev/staging/production?
- RFC shows `janus-buku-dev` as default - is this correct for all environments?
- **Action**: Confirm environment-specific bucket names

**Q2.3**: S3 object metadata requirements
- Do we need to validate ContentType when downloading?
- Are there any S3 object tags we should check?
- **Action**: Review S3 object metadata standards

> this is exisitng connection from fs-brick-service to S3. assume everything is working just fine.
---

### 3. BNPL Service Integration

**Q3.1**: BNPL endpoint availability
- Does `/merchant/enrichment-data/inquiry` endpoint already exist in BNPL or is this also being created as part of LND-4558? it is part of LND-4558
- If not created yet, what is the coordination plan between bnpl and fs-brick-service deployments? 
> need to implement LND-4558 for BNPL first, then fs-brick continue to integrate with it.
- **Action**: Verify BNPL service implementation status

**Q3.2**: los-auth-token value confirmation
- RFC mentions both `BNPL_MERCHANT_TOKEN` and `LOS_AUTH_TOKEN` must contain same value
- What is the current value in dev/staging? (or reference to secrets manager path)
- Is this token shared with other services or dedicated to fs-brick-service?
- **Action**: Secrets manager path confirmation
> there's already existing connection from fs-brick-service to BNPL. 
so use the same pattern in fs-brick-service. but for fs-bnpl need to adjust the "shouldNotFilter" part for other filter than fsbrick filter token.

**Q3.3**: BNPL service SLA and timeouts
- What is the expected p95 latency for BNPL `/merchant/enrichment-data/inquiry`?
- What timeout should fs-brick-service configure for BNPL calls? (RFC doesn't specify) 
> use existing no need to configure one.
- Does BNPL have retry logic or should fs-brick-service implement retries? 
> no need any retry mechanism.
- **Action**: Define timeout and retry configuration

---

### 4. EnrichmentDataPoint Enum & Type Mapping

**Q4.1**: Enum definition location and values
- Where should `EnrichmentDataPoint` enum be defined? (RFC mentions it but doesn't show implementation) 
> can find in existing code.
- Values: `PEFINDO`, `FDC`, `BANK_STATEMENT` - are these exact strings in database `type` column? 
> yes
- Should enum support `.toString()` mapping to database values? 
> not relevant
- **Action**: Confirm enum values match database exactly

**Q4.2**: Type validation
- Are there any other `type` values in the database beyond these 3? no
- Should the API reject unknown types or let BNPL handle validation? 
> fs-brick should reject. thats why use enum instead of string. if request lies outside enum should error.
- **Action**: Query `SELECT DISTINCT type FROM merchant_enrichment_data`

**Q4.3**: File format validation
- RFC states PEFINDO/FDC = JSON, BANK_STATEMENT = XLSX
- Should fs-brick-service validate file format before streaming? 
- Or trust S3 ContentType metadata?
> assume S3 content is always valid if any.
- **Action**: Define validation strategy

---

### 5. FeatureFlagConfiguration Implementation Details

**Q5.1**: Feature flag JSON structure
- Current `FEATURE_FLAG_CONFIG` environment variable - what does it look like today? 
> read codebase
- RFC shows example with `gt` - is this already deployed?
> yes
- **Action**: Get current production feature flag JSON

**Q5.2**: Feature flag update process
- How do we update feature flags in production? (config update, restart, or dynamic?) 
> config update will trigger restart
- Is there a UI for feature flag management or only environment variable updates? 
> no
- **Action**: Document feature flag update procedure

**Q5.3**: Feature flag testing
- How to test gradual rollout locally? 
> using development environment
- Is there a staging environment with separate feature flag config? 
> yes
- **Action**: Define testing approach for feature flags

---

### 6. EventAuditUtil Pattern Validation

**Q6.1**: Existing EventAuditUtil implementation review
- RFC references `GetFdcDataCommand.java` lines 30-72 as pattern reference
- Can we review this file to ensure we understand the pattern correctly? 
- **Action**: Read `GetFdcDataCommand.java` to validate PEA pattern

**Q6.2**: EventAuditUtil method signatures
- What are the exact method signatures for:
  - `startServiceEventAudit(requestId, partnerName, serviceName, payload)`
  - `updatePartnerLevelEventAudit(serviceName, partnerName, requestId, status, request, response)`
  - `successServiceEventAudit(auditId, metadata)`
  - `failedServiceEventAudit(auditId, errorMessage)`
- **Action**: Review EventAuditUtil class

**Q6.3**: PartnerEventAuditEntity structure
- What fields exist in `PartnerEventAuditEntity`?
- What is the data type of `messageDetail` and `partnerMessageDetail`? (TEXT, JSON, JSONB?)
- How large can these fields be? (implications for storing S3 metadata)
- **Action**: Review entity definition

> yes, can also look at Pefindo commands. Look for existing implementation for this question.

---

### 7. Multipart Response Implementation

**Q7.1**: Multipart boundary handling
- RFC shows `------WebKitFormBoundary` but doesn't specify implementation
- Should we use Spring's `MultipartBodyBuilder` or manual boundary generation? 
- **Action**: Define multipart response building approach
> can use spring's MultipartBodyBuilder.

**Q7.2**: Filename generation
- RFC shows `filename="data.json"` - should this be dynamic based on type? 
> no need. data.json would be enough
- E.g., `pefindo-{batchId}.json`, `bankstatement-{batchId}.xlsx`?
- **Action**: Define filename generation strategy

**Q7.3**: Content-Type for different file types
- RFC shows `application/octet-stream` for all files
- Should we use specific types: `application/json`, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`?
- **Action**: Define ContentType mapping
> can use application/octet-stream for all types if no impact at all.

---

## 🔍 IMPLEMENTATION DETAILS NEEDED

### 8. Testing Strategy Gaps

**Q8.1**: Test data setup
- Do we have test S3 bucket with sample files? 
> yes
- Do we have test KTP numbers with known enrichment data? 
> yes
- **Action**: Prepare test data and S3 test files

**Q8.2**: Integration test environment
- Can we use LocalStack for S3 in integration tests? 
> no
- Can we use TestContainers for PostgreSQL? 
> no
- **Action**: Set up integration test infrastructure

**Q8.3**: Performance testing
- RFC targets < 5s p95 for 10MB files
- Do we have performance testing tools set up? (JMeter, Gatling?) 
- **Action**: Define performance testing approach
> no. no need to test performance.

---

### 9. Monitoring & Observability

**Q9.1**: Metrics collection
- Are we using Prometheus/Micrometer for metrics?
- Should we add custom metrics for:
  - Request count by status code
  - File download size distribution
  - Idempotency hit rate
- **Action**: Define metrics implementation 
>no custom metrics needed.

**Q9.2**: Logging infrastructure
- Current logging framework? (Logback, Log4j2?)
- Structured logging format? (JSON?)
- Log aggregation tool? (Elasticsearch, CloudWatch?)
- **Action**: Confirm logging standards

> slf4j with logback. no specific pattern in logging, can follow pefindo or fdc.

**Q9.3**: Alerting
- What monitoring system is in use? (Grafana, Datadog, New Relic?)
- Should we set up alerts for:
  - Error rate > 1%
  - p95 latency > 5s
  - S3 404 rate (indicates data inconsistency)
- **Action**: Define alerting strategy
> datadog

---

### 10. Security & Compliance

**Q10.1**: PII handling compliance
- RFC states "Full KTP logging permitted"
- Is this confirmed with legal/compliance team?
- Any GDPR/data retention requirements for audit logs?
- **Action**: Confirm compliance approval
> no need to log full KTP. but make sure partner event audit contains all the payload including KTP

**Q10.2**: Secrets management
- Using AWS Secrets Manager or environment variables?
- Rotation policy for `los-auth-token`?
> no rotation policy.
- **Action**: Document secrets management approach

**Q10.3**: Authentication placeholder
- RFC defers auth to LND-4557
- What is the timeline for LND-4557?
- Should we implement a basic auth placeholder or truly no auth?
- **Action**: Clarify auth implementation timeline
> no auth needed for this ticket.
---

### 11. Deployment & DevOps

**Q11.1**: Deployment coordination
- Since this involves 2 services (bnpl + fs-brick-service), what is deployment order?
- Should we deploy bnpl first, then fs-brick-service?
- **Action**: Define deployment sequence
> develop bnpl first then fs-brick. but both will go live and qa tested at the same time.

**Q11.2**: Database migration
- Does bnpl need schema migrations for the new endpoint?
> no schema migrations
- Are indexes already present or do we need migration scripts?
> needed script. no indexes yet.
- **Action**: Review migration scripts

**Q11.3**: Environment variable configuration
- Full list of environment variables needed:
  - `FEATURE_FLAG_CONFIG` (enhanced with veefin-borrower-inquiry)
  - `BNPL_MERCHANT_BASE_URL`
  - `BNPL_MERCHANT_TOKEN`
  - `BNPL_S3_BUCKET_NAME`
  - `AWS_ACCESS_KEY_ID`, `AWS_SECRET_KEY`, `AWS_REGION`
  - `LOS_AUTH_TOKEN` (bnpl side)
- Do these exist in all environments or need to be created?
- **Action**: Environment variable audit
> existing. just need some adjustment  in FEATURE_FLAG_CONFIG
---

### 12. Error Handling Edge Cases

**Q12.1**: Concurrent duplicate requests
- What happens if 2 requests with same `veefin-request-id` arrive simultaneously?
- Does `startServiceEventAudit()` handle race conditions?
- **Action**: Review concurrency handling in EventAuditUtil
> No, just assume single request only. no need to handle concurrency.

**Q12.2**: S3 partial download failure
- If S3 stream fails mid-download (network interruption), what happens?
- Do we log partial state? Retry?
- **Action**: Define failure recovery strategy
> just throw exception and return 500 to client.

**Q12.3**: BNPL returns 200 but invalid JSON
- What if BNPL response is malformed?
- Should we catch `JsonProcessingException` and treat as 500?
- **Action**: Define JSON parsing error handling

> yes, treat as 500 error.

---

### 13. Code Organization Questions

**Q13.1**: Package structure
- Where should new classes be placed?
  - `com.bukuwarung.fsbrickservice.controller.veefin.BorrowerDataInquiryController`?
  - `com.bukuwarung.fsbrickservice.service.veefin.BorrowerDataServiceImpl`?
- **Action**: Confirm package naming conventions
> no specific veefin package/class name. make the implementation general
> can use 
> - `com.bukuwarung.fsbrickservice.controller.BorrowerDataInquiryController`?
> - `com.bukuwarung.fsbrickservice.service.BorrowerDataServiceImpl`?

**Q13.2**: DTO location
- RFC mentions DTOs like `BorrowerInquiryRequest`, `EnrichmentDataResponse`
- Should these be in `model.veefin` package?
- **Action**: Confirm DTO package structure
> no, make it general not veefin specific.

**Q13.3**: Exception hierarchy
- Custom exceptions mentioned:
  - `EnrichmentDataNotFoundException`
  - `DataNotReadyException`
  - `S3FileNotFoundException`
  - `BnplServiceUnavailableException`
- Should these extend common base exception?
- **Action**: Define exception hierarchy

> yes
---

### 14. Backward Compatibility & Migration

**Q14.1**: Impact on existing functionality
- Does fs-brick-service currently use S3FileService for other features?
- Will our changes affect existing S3 operations?
- **Action**: Review S3FileService usage across codebase
> no impact at all. this is new endpoint only.

**Q14.2**: BNPL backward compatibility
- Is the new BNPL endpoint backward compatible?
- Do any existing services call similar endpoints?
- **Action**: Verify no breaking changes to BNPL
> yes, no breaking changes. no existing feature will call this endpoints.
---

### 15. Performance & Scalability

**Q15.1**: Connection pool sizing
- Current RestTemplate connection pool configuration for BNPL calls?
- Should we use separate pool for this endpoint?
- **Action**: Review HTTP client configuration
> use existing.

**Q15.2**: S3 transfer acceleration
- RFC mentions S3 Transfer Acceleration as "future enhancement"
- Is this needed for initial release given 10MB target?
- **Action**: Decide on S3 optimization needs
> no need. 

**Q15.3**: Rate limiting strategy
- RFC marks rate limiting as out of scope
- Do we need basic rate limiting to protect BNPL/S3?
- **Action**: Assess rate limiting necessity
> no. no rate limiting needed.

---

## 📋 INFORMATION GATHERING CHECKLIST

### Before Implementation Begins

- [ ] **Database Schema Validation**
  - [ ] Run schema inspection on `merchant_data` and `merchant_enrichment_data`
  - [ ] Verify indexes exist
  - [ ] Get sample `s3_path` values (20+ examples)
  - [ ] Check `type` column distinct values

- [ ] **Codebase Pattern Review**
  - [ ] Read `GetFdcDataCommand.java` (PEA pattern reference)
  - [ ] Review `EventAuditUtil.java` class
  - [ ] Review `PartnerEventAuditEntity` structure
  - [ ] Review `S3FileService` implementation
  - [ ] Review `FeatureFlagConfiguration` implementation

- [ ] **Configuration & Secrets**
  - [ ] Confirm S3 bucket names per environment
  - [ ] Verify IAM permissions for S3 access
  - [ ] Get los-auth-token secrets manager path
  - [ ] Get current `FEATURE_FLAG_CONFIG` value

- [ ] **BNPL Service Coordination**
  - [ ] Confirm BNPL implementation status
  - [ ] Align on deployment timeline
  - [ ] Test BNPL endpoint in staging (if available)

- [ ] **Testing Infrastructure**
  - [ ] Set up LocalStack for S3 testing
  - [ ] Prepare test data in staging database
  - [ ] Upload test files to staging S3
  - [ ] Configure integration test environment

- [ ] **Compliance & Security**
  - [ ] Get compliance approval for KTP logging
  - [ ] Review secrets rotation policy
  - [ ] Confirm LND-4557 timeline

- [ ] **Monitoring Setup**
  - [ ] Define metrics to collect
  - [ ] Set up dashboards
  - [ ] Configure alerts

---

## 🎯 RECOMMENDED NEXT STEPS

### Phase 0: Pre-Implementation (Days 1-2)

1. **Database Discovery** (Priority: CRITICAL)
   - Run schema queries
   - Get sample data
   - Validate assumptions

2. **Codebase Pattern Study** (Priority: CRITICAL)
   - Read referenced files
   - Understand EventAuditUtil
   - Understand S3FileService

3. **Configuration Audit** (Priority: HIGH)
   - List all env vars across environments
   - Document secrets manager paths
   - Review feature flag JSON

4. **BNPL Coordination** (Priority: HIGH)
   - Sync with BNPL team on timeline
   - Agree on deployment sequence
   - Set up staging test environment

### Phase 1: Implementation Planning (Days 3-4)

1. **Technical Design Document**
   - Package structure
   - Class diagrams
   - Sequence diagrams
   - Exception hierarchy

2. **Test Plan**
   - Unit test specifications
   - Integration test specifications
   - Performance test approach
   - Test data requirements

3. **Deployment Plan**
   - Deployment sequence
   - Rollback procedures
   - Smoke test checklist

### Phase 2: Development (Days 5-10)

1. **BNPL Implementation** (if not started)
2. **FS-BRICK-SERVICE Implementation**
3. **Testing**
4. **Code Review**

---

## 🤔 OPEN DISCUSSION POINTS

### Architecture Decisions

**AD1**: Should we add circuit breaker pattern?
- RFC marks as out of scope
- But BNPL dependency failure could impact service
- **Recommendation**: Consider Resilience4j circuit breaker as enhancement
> no circuit breaker needed.

**AD2**: Should we cache BNPL responses?
- RFC doesn't mention caching
- Idempotency already handles duplicates
- **Recommendation**: No caching for initial release, add if needed
> do not cache.

**AD3**: Should we validate file content?
- RFC only validates metadata
- Should we check JSON validity for PEFINDO/FDC?
- **Recommendation**: Trust S3 content, focus on streaming
> simple validation checking file availability only.

### Implementation Choices

**IC1**: Multipart response library
- Spring `MultipartBodyBuilder`?
- Apache Commons FileUpload?
- Manual boundary generation?
- **Recommendation**: Spring MultipartBodyBuilder for consistency
> Spring multipartbodybuilder is fine. no need to use other library.

**IC2**: Exception handling strategy
- Global `@ControllerAdvice`?
- Per-controller exception handlers?
- **Recommendation**: Global exception handler for consistency
> global exception handler is fine.

**IC3**: Logging approach
- MDC (Mapped Diagnostic Context) for requestId?
- Structured JSON logs?
- **Recommendation**: MDC for correlation + structured logs
> go with recommendation

---

## 📊 RISK ASSESSMENT

### High Risk Items Requiring Immediate Attention

1. **Database Schema Unknown** (Risk: 🔴 HIGH)
   - Impact: Cannot implement without knowing actual schema
   - Mitigation: Query database immediately

2. **S3 Path Format Unknown** (Risk: 🔴 HIGH)
   - Impact: S3 download might fail if path assumptions wrong
   - Mitigation: Get real path examples

3. **BNPL Endpoint Status Unknown** (Risk: 🔴 HIGH)
   - Impact: Cannot test fs-brick-service without BNPL
   - Mitigation: Coordinate with BNPL team

### Medium Risk Items

4. **EventAuditUtil Pattern Unclear** (Risk: 🟡 MEDIUM)
   - Impact: Idempotency might not work as expected
   - Mitigation: Review reference implementation

5. **Feature Flag Testing** (Risk: 🟡 MEDIUM)
   - Impact: Gradual rollout might fail
   - Mitigation: Test in staging first

### Low Risk Items

6. **Monitoring Setup** (Risk: 🟢 LOW)
   - Impact: Observability gaps
   - Mitigation: Add incrementally

---

## 📝 SUMMARY OF QUESTIONS FOR PRODUCT/ENGINEERING TEAM

### Critical (Must Answer Before Coding)

1. What is the actual database schema for `merchant_data` and `merchant_enrichment_data`?
2. Can you provide 20+ sample `s3_path` values from the database?
3. What is the status of BNPL endpoint implementation?
4. What are the actual S3 bucket names per environment?
5. What is the current `FEATURE_FLAG_CONFIG` JSON structure?
6. Has compliance approved full KTP logging?

### High Priority (Should Answer During Early Implementation)

7. What is the exact signature of EventAuditUtil methods?
8. What timeout should we configure for BNPL calls?
9. Should we validate file ContentType before streaming?
10. What is the deployment sequence for bnpl + fs-brick-service?

### Medium Priority (Can Resolve During Development)

11. Should filenames in multipart response be dynamic?
12. Do we need custom metrics beyond basic request counters?
13. Should we implement circuit breaker pattern?

### Low Priority (Nice to Have)

14. Should we add request/response logging for debugging?
15. Do we need performance testing before initial release?
16. Should we document API in Swagger/OpenAPI?

---

## ✅ CONCLUSION

The RFCs are **comprehensive and well-structured**, but several **critical information gaps** must be filled before implementation can safely proceed.

**Estimated Time to Resolve Gaps**: 2-3 days

**Recommendation**:
1. Start with database discovery and codebase pattern review
2. Coordinate with BNPL team on implementation timeline
3. Set up test infrastructure in parallel
4. Begin implementation only after resolving CRITICAL questions

**Next Immediate Action**: Run database schema queries and review `GetFdcDataCommand.java` pattern.
