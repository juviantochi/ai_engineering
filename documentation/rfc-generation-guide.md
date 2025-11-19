# RFC Generation Guide

Use this guide whenever drafting a request for comments (RFC) document to keep proposals consistent, factual, and easy for reviewers to evaluate. The steps are intentionally generic so any domain or repository can follow the same workflow.

## 1. Clarify Scope Before Writing
- Run a short discovery with the requestor or product owner to understand the problem and desired outcome.
- Record the scope boundaries (what is in, what is explicitly out). If scope is uncertain, log the question and pause the draft until it is resolved.
- Avoid guessing. For any unknowns, capture them as open questions or `TBD` items that can be revisited.

## 2. Protect Information Integrity
- Document only what is confirmed by stakeholders, product requirements, or existing systems. Do not invent context or success metrics.
- **Never make assumptions about:**
  - Which systems or services are affected without explicit evidence (code search, configuration review, stakeholder confirmation)
  - Performance metrics (latency, throughput) unless measured or explicitly specified in requirements
  - How existing systems are used (internal vs external callers) without verification through code, logs, or documentation
  - System behavior or data flows without tracing through actual implementation
  - Filter orders or component ordering without checking existing implementations
  - Existing technical debt or planned changes in other services without explicit confirmation
- Highlight assumptions as assumptions; highlight pending inputs as `TBD`.
- Trim outdated or unverified details from earlier drafts to prevent copy-forward errors.
- When uncertain about system behavior, add it to open questions rather than stating it as fact.
- **For technical integrations**: Always verify the current state before claiming what exists:
  - Search codebase for actual filter orders before proposing new orders
  - Check actual API callers before claiming internal vs external usage
  - Verify table migrations actually exist before referencing them
  - Confirm ticket relationships (parent/subtask) before making assumptions

## 3. Handle Secrets and Configuration Safely
- Represent credentials, tokens, and environment-specific values using placeholders such as `${SERVICE_TOKEN}` or `${S3_BUCKET}`.
- Reference the system that owns the real value (e.g., “managed via AWS Secrets Manager”) instead of embedding sensitive data.
- When documenting configuration, clarify default values versus deployment overrides.

## 4. Use the Shared Template
- Start every RFC from `documentation/templates/rfc-template.md` (or your team’s canonical template).
- Update metadata (status, author, ticket id, date) before circulation.
- Leave `TBD` markers rather than removing template sections that are not yet filled; reviewers need to see what remains open.

## 5. Pre-Delivery Checklist
Before sharing the draft:
1. Verify the scope statement matches the request.
2. Ensure diagrams and data flows align with the defined scope.
3. Confirm success metrics and SLAs have a source or are labeled `TBD`.
4. Check that all placeholders follow `${PLACEHOLDER}` format and no secrets slipped into the doc.
5. List open questions or decisions required from stakeholders.
6. **Verify all factual claims**: Every statement about existing systems, performance, or usage patterns must be backed by evidence (code search, logs, metrics, or stakeholder confirmation).
7. **Remove unverified assumptions**: If you cannot prove a claim about how a system works or who uses it, either verify it or remove it from the RFC.

## 6. Communication Expectations
- Surface new questions early; waiting until review may block approvals.
- Capture dependencies (teams, systems, migrations) in their own subsection so owners can confirm feasibility.
- Note assumptions alongside a plan to validate or remove them before implementation.

## 7. General Writing Guidelines
- Prefer declarative language (“Service A calls Service B”) over speculative wording.
- Reconcile related artifacts (API docs, runbooks, dashboards) whenever the RFC redefines their behavior.
- Avoid hard-coding tool names, environments, or policies unless they are part of the requirement; generic language keeps the RFC reusable.
- Use diagrams or sequence descriptions to illustrate flows, even if only textual.

---

## 8. Organizing RFCs in the Repository

### 8.1 Main RFC Catalog
- **Central directory**: `~/Desktop/code/rfc/`
- **Naming convention**: `<ticket-id>-<short-feature-name>.md` 
- **Purpose**: Single source of truth for features that span multiple repositories or services.

### 8.2 Repository-Specific RFCs
When a proposal touches several repositories, create scoped sub-RFCs so each team can focus on its slice of work.

**Directory pattern**
```
<repo-name>/RFC/<ticket-id>-<repo-name>.md
```

**Repository RFC guidelines**
1. Limit content to changes that land in that repository.
2. Mirror the main template sections (motivation, design, rollout, risks) so reviewers see consistent structure.
3. Update metadata to include a `Repository` field and link back to the main RFC.
4. Cross-reference sibling RFCs when there are shared dependencies.
5. Cover repository-specific topics: components, schema changes, configuration, testing, rollout, and risks.
6. Exclude implementation details owned by other repositories (link to their RFC instead).

### 8.3 Suggested Workflow
1. **Draft main RFC** in the central directory capturing the end-to-end architecture.
2. **Identify impacted repositories** and the scope of work per repo.
3. **Generate repository RFCs** under each repo’s `RFC/` folder, pulling only relevant portions of the design.
4. **Maintain consistency**: shared diagrams, terminology, and success metrics should match across all RFCs.

### 8.4 Maintaining RFCs
- Synchronize updates: when the main RFC changes, update each repository RFC (and vice versa).
- Keep API docs, runbooks, and diagrams in sync whenever the design evolves.
- Preserve version history (e.g., via Git) so previous decisions remain traceable.

---

## 9. Integrating RFCs with API Documentation

### 9.1 When to Touch API Docs
- After RFC approval, before implementation starts.
- Whenever the RFC changes the contract, validation, or error semantics.
- As part of release readiness to ensure published docs match the built system.

### 9.2 Documentation Locations
- **Local markdown**: `/Users/juvianto.chi/Desktop/code/documentation/apidocs/`
- **Templates**: `/Users/juvianto.chi/Desktop/code/documentation/templates/`
- **Knowledge base**: Publish to the org’s documentation platform (e.g., Confluence) once finalized.

### 9.3 Formatting Practices
- Preserve Markdown for repo copies; follow platform-specific markup (e.g., Confluence storage format) when publishing externally.
- Provide request/response examples with placeholder secrets.
- Include an error catalogue, validation rules, and changelog sections.
- Use expandable sections or accordions where supported to keep documents scannable.

### 9.4 Publishing Tips
1. Convert Markdown to the destination format when needed.
2. Link API docs back to the RFC for traceability.
3. Apply consistent labelling/tagging so stakeholders can find documents easily.
4. Remove environment-specific SLAs from shared docs unless they are universally true.

---

## 10. Example Workflow (Generic)

1. Kick off discovery, gather requirements, and clarify scope.
2. Draft the main RFC using the template, capturing architecture, data flow, risks, and open questions.
3. Create repository-specific RFCs where multiple services are involved.
4. Review internally, address feedback, and lock the RFC once stakeholders approve.
5. Update API documentation, runbooks, and other downstream artifacts to reflect the agreed design.
6. Track implementation progress against the RFC and update the document if decisions change.

---

Following this guide keeps RFCs actionable, traceable, and reusable for future initiatives regardless of domain or technology stack.
