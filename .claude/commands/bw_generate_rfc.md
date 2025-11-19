# Generate RFC Command

You are now in RFC generation mode. Your goal is to help create a high-quality Request for Comments (RFC) document following established patterns and best practices.

## Step 1: Discovery and Clarification

Before writing any RFC content, gather essential information:

### Required Information
Ask the user to clarify:

1. **RFC Type**:
   - Is this a **repository-specific RFC** (changes in one repo only)?
   - Or a **cross-service RFC** (multiple repos/services)?

2. **Basic Metadata**:
   - Jira ticket ID (e.g., LND-4526)
   - Repository name(s) impacted
   - Brief description (1-2 sentences)

3. **Scope Boundaries**:
   - What is IN scope for this RFC?
   - What is explicitly OUT of scope?
   - Is there a parent RFC (for repository-specific RFCs)?

4. **Available Context**:
   - Are there existing implementations to reference?
   - Are there related tickets, PRDs, or documentation?
   - Are there external API docs or specifications?

### Discovery Questions

Ask targeted questions to understand:
- **Problem**: What problem does this solve?
- **Users**: Who benefits from this change?
- **Constraints**: Any technical constraints, deadlines, or dependencies?
- **Existing Systems**: What already exists that can be reused?

**CRITICAL**: If the user provides vague requirements, **STOP** and ask for clarification. Do NOT make assumptions about:
- System behavior without verification
- Performance metrics without measurements
- Integration points without confirmation
- User workflows without validation

## Step 2: Verification Phase

Before drafting content, verify all factual claims:

### Codebase Verification
- [ ] Search for existing implementations (Grep, Glob)
- [ ] Verify component locations and file paths
- [ ] Check existing filter orders, configurations, dependencies
- [ ] Confirm actual API usage patterns (internal vs external callers)

### Documentation Verification
- [ ] Read related RFCs and design docs
- [ ] Verify external API specifications if applicable
- [ ] Check existing architectural patterns
- [ ] Confirm ticket relationships (parent/subtask)

### Infrastructure Verification
For technical integrations:
- [ ] Verify current filter orders before proposing new ones
- [ ] Check actual table migrations before referencing them
- [ ] Confirm service dependencies and integration points
- [ ] Validate configuration patterns in existing code

**NEVER HALLUCINATE**: If you cannot verify a claim:
1. Mark it as `TBD` or `[Requires verification]`
2. Add to "Open Questions" section
3. Do NOT state it as fact

## Step 3: Select Template and Structure

Based on RFC type, use the appropriate template from `.claude/templates/rfc-template.md`:

### Repository-Specific RFC Structure
**Focus**: Implementation details for one repository

**Include**:
-  Scope section (clear boundaries)
-  Component Design (with code examples)
-  Scalability & Extensibility (if applicable)
-  Configuration Management (local/staging/prod)
-  Technical Decisions (with rationale)
-  Success Criteria (technical/functional/performance)

**Brief or Reference Parent RFC**:
-   Motivation (link to parent RFC for details)
-   User Benefit (reference parent RFC)
-   Rollout Plan (may be in parent RFC)

### Cross-Service RFC Structure
**Focus**: Business context and architecture

**Include**:
-  Comprehensive Motivation
-  Detailed User Benefits
-  High-Level Architecture (cross-service)
-  Data Flow (end-to-end)
-  Rollout & Migration Plan
-  Monitoring & Alerting
-  Risks & Mitigations

**Implementation Details**:
-   Reference repository-specific RFCs for detailed component design

## Step 4: Content Generation Guidelines

### Writing Standards

**Clarity**:
- Use concrete examples over abstract descriptions
- Include code snippets for technical components
- Specify file paths and locations
- Use tables for comparisons and options

**Accuracy**:
- Only document verified information
- Use placeholders for secrets: `${SECRET_NAME}`
- Mark assumptions as assumptions
- Label `TBD` items explicitly

**Completeness**:
- Address all template sections relevant to RFC type
- Include "What's Out of Scope" to set boundaries
- Document technical decisions with rationale
- Provide configuration examples for all environments

**Evidence-Based**:
- Reference existing code/patterns
- Cite external documentation
- Link to related tickets/RFCs
- Provide file paths and line numbers where applicable

### Component Design Pattern

For each major component, include:

```markdown
#### <Component Name>

**Purpose**: <What problem does it solve?>

**Location**: `<exact-file-path>`

**Design**:
```<language>
<code snippet showing key structure>
```

**Key Features**:
- <Feature #1>
- <Feature #2>

**Responsibilities**:
- <Responsibility #1>
- <Responsibility #2>
```

### Technical Decisions Pattern

For each major decision, document:

```markdown
### Decision: <Decision Title>

**Question**: Why <chose option A over option B>?

**Justification**:
-  <Benefit #1>
-  <Benefit #2>
-   <Trade-off if any>

**Performance Impact**: <Metrics if applicable>

**Alternatives Rejected**: <Why other options weren't chosen>
```

### Scalability Pattern (if applicable)

If designing reusable infrastructure:

```markdown
## Scalability & Extensibility

### Adding New <Partner/Client/Module>

**Step 1**: <Action with config example>
**Step 2**: <Action with code example>
**Step 3**: <Verification steps>

**Zero changes to**:
-  <Existing component 1>
-  <Existing component 2>

### Reusability Matrix

| Component | Reusable | Partner-Specific |
|-----------|----------|------------------|
| Base Class |  | L |
| Concrete Impl | L |  |
```

## Step 5: Quality Gates

Before finalizing the RFC, verify:

### Completeness Check
- [ ] All required sections from template included or marked N/A
- [ ] Scope clearly defined (in-scope and out-of-scope)
- [ ] Success criteria specific and measurable
- [ ] Configuration examples for all environments
- [ ] Security considerations addressed

### Accuracy Check
- [ ] All file paths verified to exist or marked as "new"
- [ ] No hallucinated APIs or behaviors
- [ ] External references properly cited
- [ ] Code examples syntactically correct
- [ ] No secrets or credentials in examples

### Clarity Check
- [ ] Technical decisions documented with rationale
- [ ] Alternatives considered and rejected with reasons
- [ ] Tables used for comparisons and options
- [ ] Diagrams or ASCII art for complex flows
- [ ] Consistent terminology throughout

### Security Check
- [ ] All secrets use `${PLACEHOLDER}` format
- [ ] Secret management strategy documented
- [ ] Authentication/authorization addressed
- [ ] Data privacy considerations included
- [ ] No PII or sensitive data in examples

## Step 6: Output and Next Steps

### Generate RFC Document

1. Create the RFC at appropriate location:
   - Repository-specific: `<repo-name>/RFC/<TICKET-ID>-<repo-name>.md`
   - Cross-service: `~/Desktop/code/rfc/<TICKET-ID>-<feature-name>.md`

2. Use the template from `.claude/templates/rfc-template.md`

3. Fill all sections based on verified information

4. Mark any `TBD` items in "Open Questions" section

### Confluence Publishing (Optional)

If user requests Confluence publishing:

1. Ask which Confluence space (e.g., "Tech", "Juvianto Chi personal space")

2. Use `mcp__atlassian__createConfluencePage` with:
   - Markdown body (will be converted automatically)
   - Appropriate space ID
   - Title: `RFC: <TICKET-ID> - <Title>`

3. Provide Confluence URL after successful creation

### Post-Generation Checklist

Present to user:

**RFC Generated**: `<file-path>`

**Next Steps**:
1. Review RFC for accuracy and completeness
2. Verify all `TBD` items and open questions
3. Share with stakeholders for feedback
4. Update status as it progresses (Draft ’ Proposed ’ Accepted)
5. Create implementation tasks from RFC

**Outstanding Items**:
- List any `TBD` items that need resolution
- List any open questions requiring external input
- List any verification tasks that couldn't be completed

## Guidelines Reference

This command follows the guidelines from:
- `documentation/rfc-generation-guide.md` - Process and best practices
- `.claude/templates/rfc-template.md` - Structure and sections
- `CLAUDE.md` (project) - Backend engineering principles

## Example Invocations

### Example 1: Repository-Specific RFC
```
User: /generate_rfc
Assistant: I'll help you generate an RFC. Let me start with some discovery questions:

1. Is this a repository-specific RFC (one repo) or cross-service RFC?
2. What's the Jira ticket ID?
3. Which repository will be impacted?
4. Can you provide a brief description (1-2 sentences)?

[Continues with discovery phase...]
```

### Example 2: With Context Provided
```
User: /generate_rfc for LND-4526, authentication filter in fs-brick-service
Assistant: Great! I'll generate a repository-specific RFC for fs-brick-service.

Let me verify the existing codebase first:
[Uses Grep/Glob to find existing filters, configurations]

Based on what I found, I'll now draft the RFC...
[Proceeds with RFC generation]
```

## Important Reminders

### DO:
 Ask clarifying questions when requirements are vague
 Verify all factual claims through code search or documentation
 Mark unverified items as `TBD` or `[Requires verification]`
 Use placeholders for all secrets and credentials
 Include code examples for technical components
 Document technical decisions with rationale
 Specify exact file paths and locations
 Create tables for comparisons and options

### DON'T:
L Hallucinate APIs, behaviors, or system details
L Make assumptions about performance without measurements
L Claim things exist without verification
L Include actual secrets or credentials
L Skip "Out of Scope" section (prevent scope creep)
L Leave open questions untracked
L Forget to document alternatives considered
L Use vague success criteria without metrics

---

**You are now in RFC generation mode. Start with Step 1: Discovery and Clarification.**