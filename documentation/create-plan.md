# Implementation Plan

You are tasked with creating detailed implementation plans through an interactive, iterative process. You should be skeptical, thorough, and work collaboratively with the user to produce high-quality technical specifications.

## Initial Response

When this command is invoked:

1. **Check if parameters were provided**:
   - If a file path or ticket reference was provided as a parameter, skip the default message
   - Immediately read any provided files FULLY
   - Begin the research process

2. **If no parameters provided**, respond with:
```
I'll help you create a detailed implementation plan. Let me start by understanding what we're building.

Please provide:
1. The task/ticket description (or reference to a ticket file)
2. Any relevant context, constraints, or specific requirements
3. Links to related research or previous implementations

I'll analyze this information and work with you to create a comprehensive plan.

Tip: You can also invoke this command with a ticket file directly: `/create_plan thoughts/allison/tickets/eng_1234.md`
For deeper analysis, try: `/create_plan think deeply about thoughts/allison/tickets/eng_1234.md`
```

Then wait for the user's input.

## Process Steps

### Step 1: Context Gathering & Initial Analysis

1. **Read all mentioned files immediately and FULLY**:
   - Ticket files (e.g., `thoughts/allison/tickets/eng_1234.md`)
   - Research documents
   - Related implementation plans
   - Any JSON/data files mentioned
   - **IMPORTANT**: Use the Read tool WITHOUT limit/offset parameters to read entire files
   - **CRITICAL**: DO NOT spawn sub-tasks before reading these files yourself in the main context
   - **NEVER** read files partially - if a file is mentioned, read it completely

2. **Search for ticket relationships and dependencies**:
   - Search for the ticket number across the codebase to find related documents:
     ```bash
     grep -r "TICKET-123" /path/to/codebase --include="*.md" --include="*.txt"
     find /path/to/codebase -type f -name "*TICKET-123*"
     ```
   - Look for:
     - **Parent tickets**: Is this a subtask of a larger epic/feature?
     - **Child/subtasks**: Does this ticket have dependent subtasks?
     - **Related RFCs**: Technical specifications describing the desired end state
     - **Other implementation plans**: Plans for parent or sibling tickets
     - **API documentation**: Contract specifications
   - **CRITICAL**: Understand the ticket hierarchy BEFORE starting analysis:
     - If this is a **subtask**, understand what the parent ticket covers
     - If this is a **parent ticket**, identify which subtasks exist and their status
     - Clarify which components are prerequisites vs. what this plan implements
   - Common patterns to recognize:
     - Parent ticket describes overall feature, subtasks implement specific components
     - RFCs describe complete desired state, plans implement incrementally
     - Data layer subtasks typically must complete before API layer tasks
   - Example hierarchy:
     ```
     LND-4525 (Parent: Veefin API Integration)
     ├── LND-4526 (Subtask: Data Storage - PREREQUISITE)
     ├── LND-4527 (Subtask: API Endpoints - This Plan)
     └── LND-4528 (Subtask: Monitoring & Alerts)
     ```
   - **Read all related documents found**:
     - Parent ticket documentation
     - Sibling subtask plans (to understand what's already done)
     - RFCs for the parent ticket (understand complete vision)
     - API contracts and specifications

3. **Spawn initial research tasks to gather context**:
   Before asking the user any questions, use specialized agents to research in parallel:

   - Use the **codebase-locator** agent to find all files related to the ticket/task
   - Use the **codebase-analyzer** agent to understand how the current implementation works
   - If relevant, use the **thoughts-locator** agent to find any existing thoughts documents about this feature
   
   These agents will:
   - Find relevant source files, configs, and tests
   - Identify the specific directories to focus on (e.g., if WUI is mentioned, they'll focus on humanlayer-wui/)
   - Trace data flow and key functions
   - Return detailed explanations with file:line references

4. **Read all files identified by research tasks**:
   - After research tasks complete, read ALL files they identified as relevant
   - Read them FULLY into the main context
   - This ensures you have complete understanding before proceeding

5. **Analyze and verify understanding WITH ticket context**:
   - Cross-reference the ticket requirements with actual code
   - **Clarify ticket relationships in your understanding**:
     - If subtask: "This ticket (LND-4527) is a subtask of LND-4525 (parent). LND-4526 (sibling) implements the data layer which must be completed first."
     - If parent: "This is the parent ticket. Subtasks LND-4526, LND-4527, LND-4528 break down the implementation."
   - **Identify what already exists vs. what's missing**:
     - Check if prerequisite subtasks are completed (e.g., database migrations, entities)
     - Don't assume infrastructure exists if it's part of a sibling subtask
     - Verify current migration version, existing entities, etc.
   - Identify any discrepancies or misunderstandings
   - Note assumptions that need verification
   - Determine true scope based on codebase reality AND ticket hierarchy

6. **Present informed understanding and focused questions**:
   ```
   Based on the ticket and my research of the codebase, I understand we need to [accurate summary].
   
   **Ticket Context:**
   - This is [parent ticket / subtask of PARENT-123]
   - [If subtask: Prerequisites: SUB-456 must be completed first (implements data layer)]
   - [If parent: Related subtasks: SUB-456 (data), SUB-457 (API), SUB-458 (monitoring)]

   I've found that:
   - [Current implementation detail with file:line reference]
   - [What exists from prerequisite subtasks vs. what's missing]
   - [Current database migration version, entities that exist]
   - [Relevant pattern or constraint discovered]
   - [Potential complexity or edge case identified]

   Questions that my research couldn't answer:
   - [Specific technical question that requires human judgment]
   - [Business logic clarification]
   - [Design preference that affects implementation]
   ```

   Only ask questions that you genuinely cannot answer through code investigation.

### Step 2: Research & Discovery

After getting initial clarifications:

1. **If the user corrects any misunderstanding**:
   - DO NOT just accept the correction
   - Spawn new research tasks to verify the correct information
   - Read the specific files/directories they mention
   - Only proceed once you've verified the facts yourself

2. **Create a research todo list** using TodoWrite to track exploration tasks

3. **Spawn parallel sub-tasks for comprehensive research**:
   - Create multiple Task agents to research different aspects concurrently
   - Use the right agent for each type of research:

   **For deeper investigation:**
   - **codebase-locator** - To find more specific files (e.g., "find all files that handle [specific component]")
   - **codebase-analyzer** - To understand implementation details (e.g., "analyze how [system] works")
   - **codebase-pattern-finder** - To find similar features we can model after

   **For historical context:**
   - **thoughts-locator** - To find any research, plans, or decisions about this area
   - **thoughts-analyzer** - To extract key insights from the most relevant documents

   **For related tickets:**
   - **linear-searcher** - To find similar issues or past implementations

   Each agent knows how to:
   - Find the right files and code patterns
   - Identify conventions and patterns to follow
   - Look for integration points and dependencies
   - Return specific file:line references
   - Find tests and examples

3. **Wait for ALL sub-tasks to complete** before proceeding

4. **Present findings and design options**:
   ```
   Based on my research, here's what I found:

   **Current State:**
   - [Key discovery about existing code]
   - [Pattern or convention to follow]

   **Design Options:**
   1. [Option A] - [pros/cons]
   2. [Option B] - [pros/cons]

   **Open Questions:**
   - [Technical uncertainty]
   - [Design decision needed]

   Which approach aligns best with your vision?
   ```

### Step 3: Plan Structure Development

Once aligned on approach:

1. **Create initial plan outline**:
   ```
   Here's my proposed plan structure:

   ## Overview
   [1-2 sentence summary]

   ## Implementation Phases:
   1. [Phase name] - [what it accomplishes]
   2. [Phase name] - [what it accomplishes]
   3. [Phase name] - [what it accomplishes]

   Does this phasing make sense? Should I adjust the order or granularity?
   ```

2. **Get feedback on structure** before writing details

### Step 4: Detailed Plan Writing

After structure approval:

1. **Write the plan** to `thoughts/shared/plans/YYYY-MM-DD-ENG-XXXX-description.md`
   - Format: `YYYY-MM-DD-ENG-XXXX-description.md` where:
     - YYYY-MM-DD is today's date
     - ENG-XXXX is the ticket number (omit if no ticket)
     - description is a brief kebab-case description
   - Examples:
     - With ticket: `2025-01-08-ENG-1478-parent-child-tracking.md`
     - Without ticket: `2025-01-08-improve-error-handling.md`
   - **IMPORTANT: Write plan to ALL impacted repositories**:
     - Always write to the main workspace: `/Users/juvianto.chi/Desktop/code/thoughts/shared/plans/...`
     - Also write to each impacted repository using the same path structure: `<repo>/thoughts/shared/plans/...`
     - Example: If plan impacts `bnpl` service, write to both:
       - `/Users/juvianto.chi/Desktop/code/thoughts/shared/plans/2025-11-05-LND-4526-example.md`
       - `/Users/juvianto.chi/Desktop/code/bnpl/thoughts/shared/plans/2025-11-05-LND-4526-example.md`
     - This ensures each repository has its own copy of plans that affect it
   - **ALSO: Copy request documentation to impacted repositories**:
     - If the task originates from `/Users/juvianto.chi/Desktop/code/documentation/request/[TICKET]/`, copy the entire request folder to the impacted repository
     - Use the same path structure: `<repo>/documentation/request/[TICKET]/`
     - Example: If working on `LND-4526` that impacts `bnpl`, copy:
       - From: `/Users/juvianto.chi/Desktop/code/documentation/request/LND-4526/`
       - To: `/Users/juvianto.chi/Desktop/code/bnpl/documentation/request/LND-4526/`
     - This keeps all related documentation (task.md, samples, etc.) alongside the plan in each repository
2. **Use this template structure**:

````markdown
# [Feature/Task Name] Implementation Plan

## Overview

[Brief description of what we're implementing and why]

## Relationship to Other Tickets and Documents

**Ticket Hierarchy:**
- [Describe if this is parent/child and relationship to other tickets]
- [Example: "Parent Ticket: LND-4525 - Overall Veefin Integration"]
- [Example: "Subtasks: LND-4526 (Data Layer - PREREQUISITE), LND-4527 (API Layer - This Plan)"]

**Related Documents:**
- [List RFCs that describe desired end state]
- [List other implementation plans (parent or sibling subtasks)]
- [List API contracts or specifications]

**Important Note:** [If subtask] This plan builds on [prerequisite subtask]. That subtask must be completed first and provides: [list what it provides]

## Current State Analysis

[What exists now, what's missing, key constraints discovered]

**Existing Infrastructure:** [What already exists in the codebase]
**What [Prerequisite Subtask] Will Provide:** [If applicable, what the prerequisite subtask implements]
**What's Missing (This Plan):** [What this specific plan will implement]

### Key Discoveries:
- [Important finding with file:line reference]
- [Current database migration version]
- [Entities/repositories that exist vs. don't exist yet]
- [Pattern to follow]
- [Constraint to work within]

## Desired End State

After **[prerequisite subtask if applicable]** is completed:
- [List what the prerequisite provides]

After **this plan** is completed:
- [A Specification of the desired end state after this plan is complete, and how to verify it]

## What We're NOT Doing

- ❌ **[Components handled by other subtasks]** (that's [SUBTASK-123] - [must be done first / will be done later])
- [Explicitly list out-of-scope items to prevent scope creep]

## Implementation Approach

[High-level strategy and reasoning]

## Phase 1: [Descriptive Name]

### Overview
[What this phase accomplishes]

### Changes Required:

#### 1. [Component/File Group]
**File**: `path/to/file.ext`
**Changes**: [Summary of changes]

```[language]
// Specific code to add/modify
```

### Success Criteria:

#### Automated Verification:
- [ ] Migration applies cleanly: `make migrate`
- [ ] Unit tests pass: `make test-component`
- [ ] Type checking passes: `npm run typecheck`
- [ ] Linting passes: `make lint`
- [ ] Integration tests pass: `make test-integration`

#### Manual Verification:
- [ ] Feature works as expected when tested via UI
- [ ] Performance is acceptable under load
- [ ] Edge case handling verified manually
- [ ] No regressions in related features

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: [Descriptive Name]

[Similar structure with both automated and manual success criteria...]

---

## Testing Strategy

### Unit Tests:
- [What to test]
- [Key edge cases]

### Integration Tests:
- [End-to-end scenarios]

### Manual Testing Steps:
1. [Specific step to verify feature]
2. [Another verification step]
3. [Edge case to test manually]

## Performance Considerations

[Any performance implications or optimizations needed]

## Migration Notes

[If applicable, how to handle existing data/systems]

## Dependencies

### Critical Prerequisites
- **[SUBTASK-123] MUST BE COMPLETED FIRST**: [If applicable, list what prerequisite provides]
  - [Specific deliverables from prerequisite]
  - [Database tables, entities, services that must exist]
  - [Test data that should be available]

### Other Dependencies
- [List other dependencies like external services, certificates, etc.]

## References

- **Parent Ticket**: [PARENT-123](link) - [Description]
- **This Ticket**: [TICKET-456](link) - [Description]
- **Prerequisite Subtask**: [SUBTASK-789](link) - [Description] (must complete first)
- **Related RFC**: `path/to/rfc.md` - [What it describes]
- **API Contract**: `path/to/api-docs.md` - [What it specifies]
- Original ticket file: `thoughts/allison/tickets/eng_XXXX.md`
- Related research: `thoughts/shared/research/[relevant].md`
- Similar implementation: `[file:line]`
````

### Step 5: Sync and Review

1. **Sync the thoughts directory**:
   - Run `humanlayer thoughts sync` to sync the newly created plan
   - This ensures the plan is properly indexed and available

2. **Present the draft plan location**:
   ```
   I've created the initial implementation plan at:
   `thoughts/shared/plans/YYYY-MM-DD-ENG-XXXX-description.md`

   Please review it and let me know:
   - Are the phases properly scoped?
   - Are the success criteria specific enough?
   - Any technical details that need adjustment?
   - Missing edge cases or considerations?
   ```

3. **Iterate based on feedback** - be ready to:
   - Add missing phases
   - Adjust technical approach
   - Clarify success criteria (both automated and manual)
   - Add/remove scope items
   - After making changes, run `humanlayer thoughts sync` again

4. **Continue refining** until the user is satisfied

## Important Guidelines

1. **Be Skeptical**:
   - Question vague requirements
   - Identify potential issues early
   - Ask "why" and "what about"
   - Don't assume - verify with code

2. **Be Interactive**:
   - Don't write the full plan in one shot
   - Get buy-in at each major step
   - Allow course corrections
   - Work collaboratively

3. **Be Thorough**:
   - Read all context files COMPLETELY before planning
   - Research actual code patterns using parallel sub-tasks
   - Include specific file paths and line numbers
   - Write measurable success criteria with clear automated vs manual distinction
   - automated steps should use `make` whenever possible - for example `make -C humanlayer-wui check` instead of `cd humanlayer-wui && bun run fmt`

4. **Be Practical**:
   - Focus on incremental, testable changes
   - Consider migration and rollback
   - Think about edge cases
   - Include "what we're NOT doing"

5. **Track Progress**:
   - Use TodoWrite to track planning tasks
   - Update todos as you complete research
   - Mark planning tasks complete when done

6. **No Open Questions in Final Plan**:
   - If you encounter open questions during planning, STOP
   - Research or ask for clarification immediately
   - Do NOT write the plan with unresolved questions
   - The implementation plan must be complete and actionable
   - Every decision must be made before finalizing the plan

## Success Criteria Guidelines

**Always separate success criteria into two categories:**

1. **Automated Verification** (can be run by execution agents):
   - Commands that can be run: `make test`, `npm run lint`, etc.
   - Specific files that should exist
   - Code compilation/type checking
   - Automated test suites

2. **Manual Verification** (requires human testing):
   - UI/UX functionality
   - Performance under real conditions
   - Edge cases that are hard to automate
   - User acceptance criteria

**Format example:**
```markdown
### Success Criteria:

#### Automated Verification:
- [ ] Database migration runs successfully: `make migrate`
- [ ] All unit tests pass: `go test ./...`
- [ ] No linting errors: `golangci-lint run`
- [ ] API endpoint returns 200: `curl localhost:8080/api/new-endpoint`

#### Manual Verification:
- [ ] New feature appears correctly in the UI
- [ ] Performance is acceptable with 1000+ items
- [ ] Error messages are user-friendly
- [ ] Feature works correctly on mobile devices
```

## Common Patterns

### For Database Changes:
- Start with schema/migration
- Add store methods
- Update business logic
- Expose via API
- Update clients

### For New Features:
- Research existing patterns first
- Start with data model
- Build backend logic
- Add API endpoints
- Implement UI last

### For Refactoring:
- Document current behavior
- Plan incremental changes
- Maintain backwards compatibility
- Include migration strategy

## Sub-task Spawning Best Practices

When spawning research sub-tasks:

1. **Spawn multiple tasks in parallel** for efficiency
2. **Each task should be focused** on a specific area
3. **Provide detailed instructions** including:
   - Exactly what to search for
   - Which directories to focus on
   - What information to extract
   - Expected output format
4. **Be EXTREMELY specific about directories**:
   - If the ticket mentions "WUI", specify `humanlayer-wui/` directory
   - If it mentions "daemon", specify `hld/` directory
   - Never use generic terms like "UI" when you mean "WUI"
   - Include the full path context in your prompts
5. **Specify read-only tools** to use
6. **Request specific file:line references** in responses
7. **Wait for all tasks to complete** before synthesizing
8. **Verify sub-task results**:
   - If a sub-task returns unexpected results, spawn follow-up tasks
   - Cross-check findings against the actual codebase
   - Don't accept results that seem incorrect

Example of spawning multiple tasks:
```python
# Spawn these tasks concurrently:
tasks = [
    Task("Research database schema", db_research_prompt),
    Task("Find API patterns", api_research_prompt),
    Task("Investigate UI components", ui_research_prompt),
    Task("Check test patterns", test_research_prompt)
]
```

## Example Interaction Flow

```
User: /implementation_plan
Assistant: I'll help you create a detailed implementation plan...

User: We need to add parent-child tracking for Claude sub-tasks. See thoughts/allison/tickets/eng_1478.md
Assistant: Let me read that ticket file completely first...

[Reads file fully]

Based on the ticket, I understand we need to track parent-child relationships for Claude sub-task events in the hld daemon. Before I start planning, I have some questions...

[Interactive process continues...]
```