# Generate QA Demo Description Command

## Command
When user says: `generate-QA-demo-description`
## Ask for
1. Jira ticket number for story/bug (if available)
2. Added information about features/PRs to include in the demo

## Behavior
Generate a QA demo ticket description following the template defined in `/templates/qa-demo-description-template.md`

## Process

### Step 1: Gather Information
Ask the user:
1. What features/PRs should be included in the QA demo?
2. What is the Jira ticket number (if available)?

### Step 2: Retrieve Implementation Details
For each PR/feature:
1. Get PR details (title, description, diff)
2. Identify key files changed
3. Extract technical details:
    - Class/method names
    - Database tables/fields
    - API endpoints
    - Configuration keys
    - Specific values/thresholds
    - DTO/model names

### Step 3: Generate Description
Follow the template structure exactly:
1. **Overview Section**:
   - Start with "This QA Demo session will validate the end-to-end functionality of [NUMBER] [type] recently merged into [service-name]:"
   - List features with bullet points: `- [Name] - [Description] (merged in PR #[NUMBER])`
2. **Scope Section** with:
   - Single `#` heading: `# [Feature Name] Testing`
   - Reference line: `Reference: PR #[NUMBER] ([full GitHub URL]) - [PR Title]`
   - "Test Coverage:" (not "Development Coverage:")
   - Categories with bullet points and specific test points
   - End with `----` separator between features

### Step 4: Output
Provide the generated description in a code block ready to paste into Jira

## Key Principles
- **Specific over generic**: Use actual class names, values, thresholds
- **Concise over elaborate**: Action-oriented, no fluff
- **Implementation-focused**: Reference real code artifacts
- **Minimal sections**: Only Overview and Scope, no extra sections
- **Database awareness**: Include table names and field names
- **Audit completeness**: Always specify what audit records should contain