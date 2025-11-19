
# Generating RFC for LND-4526

## Overview

This document describes the iterative workflow for generating the LND-4526 RFC (Request for Comments) using AI assistance. The process emphasizes clarity through collaboration, ensuring all requirements are thoroughly understood before documentation begins.

**Key Principle**: *Clarify first, document second.* This approach reduces rework and produces more comprehensive RFCs.

### Tools Used

- **GitHub Copilot with Claude model**: Used in clarification and Q&A phase up until early generation of RFC since Claude Code had not been shared before.
- **Claude Code**: Used in next phase for clarification and Q&A up until RFC generation and refinement.
- **Codex**: Some fact finding and brainstorming of current RFC.

---

## Phase 1: Initial Task Definition

### Step 1: Create Initial Task Prompt

Start with a simple task prompt located in `task.md` file (or use `requirement/` file according to your workflow; this naming predates the current workflow standard).

**Purpose**: This file serves as the living requirement document that evolves through clarification.

#### The Initial Prompt Template

```markdown
scope repository: @{repository-impacted}

You are task to help user brainstorm and clarify the request bellow.
Make sure everything is clear to be output into a requirement.
Every clarified things should update this task.md file.

background:
give brief context 2-5 sentences about the request.
(what is the problem, why is it needed, any relevant history)

The task is:
- .
- .

Make sure to:
- .
- .

Make sure not to:
- .
- .

References:
1. rfc/template.md
2. {ticket-number} ticket
3. rfc-generation-guide.md
4. etc.

**OUTPUT**
Only after the user asks to create the rfc, then :
1. output RFC (3000–5000 words) following rfc/template.md → save as `<ticket-number>.md` in `RFC/` folder  .
2. output scoped RFC in specific repository impacted by the change.
    * if more than 1 repository is impacted then output multiple RFCs with each RFC containing only relevant parts of the full RFC → save as `<ticket-number>-<repository-name>.md` in `RFC/` folder.
    * In the scoped repository RFC, include the main RFC as reference.
```

### Why This Structure?

- **Scope declaration** ensures the AI focuses on relevant codebases
- **Iterative clarification** prevents premature documentation of unclear requirements
- **Living document approach** captures evolving understanding
- **Controlled output** ensures RFC generation only happens when ready

---

## Phase 2: Iterative Clarification

### The Clarification Loop

Execute the task and prompt through an iterative dialogue:

#### What I Was Doing:

**1. Respond to Agent Questions**

- The AI will identify ambiguities and ask clarifying questions
- Provide specific, concrete answers. Make sure to have agent ask back anything that's not clarified
- Don't rush this step—thoroughness here saves time later

**2. Review Updated Task**

- Read the clarified `task.md` again after each update
- Verify your answers were captured correctly
- Check if new clarity reveals additional questions

**3. Raise Issues & Considerations**

- Think critically about edge cases
- Consider technical constraints
- Identify potential integration challenges
- Question assumptions

Example concerns to raise:
- "What happens if service X is down during this operation?"
- "Have we considered backward compatibility?"
- "What's the rollback strategy?"

**4. Iterate Until Clear**

- Return to step 1 and continue the loop
- Each cycle should reduce ambiguity
- Don't proceed until you can confidently answer "yes" to:
  - Is the technical approach clear?
  - Are all edge cases identified?
  - Are dependencies and integrations understood?
  - Is the scope well-defined?

**5. Session Summary & Learning Capture**

Once everything is clear, ask the agent to:
- Summarize the current session
- Update `rfc-generation-guide.md` with lessons learned

**Why?** This creates institutional knowledge and prevents repeating the same clarification cycles in future tasks.

> 💡 **Pro Tip**: Most of the value in this workflow comes from this phase. A well-clarified requirement practically writes itself into an RFC.

---

## Phase 3: RFC Generation

### Trigger RFC Creation

Once confident with the task definition:

**Explicitly ask**: *"Please create the RFC based on the clarified task.md"*

The agent will generate:

#### Main RFC
- File: `<ticket-number>.md` in `RFC/` folder
- Comprehensive 3000-5000 word document
- Follows `rfc/template.md` structure

#### Scoped RFCs (if multiple repositories affected)
- One per repository: `<ticket-number>-<repository-name>.md`
- Contains only repository-relevant details
- References the main RFC for full context

---

## Phase 4: RFC Review & Refinement

### Critical Review Process

After RFC generation, perform thorough review:

**1. Initial Read-Through**

- Read the entire RFC from start to finish
- Don't edit during first read—just absorb

**2. Completeness Check**

- Verify all points from clarified `task.md` are addressed
- Check that each "Make sure:" item has corresponding content
- Ensure all references are included and relevant

**3. Consistency & Logic Audit**

- Look for internal contradictions
- Verify technical accuracy
- Check that proposed solutions actually solve stated problems
- Validate that dependencies are correctly identified
- Ensure rollback/rollout strategies are realistic

**4. Revision Cycle**

- Ask for revisions if needed
- Be specific about what's wrong or missing
- Provide examples of what you expect
- Reference specific sections that need improvement
- **Note**: Most of the time revision IS needed—don't skip this

**Repeat Review After Revisions**

- Re-read the entire RFC, not just changed sections
- Context matters; changes in one section may affect others
- Continue until satisfied

**Final Approval**

- Once satisfied, mark the RFC as ready for team review
- Share with relevant stakeholders

> ⚠️ **Important**: Don't read the RFC only once. Common issues are only visible after multiple passes with different focus areas (technical accuracy, completeness, consistency, clarity).

---

## Best Practices & Lessons Learned

### Do's

- ✅ **Be specific in clarifications** - Vague answers lead to vague RFCs
- ✅ **Think about operations** - Deployment, monitoring, rollback should be considered early
- ✅ **Update learning guides** - Capture what worked and what didn't
- ✅ **Take your time in clarification** - 80% of RFC quality comes from this phase
- ✅ **Read multiple times** - Each pass reveals different issues

### Don'ts

- ❌ **Don't rush to RFC generation** - Unclear requirements = low-quality RFC
- ❌ **Don't skip the learning capture** - You'll repeat the same mistakes
- ❌ **Don't accept the first draft** - Revision is expected and valuable
- ❌ **Don't review only changed sections** - Context matters
- ❌ **Don't assume anything is "obvious"** - Explicit is better than implicit

---

## Workflow Diagram

```
┌─────────────────────┐
│  Initial task.md    │
│   with prompt       │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Clarification Loop │◄─────┐
│  - Answer questions │      │
│  - Review updates   │      │
│  - Raise issues     │      │
└──────────┬──────────┘      │
           │                 │
           │ Not clear       │
           ├─────────────────┘
           │
           │ Clear
           ▼
┌─────────────────────┐
│ Update guide & docs │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Generate RFC       │
│  - Main RFC         │
│  - Scoped RFCs      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Review & Refine    │◄─────┐
│  - Read through     │      │
│  - Check complete   │      │
│  - Find issues      │      │
└──────────┬──────────┘      │
           │                 │
           │ Issues found    │
           ├─────────────────┘
           │
           │ Satisfied
           ▼
┌─────────────────────┐
│  Share with team    │
└─────────────────────┘
```

---

## Example Timeline

### Typical Effort Distribution:

| Phase                | Time Investment   | Icon |
|----------------------|-------------------|------|
| **Clarification**    | 60-70% of time    | ⏱️   |
| **RFC Generation**   | 5-10% of time     | ⚡   |
| **Review & Revision**| 25-30% of time    | 🔍   |

> **Why this matters**: Don't be discouraged by lengthy clarification. It's the most valuable part of the process.