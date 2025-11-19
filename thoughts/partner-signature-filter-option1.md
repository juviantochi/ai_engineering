# Shared Partner Signature Filter – Option 1

Context: Instead of having one Spring Security filter per partner (@Order(3) for Veefin, @Order(4) for the next partner, etc.), we build a **single configurable filter** that can authenticate any partner hitting the shared `/fs/brick/service/**` endpoints. Below is the concrete plan.

## Filter Flow
1. Filter bean: `@Order(3)` `PartnerSignatureAuthFilter extends OncePerRequestFilter`.
2. `shouldNotFilter(HttpServletRequest request)` checks whether the request path matches any entry inside `partnerSignatureConfig.getAllPatterns()` (Ant match like `/fs/brick/service/**`). If not, skip entirely.
3. Inside `doFilterInternal`:
   - Read the neutral headers:
     * `x-partner-id`
     * `x-request-id`
     * `x-timestamp`
     * `x-signature`
   - Reject with 401 if any are missing.
   - Look up the partner using `x-partner-id`. If the id is unknown, 401.
   - Validate timestamp format and tolerance using the partner config (default 5 minutes, override per partner).
   - Validate request-id constraints (length, allowed charset).
   - Build the signing payload, e.g. `sprintf(\"%s|%s|%s|%s|%s\", partnerId, requestId, timestamp, request.getMethod(), request.getRequestURI())`.
   - Call `SignatureUtils.verifyHmac(signature, partnerSecret, payload)` and reject if it fails.
   - On success, optionally add `PartnerContext.setCurrentPartner(partnerId)` for downstream logging, then continue the chain.

## Configuration Layout
```yaml
partner-signature:
  defaultTimestampToleranceMinutes: 5
  headerNames:
    partnerId: x-partner-id
    requestId: x-request-id
    timestamp: x-timestamp
    signature: x-signature
  partners:
    veefin:
      secret: ${VEEFIN_SIGNATURE_SECRET}
      timestampToleranceMinutes: 5
      pathPatterns:
        - /fs/brick/service/veefin/**
    lenderX:
      secret: ${LENDERX_SIGNATURE_SECRET}
      timestampToleranceMinutes: 3
      pathPatterns:
        - /fs/brick/service/veefin/**
        - /fs/brick/service/gt/**
```
- For production, the `secret` entries are pulled from AWS Secrets Manager (same pattern as the RFC).
- Adding a new partner = add a new block under `partners` and redeploy.

## Helper Components
- `PartnerSignatureConfig` – loads the YAML to `Map<String, PartnerConfig>`.
- `PartnerConfig` – holds `secret`, `timestampToleranceMinutes`, `List<String> pathPatterns`, optional `headerPrefix`.
- `SignatureUtils` – already defined in RFC (HMAC SHA-256 + constant time comparison).
- `PartnerContextHolder` (optional) – stores the current partnerId in a `ThreadLocal` so controllers/services can log it.

## Edge Cases
- **Missing partner-id**: return `401 {"code":"UNAUTHORIZED","message":"Missing x-partner-id header"}`.
- **Unknown partner-id**: same 401 with `"Unknown partner id"`.
- **Path mismatch**: `shouldNotFilter` returns true, so controllers not under `/fs/brick/service/**` remain untouched.
- **Replay**: timestamp enforcement per partner. We can also optionally keep a `ConcurrentHashMap<partnerId, requestId>` to reject duplicates within the tolerance window.
- **Per-partner header prefixes**: add `headerPrefix` in config if we ever need `veefin-partner-id` etc.; the filter builds header names dynamically.

## Benefits
- One code path; any security bug fix applies to every partner.
- No risk that a Veefin-only filter blocks other partners.
- Truly configuration-driven: secrets, tolerances, and paths live in YAML/Secrets Manager.
- Easy monitoring: logs can include `partnerId` from the shared filter for both success and failure.
