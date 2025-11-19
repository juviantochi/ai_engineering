Rules for Generating Good Jira Ticket Descriptions

🎯 Core Structure Requirements

MUST HAVE:
1. Definition of Done (DoD) / Acceptance Criteria
   - Clear, measurable criteria for completion
   - Specific and testable conditions
   - Helps engineers estimate story points accurately
2. Pre-requisites
   - Dependencies on other tickets/systems
   - Required access, permissions, or resources
   - Technical dependencies or setup needed
3. Journey/Flow Details
   - Step-by-step user or system flow
   - Process sequence from start to finish
   - Integration points between systems
4. Scenarios & Edge Cases
   - Happy path scenario
   - Error scenarios
   - Edge cases and boundary conditions
   - What happens when things go wrong

📋 Content Quality Guidelines

DO:
- ✅ Keep description clean and readable (use formatting, bullets, sections)
- ✅ Include high-level implementation idea
- ✅ Clearly specify technical platform/service to be modified
- ✅ Show current state vs expected state
- ✅ Be straight to the point, avoid verbosity
- ✅ Include "Out of Scope" section to set boundaries
- ✅ Provide sufficient context for engineering funnel (RFC, implementation)

DON'T:
- ❌ Include only user stories without technical context
- ❌ Add too much irrelevant detail
- ❌ Leave technical side vague or unclear
- ❌ Omit which platform/service needs adjustment
- ❌ Skip Definition of Done
- ❌ Be verbose or rambling

📝 Template Structure

## Summary
[One-line description of what needs to be done]

## Current State
[What exists today / current behavior]

## Expected State
[What should exist / expected behavior after implementation]

## Pre-requisites
- [ ] Dependency 1
- [ ] Required access/setup
- [ ] Technical dependency

## Implementation Journey
1. Step 1: [Action]
2. Step 2: [Action]
3. Step 3: [Integration point]

## Scenarios & Edge Cases
### Happy Path
- [Normal flow description]

### Edge Cases
- Case 1: [Description]
- Case 2: [Description]

### Error Scenarios
- Error 1: [What happens and expected handling]

## Technical Details
- **Platform/Service**: [Specific service name]
- **Endpoints/Components**: [What needs modification]
- **Integration Points**: [External systems involved]

## Out of Scope
- Item 1 that will NOT be included
- Item 2 that is future work

## Definition of Done (Acceptance Criteria)
- [ ] Criterion 1 (testable)
- [ ] Criterion 2 (measurable)
- [ ] Criterion 3 (specific)
- [ ] Tests passing
- [ ] Documentation updated

⚖️ Balance Principles

Technical Clarity vs Verbosity:
- Provide enough technical detail to answer "what" and "which platform"
- Avoid over-explaining obvious steps
- Focus on critical information engineers need

Context vs Noise:
- Include context necessary for RFC/implementation
- Remove details not relevant to implementation
- Link to PRD if more context is needed rather than duplicating

User Stories vs Technical Requirements:
- User stories alone are insufficient
- Must include technical implementation context
- Balance business value with engineering actionability

🎖️ Quality Rating Alignment

The description quality should be inline with the given rating (story points):
- Small tasks (1-3 SP): Less detail needed, but DoD still required
- Medium tasks (5-8 SP): Full template with all sections
- Large tasks (13+ SP): Comprehensive detail with multiple scenarios, may need to be broken down

✅ Validation Checklist

Before finalizing, verify:
- Can an engineer estimate story points from this description?
- Is it clear which platform/service needs changes?
- Does it have DoD/Acceptance Criteria?
- Are pre-requisites listed?
- Is the journey/flow clear?
- Are edge cases considered?
- Is current vs expected state clear?
- Is it concise yet sufficient for RFC creation?

  ---
Key Principle: A good Jira description enables engineers to start RFC/implementation immediately without needing to ask clarifying questions about scope, platform, or acceptance
criteria.