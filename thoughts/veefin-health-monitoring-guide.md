# Health Monitoring for Veefin Integration (LND-4525)

**Context**: Veefin calls our API → fs-brick-service queries BNPL → BNPL queries database → fs-brick returns to Veefin

---

## What Does Health Monitoring Mean for Inbound Integration?

**Common Misconception**: "Health monitoring is only for checking if external APIs are reachable"

**Reality for Veefin (Inbound)**: Health monitoring checks if **WE CAN FULFILL VEEFIN'S REQUEST** - verifying our entire internal data pipeline is operational.

---

## Health Check Components for VeefinBorrowerInquiryProvider

### 1. **Database Connectivity** (Critical)
**What**: Can BNPL service reach its PostgreSQL database?

```java
@Override
protected ProviderHealthStatus doHealthCheck() {
  try {
    // Check 1: Database connectivity
    boolean dbHealthy = checkDatabaseHealth();
    if (!dbHealthy) {
      return ProviderHealthStatus.unhealthy(
          "Database connection failed",
          Map.of("component", "PostgreSQL", "error", "Connection timeout")
      );
    }
```

**Why Important**:
- If database is down, we can't query `merchant_credit_summary` table
- Better to fail health check than return 500 errors to Veefin
- Alerts/monitoring can trigger before Veefin even calls us

**How to Check**:
```java
// Option 1: Simple query
boolean checkDatabaseHealth() {
  try {
    jdbcTemplate.queryForObject("SELECT 1", Integer.class);
    return true;
  } catch (Exception e) {
    log.error("Database health check failed", e);
    return false;
  }
}

// Option 2: Check specific table exists
boolean checkCreditSummaryTableExists() {
  try {
    jdbcTemplate.queryForObject(
        "SELECT COUNT(*) FROM merchant_credit_summary LIMIT 1", 
        Long.class
    );
    return true;
  } catch (Exception e) {
    log.error("merchant_credit_summary table not accessible", e);
    return false;
  }
}
```

---

### 2. **BNPL Service Reachability** (Critical)
**What**: Can fs-brick-service communicate with BNPL service?

```java
    // Check 2: BNPL service reachability
    boolean bnplHealthy = checkBnplServiceHealth();
    if (!bnplHealthy) {
      return ProviderHealthStatus.unhealthy(
          "BNPL service unreachable",
          Map.of(
              "component", "BNPL Service",
              "endpoint", bnplServiceUrl + "/health",
              "error", "Connection refused or timeout"
          )
      );
    }
```

**Why Important**:
- fs-brick-service calls BNPL via RestTemplate/gRPC to get credit summary
- If BNPL service is down/restarting, all Veefin requests will fail
- Health check gives early warning before customer impact

**How to Check**:
```java
// Option 1: Call BNPL health endpoint
boolean checkBnplServiceHealth() {
  try {
    String healthUrl = bnplServiceUrl + "/actuator/health";
    ResponseEntity<Map> response = restTemplate.getForEntity(
        healthUrl, 
        Map.class
    );
    return response.getStatusCode() == HttpStatus.OK;
  } catch (Exception e) {
    log.error("BNPL service health check failed", e);
    return false;
  }
}

// Option 2: Test actual credit summary query (lightweight)
boolean checkCreditSummaryQueryWorks() {
  try {
    // Use a known test KTP or merchant_id
    creditSummaryService.getCreditSummaryByKtp("TEST_KTP_HEALTH_CHECK");
    return true;
  } catch (NotFoundException e) {
    // Expected - KTP doesn't exist, but query worked
    return true;
  } catch (Exception e) {
    // Actual failure - service down or network issue
    log.error("Credit summary query health check failed", e);
    return false;
  }
}
```

---

### 3. **Data Staleness Check** (Optional but Recommended)
**What**: Is the merchant credit summary data recent enough to be useful?

```java
    // Check 3: Data freshness (optional)
    boolean dataFresh = checkDataFreshness();
    if (!dataFresh) {
      return ProviderHealthStatus.degraded(
          "Credit summary data is stale (>30 days old)",
          Map.of(
              "component", "Data Pipeline",
              "warning", "Enrichment process may be delayed"
          )
      );
    }
```

**Why Important**:
- Veefin expects relatively recent credit data
- If enrichment process is stuck, data becomes stale
- Early warning that background jobs need attention

**How to Check**:
```java
boolean checkDataFreshness() {
  try {
    // Check most recent credit summary update
    Instant latestUpdate = jdbcTemplate.queryForObject(
        "SELECT MAX(created_at) FROM merchant_credit_summary",
        Instant.class
    );
    
    if (latestUpdate == null) {
      // No data at all
      return false;
    }
    
    Duration age = Duration.between(latestUpdate, Instant.now());
    // Consider data stale if older than 30 days
    return age.toDays() < 30;
    
  } catch (Exception e) {
    log.error("Data freshness check failed", e);
    return false;
  }
}
```

---

### 4. **Feature Flag Service** (Optional)
**What**: Can we check if Veefin integration is enabled for merchants?

```java
    // Check 4: Feature flag service
    boolean featureFlagHealthy = checkFeatureFlagService();
    if (!featureFlagHealthy) {
      return ProviderHealthStatus.degraded(
          "Feature flag service unavailable - defaulting to enabled",
          Map.of("component", "Feature Flags")
      );
    }
```

**Why Important**:
- If feature flag service is down, we can't gradual rollout
- Falls back to "enabled for all" which may not be desired
- Not critical (graceful degradation) but good to know

---

## Complete Health Check Implementation

```java
@Component
@Slf4j
public class VeefinBorrowerInquiryProvider extends AbstractCapabilityProvider {
  
  @Autowired
  private CreditSummaryService creditSummaryService;
  
  @Autowired
  private JdbcTemplate jdbcTemplate;
  
  @Value("${bnpl.service.url}")
  private String bnplServiceUrl;
  
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
    
    // Validate configuration
    ProviderConfig config = providerConfigProperties.getProviderConfig("VEEFIN_BORROWER_INQUIRY");
    if (config == null) {
      throw new ProviderInitializationException("Missing VEEFIN_BORROWER_INQUIRY config");
    }
    
    // Validate token is configured
    if (config.getAuthToken() == null || config.getAuthToken().isEmpty()) {
      throw new ProviderInitializationException("VEEFIN_TOKEN not configured");
    }
    
    log.info("[VEEFIN] Provider initialized successfully");
  }
  
  @Override
  protected ProviderHealthStatus doHealthCheck() {
    log.debug("[VEEFIN] Performing health check");
    
    Map<String, Object> healthDetails = new HashMap<>();
    List<String> failedComponents = new ArrayList<>();
    
    // 1. Check database connectivity
    boolean dbHealthy = checkDatabaseHealth();
    healthDetails.put("database", dbHealthy ? "UP" : "DOWN");
    if (!dbHealthy) {
      failedComponents.add("Database");
    }
    
    // 2. Check BNPL service reachability
    boolean bnplHealthy = checkBnplServiceHealth();
    healthDetails.put("bnplService", bnplHealthy ? "UP" : "DOWN");
    if (!bnplHealthy) {
      failedComponents.add("BNPL Service");
    }
    
    // 3. Check data freshness (non-blocking)
    boolean dataFresh = checkDataFreshness();
    healthDetails.put("dataFreshness", dataFresh ? "FRESH" : "STALE");
    if (!dataFresh) {
      healthDetails.put("warning", "Credit summary data may be outdated");
    }
    
    // 4. Check feature flag service (non-blocking)
    boolean featureFlagHealthy = checkFeatureFlagService();
    healthDetails.put("featureFlags", featureFlagHealthy ? "UP" : "DOWN");
    
    // Determine overall health status
    if (!failedComponents.isEmpty()) {
      String message = "Critical components unavailable: " + String.join(", ", failedComponents);
      log.warn("[VEEFIN] Health check failed: {}", message);
      return ProviderHealthStatus.unhealthy(message, healthDetails);
    }
    
    if (!dataFresh || !featureFlagHealthy) {
      String message = "Provider operational but with warnings";
      log.info("[VEEFIN] Health check degraded: {}", message);
      return ProviderHealthStatus.degraded(message, healthDetails);
    }
    
    log.debug("[VEEFIN] Health check passed");
    return ProviderHealthStatus.healthy("All systems operational", healthDetails);
  }
  
  private boolean checkDatabaseHealth() {
    try {
      // Simple connectivity check
      jdbcTemplate.queryForObject("SELECT 1", Integer.class);
      
      // Verify merchant_credit_summary table exists and is accessible
      Long count = jdbcTemplate.queryForObject(
          "SELECT COUNT(*) FROM merchant_credit_summary LIMIT 1",
          Long.class
      );
      
      log.debug("[VEEFIN] Database health check passed (rows: {})", count);
      return true;
    } catch (Exception e) {
      log.error("[VEEFIN] Database health check failed", e);
      return false;
    }
  }
  
  private boolean checkBnplServiceHealth() {
    try {
      // Option 1: Call BNPL actuator health endpoint
      String healthUrl = bnplServiceUrl + "/actuator/health";
      ResponseEntity<Map> response = restTemplate.getForEntity(healthUrl, Map.class);
      
      boolean healthy = response.getStatusCode() == HttpStatus.OK;
      log.debug("[VEEFIN] BNPL service health check: {}", healthy ? "PASSED" : "FAILED");
      return healthy;
      
    } catch (Exception e) {
      log.error("[VEEFIN] BNPL service health check failed", e);
      
      // Option 2: Fallback - test if service responds to any endpoint
      try {
        creditSummaryService.getCreditSummaryByKtp("__HEALTH_CHECK__");
        return true; // Service responded (even if 404)
      } catch (NotFoundException ex) {
        return true; // Expected - service is up
      } catch (Exception ex) {
        log.error("[VEEFIN] BNPL service unreachable via fallback check", ex);
        return false;
      }
    }
  }
  
  private boolean checkDataFreshness() {
    try {
      Instant latestUpdate = jdbcTemplate.queryForObject(
          "SELECT MAX(created_at) FROM merchant_credit_summary",
          Instant.class
      );
      
      if (latestUpdate == null) {
        log.warn("[VEEFIN] No credit summary data found");
        return false;
      }
      
      Duration age = Duration.between(latestUpdate, Instant.now());
      boolean fresh = age.toDays() < 30;
      
      log.debug("[VEEFIN] Data freshness check: {} days old ({})", 
          age.toDays(), fresh ? "FRESH" : "STALE");
      return fresh;
      
    } catch (Exception e) {
      log.error("[VEEFIN] Data freshness check failed", e);
      return false;
    }
  }
  
  private boolean checkFeatureFlagService() {
    try {
      // Test feature flag lookup with dummy identifier
      boolean enabled = featureFlagConfiguration.isEnabled(
          "capability_veefin_borrower_inquiry",
          "__HEALTH_CHECK__"
      );
      log.debug("[VEEFIN] Feature flag service health check passed");
      return true;
    } catch (Exception e) {
      log.warn("[VEEFIN] Feature flag service health check failed (non-critical)", e);
      return false;
    }
  }
  
  @Override
  public void shutdown() {
    log.info("[VEEFIN] Shutting down Veefin provider");
    // No resources to release (stateless)
  }
  
  // Business method
  public BorrowerInquiryResponse processBorrowerInquiry(BorrowerInquiryRequest request) {
    // ... implementation from before ...
  }
}
```

---

## Health Check Endpoints Exposed

### 1. **All Providers Health**
```bash
GET /brickclient/health/capability

Response:
{
  "timestamp": "2025-11-07T06:18:12Z",
  "totalProviders": 2,
  "healthyProviders": 2,
  "unhealthyProviders": 0,
  "providers": {
    "VIDA_POA": {
      "healthy": true,
      "status": "UP",
      "message": "Token fetch successful"
    },
    "VEEFIN_BORROWER_INQUIRY": {
      "healthy": true,
      "status": "UP",
      "message": "All systems operational",
      "details": {
        "database": "UP",
        "bnplService": "UP",
        "dataFreshness": "FRESH",
        "featureFlags": "UP"
      }
    }
  }
}
```

### 2. **Veefin Provider Specific Health**
```bash
GET /brickclient/health/capability/VEEFIN_BORROWER_INQUIRY

Response (Healthy):
{
  "timestamp": "2025-11-07T06:18:12Z",
  "providerType": "VEEFIN_BORROWER_INQUIRY",
  "providerName": "Veefin Borrower Data Inquiry",
  "healthy": true,
  "status": "UP",
  "message": "All systems operational",
  "details": {
    "database": "UP",
    "bnplService": "UP",
    "dataFreshness": "FRESH",
    "featureFlags": "UP"
  },
  "metadata": {
    "providerType": "VEEFIN_BORROWER_INQUIRY",
    "providerName": "Veefin Borrower Data Inquiry",
    "initialized": true,
    "healthy": true
  }
}

Response (Unhealthy):
{
  "timestamp": "2025-11-07T06:18:12Z",
  "providerType": "VEEFIN_BORROWER_INQUIRY",
  "providerName": "Veefin Borrower Data Inquiry",
  "healthy": false,
  "status": "DOWN",
  "message": "Critical components unavailable: Database, BNPL Service",
  "details": {
    "database": "DOWN",
    "bnplService": "DOWN",
    "dataFreshness": "UNKNOWN",
    "featureFlags": "UP"
  }
}

Response (Degraded):
{
  "timestamp": "2025-11-07T06:18:12Z",
  "providerType": "VEEFIN_BORROWER_INQUIRY",
  "providerName": "Veefin Borrower Data Inquiry",
  "healthy": true,
  "status": "DEGRADED",
  "message": "Provider operational but with warnings",
  "details": {
    "database": "UP",
    "bnplService": "UP",
    "dataFreshness": "STALE",
    "warning": "Credit summary data may be outdated",
    "featureFlags": "UP"
  }
}
```

---

## Integration with Monitoring/Alerting

### 1. **Prometheus Metrics** (If using Spring Actuator)
```yaml
# Automatically exposed by Capability Provider framework
capability_provider_health{provider="VEEFIN_BORROWER_INQUIRY"} 1.0  # 1.0 = healthy, 0.0 = unhealthy
capability_provider_component_health{provider="VEEFIN_BORROWER_INQUIRY",component="database"} 1.0
capability_provider_component_health{provider="VEEFIN_BORROWER_INQUIRY",component="bnplService"} 1.0
```

### 2. **Grafana Dashboard**
Create dashboard with:
- **Panel 1**: Veefin Provider Health (UP/DOWN/DEGRADED)
- **Panel 2**: Component Health Breakdown (database, BNPL, data freshness)
- **Panel 3**: Health Check Response Time (track latency)
- **Panel 4**: Request Success Rate (% of successful Veefin inquiries)

### 3. **PagerDuty/Slack Alerts**
```yaml
# Alert when Veefin provider is unhealthy for >5 minutes
alert: VeefinProviderUnhealthy
expr: capability_provider_health{provider="VEEFIN_BORROWER_INQUIRY"} == 0
for: 5m
annotations:
  summary: "Veefin integration is DOWN"
  description: "Critical components failed. Check /brickclient/health/capability/VEEFIN_BORROWER_INQUIRY"

# Alert when data is stale
alert: VeefinDataStale
expr: capability_provider_component_health{provider="VEEFIN_BORROWER_INQUIRY",component="dataFreshness"} == 0
for: 1h
annotations:
  summary: "Veefin credit data is stale (>30 days)"
  description: "Enrichment process may be stuck. Check merchant_credit_summary table."
```

---

## Benefits of Health Monitoring for Veefin

### 1. **Proactive Issue Detection**
- **Before**: Veefin calls us → 500 errors → angry partner
- **After**: Health check fails → alert fires → we fix before Veefin notices

### 2. **Clear Operational Dashboard**
```
Veefin Integration Status: ✅ UP
├─ Database:        ✅ UP
├─ BNPL Service:    ✅ UP
├─ Data Freshness:  ⚠️  STALE (28 days)
└─ Feature Flags:   ✅ UP
```

### 3. **Faster Troubleshooting**
When Veefin reports an issue:
1. Check health endpoint: `/brickclient/health/capability/VEEFIN_BORROWER_INQUIRY`
2. See exactly which component failed: "Database connection timeout"
3. No need to dig through logs or test manually

### 4. **SLA Compliance**
- Track uptime: "Veefin integration was healthy 99.5% of the month"
- Prove to partner that our service is reliable
- Identify patterns: "Database goes down every Sunday 2 AM during backups"

### 5. **Gradual Degradation**
```
Status: DEGRADED (not DOWN)
- Database: ✅ Working
- BNPL: ✅ Working
- Data: ⚠️ Stale (but still usable)
- Feature Flags: ❌ Unavailable (defaulting to enabled)

→ Continue serving Veefin requests (with warnings in logs)
→ Alert ops team (non-critical)
```

---

## Health Check Best Practices

### 1. **Keep Checks Lightweight**
```java
// ❌ BAD: Expensive health check
boolean checkHealth() {
  // This queries millions of rows!
  creditSummaryService.getAllCreditSummaries(); 
}

// ✅ GOOD: Lightweight check
boolean checkHealth() {
  jdbcTemplate.queryForObject("SELECT 1", Integer.class);
}
```

### 2. **Don't Modify State**
```java
// ❌ BAD: Health check inserts data
boolean checkHealth() {
  creditSummaryRepository.save(new CreditSummary()); // Side effect!
}

// ✅ GOOD: Read-only check
boolean checkHealth() {
  jdbcTemplate.queryForObject("SELECT COUNT(*) FROM merchant_credit_summary LIMIT 1", Long.class);
}
```

### 3. **Cache Health Status**
```java
// ✅ GOOD: Cache health check results for 30 seconds
private volatile ProviderHealthStatus cachedHealth;
private volatile Instant lastHealthCheck = Instant.MIN;

@Override
protected ProviderHealthStatus doHealthCheck() {
  if (Duration.between(lastHealthCheck, Instant.now()).getSeconds() < 30) {
    return cachedHealth; // Return cached result
  }
  
  // Perform actual health check
  ProviderHealthStatus freshHealth = performActualHealthCheck();
  cachedHealth = freshHealth;
  lastHealthCheck = Instant.now();
  return freshHealth;
}
```

### 4. **Fail Gracefully**
```java
// ✅ GOOD: If health check itself fails, assume unhealthy
@Override
protected ProviderHealthStatus doHealthCheck() {
  try {
    return performHealthCheck();
  } catch (Exception e) {
    log.error("[VEEFIN] Health check threw exception", e);
    return ProviderHealthStatus.unhealthy(
        "Health check failed: " + e.getMessage(),
        Map.of("exception", e.getClass().getSimpleName())
    );
  }
}
```

---

## Summary

### What Health Monitoring Checks for Veefin

1. ✅ **Database connectivity** - Can we query `merchant_credit_summary`?
2. ✅ **BNPL service reachability** - Can fs-brick call BNPL?
3. ✅ **Data freshness** - Is credit data recent?
4. ✅ **Feature flag service** - Can we check enabled/disabled status?

### What Health Monitoring Provides

1. 📊 **Operational dashboard** - Real-time status of Veefin integration
2. 🚨 **Proactive alerts** - Know about issues before Veefin does
3. 🔍 **Faster debugging** - Pinpoint exact component that failed
4. 📈 **SLA tracking** - Prove reliability to partner
5. ⚠️  **Graceful degradation** - Continue serving with warnings

### Health Endpoint URLs

```bash
# All providers
GET /brickclient/health/capability

# Veefin specific
GET /brickclient/health/capability/VEEFIN_BORROWER_INQUIRY

# Registry status
GET /brickclient/health/capability/registry/status
```

**Result**: You have comprehensive health monitoring for Veefin integration, with early warning system and clear operational visibility! 🎯
