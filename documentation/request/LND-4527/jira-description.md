# LND-4527: Create API Documentation for Borrowers Data Inquiry Endpoint

**Tickets:** LND-4527  
**Parent:** LND-4525

## Context

We need to create formal API documentation for a new partner integration endpoint that will allow Veefin to query borrower credit information using KTP numbers. This documentation will serve as the contract between our team and Veefin before implementation begins.

## Scope

Create a complete API documentation file at `documentation/apidocs/borrowers-data-inquiry-api-docs.md` that includes:

### 1. Overview Section
- Service name and ownership
- Intended audience (Veefin)
- Version and update tracking

### 2. Authentication & Authorization
- Custom header scheme (`x-brick-token`)
- IP whitelisting requirements

### 3. Environment Details
- Staging and Production base URLs
- Authentication notes per environment
- Support channels and SLAs

### 4. Endpoint Specification
- Full endpoint path: `POST /fs/brick/service/veefin/v1/inquiry-borrowers`
- Business description and use case
- Prerequisites

### 5. Request/Response Contract
- Request payload structure (ktpNumber, requestId)
- Success response format with credit metrics:
  - `jmlFasilitas` (total facilities)
  - `jmlHariTunggakanTerburuk` (worst overdue days)
  - `jmlSaldoTerutang` (total outstanding balance)
- Complete error handling with HTTP codes (200, 400, 401, 404, 500)

### 6. Data Validation Rules
- KTP number format requirements (16 digits)
- Request ID format (UUID v4)

### 7. Integration Notes
- Idempotency behavior
- Security considerations
- PII handling compliance

### 8. Change Log
- Version tracking for future updates

## Acceptance Criteria

- [ ] Documentation file created at specified path
- [ ] All sections complete and reviewed
- [ ] Request/response examples are valid JSON
- [ ] Error scenarios comprehensively documented
- [ ] Validation rules clearly specified
- [ ] Document reviewed and approved by Tech Lead
- [ ] Document shared with Veefin for review and sign-off

## Dependencies

- None (this is a documentation-only task)

## Notes

- This documentation must be approved before LND-4528 (API implementation) begins
- The API contract defined here will be the source of truth for implementation
- Version v1.0.0 marks the initial contract definition
