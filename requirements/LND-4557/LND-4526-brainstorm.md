# LND-4526 Implementation - Discovery Questions

**RFC**: Veefin Signature Authentication Filter (fs-brick-service)
**Approach**: A (Base Class + Multiple Filter Instances)
**Status**: Accepted (13-Nov-2024)
**Session Date**: 2025-11-14

---

## 🎯 Implementation Scope

### RFC Summary
Implementing partner signature authentication for `fs-brick-service` using **Approach A**:
- Base class: `AbstractPartnerSignatureFilter` (template method pattern)
- Concrete filter: `VeefinSignatureAuthenticationFilter` (@Order(3))
- Security guard: `PartnerCatchAllFilter` (defense-in-depth)
- Configuration: YAML-based per-partner settings
- Security: HMAC-SHA256 signature verification with timestamp validation

**Key Decision Rationale**: Limited blast radius (1 partner, growing to 2-3) makes strong isolation and defense-in-depth security more valuable than configuration-driven scalability.

---

## ❓ Discovery Questions

### 📍 Repository & Environment Context

**1. Where is the `fs-brick-service` repository located?**
- [x] Do you have it cloned locally at `/Users/juvianto.chi/Desktop/code/`?
- [x] What's the exact path to this Spring Boot service? `/Users/juvianto.chi/Desktop/code/fs-brick-service`
- [x] Is it a monorepo or standalone service? mono repo

**2. What's the current state of the codebase?**
- [x] Is this a greenfield implementation or are there existing authentication filters? there's already some filter implemented. no need to touch them and I want them to still be working fine.
- [x] What Spring Boot version is being used? 2.6.3
- [x] What's the current package structure (e.g., `com.bukuwarung.fsbrickservice.*`)?
- [x] Are there existing security configurations I should be aware of? there are some filters with @Order annotation and some with no specified order.

**3. What's the current authentication/authorization setup?**
- [x] Existing filter chain configuration? Yes, there are some filters with @Order annotation and some with no specified order.
- [x] Other authentication mechanisms in place (JWT, OAuth, etc.)? nothing significant. mostly static token
- [x] Security configuration class location? `src/main/java/com/bukuwarung/fsbrickservice/filter`

---

### 🎯 Implementation Scope

**4. What phase should I focus on?**
- [ ] **Phase 1 only**: Just the Veefin filter implementation (initial partner)
- [ ] **Phase 2**: Full framework (base class + Veefin + catch-all filter)
- [ ] **Phase 3**: Complete with tests and documentation
- [x] **All phases**: End-to-end implementation with deployment readiness

**5. Implementation priorities?**
- [ ] Speed (working prototype quickly) vs. Completeness (production-ready with all tests)
- [ ] Should I prioritize core functionality first, then tests?
- [x] Or implement with TDD approach (tests alongside code)?

---

### 🔧 Technical Prerequisites

**6. Do you have the Veefin integration details?**
- [x] Header specifications confirmed:
  - `veefin-x-partner-id`
  - `veefin-x-request-id`
  - `veefin-x-timestamp`
  - `veefin-x-signature`
- [x] Signature payload format: `partnerId|requestId|timestamp`?
- [x] HMAC algorithm: HMAC-SHA256 confirmed?
- [x] Timestamp format: Unix epoch (seconds)?
- [x] Protected paths: `/fs/brick/service/veefin/**` confirmed? based on the configuration yaml. for the catch-all filter, it should protect all paths configured for all partners.

**7. Configuration management approach?**
- [ ] Spring Cloud Config, environment variables, or local YAML?
- [x] Where should `VEEFIN_PARTNER_SECRET` be stored?
  secret manager retrieved in env variable yaml using ${VEEFIN_PARTNER_SECRET}
  - Secrets manager (AWS Secrets Manager, Vault)?
  - Environment variable?
  - Local properties for dev environment?
- [x] Config hot-reload requirements? reload once in startup.

**8. Testing infrastructure?**
- [x] Existing test patterns in the codebase I should follow? **YES - See analysis below**
- [x] JUnit version (JUnit 5 Jupiter or JUnit 4)? **JUnit 5 Jupiter** (build.gradle: junit_jupiter_version = '5.8.1')
- [x] Mocking framework: Mockito? MockMvc for integration tests? **Mockito 3.12.4** for unit tests
- [x] Test coverage requirements (RFC mentions >85%)? **Target >85% based on RFC**
- [x] Existing test utilities or base test classes? **YES - See detailed analysis below**

### 📊 Test Infrastructure Analysis

**Testing Framework Stack:**
```gradle
JUnit 5 Jupiter: 5.8.1
Mockito Core: 3.12.4
Mockito Inline: 5.2.0
Spring Boot Test: 2.6.3 (includes MockMvc, WebTestClient)
H2 Database: For test database
```

**Test Patterns Found:**

1. **Test Class Naming**: `{ClassName}Test.java` (e.g., `CvvEncryptionUtilTest`)

2. **Test Structure**: Standard AAA pattern
   ```java
   // Given - Setup and arrange
   // When - Execute action
   // Then - Verify results
   ```

3. **JUnit 5 Features Used**:
   - `@BeforeEach` for setup
   - `@Test` for individual test cases
   - `@ParameterizedTest` with various sources:
     - `@MethodSource` for complex parameters
     - `@CsvSource` for simple data sets
     - `@ValueSource` for single parameters
   - Static imports for assertions: `assertEquals`, `assertNotNull`, `assertThrows`, etc.

4. **Test Utilities**:
   - **TestConstants.java**: Centralized test data constants
     - Location: `src/test/java/com/bukuwarung/fsbrickservice/constants/TestConstants.java`
     - Pattern: `public static final` constants
     - Categories: Feature flags, user data, business data, etc.

5. **Existing Filters** (Reference for Implementation):
   - `FsBrickTokenFilter` (@Order(1)) - uses `OncePerRequestFilter`
   - `FirebaseTokenFilter` - authentication filter pattern
   - `LosBrickTokenFilter` - path-based filtering
   - `PanaceaTokenFilter` - token validation

6. **Filter Testing Pattern** (Expected):
   - Currently: No filter tests found (filters excluded from coverage in sonarqube config)
   - Recommendation: Create filter tests despite coverage exclusion for security validation

7. **Code Quality Tools**:
   - **Spotless**: Google Java Format enforced
   - **JaCoCo**: Test coverage reports
   - **SonarQube**: Code quality analysis
   - Coverage exclusions: `**/filter/**` (filters excluded but we should test anyway)

**Recommended Test Structure for LND-4526:**

```
src/test/java/com/bukuwarung/fsbrickservice/
├── filter/
│   ├── AbstractPartnerSignatureFilterTest.java
│   ├── VeefinSignatureAuthenticationFilterTest.java
│   └── PartnerCatchAllFilterTest.java
├── config/
│   └── PartnerSignatureConfigTest.java
├── util/
│   └── SignatureUtilsTest.java  (similar to CvvEncryptionUtilTest)
└── constants/
    └── PartnerTestConstants.java (extend TestConstants pattern)
```

**Test Coverage Strategy**:
- Unit tests: >85% for utility classes (SignatureUtils)
- Filter tests: Focus on critical paths (valid/invalid signatures, timestamp validation)
- Integration tests: End-to-end authentication flows (optional but recommended)
- Security tests: Malicious input handling, bypass attempts

**Mocking Strategy** (following existing patterns):
- Use Mockito for external dependencies
- No mocking for utility classes (direct instantiation)
- Spring test utilities for filter chain testing

---

### 🔐 Security & Compliance

**9. Are there existing security patterns to follow?**
- [x] Existing signature verification utilities in the codebase?
- [x] Crypto libraries already in use (BouncyCastle, javax.crypto)? yes javax.crypto already in use
- [x] Error response format standards (RFC shows JSON error format)?
- [x] Logging standards:
  - What to log (authentication attempts, failures)? authentication failures log with ERROR, for successful attempts log with DEBUG.
  - What to sanitize (secrets, PII)? no
  - Log format (structured JSON logs)? plain text logs. simple, concise and informative 

**10. Security review requirements?**
- [x] Who needs to review the security implementation? peers
- [x] Threat model validation needed? no
- [x] Penetration testing planned? no

---

### 📊 Success Criteria

**11. What does "done" look like for you?**
- [ ] Working code only?
- [x] Working code + unit tests?
- [ ] Working code + unit tests + integration tests?
- [ ] Full implementation with:
  - Code implementation
  - Unit tests (>85% coverage)
  - Integration tests
  - API documentation
  - Configuration guide
  - Operational runbook
  - Monitoring/metrics setup

**12. Deployment readiness?**
- [x] Should I include deployment configuration (application.yml, profiles)?
- [x] Feature flag support needed? no
- [x] Rollback plan implementation details? no details.

---

### 🚀 Delivery & Timeline

**13. Timeline expectations?**
- [ ] RFC estimates 20 business days (4 weeks) for full implementation
- [x] What's your target delivery date? done (code + unit tests) in 3 days.
- [x] Incremental delivery preferred or complete implementation? incremental delivery. always verify first only then continue.

**14. Integration testing environment?**
- [x] Do you have access to Veefin staging environment? no
- [x] Mock integration acceptable for initial implementation? yes
- [x] Local testing approach? dummy secrets setup using variables. 

---

## 💡 What I Can Deliver

Based on RFC Approach A, I can provide:

### ✅ Core Implementation
- [x] `AbstractPartnerSignatureFilter` base class
  - Template method pattern
  - Extensible validation logic
  - Request attribute management
- [x] `VeefinSignatureAuthenticationFilter` concrete implementation
  - @Order(3) configuration
  - Partner-specific header handling
- [x] `PartnerCatchAllFilter` security guard
  - @Order(LOWEST_PRECEDENCE - 10)
  - Dynamic pattern collection from config
  - Authentication validation safety net
- [x] `PartnerSignatureConfig` configuration binding
  - @ConfigurationProperties
  - Per-partner settings
- [x] `SignatureUtils` HMAC verification utilities
  - HMAC-SHA256 implementation
  - Signature payload construction

### ✅ Configuration
- [x] `application.yml` configuration template
- [ ] Environment-specific profiles (dev, staging, prod)
- [ ] Secret management integration

### ✅ Testing Strategy
- [x] Unit tests for each component
  - Base filter validation logic
  - Veefin filter customization
  - Catch-all filter protection
  - Signature utilities
- [ ] Integration tests
  - End-to-end authentication flows
  - Multi-partner scenarios (for future scalability)
  - Path protection validation
- [x] Security validation tests
  - Invalid signature rejection
  - Timestamp expiration
  - Missing headers handling

### ✅ Documentation
- [ ] API specifications (header requirements)
- [ ] Configuration guide
- [ ] Operational runbook
- [x] Code documentation (Javadoc)

---

## 🔍 Next Steps

**Once you provide answers to the questions above, I will:**

1. **Create detailed implementation plan** with specific file locations and code structure
2. **Generate implementation tasks** broken down by component
3. **Provide code templates** following existing codebase patterns
4. **Design test strategy** aligned with your testing infrastructure
5. **Create configuration templates** for all environments

**Priority**: Please answer questions 1-2 (repository location and current state) first so I can inspect the codebase and provide context-aware implementation guidance.

---

## 📝 Notes & Decisions

*This section will be updated as questions are answered and decisions are made*

### Answered Questions
<!-- Fill in as questions are answered -->

### Key Decisions Made
<!-- Track important implementation decisions -->

### Blockers & Dependencies
<!-- Track any blockers that need resolution -->

### Open Issues
<!-- Track questions that arise during implementation -->
