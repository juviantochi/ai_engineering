# Capability Provider Pattern - Complete Guide

**Repository**: `fs-brick-service`  
**Pattern**: Plugin Architecture for External Service Integrations  
**Status**: Production (Currently used by VIDA PoA)

---

## What is Capability Provider?

**Capability Provider** is an architectural pattern that standardizes how `fs-brick-service` integrates with external services (third-party APIs, partners, vendors). Think of it as a **plugin system** that makes it easy to add new external service integrations without changing core application code.

### Core Concept

Instead of writing custom integration code scattered throughout the codebase for each partner, we define a standard interface (`CapabilityProvider`) that all integrations must implement. This provides:
- **Consistent lifecycle management** (initialize → operate → shutdown)
- **Automatic health monitoring** (are external services reachable?)
- **Centralized registry** (lookup any provider by name)
- **Feature flag integration** (enable/disable per user/merchant)
- **Automatic audit logging** (track all operations)
- **Built-in resilience** (circuit breakers, retries)

---

## Key Components

### 1. **CapabilityProvider Interface**

The contract that all external service integrations must implement:

```java
public interface CapabilityProvider {
  String getProviderType();        // Unique ID: "VIDA_POA", "VEEFIN_CREDIT"
  String getProviderName();        // Human name: "VIDA Power of Attorney"
  
  void initialize();               // Setup: validate config, fetch tokens, etc.
  ProviderHealthStatus checkHealth(); // Verify: is the external service reachable?
  void shutdown();                 // Cleanup: release resources
  
  boolean isEnabled(String id);    // Feature flag: enabled for this merchant?
  Map<String, Object> getMetadata(); // Config info for monitoring
}
```

### 2. **AbstractCapabilityProvider (Base Class)**

Abstract class that provides common functionality:

```java
@Component
public abstract class AbstractCapabilityProvider implements CapabilityProvider {
  // Shared infrastructure (auto-injected by Spring)
  @Autowired protected RestTemplate restTemplate;
  @Autowired protected ObjectMapper objectMapper;
  @Autowired protected EventAuditUtil eventAuditUtil;
  @Autowired protected FeatureFlagConfiguration featureFlagConfiguration;
  
  // Automatic lifecycle hooks
  @PostConstruct  // Called when Spring creates this bean
  public final void postConstruct() {
    initialize();                              // 1. Provider-specific setup
    CapabilityProviderRegistry.getInstance()
        .registerProvider(this);               // 2. Register in central registry
    checkHealth();                             // 3. Initial health check
  }
  
  @PreDestroy  // Called when Spring shuts down
  public final void preDestroy() {
    shutdown();  // Cleanup
  }
}
```

**What it provides**:
- ✅ Auto-injection of shared infrastructure (RestTemplate, ObjectMapper, etc.)
- ✅ Automatic registration in registry during Spring startup
- ✅ Automatic health check execution
- ✅ Feature flag integration out-of-the-box
- ✅ Consistent logging with provider type prefix

### 3. **CapabilityProviderRegistry (Singleton)**

Centralized lookup for all providers:

```java
// Singleton pattern
CapabilityProviderRegistry registry = CapabilityProviderRegistry.getInstance();

// Lookup provider by type
Optional<CapabilityProvider> provider = registry.getProvider("VIDA_POA");

// Check if provider is healthy
boolean healthy = registry.isProviderHealthy("VIDA_POA");

// Get all providers
Map<String, CapabilityProvider> all = registry.getAllProviders();
```

**What it provides**:
- ✅ Thread-safe concurrent access (uses `ConcurrentHashMap`)
- ✅ Lookup provider by type ID
- ✅ Health status aggregation across all providers
- ✅ Dynamic provider registration/unregistration

### 4. **Example: VidaPoAProvider**

Real-world implementation for VIDA electronic signature service:

```java
@Component  // Spring will create this as a bean
@Slf4j
public class VidaPoAProvider extends AbstractCapabilityProvider {
  
  // OAuth token caching
  private final AtomicReference<String> cachedAccessToken = new AtomicReference<>();
  private final AtomicReference<Instant> tokenExpiry = new AtomicReference<>();
  
  @Override
  public String getProviderType() {
    return "VIDA_POA";  // Unique identifier
  }
  
  @Override
  public String getProviderName() {
    return "VIDA Power of Attorney Service";
  }
  
  @Override
  public void initialize() {
    log.info("[VIDA_POA] Initializing provider");
    
    // 1. Validate configuration
    ProviderConfig config = providerConfigProperties.getProviderConfig("VIDA_POA");
    if (config == null || config.getClientId() == null) {
      throw new ProviderInitializationException("Missing VIDA_POA config");
    }
    
    // 2. Pre-fetch OAuth token (optional, for faster first request)
    try {
      getAccessToken(config);
      log.info("[VIDA_POA] Pre-fetched OAuth token successfully");
    } catch (Exception e) {
      log.warn("[VIDA_POA] Failed to pre-fetch token, will retry on demand", e);
    }
  }
  
  @Override
  protected ProviderHealthStatus doHealthCheck() {
    try {
      // Lightweight check: can we fetch a token?
      ProviderConfig config = getVidaPoaConfig();
      String token = getAccessToken(config);
      
      return ProviderHealthStatus.healthy("Token fetch successful");
    } catch (Exception e) {
      return ProviderHealthStatus.unhealthy("Token fetch failed: " + e.getMessage());
    }
  }
  
  @Override
  public void shutdown() {
    log.info("[VIDA_POA] Shutting down provider");
    cachedAccessToken.set(null);  // Clear cached token
    tokenExpiry.set(Instant.now());
  }
  
  // Business methods (called by controllers/services)
  public VidaPoASignatureResponse requestPoASignature(PoASignatureRequest request) {
    // 1. Get OAuth token (uses cache if available)
    String token = getAccessToken(getVidaPoaConfig());
    
    // 2. Call VIDA API
    HttpHeaders headers = new HttpHeaders();
    headers.setBearerAuth(token);
    
    ResponseEntity<VidaPoASignatureResponse> response = restTemplate.exchange(
        config.getBaseUrl() + "/sign",
        HttpMethod.POST,
        new HttpEntity<>(request, headers),
        VidaPoASignatureResponse.class
    );
    
    // 3. Audit log (automatic via eventAuditUtil)
    eventAuditUtil.logEvent("VIDA_POA_SIGNATURE_REQUEST", request.getLoanId(), response);
    
    return response.getBody();
  }
}
```

---

## How It Works: Lifecycle Flow

```
Application Startup (Spring Boot)
    │
    ├─ Spring scans for @Component classes
    │
    ├─ Finds VidaPoAProvider (extends AbstractCapabilityProvider)
    │
    ├─ Spring creates VidaPoAProvider bean
    │
    ▼
AbstractCapabilityProvider.postConstruct() [Automatic]
    │
    ├─ Step 1: Call VidaPoAProvider.initialize()
    │           └─> Validate config, pre-fetch OAuth token
    │
    ├─ Step 2: Register in CapabilityProviderRegistry
    │           └─> registry.registerProvider(this)
    │           └─> Now lookup available: registry.getProvider("VIDA_POA")
    │
    ├─ Step 3: Initial health check
    │           └─> VidaPoAProvider.doHealthCheck()
    │           └─> Test if VIDA API is reachable
    │
    └─> Provider is READY for use
    
During Runtime
    │
    ├─ Controller receives request
    ├─ Looks up provider: registry.getProvider("VIDA_POA")
    ├─ Calls: vidaPoAProvider.requestPoASignature(...)
    └─> Provider handles OAuth, API call, audit logging
    
Application Shutdown
    │
    ├─ Spring calls @PreDestroy hooks
    ▼
AbstractCapabilityProvider.preDestroy() [Automatic]
    │
    └─ VidaPoAProvider.shutdown()
        └─> Clear cached tokens, release resources
```

---

## Benefits of Capability Provider Pattern

### 1. **Standardization**
- All external integrations follow the same contract
- Consistent error handling, logging, monitoring
- Easy to understand how any integration works

### 2. **Isolation**
- Each provider is self-contained
- Failures in one provider don't affect others
- Easy to enable/disable via feature flags

### 3. **Discoverability**
- Central registry: `getAllProviders()` → see all integrations
- Health dashboard: check which providers are healthy
- Metadata API: get config info for debugging

### 4. **Testability**
- Easy to mock providers in tests
- Health checks can be run independently
- Registry can be cleared for unit tests

### 5. **Scalability**
- Add new providers without changing core code
- Feature flags enable gradual rollout per merchant
- Circuit breakers prevent cascading failures

### 6. **Observability**
- Automatic health monitoring
- Centralized audit logging via `eventAuditUtil`
- Metadata endpoint shows provider status

---

## Can LND-4525 Use Capability Provider?

**YES! Even though Veefin calls US (inbound), Capability Provider is still highly beneficial.**

### Important: Direction Doesn't Matter

**Common Misconception**: "Capability Provider is only for when WE call external services (outbound)"

**Reality**: Capability Provider works great for BOTH directions:
- ✅ **Outbound**: We call external API (e.g., VIDA PoA - we call VIDA to request signatures)
- ✅ **Inbound**: External partner calls us (e.g., Veefin - Veefin calls us to query borrower data)

**Real Example from Codebase:**

```java
// VidaPoAController - Outbound integration (we call VIDA)
@RestController
@RequestMapping("/fs/brick/service/vida-poa")
public class VidaPoAController {
  
  private final VidaPoAProvider vidaPoAProvider;  // ← Uses Capability Provider
  
  @PostMapping("/signature")
  public ResponseEntity<Object> requestPoaSignature(@RequestBody PoASignatureRequest request) {
    // OUR service calls VIDA API (outbound)
    VidaPoASignatureResponse response = vidaPoAProvider.requestPoASignature(request);
    return ResponseEntity.ok(response);
  }
}

// VeefinBorrowerInquiryController - Inbound integration (Veefin calls us)
@RestController
@RequestMapping("/fs/brick/service/veefin")
public class VeefinBorrowerInquiryController {
  
  private final VeefinBorrowerInquiryProvider veefinProvider;  // ← ALSO uses Capability Provider!
  
  @PostMapping("/v1/inquiry-borrowers")
  public ResponseEntity<BorrowerInquiryResponse> inquireBorrower(@RequestBody BorrowerInquiryRequest request) {
    // VEEFIN calls our API (inbound), but we still use provider pattern
    BorrowerInquiryResponse response = veefinProvider.processBorrowerInquiry(request);
    return ResponseEntity.ok(response);
  }
}
```

### Why Veefin Should Use Capability Provider (Inbound Use Case)

1. **Veefin is an External Partner Integration**
   - Even though Veefin calls US, it's still a partner integration that needs:
     - Authentication (veefin-x-token filter)
     - Rate limiting per partner
     - Partner-specific business logic
     - Audit logging for compliance
   - Capability Provider centralizes ALL partner-specific concerns

2. **Feature Flag Requirements**
   - Enable/disable Veefin integration per merchant/KTP
   - Gradual rollout: test with subset of borrowers first
   - Emergency kill switch if issues arise
   - Example: `if (!provider.isEnabled(ktpNumber)) { return disabled response; }`

3. **Health Monitoring**
   - Is the integration working end-to-end?
   - Can we reach BNPL credit summary service?
   - Is the database available?
   - Health dashboard shows: `GET /brickclient/health/capability`

4. **Audit Requirements**
   - Track every Veefin inquiry request
   - Log request/response for compliance
   - Record which data sources were used (PEFINDO/FDC)
   - Automatic via `eventAuditUtil` built into provider

5. **Configuration Management**
   - Veefin authentication token (from AWS Secrets Manager)
   - Timeouts for credit summary queries
   - Circuit breaker thresholds (if BNPL service is slow)
   - Feature flag configuration
   - All centralized in one place: `capability.providers.configs.VEEFIN_BORROWER_INQUIRY`

6. **Business Logic Isolation**
   - All Veefin-specific logic in ONE place (VeefinBorrowerInquiryProvider)
   - Controller stays thin (just authentication + delegation)
   - Easy to test, mock, and maintain
   - Clear separation: Controller handles HTTP, Provider handles business logic

### Proposed: VeefinBorrowerInquiryProvider

```java
@Component
@Slf4j
public class VeefinBorrowerInquiryProvider extends AbstractCapabilityProvider {
  
  @Autowired
  private CreditSummaryService creditSummaryService;  // BNPL internal service
  
  @Override
  public String getProviderType() {
    return "VEEFIN_BORROWER_INQUIRY";
  }
  
  @Override
  public String getProviderName() {
    return "Veefin Borrower Data Inquiry";
  }
  
  @Override
  public void initialize() {
    log.info("[VEEFIN] Initializing Veefin provider");
    
    // 1. Validate configuration
    ProviderConfig config = providerConfigProperties.getProviderConfig("VEEFIN_BORROWER_INQUIRY");
    if (config == null) {
      throw new ProviderInitializationException("Missing VEEFIN_BORROWER_INQUIRY config");
    }
    
    // 2. Validate token is configured
    if (config.getAuthToken() == null || config.getAuthToken().isEmpty()) {
      throw new ProviderInitializationException("VEEFIN_TOKEN not configured");
    }
    
    log.info("[VEEFIN] Provider initialized successfully");
  }
  
  @Override
  protected ProviderHealthStatus doHealthCheck() {
    try {
      // Health check: can we query credit summary service?
      // (Lightweight check, not full query)
      // This verifies the ENTIRE integration is working:
      //   - Database connectivity
      //   - BNPL service availability
      //   - Configuration validity
      boolean bnplServiceHealthy = creditSummaryService.isHealthy();
      
      if (bnplServiceHealthy) {
        return ProviderHealthStatus.healthy("BNPL service reachable");
      } else {
        return ProviderHealthStatus.unhealthy("BNPL service unavailable");
      }
    } catch (Exception e) {
      return ProviderHealthStatus.unhealthy("Health check failed: " + e.getMessage());
    }
  }
  
  @Override
  public void shutdown() {
    log.info("[VEEFIN] Shutting down Veefin provider");
    // No resources to release (stateless)
  }
  
  /**
   * Business method: Handle borrower inquiry from Veefin.
   * 
   * This is called by VeefinBorrowerInquiryController after authentication.
   */
  public BorrowerInquiryResponse processBorrowerInquiry(BorrowerInquiryRequest request) {
    String ktpNumber = request.getKtpNumber();
    String requestId = request.getRequestId();
    
    log.info("[VEEFIN] Processing borrower inquiry - requestId={}, ktp={}", requestId, ktpNumber);
    
    try {
      // 1. Check if enabled for this merchant (via feature flag)
      if (!isEnabled(ktpNumber)) {
        log.warn("[VEEFIN] Provider disabled for ktp={}", ktpNumber);
        throw new ProviderDisabledException("Veefin provider disabled for this merchant");
      }
      
      // 2. Query credit summary from BNPL service
      CreditSummaryResponse creditSummary = creditSummaryService.getCreditSummaryByKtp(ktpNumber);
      
      // 3. Transform to Veefin response format
      BorrowerInquiryResponse response = BorrowerInquiryResponse.builder()
          .requestId(requestId)
          .ktpNumber(ktpNumber)
          .totalFasilitas(creditSummary.getTotalFasilitas())
          .jmlHariTunggakanTerburuk(creditSummary.getJmlHariTunggakanTerburuk())
          .jmlSaldoTerutang(creditSummary.getJmlSaldoTerutang())
          .dataSource(creditSummary.getDataSource())
          .timestamp(Instant.now())
          .build();
      
      // 4. Audit log (automatic via eventAuditUtil)
      eventAuditUtil.logEvent(
          "VEEFIN_BORROWER_INQUIRY_SUCCESS",
          requestId,
          Map.of(
              "ktp", ktpNumber,
              "totalFasilitas", creditSummary.getTotalFasilitas(),
              "dataSource", creditSummary.getDataSource()
          )
      );
      
      log.info("[VEEFIN] Inquiry successful - requestId={}", requestId);
      return response;
      
    } catch (CreditSummaryNotFoundException e) {
      log.warn("[VEEFIN] Credit summary not found - requestId={}, ktp={}", requestId, ktpNumber);
      
      // Audit log failure
      eventAuditUtil.logEvent(
          "VEEFIN_BORROWER_INQUIRY_NOT_FOUND",
          requestId,
          Map.of("ktp", ktpNumber, "error", e.getMessage())
      );
      
      throw new BorrowerDataNotFoundException("No credit data found for KTP: " + ktpNumber);
      
    } catch (Exception e) {
      log.error("[VEEFIN] Inquiry failed - requestId={}", requestId, e);
      
      // Audit log error
      eventAuditUtil.logEvent(
          "VEEFIN_BORROWER_INQUIRY_ERROR",
          requestId,
          Map.of("ktp", ktpNumber, "error", e.getMessage())
      );
      
      throw new ProviderRuntimeException("Borrower inquiry failed: " + e.getMessage(), e);
    }
  }
}
```

### Controller Using Provider (Inbound Request Handling)

```java
@RestController
@RequestMapping("/fs/brick/service/veefin")
@RequiredArgsConstructor
@Slf4j
public class VeefinBorrowerInquiryController {
  
  // Option 1: Direct injection (simpler, recommended)
  private final VeefinBorrowerInquiryProvider veefinProvider;
  
  // Option 2: Registry lookup (more dynamic, useful if provider type is determined at runtime)
  // private final CapabilityProviderRegistry registry;
  
  /**
   * Handle inbound borrower inquiry requests from Veefin.
   * 
   * Flow:
   * 1. VeefinTokenFilter validates veefin-x-token header (authentication)
   * 2. Controller receives authenticated request
   * 3. Delegates to VeefinBorrowerInquiryProvider (business logic)
   * 4. Provider queries BNPL service, transforms data, logs audit trail
   * 5. Returns standardized response to Veefin
   */
  @PostMapping("/v1/inquiry-borrowers")
  public ResponseEntity<BorrowerInquiryResponse> inquireBorrower(
      @RequestBody @Valid BorrowerInquiryRequest request) {
    
    log.info("[VEEFIN] Received borrower inquiry - requestId={}", request.getRequestId());
    
    try {
      // Option 1: Direct injection (simpler)
      BorrowerInquiryResponse response = veefinProvider.processBorrowerInquiry(request);
      
      /* Option 2: Registry lookup (if provider type is dynamic)
      VeefinBorrowerInquiryProvider provider = (VeefinBorrowerInquiryProvider) registry
          .getProvider("VEEFIN_BORROWER_INQUIRY")
          .orElseThrow(() -> new ProviderNotFoundException("Veefin provider not available"));
      
      BorrowerInquiryResponse response = provider.processBorrowerInquiry(request);
      */
      
      log.info("[VEEFIN] Inquiry completed successfully - requestId={}", request.getRequestId());
      return ResponseEntity.ok(response);
      
    } catch (BorrowerDataNotFoundException e) {
      log.warn("[VEEFIN] Borrower not found - requestId={}", request.getRequestId());
      return ResponseEntity
          .status(HttpStatus.NOT_FOUND)
          .body(BorrowerInquiryResponse.error(request.getRequestId(), "BORROWER_NOT_FOUND"));
          
    } catch (ProviderDisabledException e) {
      log.warn("[VEEFIN] Provider disabled - requestId={}", request.getRequestId());
      return ResponseEntity
          .status(HttpStatus.FORBIDDEN)
          .body(BorrowerInquiryResponse.error(request.getRequestId(), "FEATURE_DISABLED"));
          
    } catch (Exception e) {
      log.error("[VEEFIN] Inquiry failed - requestId={}", request.getRequestId(), e);
      return ResponseEntity
          .status(HttpStatus.INTERNAL_SERVER_ERROR)
          .body(BorrowerInquiryResponse.error(request.getRequestId(), "INTERNAL_ERROR"));
    }
  }
}
```

**Key Points for Inbound Use Case:**
1. **Controller stays thin** - Just HTTP handling, error mapping, logging
2. **Provider contains business logic** - Credit queries, transformations, audit logging
3. **Filter handles authentication** - VeefinTokenFilter validates `veefin-x-token`
4. **Provider handles feature flags** - `isEnabled(ktpNumber)` checks if enabled for this borrower
5. **Automatic audit trail** - Provider logs via `eventAuditUtil` automatically

### Configuration (application.yml)

```yaml
capability:
  providers:
    configs:
      VEEFIN_BORROWER_INQUIRY:
        enabled: true
        baseUrl: N/A  # Veefin calls us, we don't call Veefin (yet)
        authToken: ${VEEFIN_TOKEN}  # From AWS Secrets Manager
        timeoutMs: 30000
        retryAttempts: 2
        circuitBreaker:
          failureThreshold: 5
          waitDurationMs: 60000
```

---

## Benefits for LND-4525 (Veefin)

### 1. **Feature Flag Control**
```java
// Enable Veefin for specific merchants only
if (provider.isEnabled(merchantId)) {
  // Process inquiry
} else {
  // Return "feature not available"
}
```

### 2. **Health Dashboard**
```bash
GET /fs/brick/service/health/providers

Response:
{
  "totalProviders": 2,
  "healthyProviders": 2,
  "unhealthyProviders": 0,
  "providers": {
    "VIDA_POA": {
      "healthy": true,
      "message": "Token fetch successful"
    },
    "VEEFIN_BORROWER_INQUIRY": {
      "healthy": true,
      "message": "BNPL service reachable"
    }
  }
}
```

### 3. **Automatic Audit Logging**
Every Veefin request/response automatically logged via `eventAuditUtil`:
- Request timestamp
- KTP number (masked for privacy)
- Response status (success/failure/not found)
- Data source (PEFINDO/FDC)
- Processing time

### 4. **Circuit Breaker Protection**
If BNPL credit summary service is down:
- Circuit breaker opens after 5 failures
- Veefin requests fail fast (no cascading delays)
- Auto-recovery after 60 seconds

### 5. **Centralized Configuration**
All Veefin settings in one place:
- Token management
- Timeouts
- Retry policies
- Feature flags
- Circuit breaker thresholds

---

## When NOT to Use Capability Provider

❌ **Internal Service Communication (Same Organization)**
- Don't use for BNPL → fs-brick-service calls (both BukuWarung services)
- Don't use for microservice-to-microservice within same organization
- Pattern is for **external partner** integrations only

❌ **Simple CRUD Operations**
- Don't wrap JPA repositories in providers
- Don't use for basic database queries
- Use for **partner integrations** with business logic, not data access layers

❌ **Simple Utility Services**
- Don't use for sending emails
- Don't use for SMS notifications (unless partner-specific with SLAs)
- Capability Provider is for **partner integrations**, not utilities

❌ **When You Don't Need These Features**
- No feature flags required
- No health monitoring needed
- No audit logging required
- No partner-specific business logic
- No circuit breaker/retry logic
- → Then don't use Capability Provider (it's overkill)

---

## Decision Framework: Should I Use Capability Provider?

Ask yourself these questions:

### 1. Is This an External Partner Integration?
- ✅ **YES**: Third-party partner, vendor, external API
- ❌ **NO**: Internal microservice, database, utility

### 2. Do I Need Feature Flags?
- ✅ **YES**: Need to enable/disable per merchant/user, gradual rollout
- ❌ **NO**: Always on, no need for gradual rollout

### 3. Do I Need Health Monitoring?
- ✅ **YES**: Need to check if integration is working, alert on failures
- ❌ **NO**: Fire and forget, don't care about health

### 4. Do I Need Audit Logging?
- ✅ **YES**: Compliance requirement, need to track every operation
- ❌ **NO**: No regulatory requirements

### 5. Is There Complex Business Logic?
- ✅ **YES**: Data transformations, partner-specific rules, multi-step workflows
- ❌ **NO**: Simple pass-through, basic CRUD

### 6. Do I Need Resilience Patterns?
- ✅ **YES**: Circuit breaker, retries, timeout handling critical
- ❌ **NO**: Failures are acceptable, no retry needed

**If you answered YES to 3+ questions → Use Capability Provider**  
**If you answered YES to 1-2 questions → Maybe, consider complexity trade-off**  
**If you answered YES to 0 questions → Don't use it**

---

## Summary

### What Capability Provider Does

1. **Standardizes external integrations** (consistent interface)
2. **Automatic lifecycle management** (Spring @PostConstruct/@PreDestroy)
3. **Centralized registry** (lookup any provider by name)
4. **Health monitoring** (is the external service reachable?)
5. **Feature flags** (enable/disable per merchant)
6. **Audit logging** (track all operations automatically)
7. **Resilience** (circuit breakers, retries, graceful degradation)

### Why Use for LND-4525 (Veefin) - Decision Matrix

| Question | Answer | Justification |
|----------|--------|---------------|
| **Is this an external partner?** | ✅ YES | Veefin is external partner (not BukuWarung service) |
| **Need feature flags?** | ✅ YES | Gradual rollout per merchant, emergency kill switch |
| **Need health monitoring?** | ✅ YES | Track if BNPL service is healthy, alert on failures |
| **Need audit logging?** | ✅ YES | Compliance requirement for credit data access |
| **Complex business logic?** | ✅ YES | Credit summary query, data transformation, multi-source handling |
| **Need resilience patterns?** | ✅ YES | Circuit breaker if BNPL service slow, timeout handling |

**Score: 6/6 → Capability Provider is HIGHLY RECOMMENDED**

Additional Benefits:
- ✅ Consistent with existing patterns (VIDA PoA already uses it)
- ✅ Centralizes all Veefin-specific logic in one provider class
- ✅ Easy to add future Veefin features (webhooks, callbacks, new endpoints)
- ✅ Clear separation of concerns (Controller = HTTP, Provider = business logic)

### Implementation Checklist for LND-4525

- [ ] Create `VeefinBorrowerInquiryProvider extends AbstractCapabilityProvider`
- [ ] Implement `initialize()`, `doHealthCheck()`, `shutdown()`
- [ ] Add business method: `processBorrowerInquiry(request)`
- [ ] Update controller to use `CapabilityProviderRegistry.getProvider("VEEFIN_BORROWER_INQUIRY")`
- [ ] Add provider config to `application.yml` under `capability.providers.configs`
- [ ] Configure feature flags for gradual rollout
- [ ] Test health endpoint: `/fs/brick/service/health/providers`
- [ ] Test audit logs in partner_event_audit table

**Result**: Veefin integration follows production-ready pattern with monitoring, resilience, and observability built-in! 🎉

---

## Appendix: Inbound vs Outbound Comparison

### Outbound Integration (We Call External Service)

**Example**: VIDA PoA - We call VIDA API to request electronic signatures

```
BukuWarung Service (fs-brick-service)
    │
    ├─ VidaPoAController receives request from internal service
    ├─ Delegates to VidaPoAProvider
    │
    └─> VidaPoAProvider.requestPoASignature()
            │
            ├─ Fetch OAuth token from VIDA
            ├─ Call VIDA API (outbound HTTP request)
            ├─ Poll for signature status
            ├─ Download signed document
            └─> Return response to controller

Direction: fs-brick-service → VIDA API
```

**Provider Responsibilities**:
- OAuth token management (fetch, cache, refresh)
- Call external API (outbound RestTemplate)
- Handle external API errors (retry, circuit breaker)
- Poll external service for status updates
- Transform external API response to internal format
- Audit log all operations

---

### Inbound Integration (External Service Calls Us)

**Example**: Veefin - Veefin calls our API to query borrower credit data

```
Veefin Partner (External)
    │
    ├─ Veefin sends HTTP request to our API
    │
    ▼
BukuWarung Service (fs-brick-service)
    │
    ├─ VeefinTokenFilter authenticates request (veefin-x-token)
    ├─ VeefinBorrowerInquiryController receives authenticated request
    ├─ Delegates to VeefinBorrowerInquiryProvider
    │
    └─> VeefinBorrowerInquiryProvider.processBorrowerInquiry()
            │
            ├─ Check feature flag (enabled for this borrower?)
            ├─ Query BNPL credit summary service (internal call)
            ├─ Transform internal data to Veefin format
            ├─ Audit log the inquiry
            └─> Return response to controller → Veefin

Direction: Veefin → fs-brick-service
```

**Provider Responsibilities**:
- Feature flag checking (enabled for this request?)
- Query internal services (BNPL credit summary)
- Transform internal data to partner-expected format
- Handle internal service errors (circuit breaker, graceful degradation)
- Audit log all partner inquiries
- Health check (verify internal services are healthy)

---

### Key Difference: Direction Doesn't Change the Pattern

| Aspect | Outbound (VIDA) | Inbound (Veefin) | Provider Handles Both |
|--------|----------------|------------------|------------------------|
| **HTTP Direction** | We call them | They call us | ✅ |
| **Authentication** | OAuth 2.0 to VIDA | veefin-x-token from Veefin | ✅ |
| **Feature Flags** | Enable VIDA per merchant | Enable Veefin per borrower | ✅ |
| **Health Checks** | Can we reach VIDA? | Can we reach BNPL? | ✅ |
| **Audit Logging** | Log all VIDA API calls | Log all Veefin inquiries | ✅ |
| **Circuit Breaker** | If VIDA is down | If BNPL is slow | ✅ |
| **Business Logic** | OAuth + API call + polling | Feature flag + query + transform | ✅ |
| **Error Handling** | VIDA API errors | BNPL service errors | ✅ |

**Conclusion**: Capability Provider pattern works for BOTH directions. It's about encapsulating **partner-specific integration logic**, regardless of who initiates the HTTP request.

---

## Real-World Analogy

Think of Capability Provider as a **"Partner Integration Module"**:

### Without Capability Provider (Bad)
```
Controller has everything mixed together:
- HTTP handling
- Authentication logic
- Partner-specific business rules
- Feature flag checks
- Audit logging
- Error handling
- Health checks

→ Hard to test, hard to maintain, hard to reuse
```

### With Capability Provider (Good)
```
Controller: Thin HTTP layer
    │
    └─> Delegates to Provider
            │
            ├─ Provider: All partner-specific logic encapsulated
            │   - Business rules
            │   - Feature flags
            │   - Audit logging
            │   - Health checks
            │   - Error handling
            │
            └─> Easy to test, maintain, reuse, monitor
```

**The direction of the HTTP request doesn't matter. What matters is having a clean, standardized way to handle partner integrations!** 🎯
