# QA Demo Description Template

## Template Structure

When generating QA demo descriptions with `generate-QA-demo-description`, follow this structure:

### 1. Overview Section
Start with:
```
This QA Demo session will validate the end-to-end functionality of [NUMBER] [integration/feature type] recently merged into [service-name]:
  - [Feature 1] - [Brief description] (merged in PR #[NUMBER])
  - [Feature 2] - [Brief description] (merged in PR #[NUMBER])
```

### 2. Scope Section

Each feature should follow this format:

```
# [Feature Name] Testing

Reference: PR #[NUMBER] ([GitHub URL]) - [PR Title]

Test Coverage:
  - **[Category 1]**
    - [Specific test point 1]
    - [Specific test point 2] (include specific values/thresholds when applicable)
  - **[Category 2]**
    - [Specific test point 1]
    - [Specific test point 2]

----
```

### Key Guidelines:

1. **Overview Format**
   - Always start with: "This QA Demo session will validate the end-to-end functionality of..."
   - Count the number of integrations/features being tested
   - Use bullet points with precise descriptions and PR references
   - Format: `- [Name] - [Description] (merged in PR #[NUMBER])`

2. **Scope Section Structure**
   - Use single `#` heading for each feature: `# [Feature Name] Testing`
   - Reference line: `Reference: PR #[NUMBER] ([full GitHub URL]) - [PR Title]`
   - Use "Test Coverage:" (not "Development Coverage:")
   - Use single-level bullet points with bold categories
   - End each feature section with `----` separator

3. **Be Specific with Values**
   - Include exact values, thresholds, or timing requirements
   - Example: "15 seconds before expiration" not "before expiration"
   - Example: "status IN_PROGRESS" not just "status"
   - Use double curly braces for code/table references: `{{merchant_data_enrichment}}`

4. **Focus on Implementation Details**
   - Reference actual class names, method names, DTOs when relevant
   - Example: `{{GetFdcDataCommand}}`, `{{GetFdcDataResponse}}`
   - Use double curly braces for code references: `{{GetFdcDataResponse}}`
   - Reference endpoints: `{{/leads-enrichment}}`

5. **Database/Infrastructure Checks**
   - Specify table names and field names with double curly braces
   - Example: "{{merchant_data_enrichment}} record insertion with status IN_PROGRESS"
   - Example: "{{partner_event_audit}} creation"

6. **Essential Test Categories (adjust per feature):**
   - Authentication & Token Management (if applicable)
   - Credit Search API / API/Endpoint Testing
   - Data Validation & Parsing / Data Retrieval
   - Storage/Persistence (S3, Database) / S3 Document Storage
   - Event Auditing
   - Error Scenarios
   - State Machine Flow (if applicable)

7. **Audit Trail Requirements**
   - Always specify what should be in audit records:
     - request ID, partner type, request/response payloads, partner request/response payloads, status

8. **State Machine Testing Format**
   - Use nested bullet points with `*` and `**`
   - Example: "* Verify {{merchant_data_enrichment}} record insertion with status IN_PROGRESS"
   - Example: "** Test {{/leads-enrichment}} endpoint with valid state transition requests"
   - Mention current implementation state: "(for now it is only {{GetFdcDataCommand}})"

9. **Reference Format**
   - Always include full GitHub PR links in Reference line
   - Format: `Reference: PR #[NUMBER] ([https://github.com/[org]/[repo]/pull/[NUMBER]]) - [Title]`

10. **Special Formatting Notes**
    - Use `----` as section separator between features
    - Use double curly braces for all code/technical references
    - Keep bullet point hierarchy consistent within each feature
    - Use "Verify event ID=" for specific validation points

## Information Gathering Process

When user requests `generate-QA-demo-description`:

1. **Identify PRs/Features**
   - Ask which PRs or features to include
   - Get PR numbers or feature names

2. **Retrieve PR Information**
   - Get PR title, description, and diff
   - Identify changed files and key implementations

3. **Extract Technical Details**
   - Class names, method names, DTOs
   - Database tables and fields
   - API endpoints
   - Configuration values
   - Specific thresholds/timeouts

4. **Map to Test Categories**
   - Identify which standard categories apply
   - Extract specific test points from implementation
   - Include actual code references

5. **Generate Concise Description**
   - Follow template structure
   - Use specific values from code
   - Keep bullet points focused
   - Include all PR references with links
