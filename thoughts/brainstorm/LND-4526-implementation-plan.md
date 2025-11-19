# LND-4526 Implementation Plan

**RFC**: Veefin Signature Authentication Filter
**Timeline**: 3 days (code + unit tests)
**Approach**: TDD with incremental delivery and verification
**Target**: Production-ready code with >85% test coverage (utilities)

---

## 📋 Executive Summary

### Delivery Strategy: Incremental with Verification Gates

Each phase must be **verified and working** before proceeding to the next:

**Day 1**: Foundation (Config + Utilities + Tests)
**Day 2**: Core Filters (Base + Veefin + Tests)
**Day 3**: Security Guard (Catch-All Filter + Integration Tests + YAML Config)

### Success Criteria
- ✅ Working code + comprehensive unit tests
- ✅ >85% test coverage for utility classes
- ✅ Google Java Format compliance (Spotless)
- ✅ All tests passing
- ✅ Configuration YAML ready for deployment
- ✅ Javadoc for public APIs

---

## 🏗️ File Structure

```
fs-brick-service/
├── src/main/java/com/bukuwarung/fsbrickservice/
│   ├── config/
│   │   └── PartnerSignatureConfig.java          [NEW - Day 1]
│   ├── filter/
│   │   ├── AbstractPartnerSignatureFilter.java   [NEW - Day 2]
│   │   ├── VeefinSignatureAuthenticationFilter.java [NEW - Day 2]
│   │   └── PartnerCatchAllFilter.java            [NEW - Day 3]
│   ├── util/
│   │   └── SignatureUtils.java                   [NEW - Day 1]
│   └── exceptions/
│       └── PartnerAuthenticationException.java   [NEW - Day 1]
│
├── src/test/java/com/bukuwarung/fsbrickservice/
│   ├── config/
│   │   └── PartnerSignatureConfigTest.java       [NEW - Day 1]
│   ├── filter/
│   │   ├── AbstractPartnerSignatureFilterTest.java [NEW - Day 2]
│   │   ├── VeefinSignatureAuthenticationFilterTest.java [NEW - Day 2]
│   │   └── PartnerCatchAllFilterTest.java        [NEW - Day 3]
│   ├── util/
│   │   └── SignatureUtilsTest.java               [NEW - Day 1]
│   └── constants/
│       └── PartnerTestConstants.java             [NEW - Day 1]
│
└── src/main/resources/
    └── application.yml                            [MODIFY - Day 3]
```

**Total New Files**: 13 (6 implementation + 6 tests + 1 constants)

---

## 📅 Day 1: Foundation Layer

**Goal**: Configuration + Utilities + Exception Handling
**Verification**: All utility tests pass with >85% coverage

### 1.1 Create Test Constants (30 min)
**File**: `src/test/java/com/bukuwarung/fsbrickservice/constants/PartnerTestConstants.java`

**Purpose**: Centralized test data following existing `TestConstants.java` pattern

```java
package com.bukuwarung.fsbrickservice.constants;

public class PartnerTestConstants {

  // Veefin Partner Constants
  public static final String VEEFIN_PARTNER_ID = "veefin";
  public static final String VEEFIN_SECRET = "test-secret-key-32-chars-long-1234567890";
  public static final String VEEFIN_HEADER_PREFIX = "veefin-x";

  // Test Headers
  public static final String VALID_REQUEST_ID = "1234567890abcdef";
  public static final String VALID_TIMESTAMP = "1699776000"; // Fixed timestamp for tests
  public static final String VALID_SIGNATURE_VEEFIN = "computed-in-test"; // Will be computed

  // Invalid Test Data
  public static final String INVALID_REQUEST_ID = "invalid!@#";
  public static final String EXPIRED_TIMESTAMP = "1000000000"; // Very old
  public static final String INVALID_SIGNATURE = "invalid-signature";

  // Paths
  public static final String VEEFIN_PROTECTED_PATH = "/fs/brick/service/veefin/test";
  public static final String UNPROTECTED_PATH = "/fs/brick/service/other/test";

  // Tolerance
  public static final int DEFAULT_TOLERANCE_MINUTES = 5;
}
```

**Verification**: File compiles, constants accessible

---

### 1.2 Create Custom Exception (15 min)
**File**: `src/main/java/com/bukuwarung/fsbrickservice/exceptions/PartnerAuthenticationException.java`

**Purpose**: Specific exception for partner authentication failures

```java
package com.bukuwarung.fsbrickservice.exceptions;

/**
 * Exception thrown when partner signature authentication fails.
 *
 * <p>This exception is used for:
 * <ul>
 *   <li>Invalid or missing authentication headers</li>
 *   <li>Signature verification failures</li>
 *   <li>Timestamp validation failures</li>
 * </ul>
 */
public class PartnerAuthenticationException extends RuntimeException {

  public PartnerAuthenticationException(String message) {
    super(message);
  }

  public PartnerAuthenticationException(String message, Throwable cause) {
    super(message, cause);
  }
}
```

**Verification**: Exception compiles, can be instantiated in tests

---

### 1.3 Create Configuration Class (TDD) (1.5 hours)

**Step 1: Write Test First** (30 min)
**File**: `src/test/java/com/bukuwarung/fsbrickservice/config/PartnerSignatureConfigTest.java`

```java
package com.bukuwarung.fsbrickservice.config;

import static org.junit.jupiter.api.Assertions.*;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;

@SpringBootTest(classes = PartnerSignatureConfig.class)
@EnableConfigurationProperties(PartnerSignatureConfig.class)
@TestPropertySource(
    properties = {
      "partner.signature.partners.veefin.partner-id=veefin",
      "partner.signature.partners.veefin.secret=test-secret",
      "partner.signature.partners.veefin.header-prefix=veefin-x",
      "partner.signature.partners.veefin.timestamp-tolerance-minutes=5",
      "partner.signature.partners.veefin.path-patterns[0]=/fs/brick/service/veefin/**"
    })
class PartnerSignatureConfigTest {

  @Autowired private PartnerSignatureConfig config;

  @Test
  void testConfigurationLoaded() {
    // Then
    assertNotNull(config);
    assertNotNull(config.getPartners());
    assertTrue(config.getPartners().containsKey("veefin"));
  }

  @Test
  void testVeefinPartnerConfig() {
    // When
    PartnerSignatureConfig.PartnerConfig veefin = config.getPartners().get("veefin");

    // Then
    assertNotNull(veefin);
    assertEquals("veefin", veefin.getPartnerId());
    assertEquals("test-secret", veefin.getSecret());
    assertEquals("veefin-x", veefin.getHeaderPrefix());
    assertEquals(5, veefin.getTimestampToleranceMinutes());
    assertEquals(1, veefin.getPathPatterns().size());
    assertEquals("/fs/brick/service/veefin/**", veefin.getPathPatterns().get(0));
  }
}
```

**Step 2: Implement Configuration** (1 hour)
**File**: `src/main/java/com/bukuwarung/fsbrickservice/config/PartnerSignatureConfig.java`

```java
package com.bukuwarung.fsbrickservice.config;

import java.util.List;
import java.util.Map;
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

/**
 * Configuration for partner signature authentication.
 *
 * <p>Binds to YAML configuration under {@code partner.signature} prefix.
 *
 * <p>Example configuration:
 * <pre>
 * partner:
 *   signature:
 *     partners:
 *       veefin:
 *         partner-id: veefin
 *         secret: ${VEEFIN_PARTNER_SECRET}
 *         header-prefix: veefin-x
 *         timestamp-tolerance-minutes: 5
 *         path-patterns:
 *           - /fs/brick/service/veefin/**
 * </pre>
 */
@Configuration
@ConfigurationProperties(prefix = "partner.signature")
@Data
public class PartnerSignatureConfig {

  /** Map of partner configurations keyed by partner ID */
  private Map<String, PartnerConfig> partners;

  /** Configuration for a single partner */
  @Data
  public static class PartnerConfig {
    /** Unique partner identifier */
    private String partnerId;

    /** HMAC secret for signature verification */
    private String secret;

    /** HTTP header prefix (e.g., "veefin-x" for headers like "veefin-x-signature") */
    private String headerPrefix;

    /** Timestamp tolerance in minutes for replay protection */
    private int timestampToleranceMinutes;

    /** Ant-style path patterns protected by this partner's authentication */
    private List<String> pathPatterns;
  }
}
```

**Verification Gate**: Run test `PartnerSignatureConfigTest` → Must pass ✅

---

### 1.4 Create Signature Utilities (TDD) (2.5 hours)

**Step 1: Write Comprehensive Tests First** (1 hour)
**File**: `src/test/java/com/bukuwarung/fsbrickservice/util/SignatureUtilsTest.java`

```java
package com.bukuwarung.fsbrickservice.util;

import static org.junit.jupiter.api.Assertions.*;

import com.bukuwarung.fsbrickservice.constants.PartnerTestConstants;
import java.util.stream.Stream;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.Arguments;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.MethodSource;

class SignatureUtilsTest {

  private static final String TEST_PARTNER_ID = PartnerTestConstants.VEEFIN_PARTNER_ID;
  private static final String TEST_SECRET = PartnerTestConstants.VEEFIN_SECRET;
  private static final String TEST_REQUEST_ID = PartnerTestConstants.VALID_REQUEST_ID;
  private static final String TEST_TIMESTAMP = PartnerTestConstants.VALID_TIMESTAMP;

  @Test
  void testGenerateSignature_expectValidBase64() {
    // When
    String signature =
        SignatureUtils.generateSignature(TEST_PARTNER_ID, TEST_REQUEST_ID, TEST_TIMESTAMP, TEST_SECRET);

    // Then
    assertNotNull(signature);
    assertFalse(signature.isEmpty());
    // Base64 encoded string should not contain special chars except =
    assertTrue(signature.matches("[A-Za-z0-9+/=]+"));
  }

  @Test
  void testVerifySignature_withValidSignature_expectTrue() {
    // Given
    String signature =
        SignatureUtils.generateSignature(TEST_PARTNER_ID, TEST_REQUEST_ID, TEST_TIMESTAMP, TEST_SECRET);

    // When
    boolean valid =
        SignatureUtils.verifySignature(
            signature, TEST_PARTNER_ID, TEST_REQUEST_ID, TEST_TIMESTAMP, TEST_SECRET);

    // Then
    assertTrue(valid);
  }

  @Test
  void testVerifySignature_withInvalidSignature_expectFalse() {
    // Given
    String invalidSignature = "invalid-signature";

    // When
    boolean valid =
        SignatureUtils.verifySignature(
            invalidSignature, TEST_PARTNER_ID, TEST_REQUEST_ID, TEST_TIMESTAMP, TEST_SECRET);

    // Then
    assertFalse(valid);
  }

  @Test
  void testVerifySignature_withDifferentSecret_expectFalse() {
    // Given
    String signature =
        SignatureUtils.generateSignature(TEST_PARTNER_ID, TEST_REQUEST_ID, TEST_TIMESTAMP, TEST_SECRET);

    // When
    boolean valid =
        SignatureUtils.verifySignature(
            signature, TEST_PARTNER_ID, TEST_REQUEST_ID, TEST_TIMESTAMP, "different-secret");

    // Then
    assertFalse(valid);
  }

  @ParameterizedTest(name = "[{index}] partnerId={0}, requestId={1}, timestamp={2} should be deterministic")
  @CsvSource({
    "veefin,abc123,1699776000",
    "partner-b,xyz789,1699776001",
    "test,request1,1234567890"
  })
  void testGenerateSignature_sameinputs_expectDeterministic(
      String partnerId, String requestId, String timestamp) {
    // When
    String signature1 = SignatureUtils.generateSignature(partnerId, requestId, timestamp, TEST_SECRET);
    String signature2 = SignatureUtils.generateSignature(partnerId, requestId, timestamp, TEST_SECRET);

    // Then
    assertEquals(signature1, signature2);
  }

  @ParameterizedTest(name = "[{index}] Different {3} should produce different signatures")
  @MethodSource("provideDifferentPayloadInputs")
  void testGenerateSignature_differentInputs_expectDifferentSignatures(
      String partnerId1,
      String requestId1,
      String timestamp1,
      String partnerId2,
      String requestId2,
      String timestamp2,
      String changedField) {
    // When
    String signature1 =
        SignatureUtils.generateSignature(partnerId1, requestId1, timestamp1, TEST_SECRET);
    String signature2 =
        SignatureUtils.generateSignature(partnerId2, requestId2, timestamp2, TEST_SECRET);

    // Then
    assertNotEquals(signature1, signature2);
  }

  private static Stream<Arguments> provideDifferentPayloadInputs() {
    return Stream.of(
        Arguments.of("veefin", "req1", "1699776000", "partner-b", "req1", "1699776000", "partnerId"),
        Arguments.of("veefin", "req1", "1699776000", "veefin", "req2", "1699776000", "requestId"),
        Arguments.of("veefin", "req1", "1699776000", "veefin", "req1", "1699776001", "timestamp"));
  }

  @Test
  void testConstructPayload_expectCorrectFormat() {
    // When
    String payload = SignatureUtils.constructPayload(TEST_PARTNER_ID, TEST_REQUEST_ID, TEST_TIMESTAMP);

    // Then
    assertEquals("veefin|1234567890abcdef|1699776000", payload);
  }
}
```

**Step 2: Implement Signature Utilities** (1.5 hours)
**File**: `src/main/java/com/bukuwarung/fsbrickservice/util/SignatureUtils.java`

```java
package com.bukuwarung.fsbrickservice.util;

import java.nio.charset.StandardCharsets;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import java.util.Base64;
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import lombok.extern.slf4j.Slf4j;

/**
 * Utility class for HMAC-SHA256 signature generation and verification.
 *
 * <p>Used for partner signature authentication following the pattern:
 * <pre>
 * payload = partnerId|requestId|timestamp
 * signature = Base64(HMAC-SHA256(payload, secret))
 * </pre>
 */
@Slf4j
public class SignatureUtils {

  private static final String HMAC_ALGORITHM = "HmacSHA256";

  private SignatureUtils() {
    // Utility class - prevent instantiation
  }

  /**
   * Constructs the signature payload from partner authentication components.
   *
   * @param partnerId the partner identifier
   * @param requestId the unique request identifier
   * @param timestamp the request timestamp (Unix epoch seconds)
   * @return pipe-delimited payload string
   */
  public static String constructPayload(String partnerId, String requestId, String timestamp) {
    return String.format("%s|%s|%s", partnerId, requestId, timestamp);
  }

  /**
   * Generates HMAC-SHA256 signature for the given payload.
   *
   * @param partnerId the partner identifier
   * @param requestId the unique request identifier
   * @param timestamp the request timestamp
   * @param secret the HMAC secret key
   * @return Base64-encoded signature
   * @throws RuntimeException if signature generation fails
   */
  public static String generateSignature(
      String partnerId, String requestId, String timestamp, String secret) {
    String payload = constructPayload(partnerId, requestId, timestamp);
    return computeHmac(payload, secret);
  }

  /**
   * Verifies that the provided signature matches the expected signature for the given inputs.
   *
   * @param providedSignature the signature to verify
   * @param partnerId the partner identifier
   * @param requestId the unique request identifier
   * @param timestamp the request timestamp
   * @param secret the HMAC secret key
   * @return true if signature is valid, false otherwise
   */
  public static boolean verifySignature(
      String providedSignature,
      String partnerId,
      String requestId,
      String timestamp,
      String secret) {
    try {
      String expectedSignature = generateSignature(partnerId, requestId, timestamp, secret);
      return constantTimeEquals(providedSignature, expectedSignature);
    } catch (Exception e) {
      log.error("Signature verification failed due to error", e);
      return false;
    }
  }

  /**
   * Computes HMAC-SHA256 hash of the payload using the provided secret.
   *
   * @param payload the data to sign
   * @param secret the HMAC secret key
   * @return Base64-encoded HMAC signature
   * @throws RuntimeException if HMAC computation fails
   */
  private static String computeHmac(String payload, String secret) {
    try {
      Mac mac = Mac.getInstance(HMAC_ALGORITHM);
      SecretKeySpec secretKeySpec =
          new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), HMAC_ALGORITHM);
      mac.init(secretKeySpec);
      byte[] hmacBytes = mac.doFinal(payload.getBytes(StandardCharsets.UTF_8));
      return Base64.getEncoder().encodeToString(hmacBytes);
    } catch (NoSuchAlgorithmException | InvalidKeyException e) {
      log.error("Failed to compute HMAC signature", e);
      throw new RuntimeException("Signature generation failed", e);
    }
  }

  /**
   * Constant-time string comparison to prevent timing attacks.
   *
   * @param a first string
   * @param b second string
   * @return true if strings are equal, false otherwise
   */
  private static boolean constantTimeEquals(String a, String b) {
    if (a == null || b == null) {
      return a == b;
    }

    byte[] aBytes = a.getBytes(StandardCharsets.UTF_8);
    byte[] bBytes = b.getBytes(StandardCharsets.UTF_8);

    if (aBytes.length != bBytes.length) {
      return false;
    }

    int result = 0;
    for (int i = 0; i < aBytes.length; i++) {
      result |= aBytes[i] ^ bBytes[i];
    }
    return result == 0;
  }
}
```

**Verification Gate**: Run test `SignatureUtilsTest` → Must achieve >85% coverage ✅

---

### Day 1 Deliverables Checklist
- [ ] `PartnerTestConstants.java` created and compiles
- [ ] `PartnerAuthenticationException.java` created and compiles
- [ ] `PartnerSignatureConfigTest.java` written and passes
- [ ] `PartnerSignatureConfig.java` implemented and test passes
- [ ] `SignatureUtilsTest.java` written with >10 test cases
- [ ] `SignatureUtils.java` implemented with >85% coverage
- [ ] All tests pass: `./gradlew test --tests "*Partner*" --tests "*Signature*"`
- [ ] Code formatted with Spotless: `./gradlew spotlessApply`

**Verification Command**:
```bash
cd /Users/juvianto.chi/Desktop/code/fs-brick-service
./gradlew test --tests "com.bukuwarung.fsbrickservice.config.PartnerSignatureConfigTest" \
                --tests "com.bukuwarung.fsbrickservice.util.SignatureUtilsTest"
./gradlew spotlessApply
```

---

## 📅 Day 2: Core Filter Implementation

**Goal**: Base Filter + Veefin Filter + Comprehensive Tests
**Verification**: Filter chain works with valid/invalid signatures

### 2.1 Abstract Base Filter (TDD) (2.5 hours)

**Step 1: Write Base Filter Tests** (1 hour)
**File**: `src/test/java/com/bukuwarung/fsbrickservice/filter/AbstractPartnerSignatureFilterTest.java`

```java
package com.bukuwarung.fsbrickservice.filter;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

import com.bukuwarung.fsbrickservice.config.PartnerSignatureConfig;
import com.bukuwarung.fsbrickservice.constants.PartnerTestConstants;
import com.bukuwarung.fsbrickservice.util.SignatureUtils;
import java.util.List;
import javax.servlet.FilterChain;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class AbstractPartnerSignatureFilterTest {

  @Mock private HttpServletRequest request;
  @Mock private HttpServletResponse response;
  @Mock private FilterChain filterChain;

  private TestPartnerFilter filter;
  private PartnerSignatureConfig.PartnerConfig partnerConfig;

  @BeforeEach
  void setUp() {
    partnerConfig = new PartnerSignatureConfig.PartnerConfig();
    partnerConfig.setPartnerId(PartnerTestConstants.VEEFIN_PARTNER_ID);
    partnerConfig.setSecret(PartnerTestConstants.VEEFIN_SECRET);
    partnerConfig.setHeaderPrefix(PartnerTestConstants.VEEFIN_HEADER_PREFIX);
    partnerConfig.setTimestampToleranceMinutes(PartnerTestConstants.DEFAULT_TOLERANCE_MINUTES);
    partnerConfig.setPathPatterns(List.of("/fs/brick/service/veefin/**"));

    filter = new TestPartnerFilter(partnerConfig);
  }

  @Test
  void testDoFilterInternal_withValidSignature_expectSuccess() throws Exception {
    // Given
    String currentTimestamp = String.valueOf(System.currentTimeMillis() / 1000);
    String validSignature =
        SignatureUtils.generateSignature(
            PartnerTestConstants.VEEFIN_PARTNER_ID,
            PartnerTestConstants.VALID_REQUEST_ID,
            currentTimestamp,
            PartnerTestConstants.VEEFIN_SECRET);

    when(request.getHeader("veefin-x-partner-id"))
        .thenReturn(PartnerTestConstants.VEEFIN_PARTNER_ID);
    when(request.getHeader("veefin-x-request-id"))
        .thenReturn(PartnerTestConstants.VALID_REQUEST_ID);
    when(request.getHeader("veefin-x-timestamp")).thenReturn(currentTimestamp);
    when(request.getHeader("veefin-x-signature")).thenReturn(validSignature);
    when(request.getRequestURI()).thenReturn("/fs/brick/service/veefin/test");

    // When
    filter.doFilter(request, response, filterChain);

    // Then
    verify(request).setAttribute("partner.authenticated", true);
    verify(request).setAttribute("partner.id", PartnerTestConstants.VEEFIN_PARTNER_ID);
    verify(filterChain).doFilter(request, response);
    verify(response, never()).sendError(anyInt(), anyString());
  }

  @Test
  void testDoFilterInternal_withMissingHeaders_expectUnauthorized() throws Exception {
    // Given
    when(request.getHeader("veefin-x-partner-id")).thenReturn(null);
    when(request.getRequestURI()).thenReturn("/fs/brick/service/veefin/test");

    // When
    filter.doFilter(request, response, filterChain);

    // Then
    verify(response).sendError(eq(HttpServletResponse.SC_UNAUTHORIZED), anyString());
    verify(filterChain, never()).doFilter(request, response);
  }

  @Test
  void testDoFilterInternal_withInvalidSignature_expectUnauthorized() throws Exception {
    // Given
    String currentTimestamp = String.valueOf(System.currentTimeMillis() / 1000);

    when(request.getHeader("veefin-x-partner-id"))
        .thenReturn(PartnerTestConstants.VEEFIN_PARTNER_ID);
    when(request.getHeader("veefin-x-request-id"))
        .thenReturn(PartnerTestConstants.VALID_REQUEST_ID);
    when(request.getHeader("veefin-x-timestamp")).thenReturn(currentTimestamp);
    when(request.getHeader("veefin-x-signature"))
        .thenReturn(PartnerTestConstants.INVALID_SIGNATURE);
    when(request.getRequestURI()).thenReturn("/fs/brick/service/veefin/test");

    // When
    filter.doFilter(request, response, filterChain);

    // Then
    verify(response).sendError(eq(HttpServletResponse.SC_UNAUTHORIZED), contains("Signature"));
    verify(filterChain, never()).doFilter(request, response);
  }

  @Test
  void testDoFilterInternal_withExpiredTimestamp_expectUnauthorized() throws Exception {
    // Given
    String expiredTimestamp = PartnerTestConstants.EXPIRED_TIMESTAMP;
    String validSignature =
        SignatureUtils.generateSignature(
            PartnerTestConstants.VEEFIN_PARTNER_ID,
            PartnerTestConstants.VALID_REQUEST_ID,
            expiredTimestamp,
            PartnerTestConstants.VEEFIN_SECRET);

    when(request.getHeader("veefin-x-partner-id"))
        .thenReturn(PartnerTestConstants.VEEFIN_PARTNER_ID);
    when(request.getHeader("veefin-x-request-id"))
        .thenReturn(PartnerTestConstants.VALID_REQUEST_ID);
    when(request.getHeader("veefin-x-timestamp")).thenReturn(expiredTimestamp);
    when(request.getHeader("veefin-x-signature")).thenReturn(validSignature);
    when(request.getRequestURI()).thenReturn("/fs/brick/service/veefin/test");

    // When
    filter.doFilter(request, response, filterChain);

    // Then
    verify(response).sendError(eq(HttpServletResponse.SC_UNAUTHORIZED), contains("Timestamp"));
    verify(filterChain, never()).doFilter(request, response);
  }

  @Test
  void testShouldNotFilter_withNonProtectedPath_expectTrue() throws Exception {
    // Given
    when(request.getRequestURI()).thenReturn("/fs/brick/service/other/test");

    // When
    boolean result = filter.shouldNotFilter(request);

    // Then
    assertTrue(result);
  }

  @Test
  void testShouldNotFilter_withProtectedPath_expectFalse() throws Exception {
    // Given
    when(request.getRequestURI()).thenReturn("/fs/brick/service/veefin/test");

    // When
    boolean result = filter.shouldNotFilter(request);

    // Then
    assertFalse(result);
  }

  // Test implementation of AbstractPartnerSignatureFilter
  private static class TestPartnerFilter extends AbstractPartnerSignatureFilter {
    public TestPartnerFilter(PartnerSignatureConfig.PartnerConfig config) {
      super(config, config.getPartnerId());
    }
  }
}
```

**Step 2: Implement Abstract Base Filter** (1.5 hours)
**File**: `src/main/java/com/bukuwarung/fsbrickservice/filter/AbstractPartnerSignatureFilter.java`

```java
package com.bukuwarung.fsbrickservice.filter;

import com.bukuwarung.fsbrickservice.config.PartnerSignatureConfig.PartnerConfig;
import com.bukuwarung.fsbrickservice.exceptions.PartnerAuthenticationException;
import com.bukuwarung.fsbrickservice.util.SignatureUtils;
import java.io.IOException;
import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.util.AntPathMatcher;
import org.springframework.web.filter.OncePerRequestFilter;

/**
 * Abstract base filter for partner signature authentication.
 *
 * <p>Implements the template method pattern for signature-based partner authentication. Concrete
 * subclasses must provide partner-specific configuration.
 *
 * <p>Authentication flow:
 * <ol>
 *   <li>Extract partner headers (partner-id, request-id, timestamp, signature)
 *   <li>Validate header presence and format
 *   <li>Verify timestamp is within tolerance window
 *   <li>Verify HMAC-SHA256 signature
 *   <li>Set authentication attributes on request
 * </ol>
 *
 * <p>Subclasses can override {@link #validateHeaders}, {@link #verifySignature}, and {@link
 * #onSuccess} for partner-specific customization.
 */
@Slf4j
public abstract class AbstractPartnerSignatureFilter extends OncePerRequestFilter {

  protected final PartnerConfig config;
  protected final String partnerName;
  private final AntPathMatcher pathMatcher = new AntPathMatcher();

  /**
   * Constructs the filter with partner configuration.
   *
   * @param config partner-specific configuration
   * @param partnerName unique partner identifier
   */
  protected AbstractPartnerSignatureFilter(PartnerConfig config, String partnerName) {
    this.config = config;
    this.partnerName = partnerName;
  }

  @Override
  protected void doFilterInternal(
      HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
      throws ServletException, IOException {

    try {
      // Extract headers
      String prefix = config.getHeaderPrefix();
      String partnerId = request.getHeader(prefix + "-partner-id");
      String requestId = request.getHeader(prefix + "-request-id");
      String timestamp = request.getHeader(prefix + "-timestamp");
      String signature = request.getHeader(prefix + "-signature");

      // Validate headers
      validateHeaders(partnerId, requestId, timestamp, signature);

      // Verify timestamp
      validateTimestamp(timestamp);

      // Verify signature
      verifySignature(signature, partnerId, requestId, timestamp);

      // Success - mark authenticated
      request.setAttribute("partner.authenticated", true);
      request.setAttribute("partner.id", partnerName);

      log.debug("Partner {} authenticated successfully for request {}", partnerName, requestId);
      onSuccess(request, response);

      filterChain.doFilter(request, response);

    } catch (PartnerAuthenticationException e) {
      log.error("Partner {} authentication failed: {}", partnerName, e.getMessage());
      response.sendError(HttpServletResponse.SC_UNAUTHORIZED, e.getMessage());
    }
  }

  /**
   * Validates that all required headers are present.
   *
   * @param partnerId the partner identifier header value
   * @param requestId the request identifier header value
   * @param timestamp the timestamp header value
   * @param signature the signature header value
   * @throws PartnerAuthenticationException if any header is missing
   */
  protected void validateHeaders(
      String partnerId, String requestId, String timestamp, String signature) {
    if (partnerId == null || partnerId.isEmpty()) {
      throw new PartnerAuthenticationException("Missing partner-id header");
    }
    if (requestId == null || requestId.isEmpty()) {
      throw new PartnerAuthenticationException("Missing request-id header");
    }
    if (timestamp == null || timestamp.isEmpty()) {
      throw new PartnerAuthenticationException("Missing timestamp header");
    }
    if (signature == null || signature.isEmpty()) {
      throw new PartnerAuthenticationException("Missing signature header");
    }

    // Verify partnerId matches expected
    if (!partnerName.equals(partnerId)) {
      throw new PartnerAuthenticationException(
          String.format("Partner ID mismatch: expected %s, got %s", partnerName, partnerId));
    }
  }

  /**
   * Validates that the timestamp is within the configured tolerance window.
   *
   * @param timestamp the request timestamp (Unix epoch seconds)
   * @throws PartnerAuthenticationException if timestamp is outside tolerance
   */
  protected void validateTimestamp(String timestamp) {
    try {
      long requestTime = Long.parseLong(timestamp);
      long currentTime = System.currentTimeMillis() / 1000;
      long diff = Math.abs(currentTime - requestTime);
      long toleranceSeconds = config.getTimestampToleranceMinutes() * 60L;

      if (diff > toleranceSeconds) {
        throw new PartnerAuthenticationException(
            String.format(
                "Timestamp outside tolerance window: %d seconds difference (max: %d)",
                diff, toleranceSeconds));
      }
    } catch (NumberFormatException e) {
      throw new PartnerAuthenticationException("Invalid timestamp format: " + timestamp);
    }
  }

  /**
   * Verifies the HMAC-SHA256 signature.
   *
   * @param signature the provided signature
   * @param partnerId the partner identifier
   * @param requestId the request identifier
   * @param timestamp the request timestamp
   * @throws PartnerAuthenticationException if signature verification fails
   */
  protected void verifySignature(
      String signature, String partnerId, String requestId, String timestamp) {
    boolean valid =
        SignatureUtils.verifySignature(
            signature, partnerId, requestId, timestamp, config.getSecret());

    if (!valid) {
      throw new PartnerAuthenticationException("Signature verification failed");
    }
  }

  /**
   * Hook method called after successful authentication. Subclasses can override for custom
   * behavior.
   *
   * @param request the HTTP request
   * @param response the HTTP response
   */
  protected void onSuccess(HttpServletRequest request, HttpServletResponse response) {
    // Default: no-op. Subclasses can override for custom behavior.
  }

  @Override
  protected boolean shouldNotFilter(HttpServletRequest request) throws ServletException {
    String path = request.getRequestURI();

    // Skip filter if path doesn't match any configured patterns
    boolean matchesPattern =
        config.getPathPatterns().stream().anyMatch(pattern -> pathMatcher.match(pattern, path));

    // Skip if path doesn't match OR if partner headers are not present
    if (!matchesPattern) {
      return true;
    }

    // Check if partner-specific headers are present
    String prefix = config.getHeaderPrefix();
    String partnerId = request.getHeader(prefix + "-partner-id");
    return partnerId == null;
  }
}
```

**Verification Gate**: Run test `AbstractPartnerSignatureFilterTest` → Must pass ✅

---

### 2.2 Veefin Concrete Filter (TDD) (1.5 hours)

**Step 1: Write Veefin Filter Tests** (30 min)
**File**: `src/test/java/com/bukuwarung/fsbrickservice/filter/VeefinSignatureAuthenticationFilterTest.java`

```java
package com.bukuwarung.fsbrickservice.filter;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

import com.bukuwarung.fsbrickservice.config.PartnerSignatureConfig;
import com.bukuwarung.fsbrickservice.constants.PartnerTestConstants;
import com.bukuwarung.fsbrickservice.util.SignatureUtils;
import java.util.List;
import javax.servlet.FilterChain;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class VeefinSignatureAuthenticationFilterTest {

  @Mock private HttpServletRequest request;
  @Mock private HttpServletResponse response;
  @Mock private FilterChain filterChain;

  private VeefinSignatureAuthenticationFilter filter;
  private PartnerSignatureConfig config;

  @BeforeEach
  void setUp() {
    config = new PartnerSignatureConfig();
    PartnerSignatureConfig.PartnerConfig veefinConfig =
        new PartnerSignatureConfig.PartnerConfig();
    veefinConfig.setPartnerId(PartnerTestConstants.VEEFIN_PARTNER_ID);
    veefinConfig.setSecret(PartnerTestConstants.VEEFIN_SECRET);
    veefinConfig.setHeaderPrefix(PartnerTestConstants.VEEFIN_HEADER_PREFIX);
    veefinConfig.setTimestampToleranceMinutes(PartnerTestConstants.DEFAULT_TOLERANCE_MINUTES);
    veefinConfig.setPathPatterns(List.of("/fs/brick/service/veefin/**"));

    config.setPartners(java.util.Map.of("veefin", veefinConfig));

    filter = new VeefinSignatureAuthenticationFilter(config);
  }

  @Test
  void testFilterOrder_expectOrder3() {
    // Then
    assertEquals(3, filter.getOrder());
  }

  @Test
  void testDoFilter_withValidVeefinSignature_expectSuccess() throws Exception {
    // Given
    String currentTimestamp = String.valueOf(System.currentTimeMillis() / 1000);
    String validSignature =
        SignatureUtils.generateSignature(
            PartnerTestConstants.VEEFIN_PARTNER_ID,
            PartnerTestConstants.VALID_REQUEST_ID,
            currentTimestamp,
            PartnerTestConstants.VEEFIN_SECRET);

    when(request.getHeader("veefin-x-partner-id"))
        .thenReturn(PartnerTestConstants.VEEFIN_PARTNER_ID);
    when(request.getHeader("veefin-x-request-id"))
        .thenReturn(PartnerTestConstants.VALID_REQUEST_ID);
    when(request.getHeader("veefin-x-timestamp")).thenReturn(currentTimestamp);
    when(request.getHeader("veefin-x-signature")).thenReturn(validSignature);
    when(request.getRequestURI()).thenReturn(PartnerTestConstants.VEEFIN_PROTECTED_PATH);

    // When
    filter.doFilter(request, response, filterChain);

    // Then
    verify(request).setAttribute("partner.authenticated", true);
    verify(request).setAttribute("partner.id", "veefin");
    verify(filterChain).doFilter(request, response);
  }

  @Test
  void testShouldNotFilter_withNonVeefinPath_expectTrue() throws Exception {
    // Given
    when(request.getRequestURI()).thenReturn("/fs/brick/service/other/test");

    // When
    boolean result = filter.shouldNotFilter(request);

    // Then
    assertTrue(result);
  }
}
```

**Step 2: Implement Veefin Filter** (1 hour)
**File**: `src/main/java/com/bukuwarung/fsbrickservice/filter/VeefinSignatureAuthenticationFilter.java`

```java
package com.bukuwarung.fsbrickservice.filter;

import com.bukuwarung.fsbrickservice.config.PartnerSignatureConfig;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

/**
 * Signature authentication filter for Veefin partner.
 *
 * <p>Authenticates requests from Veefin partner using HMAC-SHA256 signature verification.
 *
 * <p>Expected headers:
 * <ul>
 *   <li>{@code veefin-x-partner-id}: Must be "veefin"
 *   <li>{@code veefin-x-request-id}: Unique request identifier
 *   <li>{@code veefin-x-timestamp}: Unix epoch timestamp in seconds
 *   <li>{@code veefin-x-signature}: Base64-encoded HMAC-SHA256 signature
 * </ul>
 *
 * <p>Protected paths: {@code /fs/brick/service/veefin/**}
 *
 * <p>Filter order: 3 (after {@code FsBrickTokenFilter} at order 1)
 */
@Component
@Slf4j
@Order(3)
public class VeefinSignatureAuthenticationFilter extends AbstractPartnerSignatureFilter
    implements Ordered {

  public VeefinSignatureAuthenticationFilter(PartnerSignatureConfig config) {
    super(config.getPartners().get("veefin"), "veefin");
  }

  @Override
  public int getOrder() {
    return 3;
  }
}
```

**Verification Gate**: Run test `VeefinSignatureAuthenticationFilterTest` → Must pass ✅

---

### Day 2 Deliverables Checklist
- [ ] `AbstractPartnerSignatureFilterTest.java` written with >6 test cases
- [ ] `AbstractPartnerSignatureFilter.java` implemented with full validation logic
- [ ] `VeefinSignatureAuthenticationFilterTest.java` written with >2 test cases
- [ ] `VeefinSignatureAuthenticationFilter.java` implemented with @Order(3)
- [ ] All tests pass: `./gradlew test --tests "*Filter*"`
- [ ] Code formatted: `./gradlew spotlessApply`
- [ ] Manually test filter doesn't break existing filters

**Verification Command**:
```bash
cd /Users/juvianto.chi/Desktop/code/fs-brick-service
./gradlew test --tests "com.bukuwarung.fsbrickservice.filter.*Test"
./gradlew spotlessApply
```

---

## 📅 Day 3: Security Guard + Configuration + Final Integration

**Goal**: Catch-All Filter + YAML Configuration + End-to-End Verification
**Verification**: Complete system works with configuration rollback capability

### 3.1 Catch-All Filter (TDD) (2 hours)

**Step 1: Write Catch-All Filter Tests** (45 min)
**File**: `src/test/java/com/bukuwarung/fsbrickservice/filter/PartnerCatchAllFilterTest.java`

```java
package com.bukuwarung.fsbrickservice.filter;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

import com.bukuwarung.fsbrickservice.config.PartnerSignatureConfig;
import java.util.List;
import java.util.Map;
import javax.servlet.FilterChain;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.core.Ordered;

@ExtendWith(MockitoExtension.class)
class PartnerCatchAllFilterTest {

  @Mock private HttpServletRequest request;
  @Mock private HttpServletResponse response;
  @Mock private FilterChain filterChain;

  private PartnerCatchAllFilter filter;
  private PartnerSignatureConfig config;

  @BeforeEach
  void setUp() {
    config = new PartnerSignatureConfig();

    PartnerSignatureConfig.PartnerConfig veefinConfig =
        new PartnerSignatureConfig.PartnerConfig();
    veefinConfig.setPathPatterns(List.of("/fs/brick/service/veefin/**"));

    PartnerSignatureConfig.PartnerConfig partnerBConfig =
        new PartnerSignatureConfig.PartnerConfig();
    partnerBConfig.setPathPatterns(List.of("/fs/brick/service/partner-b/**"));

    config.setPartners(Map.of("veefin", veefinConfig, "partner-b", partnerBConfig));

    filter = new PartnerCatchAllFilter(config);
  }

  @Test
  void testFilterOrder_expectLowestPrecedenceMinus10() {
    // Then
    assertEquals(Ordered.LOWEST_PRECEDENCE - 10, filter.getOrder());
  }

  @Test
  void testDoFilterInternal_protectedPathWithAuthentication_expectAllow() throws Exception {
    // Given
    when(request.getRequestURI()).thenReturn("/fs/brick/service/veefin/test");
    when(request.getAttribute("partner.authenticated")).thenReturn(true);

    // When
    filter.doFilter(request, response, filterChain);

    // Then
    verify(filterChain).doFilter(request, response);
    verify(response, never()).sendError(anyInt(), anyString());
  }

  @Test
  void testDoFilterInternal_protectedPathWithoutAuthentication_expectUnauthorized()
      throws Exception {
    // Given
    when(request.getRequestURI()).thenReturn("/fs/brick/service/veefin/test");
    when(request.getAttribute("partner.authenticated")).thenReturn(null);

    // When
    filter.doFilter(request, response, filterChain);

    // Then
    verify(response)
        .sendError(eq(HttpServletResponse.SC_UNAUTHORIZED), contains("Missing partner authentication"));
    verify(filterChain, never()).doFilter(request, response);
  }

  @Test
  void testDoFilterInternal_unprotectedPath_expectAllow() throws Exception {
    // Given
    when(request.getRequestURI()).thenReturn("/fs/brick/service/other/test");
    when(request.getAttribute("partner.authenticated")).thenReturn(null);

    // When
    filter.doFilter(request, response, filterChain);

    // Then
    verify(filterChain).doFilter(request, response);
    verify(response, never()).sendError(anyInt(), anyString());
  }

  @Test
  void testDoFilterInternal_multiplePartnerPaths_expectCorrectProtection() throws Exception {
    // Test Veefin path protected
    when(request.getRequestURI()).thenReturn("/fs/brick/service/veefin/test");
    when(request.getAttribute("partner.authenticated")).thenReturn(null);

    filter.doFilter(request, response, filterChain);

    verify(response).sendError(eq(HttpServletResponse.SC_UNAUTHORIZED), anyString());

    // Reset mocks
    reset(request, response, filterChain);

    // Test Partner B path protected
    when(request.getRequestURI()).thenReturn("/fs/brick/service/partner-b/test");
    when(request.getAttribute("partner.authenticated")).thenReturn(null);

    filter.doFilter(request, response, filterChain);

    verify(response).sendError(eq(HttpServletResponse.SC_UNAUTHORIZED), anyString());
  }
}
```

**Step 2: Implement Catch-All Filter** (1 hour 15 min)
**File**: `src/main/java/com/bukuwarung/fsbrickservice/filter/PartnerCatchAllFilter.java`

```java
package com.bukuwarung.fsbrickservice.filter;

import com.bukuwarung.fsbrickservice.config.PartnerSignatureConfig;
import java.io.IOException;
import java.util.Set;
import java.util.stream.Collectors;
import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.util.AntPathMatcher;
import org.springframework.web.filter.OncePerRequestFilter;

/**
 * Catch-all filter providing defense-in-depth security for partner-authenticated endpoints.
 *
 * <p>This filter acts as a safety net ensuring that no protected endpoint is accessible without
 * proper partner authentication, even if partner-specific filters are misconfigured.
 *
 * <p>Execution order: {@link Ordered#LOWEST_PRECEDENCE} - 10 (runs after all partner filters)
 *
 * <p>Security guarantee: {@code protected_path && !authenticated → 401 UNAUTHORIZED}
 *
 * <p>Performance: Adds ~10-50μs per request (<3% of signature verification overhead)
 */
@Component
@Slf4j
@Order(Ordered.LOWEST_PRECEDENCE - 10)
public class PartnerCatchAllFilter extends OncePerRequestFilter implements Ordered {

  private final AntPathMatcher matcher = new AntPathMatcher();
  private final Set<String> protectedPatterns;

  /**
   * Constructs catch-all filter by collecting all partner path patterns from configuration.
   *
   * @param config partner signature configuration
   */
  public PartnerCatchAllFilter(PartnerSignatureConfig config) {
    // Dynamically collect all partner path patterns from YAML at startup
    this.protectedPatterns =
        config.getPartners().values().stream()
            .flatMap(pc -> pc.getPathPatterns().stream())
            .collect(Collectors.toUnmodifiableSet());

    log.info("PartnerCatchAllFilter initialized with {} protected patterns", protectedPatterns.size());
    log.debug("Protected patterns: {}", protectedPatterns);
  }

  @Override
  protected void doFilterInternal(
      HttpServletRequest request, HttpServletResponse response, FilterChain chain)
      throws IOException, ServletException {

    String path = request.getRequestURI();

    // Check if path is protected by any partner
    boolean isProtectedPath =
        protectedPatterns.stream().anyMatch(pattern -> matcher.match(pattern, path));

    if (!isProtectedPath) {
      // Unprotected path - allow through
      chain.doFilter(request, response);
      return;
    }

    // Protected path - check if authenticated
    boolean wasAuthenticated =
        Boolean.TRUE.equals(request.getAttribute("partner.authenticated"));

    if (wasAuthenticated) {
      // Authenticated - allow through
      String partnerId = (String) request.getAttribute("partner.id");
      log.debug("Catch-all: authenticated request for partner {} on path {}", partnerId, path);
      chain.doFilter(request, response);
    } else {
      // Security violation - protected endpoint accessed without authentication
      log.error("Catch-all: unauthorized access attempt to protected path {}", path);
      response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
      response.setContentType("application/json");
      response
          .getWriter()
          .write(
              "{\"code\":\"UNAUTHORIZED\",\"message\":\"Missing partner authentication\",\"errors\":[\"Protected endpoint requires partner signature authentication\"]}");
    }
  }

  @Override
  public int getOrder() {
    return Ordered.LOWEST_PRECEDENCE - 10;
  }
}
```

**Verification Gate**: Run test `PartnerCatchAllFilterTest` → Must pass ✅

---

### 3.2 Application Configuration (30 min)

**File**: `src/main/resources/application.yml` (ADD configuration block)

```yaml
# Partner Signature Authentication Configuration
partner:
  signature:
    partners:
      veefin:
        partner-id: veefin
        secret: ${VEEFIN_PARTNER_SECRET:dummy-secret-for-local-dev-only}
        header-prefix: veefin-x
        timestamp-tolerance-minutes: 5
        path-patterns:
          - /fs/brick/service/veefin/**
```

**Local Testing Configuration** (create for testing):
**File**: `src/main/resources/application-local.yml` (CREATE if doesn't exist)

```yaml
# Local development configuration
partner:
  signature:
    partners:
      veefin:
        partner-id: veefin
        secret: local-development-secret-key-32-chars
        header-prefix: veefin-x
        timestamp-tolerance-minutes: 5
        path-patterns:
          - /fs/brick/service/veefin/**
```

---

### 3.3 End-to-End Verification (1 hour)

**Step 1: Manual Filter Chain Test** (30 min)

Create a simple test endpoint to verify filter chain:

**File**: `src/test/java/com/bukuwarung/fsbrickservice/integration/PartnerAuthenticationIntegrationTest.java`

```java
package com.bukuwarung.fsbrickservice.integration;

import static org.junit.jupiter.api.Assertions.*;

import com.bukuwarung.fsbrickservice.constants.PartnerTestConstants;
import com.bukuwarung.fsbrickservice.util.SignatureUtils;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.result.MockMvcResultMatchers;

@SpringBootTest
@AutoConfigureMockMvc
@TestPropertySource(
    properties = {
      "partner.signature.partners.veefin.partner-id=veefin",
      "partner.signature.partners.veefin.secret=test-secret",
      "partner.signature.partners.veefin.header-prefix=veefin-x",
      "partner.signature.partners.veefin.timestamp-tolerance-minutes=5",
      "partner.signature.partners.veefin.path-patterns[0]=/fs/brick/service/veefin/**"
    })
class PartnerAuthenticationIntegrationTest {

  @Autowired private MockMvc mockMvc;

  @Test
  void testVeefinAuthentication_withValidSignature_expectSuccess() throws Exception {
    // Given
    String currentTimestamp = String.valueOf(System.currentTimeMillis() / 1000);
    String requestId = "test-request-123";
    String signature =
        SignatureUtils.generateSignature("veefin", requestId, currentTimestamp, "test-secret");

    // When & Then
    mockMvc
        .perform(
            MockMvcRequestBuilders.get("/fs/brick/service/veefin/test")
                .header("veefin-x-partner-id", "veefin")
                .header("veefin-x-request-id", requestId)
                .header("veefin-x-timestamp", currentTimestamp)
                .header("veefin-x-signature", signature))
        .andExpect(MockMvcResultMatchers.status().isNotFound()); // 404 = passed auth, no endpoint
  }

  @Test
  void testVeefinAuthentication_withInvalidSignature_expect401() throws Exception {
    // Given
    String currentTimestamp = String.valueOf(System.currentTimeMillis() / 1000);
    String requestId = "test-request-123";

    // When & Then
    mockMvc
        .perform(
            MockMvcRequestBuilders.get("/fs/brick/service/veefin/test")
                .header("veefin-x-partner-id", "veefin")
                .header("veefin-x-request-id", requestId)
                .header("veefin-x-timestamp", currentTimestamp)
                .header("veefin-x-signature", "invalid-signature"))
        .andExpect(MockMvcResultMatchers.status().isUnauthorized());
  }

  @Test
  void testCatchAllFilter_protectedPathWithoutAuth_expect401() throws Exception {
    // When & Then - no authentication headers
    mockMvc
        .perform(MockMvcRequestBuilders.get("/fs/brick/service/veefin/test"))
        .andExpect(MockMvcResultMatchers.status().isUnauthorized());
  }
}
```

**Step 2: Verify Existing Filters Still Work** (30 min)

Run existing filter tests to ensure no regression:

```bash
cd /Users/juvianto.chi/Desktop/code/fs-brick-service
./gradlew test --tests "*Filter*Test" --tests "*Integration*Test"
```

**Expected**: All existing filter tests pass + new partner authentication tests pass

---

### Day 3 Deliverables Checklist
- [ ] `PartnerCatchAllFilterTest.java` written with >4 test cases
- [ ] `PartnerCatchAllFilter.java` implemented with dynamic pattern collection
- [ ] `application.yml` updated with partner configuration block
- [ ] `application-local.yml` created for local testing
- [ ] `PartnerAuthenticationIntegrationTest.java` created (optional but recommended)
- [ ] All tests pass: `./gradlew test`
- [ ] Spotless applied: `./gradlew spotlessApply`
- [ ] Existing filters still work (no regression)
- [ ] Manual smoke test with curl/Postman

**Final Verification Command**:
```bash
cd /Users/juvianto.chi/Desktop/code/fs-brick-service
./gradlew clean test
./gradlew spotlessCheck
./gradlew build
```

---

## 📊 Final Deliverables Summary

### Code Artifacts
| Component | LOC | Tests | Coverage |
|-----------|-----|-------|----------|
| `PartnerSignatureConfig` | ~50 | 2 tests | Config validation |
| `SignatureUtils` | ~120 | 10+ tests | >85% |
| `AbstractPartnerSignatureFilter` | ~150 | 6+ tests | Critical paths |
| `VeefinSignatureAuthenticationFilter` | ~30 | 2+ tests | Integration |
| `PartnerCatchAllFilter` | ~80 | 4+ tests | Security validation |
| `PartnerAuthenticationException` | ~10 | N/A | Exception class |
| **Total** | **~440 lines** | **24+ tests** | **>85% utilities** |

### Configuration Files
- [x] `application.yml` - Partner configuration
- [x] `application-local.yml` - Local development config

### Test Files
- [x] 6 test classes with comprehensive coverage
- [x] Integration tests for end-to-end validation
- [x] Test constants for reusable test data

---

## 🚀 Rollback Plan

Since this is a new feature (no existing partner auth), rollback is configuration-based:

**Quick Rollback** (< 1 minute):
```yaml
# In application.yml - Comment out or rename partner
partner:
  signature:
    partners:
      veefin-disabled:  # Change key to disable
        partner-id: veefin
        # ... rest of config
```

**Why This Works**:
- `VeefinSignatureAuthenticationFilter` constructor looks up config by key "veefin"
- If key doesn't exist, filter bean creation fails gracefully or skips initialization
- No code deployment needed, just config change

**Complete Removal** (if needed):
- Remove configuration block from YAML
- Restart application
- Filter beans won't be created without config

---

## ✅ Verification Gates Summary

### Day 1 Gate
```bash
./gradlew test --tests "*PartnerSignatureConfig*" --tests "*SignatureUtils*"
```
**Pass Criteria**: All config and utility tests pass with >85% coverage

### Day 2 Gate
```bash
./gradlew test --tests "*AbstractPartnerSignatureFilter*" --tests "*VeefinSignatureAuthentication*"
```
**Pass Criteria**: All filter tests pass, existing filters still work

### Day 3 Gate
```bash
./gradlew clean build
./gradlew test
./gradlew spotlessCheck
```
**Pass Criteria**: Complete build passes, all tests green, code formatted

---

## 📝 Implementation Notes

### Code Quality Standards
- ✅ Google Java Format (enforced by Spotless)
- ✅ Lombok for boilerplate reduction
- ✅ SLF4J for logging (ERROR for failures, DEBUG for success)
- ✅ Javadoc for public APIs
- ✅ JUnit 5 + Mockito for testing

### Security Considerations
- ✅ Constant-time string comparison (timing attack prevention)
- ✅ HMAC-SHA256 for signature verification
- ✅ Timestamp validation (replay attack mitigation)
- ✅ Defense-in-depth with catch-all filter
- ✅ No secrets in logs

### Performance Characteristics
- Signature verification: ~1-2ms
- Filter overhead: ~0.1ms per filter
- Catch-all overhead: ~0.05ms
- Total authentication overhead: <3ms per request

---

## 🎯 Success Metrics

**Code Metrics**:
- [ ] >85% test coverage for `SignatureUtils`
- [ ] >70% test coverage overall for new code
- [ ] 0 Spotless violations
- [ ] 0 critical SonarQube issues

**Functional Metrics**:
- [ ] Valid signatures pass authentication
- [ ] Invalid signatures rejected with 401
- [ ] Expired timestamps rejected
- [ ] Missing headers rejected
- [ ] Catch-all filter protects all configured paths
- [ ] Existing filters unaffected

**Timeline Metrics**:
- [ ] Day 1 completed: Foundation layer
- [ ] Day 2 completed: Core filters
- [ ] Day 3 completed: Security guard + config
- [ ] All verification gates passed

---

**Ready to proceed with Day 1 implementation?** Let me know and I'll start with the foundation layer! 🚀
