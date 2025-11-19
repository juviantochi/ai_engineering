## Scope
**Repository**: `@fs-brick-service`  
**Parent Ticket**: [LND-4525](https://bukuwarung.atlassian.net/browse/LND-4525) - Veefin Borrower Data Inquiry API

## Objective
Implement a signature-based authentication filter (`VeefinSignatureAuthenticationFilter`) for Veefin partner integration to secure the borrower inquiry API endpoint. This filter implements AES-encrypted signature validation, providing stronger security than static token authentication.

## Background
Veefin is a new lending partner that requires access to borrower credit information through our FS Brick service. They will send requests to `/fs/brick/service/veefin/v1/*` endpoints. To ensure secure authentication, Veefin will generate an encrypted signature using their secret key, request timestamp, and request ID. Our filter will validate this signature before allowing access.

## Requirements

### Functional Requirements

#### 1. Filter Implementation: `VeefinSignatureAuthenticationFilter`
Create a filter that extends `OncePerRequestFilter` from Spring and validates requests to `/fs/brick/service/veefin/**` paths.

**Required Headers**:
- `veefin-x-partner-id` - Partner identifier (currently only "veefin")
- `veefin-x-request-id` - Unique request identifier (used as AES IV, must be 16 characters)
- `veefin-x-timestamp` - Unix epoch timestamp in seconds
- `veefin-x-signature` - Base64-encoded encrypted signature

**Validation Flow**:
1. Extract all required headers (return 401 if any missing/empty)
2. Validate timestamp is within 5 minutes of current time (prevent replay attacks)
3. Validate signature is valid Base64 encoding
4. Retrieve partner secret from configuration using `partner-id`
5. Derive AES key from partner-id and timestamp using PBKDF2
6. Decrypt signature using AES/CBC/PKCS5Padding with request-id as IV
7. Compare decrypted value with stored partner secret
8. If valid, allow request to proceed; otherwise return 401

**Signature Generation (Veefin Side)**:
```java
// Key derivation
SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
KeySpec spec = new PBEKeySpec(partnerId.toCharArray(), timestamp.getBytes(), 65536, 256);
SecretKey key = new SecretKeySpec(factory.generateSecret(spec).getEncoded(), "AES");

// Encryption
Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
IvParameterSpec ivSpec = new IvParameterSpec(requestId.getBytes()); // requestId must be 16 chars
cipher.init(Cipher.ENCRYPT_MODE, key, ivSpec);
byte[] encrypted = cipher.doFinal(partnerSecret.getBytes());
String signature = Base64.getEncoder().encodeToString(encrypted);
```

#### 2. Security Configuration
- Partner secrets stored in AWS Secrets Manager: `${VEEFIN_PARTNER_SECRET}`
- Partner ID configuration: `${VEEFIN_PARTNER_ID:veefin}`
- Filter integrated into Spring Security filter chain
- Applied only to Veefin-specific paths: `/fs/brick/service/veefin/**`
- Not impact other endpoints and other mechanism

#### 3. Error Handling
Return 401 Unauthorized with appropriate messages:
- Missing header → `"Missing required authentication header: {header-name}"`
- Empty header → `"Empty authentication header: {header-name}"`
- Expired timestamp → `"Request is expired (timestamp difference > 5 minutes)"`
- Invalid signature format → `"Invalid signature format (not valid Base64)"`
- Signature verification failed → `"Signature verification failed"`
- Response format: `{"code": 401, "message": "Unauthorized - {detail}"}`

#### 4. Logging & Audit Trail
Log all authentication attempts with:
- Partner ID
- Request ID
- Timestamp
- Validation result (success (log level : DEBUG) /failure reason (log level : WARN))
- IP address (if available)

### Non-Functional Requirements
- No impact on other endpoints (non-Veefin paths)
- Comprehensive unit and integration tests
- Clear audit logging for security monitoring
- Support for future partner additions (extensible design)

## What This Task Does NOT Include
- ❌ Business logic or controller implementation (separate task)
- ❌ Database queries or data layer (separate task)  
- ❌ BNPL service integration (separate task)
- ❌ Key rotation mechanism (static secrets for now)
- ❌ Multiple partner support (designed for extensibility but only Veefin implemented)

## Technical Details

### Files to Create
- `VeefinSignatureAuthenticationFilter.java` - Main filter implementation
- `VeefinPartnerConfig.java` - Configuration properties for partner credentials
- `VeefinSignatureAuthenticationFilterTest.java` - Unit tests
- `VeefinSecurityIntegrationTest.java` - Integration tests
- `SignatureUtils.java` - Shared utilities for key derivation and encryption/decryption

### Files to Modify
- `application.yml` - Add Veefin partner configuration

### Configuration Structure
```yaml
veefin:
  partner:
    id: ${VEEFIN_PARTNER_ID:veefin}
    secret: ${VEEFIN_PARTNER_SECRET:default-secret-for-local-testing}
  security:
    header-prefix: veefin-x
    timestamp-tolerance-minutes: 5
```

### AWS Secrets Manager
Secret name: `fs-brick-service/veefin/partner-credentials`
```json
{
  "partnerId": "veefin",
  "partnerSecret": "actual-secret-value-here"
}
```

## Success Criteria
- ✅ Filter validates signature using AES encryption correctly
- ✅ Returns 401 for missing/invalid/expired requests
- ✅ Allows valid requests to proceed
- ✅ Timestamp validation prevents replay attacks (5-minute window)
- ✅ minimum of 80% unit test coverage
- ✅ Integration tests pass with real encryption/decryption
- ✅ Spotless code formatting passes
- ✅ Audit logging captures all authentication failure
- ✅ Documentation includes signature generation example for Veefin

**REFERENCES**
1. rfc/template.md
2. LND-4526 ticket

## Implementation Status
⏳ Implementation pending - Awaiting confirmation before proceeding

**OUTPUT** 
Only after the user asks to create the rfc, then :
1. output RFC (3000–5000 words) following rfc/template.md → save as `<ticket-number>.md` in `RFC/` folder  .
2. output scoped RFC in specific repository impacted by the change. 
   * if more than 1 repository is impacted then output multiple RFCs with each RFC containing only relevant parts of the full RFC → save as `<ticket-number>-<repository-name>.md` in `RFC/` folder.
   * In the scoped repository RFC, include the main RFC as reference.

Audience: Senior Engineers, Architects, Product Team  
Focus: Practical implementation, clear trade-offs, safe migration plan

## Notes for Implementation
1. **requestId must be exactly 16 characters** 
2. **Base64 validation** - Use regex pattern to validate before decryption
3. **Exception handling** - Wrap crypto exceptions into custom `UnauthorisedException`
5. **Veefin documentation** - Provide clear example code for signature generation
