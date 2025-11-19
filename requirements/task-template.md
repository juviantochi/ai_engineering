scope repository: @{repository-impacted}

You are task to help user brainstorm and clarify the request bellow.
Make sure everything is clear to be output into a requirement.
Every clarified things should update this task.md file.

background:
Currently this is supposed to be supported in the /upload/file endpoint and currently there's no validation other than file type.

The task is:
- to have it only accepts PDFs and validate PDF is not corrupted and can be opened.
- make sure to also create unit test for edge cases like corrupted PDF, empty PDF, very large PDF.

Make sure what will be out of scope if not sure. 
DO NOT ASUME anything. ALWAYS ASK FOR USER CLARIFICATION IF NEEDED.

References:
1. {ticket-number} ticket
2. etc.

**OUTPUT**
Decent requirement and put it in requirement.md file under the same folder.
requirement should have these sections:
- Title
- Background
- Objective
- Scope (Whats in and out of scope)
- Requirements
- References
