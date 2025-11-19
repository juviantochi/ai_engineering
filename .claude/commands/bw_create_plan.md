# Implementation Planning Task Instructions

## Purpose
Create a detailed implementation plan before starting any coding work, allowing for review and refinement before execution.

## Quick Start

When you receive a planning request:
check if any reference are given.
1. **Fetch Jira ticket** - Get all details from the ticket
2. **Ask clarifying questions** - If ANYTHING is unclear or ambiguous, STOP and ask the user:
    - Missing acceptance criteria details
    - Unclear business requirements
    - Ambiguous technical specifications
    - Unknown integration points
    - Unclear error handling expectations
3. **Analyze codebase** - Find similar patterns
4. **Create plan** - Generate implementation plan in `plans/` directory
5. **Save to file** - Use format: `plans/{JIRA-NUM}-{feature-name}.md`

**Example filename**: `plans/BUKU-11362-app-version-routing.md`

## Step-by-Step Process

### Step 1: Fetch Jira Ticket
```
1. Get Jira ticket via MCP tool
2. Read description and acceptance criteria
3. Check linked tickets/dependencies
```

**→ If ANY requirement is unclear: STOP and ASK USER**

---

### Step 2: Analyze Current Codebase

**Find similar implementations:**

```bash
# Search for similar features
Grep: "pattern-to-search" --type java

# Find similar controllers
Glob: **/controller/**/*Controller.java

# Find similar services/utilities
Glob: **/service/**/*Service.java
Glob: **/util/**/*Utils.java

# Find similar configs
Glob: **/config/**/*Config.java
```

**Identify patterns to reuse:**
- How are similar features implemented?
- What naming conventions are used?
- What package structure is followed?
- How are similar features tested?

**→ If unsure about patterns: ASK USER**

---

### Step 3: Identify External Dependencies

**Check what the feature needs to call:**

#### HTTP Calls
```bash
# Find existing HTTP clients
Glob: **/client/**/*Client.java
Grep: "RestTemplate|WebClient|HttpClient" --type java

# Check which external services are called
```

**Questions to answer:**
- Which external API endpoints will be called?
- What are the timeout requirements?
- How to handle failures? (retry, circuit breaker)
- Is there existing client code to reuse?

#### Database Calls
```bash
# Find existing repositories
Glob: **/repository/**/*Repository.java

# Check existing entities
Glob: **/entity/**/*Entity.java

# Review migration files
Glob: **/db/migration/**/*.sql
```

**Questions to answer:**
- Which tables need to be created/modified?
- What indexes are needed?
- Are there existing entities to reuse?
- Is backward compatibility needed?

#### Message Queues (if applicable)
```bash
# Find queue producers/consumers
Grep: "@RabbitListener|@KafkaListener" --type java
```

#### Other Integrations
- Redis/Cache calls
- S3/File storage
- Third-party services

**Document all external dependencies:**
```markdown
### External Dependencies
1. **HTTP**: External Service X
   - Endpoint: POST /api/v1/endpoint
   - Timeout: 30s
   - Existing client: `ExternalServiceClient.java` ✅

2. **Database**: transactions table
   - New table or modify existing?
   - Migration needed: ✅

3. **Cache**: Redis (if needed)
   - Key pattern: feature:id
   - TTL: 5 minutes
```

**→ If any external dependency is unclear: ASK USER**

---

### Step 4: Identify Unclear Requirements & Ask Questions

**IMPORTANT**: Stop and ask about:

**Business Logic:**
- ❓ Unclear business rules or calculations
- ❓ Missing edge case handling
- ❓ Unclear validation rules

**Technical Specifications:**
- ❓ Which external service endpoint to call?
- ❓ What's the timeout/retry strategy?
- ❓ How to handle external service failures?
- ❓ Database changes needed?
- ❓ Backward compatibility required?

**Security & Performance:**
- ❓ Authentication/authorization required?
- ❓ Expected load/throughput?
- ❓ Caching needed?
- ❓ PII data handling?

**Configuration:**
- ❓ Should this be configurable per environment?
- ❓ Default values for configs?
- ❓ Feature flags needed?

**Example questions:**
- "Should we support backward compatibility with version X?"
- "What should happen if the external service times out?"
- "Should this be configurable per environment?"
- "What's the expected load/throughput?"
- "Which external API endpoint should we call for this?"
- "Do we need a new database table or modify existing one?"

**→ ALWAYS ask if unsure. Never assume!**

---

### Step 5: Create Plan Document

**Location**: `plans/{JIRA-NUM}-{short-feature-name}.md`

**Filename examples:**
- ✅ `plans/BUKU-11362-app-version-routing.md`
- ✅ `plans/BUKU-11500-payment-timeout.md`
- ❌ `plans/add-feature.md` (missing JIRA number)

**Filename rules:**
- Start with JIRA ticket number
- 2-4 words describing feature
- Use kebab-case
- Keep it short and clear

## Plan Document Template

Save to: `plans/{JIRA-NUM}-{feature-name}.md`

```markdown
# {JIRA-NUM}: {Feature Title}

**Jira**: https://bukuwarung.atlassian.net/browse/{JIRA-NUM}
**Created**: {Date}
**Status**: 📝 Planning

---

## What We're Building

{1-2 sentence summary of the feature}

## Requirements from Ticket

**Key Requirements:**
- Requirement 1
- Requirement 2

**Acceptance Criteria:**
- [ ] AC 1
- [ ] AC 2

**Dependencies/Blockers:**
- None OR list blockers

---

## Codebase Analysis

### Similar Implementations Found
- **File**: `path/to/similar/Example.java:123`
  - Pattern to reuse: {describe pattern}
  - Why similar: {explain similarity}

**Example:**
- **File**: `adapters/api/src/.../controller/BalanceController.java:45`
  - Pattern: Header extraction and validation
  - Reuse: Header parsing logic

### Affected Modules
- `common/` - Add AppVersionConfig and AppVersionUtils
- `adapters/api/` - Modify 6 controllers to capture header
- `adapters/persistence/` - No changes
- `core/` - No changes

---

## External Dependencies

### HTTP Calls (if needed)
**None** OR list:

1. **Service Name**: {e.g., Payment Gateway}
   - **Endpoint**: `POST https://api.external.com/v1/endpoint`
   - **Timeout**: 30 seconds
   - **Retry**: 3 attempts with exponential backoff
   - **Circuit Breaker**: Yes/No
   - **Error Handling**: Return error code XXX
   - **Existing Client**: `ExternalServiceClient.java` OR "Need to create new client"
   - **Auth Required**: API Key in header

### Database Calls

**Tables to Create:**
- None OR list new tables

**Tables to Modify:**
- `transactions` - Add column `app_version_code VARCHAR(50)`
  - Index: Add index on `app_version_code`
  - Default value: NULL for backward compatibility

**Existing Entities to Reuse:**
- `Transaction.java` - Add field `appVersionCode`

**Migration File:**
- `V2025010812000000__add_app_version_to_transactions.sql`

### Cache/Redis (if needed)
**None** OR:
- **Key Pattern**: `feature:{id}`
- **TTL**: 5 minutes
- **Invalidation**: On update/delete

### Message Queue (if needed)
**None** OR:
- **Topic/Queue**: `feature-events`
- **Message Format**: JSON
- **Consumer**: `FeatureEventConsumer.java`

### Third-Party Libraries (if new dependencies needed)
**None** OR:
```gradle
implementation 'group:artifact:version'  // Purpose
```

---

## Implementation Plan

### 1. Configuration (if needed)
**File**: `common/src/main/.../config/FeatureConfig.java`
```java
@Configuration
public class FeatureConfig {
    @Value("${config.property:defaultValue}")
    private String property;
}
```

**Environment Variables:**
```properties
CONFIG_PROPERTY=value  # Description
```

### 2. Domain Layer (if needed)
**Location**: `core/src/main/java/.../domain/`

**Create/Modify:**
- `Entity.java` - {purpose}
- `ValueObject.java` - {purpose}
- `DomainService.java` - {purpose}

**Business Rules:**
1. Rule description
2. Rule description

### 3. Application Layer (if needed)
**Location**: `core/src/main/java/.../application/`

**Create/Modify:**
- `UseCase.java` - {purpose}
- `RequestDTO.java` - {purpose}
- `ResponseDTO.java` - {purpose}

### 4. Adapter Layer

**REST Controllers** (`adapters/api/src/main/java/.../controller/`)
- `FeatureController.java`
    - Endpoint: `POST /api/v1/feature`
    - Request: `{json example}`
    - Response: `{json example}`

**Repositories** (`adapters/persistence/src/main/java/.../repository/`)
- `FeatureRepository.java` - {purpose}

**Database Migration** (if needed)
- File: `adapters/persistence/src/main/resources/db/migration/V{YYYYMMDDHHMMSS}__{description}.sql`
- Changes: Create/modify tables, add indexes

**External Integrations** (if needed)
- `ExternalServiceClient.java` - {purpose}

---

## Step-by-Step Implementation

**Follow this order:**

### Phase 1: Foundation ⚙️
- [ ] Create configuration class with env variables
- [ ] Create database migration (if needed)
- [ ] Run migration: `./gradlew flywayMigrate` (use gradlew for database tasks)
- [ ] Verify DB changes

### Phase 2: Domain Logic 🧠
- [ ] Create/modify domain entities
- [ ] Implement business rules
- [ ] Write domain unit tests
- [ ] Run tests: `make test`

### Phase 3: Application Layer 📦
- [ ] Create use cases
- [ ] Create DTOs and mappers
- [ ] Implement validation
- [ ] Write application unit tests

### Phase 4: Adapters 🔌
- [ ] Create/modify controllers
- [ ] Create/modify repositories
- [ ] Implement external integrations (if any)
- [ ] Add logging with correlation IDs

### Phase 5: Testing ✅
- [ ] Write integration tests
- [ ] Write controller tests with MockMvc
- [ ] Test error scenarios
- [ ] Manual testing: happy path
- [ ] Manual testing: edge cases

### Phase 6: Quality Checks 🎯
- [ ] Format code: `make format`
- [ ] Run all tests: `make test`
- [ ] Check coverage: `make coverage`
- [ ] Run quality checks: `make check`

---

## Testing Strategy

### Unit Tests
**Location**: `src/test/java/` (mirror package structure)

**Key scenarios:**
- Happy path with valid inputs
- Invalid inputs and validation
- Edge cases and boundary conditions
- Error handling

**Mock strategy:**
- Mock external dependencies (repositories, HTTP clients)
- Use real domain logic

### Integration Tests
**Annotation**: `@SpringBootTest`

**Key scenarios:**
- End-to-end API flow
- Database integration
- External service integration (with Testcontainers/mocks)

**Test data:**
- Use dedicated test accounts/IDs
- Clean up after each test

---

## Configuration

**New Environment Variables:**
```bash
# Feature toggle (if needed)
FEATURE_ENABLED=true

# Feature-specific config
FEATURE_TIMEOUT_MS=30000
FEATURE_MAX_RETRIES=3
```

**Add to `.env.example`**

---

## Security & Performance

### Security
- [ ] Input validation on all endpoints
- [ ] Authentication required: {Yes/No}
- [ ] Authorization level: {e.g., USER, ADMIN}
- [ ] No PII in logs
- [ ] Sanitize error messages

### Performance
- **Expected load**: {e.g., 100 req/sec}
- **Timeout**: {e.g., 30 seconds}
- **Caching**: {needed? strategy?}
- **Database indexes**: {list if needed}

---

## Risks & Questions

### Risks
1. **Risk**: {description}
    - **Mitigation**: {how to handle}

### Open Questions
- ❓ Question 1?
- ❓ Question 2?

**→ Ask user for clarification on these before proceeding**

---

## Estimated Effort

- **Complexity**: Low | Medium | High
- **Estimated Time**: {X} days
- **Dependencies**: {list or "None"}

---

## References

- **Similar code**: `path/to/reference/file.java:123`
- **Related ticket**: JIRA-XXX
- **Documentation**: {link if applicable}

---

**Next Steps:**
1. Review this plan
2. Answer open questions
3. Update status to "✅ Approved"
4. Start implementation
```

---

## Important Guidelines

### Always Ask Questions First! ❓

**Before creating the plan, if you encounter:**
- Unclear requirements → **ASK USER**
- Ambiguous acceptance criteria → **ASK USER**
- Unknown business rules → **ASK USER**
- Missing technical details → **ASK USER**
- Unclear error handling → **ASK USER**

**Don't assume or guess!**

### Writing the Plan

**Be Specific:**
- ✅ `common/src/main/java/com/bukuwarung/edc/config/AppVersionConfig.java`
- ❌ "Create a config file somewhere"

**Use Concrete Examples:**
- ✅ Show actual code snippets
- ✅ Show exact file paths
- ✅ Show example JSON requests/responses

**Reference Existing Code:**
- ✅ `adapters/api/src/main/java/.../BalanceController.java:45`
- Use line numbers when possible

**Break Down Tasks:**
- Each checkbox should be ~30 min to 2 hours of work
- If bigger, break it down further

### Checklist Before Finishing Plan

- [ ] All unclear requirements have been clarified with user
- [ ] File paths are specific and accurate
- [ ] Code examples are included
- [ ] Similar implementations are referenced
- [ ] Risks are identified
- [ ] Backward compatibility is considered
- [ ] Security implications are noted
- [ ] Performance impact is considered

---

## Workflow

**Complete Process:**

1. **User requests `/plan {JIRA-NUM}`**

2. **Fetch Jira ticket** → Get requirements
   - If unclear → STOP and ASK USER

3. **Analyze codebase** → Find similar patterns
   ```bash
   # Search for similar implementations
   Glob/Grep to find existing code
   ```
- If unsure about patterns → ASK USER

4. **Identify external dependencies**
    - HTTP calls needed?
    - Database changes?
    - Cache/Queue/Third-party?
    - Check existing clients/repositories
    - If unclear which endpoint/table → ASK USER

5. **Ask clarifying questions** for any:
    - Unclear business logic
    - Missing technical specs
    - Unknown external endpoints
    - Unclear error handling

6. **Generate plan** with:
    - Similar implementations referenced
    - External dependencies documented
    - Specific file paths
    - Concrete examples

7. **Save to** `plans/{JIRA-NUM}-{feature}.md`

8. **User reviews** and approves

9. **Update status** to "✅ Approved"

10. **Start coding** using `/code {JIRA-NUM}`

---

## Examples

### Good Plan File Path
✅ `plans/BUKU-11362-app-version-routing.md`
✅ `plans/BUKU-11500-payment-timeout-handling.md`

### Bad Plan File Path
❌ `plans/add-version-check.md` (missing JIRA number)
❌ `plans/BUKU-11362-add-app-version-check-to-route-requests-to-nobu-or-aj-based-on-threshold.md` (too long)

### Good Questions to Ask User
✅ "Should the version threshold be configurable per environment or hardcoded?"
✅ "What should happen if the version header is malformed? Return error or default to legacy?"
✅ "Do we need to store the version in transaction records for auditing?"

### Bad Assumptions (Don't Do This!)
❌ "I'll assume we need to support JSON and XML formats" (ask first!)
❌ "Probably needs authentication" (confirm first!)
❌ "I'll add caching since it might be useful" (verify requirement!)