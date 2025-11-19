## Scope
**Repository**: `@fs-brick-service`  
**Parent Ticket**: [LND-4525](https://bukuwarung.atlassian.net/browse/LND-4525) - Veefin Borrower Data Inquiry API

## Objective
Implement a static token authentication filter (`VeefinTokenFilter`) for Veefin partner integration to secure the borrower inquiry API endpoint.

## Background
Veefin is a new lending partner that requires access to borrower credit information through our FS Brick service. They will send requests to `/fs/brick/service/veefin/v1/*` endpoints and need a simple, secure authentication mechanism using a static token in the request header.

## Requirements

### Functional Requirements
1. **Filter Implementation**: Create `VeefinTokenFilter` that:
   - Extends `OncePerRequestFilter` from Spring
   - Validates `veefin-x-token` header for all `/fs/brick/service/veefin/**` paths
   - Returns 401 Unauthorized if token is missing or invalid
   - Allows request to proceed to controller if token is valid
   - Logs all authentication attempts for audit trail

2. **Security Configuration**:
   - Token value stored in AWS Secrets Manager (`${VEEFIN_X_TOKEN}`)
   - Filter integrated into Spring Security filter chain
   - Applied only to Veefin-specific paths

3. **Error Handling**:
   - Missing token → 401 with message: "Missing authentication token"
   - Invalid token → 401 with message: "Invalid authentication token"
   - Response format: `{"code": 401, "message": "Unauthorized - {detail}"}`

### Non-Functional Requirements
- Token validation completes in <10ms
- No impact on other endpoints (non-Veefin paths)
- Comprehensive unit and integration tests
- Clear audit logging for security monitoring

## What This Task Does NOT Include
- ❌ Business logic or controller implementation (separate task)
- ❌ Database queries or data layer (separate task)
- ❌ BNPL service integration (separate task)
- ❌ Capability Provider pattern (separate task)
- ❌ Circuit breaker or health monitoring (separate task)

## Scope: Filter ONLY
This task implements **only the authentication filter**. The filter's responsibility is to:
1. Extract the `veefin-x-token` header
2. Validate it against configured token
3. Return 401 if invalid
4. Allow request to proceed if valid

## Technical Details

### Files to Create
- `VeefinTokenFilter.java` - Main filter implementation
- `VeefinTokenFilterTest.java` - Unit tests
- `VeefinSecurityIntegrationTest.java` - Integration tests

### Files to Modify
- `SecurityConfiguration.java` - Register filter in Spring Security chain
- `application.yml` - Add Veefin token configuration

### Configuration
```yaml
veefin:
  security:
    token: ${VEEFIN_X_TOKEN:default-token-for-local-testing}
    header-name: veefin-x-token
```

## Success Criteria
- ✅ Filter validates `veefin-x-token` header correctly
- ✅ Returns 401 for missing/invalid tokens
- ✅ Allows valid requests to proceed
- ✅ Filter overhead < 10ms
- ✅ 100% unit test coverage
- ✅ Integration tests pass
- ✅ Spotless code formatting passes
- ✅ Audit logging captures all authentication attempts

## Related Documents
- **Parent Ticket**: [LND-4525](https://bukuwarung.atlassian.net/browse/LND-4525)
- **Parent Plan**: `thoughts/shared/plans/2025-11-07-LND-4525-veefin-borrower-inquiry-api.md`

## Implementation Status
⏳ Implementation pending