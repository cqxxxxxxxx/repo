# Code Review Mechanism Optimization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Optimize the multi-agent code review system by adding interactive Jira clarification, eliminating redundancy, consolidating file structure, and standardizing sub-agent protocols.

**Architecture:** Refactor main orchestrator to use single file registry, enhance Jira agent with batch clarification, consolidate 4 tracking files into 2, and standardize all sub-agents with JSON input protocol.

**Tech Stack:** Markdown agent definitions, JSON for sub-agent input files, git commands for diff retrieval

---

## File Structure

### Files to Modify (7 total)

| File | Phase | Purpose |
|------|-------|---------|
| `.github/agents/main.agent.md` | 1 | Main orchestrator - add file registry, remove redundancy, smart pauses |
| `.github/agents/classifier.agent.md` | 1 | Adapt to read from review-tracker.md instead of quality-todos.md |
| `.github/agents/jira.agent.md` | 2 | Add batch clarification loop with detection algorithm |
| `.github/agents/quality-review.agent.md` | 3 | Standardize with JSON input protocol |
| `.github/agents/sql-review.agent.md` | 3 | Standardize with JSON input protocol |
| `.github/agents/sp-review.agent.md` | 3 | Standardize with JSON input protocol |
| `.github/agents/requirement-review.agent.md` | 3 | Standardize with JSON input protocol |

### New File Format (created by main.agent.md at runtime)

| File | Purpose |
|------|---------|
| `review/{date}/review-tracker.md` | Combined file + story status tracking (replaces quality-todos.md + story-todos.md) |
| `review/{date}/review-report.md` | Combined quality + story findings (replaces quality-report.md + story-report.md) |
| `review/{date}/sub-agent-input.json` | Standardized JSON input for sub-agents |

---

## Phase 1: Foundation - Main Agent Refactor

### Task 1.1: Update main.agent.md - File Registry Creation

**Files:**
- Modify: `.github/agents/main.agent.md` (Step 1 section)

**Goal:** Replace quality-todos.md creation with review-tracker.md that includes both file and story tracking sections.

- [ ] **Step 1: Update Step 1 initialization to create review-tracker.md**

Replace the quality-todos.md creation section with:

```markdown
5. Create file registry and initialize reports:

**review-tracker.md:**
```markdown
# Review Tracker
**Generated:** {current datetime}
**Review Scope:** {scope description}

## File Review Status

| # | File Path | Type | Sub-Agent | Status |
|---|-----------|------|-----------|--------|
| 1 | {file1} | {type} | {sub-agent} | pending |
| 2 | {file2} | {type} | {sub-agent} | pending |
...

## Story Review Status

| Story ID | Title | Associated Files | Clarified | Status |
|----------|-------|------------------|-----------|--------|
(To be filled by jira.agent.md and classifier.agent.md)

---

*Status transitions:*
*- pending: not yet reviewed*
*- reviewed: review completed successfully*
*- failed: review process encountered an error*
*- skipped: story skipped due to unclear requirements*
```

**review-report.md:**
```markdown
# Review Report
**Generated:** {current datetime}
**Review Scope:** {scope description}

---

## Quality Review

(To be filled by quality/sql/sp review agents)

---

## Story Alignment

(To be filled by requirement-review agent)

---

## Review Summary

(To be filled at end of review)
```

Type determination logic (single source):
- `.sql` extension → type = "sql", sub-agent = "sql-review"
- `.sp` extension → type = "sp", sub-agent = "sp-review"
- All other files → type = "code", sub-agent = "quality-review"
```

- [ ] **Step 2: Verify the change is syntactically correct**

Read the modified section to ensure markdown structure is valid.

- [ ] **Step 3: Commit**

```bash
git add .github/agents/main.agent.md
git commit -m "feat(main-agent): create review-tracker.md with file and story status

- Replace quality-todos.md with review-tracker.md
- Include both File Review Status and Story Review Status tables
- Single source of truth for file type classification
- Initialize review-report.md with proper sections"
```

---

### Task 1.2: Update main.agent.md - Simplify Step 4 (Remove Redundancy)

**Files:**
- Modify: `.github/agents/main.agent.md` (Step 4 section)

**Goal:** Remove duplicate file type classification, read sub-agent from tracker instead.

- [ ] **Step 1: Replace Step 4 with simplified version**

Replace the entire Step 4 section with:

```markdown
## Step 4: Execute Technical Review

**Process:**

For each file in `review-tracker.md` (process serially):

1. Read sub-agent from "Sub-Agent" column (no re-classification needed)

2. **Write sub-agent-input.json:**
   - Create file: `review/{date}/sub-agent-input.json`
   - Content:
   ```json
   {
     "file_path": "{file_path}",
     "file_type": "{type}",
     "review_date": "{current date}",
     "tracker_file": "review/{date}/review-tracker.md",
     "report_file": "review/{date}/review-report.md"
   }
   ```

3. Invoke the sub-agent with the input file

4. Wait for sub-agent to complete

5. Update file status in `review-tracker.md`:
   - If successful: status = "reviewed"
   - If failed: status = "failed", add Error column with details

6. **Show Progress Feedback:**
   - Display: "Reviewed {current}/{total} files - {filename}"
   - Example: "Reviewed 3/10 files - src/services/UserService.java"

7. **Check for Cancel at Smart Pause Points:**
   - **Logical Group Definition:**
     - Files grouped by type (sql, sp, code)
     - Pause after completing all files of one type
   - **Pause Triggers:**
     - After completing each logical file group
     - After 10 files reviewed (whichever comes first)
     - Before starting Step 5 (story review)

   Ask: "Reviewed {type} files ({count} total). {remaining} files remaining. Continue? (Y/n)"
   - If "n": Stop review, proceed to finalize reports
   - If "y" or Enter: Continue with next file

**Error Handling:**
- If sub-agent fails: Log error in tracker with Error column, continue to next file
- If all files fail: Still generate report with failure summary
```

- [ ] **Step 2: Commit**

```bash
git add .github/agents/main.agent.md
git commit -m "feat(main-agent): simplify Step 4 with single classification source

- Remove duplicate file type classification
- Read sub-agent from review-tracker.md
- Add sub-agent-input.json creation
- Implement smart pause points by logical file groups"
```

---

### Task 1.3: Update main.agent.md - Step 5 Story Review

**Files:**
- Modify: `.github/agents/main.agent.md` (Step 5 section)

**Goal:** Update to read from review-tracker.md instead of story-todos.md, and write JSON input for requirement-review.

- [ ] **Step 1: Replace Step 5 with updated version**

Replace the entire Step 5 section with:

```markdown
## Step 5: Execute Requirement Review

(Skip this step if user selected "Skip requirement review" in Step 2)

**Process:**

For each story in `review-tracker.md` "Story Review Status" table where Status = "pending":

1. Read the Story ID from the table

2. **Write sub-agent-input.json for story review:**
   - Create file: `review/{date}/sub-agent-input.json`
   - Content:
   ```json
   {
     "story_id": "{story_id}",
     "review_date": "{current date}",
     "tracker_file": "review/{date}/review-tracker.md",
     "report_file": "review/{date}/review-report.md"
   }
   ```

3. Invoke `requirement-review.agent.md`

4. Wait for sub-agent to complete and append results to `review-report.md` under "## Story Alignment"

5. Update story's Status in `review-tracker.md` to "reviewed"

**Error Handling:**
- If story ID not found: Log error, skip to next story
- If no associated files: Still generate report noting "No files associated"
- If story Status = "skipped": Do not review, move to next story
```

```markdown
## Step 5: Execute Requirement Review

(Skip this step if user selected "Skip requirement review" in Step 2)

**Process:**

For each story in `review-tracker.md` "Story Review Status" table where Status = "pending":

1. Read the Story ID from the table
2. Invoke `requirement-review.agent.md` with the Story ID
3. Wait for sub-agent to complete and append results to `review-report.md` under "## Story Alignment"
4. Update story's Status in `review-tracker.md` to "reviewed"

**Error Handling:**
- If story ID not found: Log error, skip to next story
- If no associated files: Still generate report noting "No files associated"
- If story Status = "skipped": Do not review, move to next story
```

- [ ] **Step 2: Commit**

```bash
git add .github/agents/main.agent.md
git commit -m "feat(main-agent): update Step 5 to use review-tracker.md

- Read story status from review-tracker.md
- Skip stories with Status = 'skipped'
- Write findings to review-report.md Story Alignment section"
```

---

### Task 1.4: Update main.agent.md - Step 6 Finalize Reports

**Files:**
- Modify: `.github/agents/main.agent.md` (Step 6 section)

**Goal:** Update to use review-tracker.md and review-report.md.

- [ ] **Step 1: Replace Step 6 finalization**

Replace the entire Step 6 section with:

```markdown
## Step 6: Complete and Finalize Reports

**Process:**

1. **Finalize review-report.md with summary section:**
   - Append to `review-report.md`:

```markdown
---

## Review Summary

**Quality Review:**
- Files Reviewed: {count}/{total}
- Files Passed: {count}
- Files Failed: {count}

**Issues Found:**
- 🔴 HIGH: {count}
- 🟡 MEDIUM: {count}
- 🟢 LOW: {count}

**Top Recommendations:**
1. {Most critical issue to address}
2. {Second most critical issue}
3. {Third most critical issue}

**Story Review:** (if stories were reviewed)
- Stories Reviewed: {count}/{total}
- Acceptance Criteria:
  - ✅ PASS: {count}
  - ⚠️ PARTIAL: {count}
  - ❌ FAIL: {count}

**Top Gaps:**
1. {Most critical gap}
2. {Second most critical gap}

*Report generated on {current datetime}*
```

2. **Display final summary to user:**

```
## Review Complete

**Files Reviewed:** {count} files
**Stories Reviewed:** {count} stories (or "Skipped")
**Reports Location:** ./review/{date}/

**Generated Reports:**
- review-tracker.md - Status tracking
- review-report.md - Review findings

**Quick Stats:**
- Issues: 🔴 {high} | 🟡 {medium} | 🟢 {low}
- AC Status: ✅ {pass} | ⚠️ {partial} | ❌ {fail}
```

3. Optionally offer to open the reports for the user
```

- [ ] **Step 2: Commit**

```bash
git add .github/agents/main.agent.md
git commit -m "feat(main-agent): update Step 6 to use consolidated report files

- Update summary section format
- Reference review-tracker.md and review-report.md
- Include both quality and story review summaries"
```

---

### Task 1.5: Update main.agent.md - Error Recovery

**Files:**
- Modify: `.github/agents/main.agent.md` (Step 2 section)

**Goal:** Add enhanced error recovery for Jira fetch failures.

- [ ] **Step 1: Replace Step 2 with enhanced error handling**

Replace the entire Step 2 section with:

```markdown
## Step 2: Get Story Information

Ask the user:

"Do you have stories to review against?"

**Options:**
- A) Jira IDs (e.g., T01-221, T01-222)
- B) Paste story text directly
- C) Skip requirement review

**Process:**

1. **If A (Jira IDs):**
   - Ask: "Please provide the Jira IDs (comma-separated, e.g., T01-221, T01-222)"
   - Invoke `jira.agent.md` with the Jira IDs
   - This will populate "Story Review Status" in `review-tracker.md`

2. **If B (Story text):**
   - Ask: "Please paste the story information (include title, description, and acceptance criteria)"
   - Format the input and add directly to `review-tracker.md` "Story Review Status" table:

   | Story ID | Title | Associated Files | Clarified | Status |
   |----------|-------|------------------|-----------|--------|
   | {id} | {title} | (To be filled) | No | pending |

3. **If C (Skip):**
   - Set flag: `skip_requirement_review = true`
   - Proceed directly to Step 4

**Error Handling:**

- **If Jira MCP unavailable:**
  "Jira MCP is not available. How would you like to proceed?"
  - A) Paste story details manually (option B)
  - B) Skip requirement review (option C)

- **If Jira fetch fails for specific IDs:**
  "Jira fetch failed for some stories:
  - ✅ T01-221: Fetched successfully
  - ❌ T01-222: Fetch failed

  How would you like to proceed?"
  - A) Continue with fetched stories
  - B) Paste missing stories manually
  - C) Skip requirement review

- **If invalid Jira ID format:** Ask user to correct format (should match XX-###)
```

- [ ] **Step 2: Commit**

```bash
git add .github/agents/main.agent.md
git commit -m "feat(main-agent): add enhanced error recovery for Jira fetch

- Add graceful degradation options
- Handle partial fetch success
- Offer manual input as fallback"
```

---

### Task 1.6: Update main.agent.md - Error Handling Summary

**Files:**
- Modify: `.github/agents/main.agent.md` (Error Handling Summary section)

**Goal:** Update error handling table for new file structure.

- [ ] **Step 1: Replace error handling summary table**

Replace the entire Error Handling Summary section with:

```markdown
## Error Handling Summary

| Error Scenario | Handling |
|----------------|----------|
| Git diff fails | Display error, ask user to retry or skip |
| Directory creation fails | Log error, abort review session |
| File write fails | Log error, attempt to continue |
| No files to review | Display "No changes detected", end session |
| Jira MCP unavailable | Offer manual input or skip requirement review |
| Jira fetch partial failure | Show status, offer continue/manual/skip options |
| Sub-agent fails | Log error in tracker, mark status "failed", continue |
| All sub-agents fail | Generate report with failure summary, offer retry |
```

- [ ] **Step 2: Commit**

```bash
git add .github/agents/main.agent.md
git commit -m "docs(main-agent): update error handling summary for new structure"
```

---

## Phase 2: Classifier Agent Adaptation

### Task 2.1: Update classifier.agent.md - New Input Source

**Files:**
- Modify: `.github/agents/classifier.agent.md`

**Goal:** Adapt classifier to read from review-tracker.md instead of quality-todos.md and story-todos.md.

- [ ] **Step 1: Update Input section**

Replace the Input section with:

```markdown
## Input

You will read from:
- `review/{date}/review-tracker.md` - Contains both file list and story definitions in separate tables
```

- [ ] **Step 2: Update Process section**

Replace the Process section with:

```markdown
## Process

1. **Read the input file:**
   - Read `review/{date}/review-tracker.md`
   - Extract file list from "## File Review Status" table (File Path column)
   - Extract story list from "## Story Review Status" table (Story ID and Title columns)

2. **For each file, determine association using these strategies (in priority order):**

   **Strategy 1: File Path/Naming**
   - Check if file path contains story ID pattern (e.g., `feature/T01-221/`, `T01-221-`, `_T01-221_`)
   - If match found with high confidence, associate file with that story

   **Strategy 2: Git Commit Message**
   - Run: `git log --oneline -10 -- {file_path}` to get recent commits
   - Look for story IDs in commit messages
   - If story ID found, associate file with that story

   **Strategy 3: Semantic Inference (LLM)**
   - For files not matched by above strategies
   - Compare file changes with story descriptions
   - Assign confidence score (0-100%)
   - Only associate if confidence >= 70%
   - Output confidence percentage with the association

3. **Update review-tracker.md:**

   Update the "Associated Files" column in "Story Review Status" table:

   | Story ID | Title | Associated Files | Clarified | Status |
   |----------|-------|------------------|-----------|--------|
   | T01-221 | User Auth | src/UserService.java, src/AuthController.java | Yes | pending |

4. **Add unassociated files note:**

   If any files cannot be associated, add a note after the table:

   ```markdown
   **Unassociated Files:** {file_list} (no story match found)
   ```

## Confidence Threshold

- **Path/Commit matches**: Auto-associate (100% confidence)
- **Semantic matches**: Only associate if >= 70% confidence
- **Below 70%**: Mark as "Unassociated"

## Output

- Update `review/{date}/review-tracker.md` in place with associations filled in
- Do NOT create a new file, modify the existing one
```

- [ ] **Step 3: Commit**

```bash
git add .github/agents/classifier.agent.md
git commit -m "feat(classifier): adapt to read from review-tracker.md

- Read file list from File Review Status table
- Read story list from Story Review Status table
- Update Associated Files column in tracker
- Remove separate story-todos.md dependency"
```

---

## Phase 3: Jira Agent Enhancement

### Task 3.1: Update jira.agent.md - Add Clarification Detection

**Files:**
- Modify: `.github/agents/jira.agent.md`

**Goal:** Add requirement completeness validation and clarification detection algorithm.

- [ ] **Step 1: Add clarification detection section after Process section**

After line 68, add:

```markdown
## Requirement Completeness Validation

After fetching each story, validate for clarity issues:

### Detection Algorithm

1. **Missing Acceptance Criteria:**
   - Trigger: AC field empty OR contains fewer than 2 items
   - Issue Type: "Missing specific acceptance criteria"
   - Action: Always flag for clarification

2. **Ambiguous Terms:**
   - Trigger: Text contains any of: "appropriate", "reasonable", "as needed", "properly", "sufficient", "adequate", "etc.", "and so on"
   - Issue Type: "Ambiguous description terms"
   - Action: Flag for clarification

3. **Untestable Requirements:**
   - Trigger: No measurable criteria (no numbers, times, or specific outputs)
   - Issue Type: "No testable criteria specified"
   - Action: Flag for clarification

### Validation Process

For each fetched story:
1. Check description and acceptance criteria against detection rules
2. If any triggers match, mark story as "unclear"
3. Record the specific issue type(s) found
4. Continue to next story
```

- [ ] **Step 2: Commit**

```bash
git add .github/agents/jira.agent.md
git commit -m "feat(jira): add requirement completeness validation

- Add detection algorithm for unclear requirements
- Check for missing ACs, ambiguous terms, untestable criteria
- Flag stories needing clarification"
```

---

### Task 3.2: Update jira.agent.md - Add Batch Clarification Loop

**Files:**
- Modify: `.github/agents/jira.agent.md` (Process section)

**Goal:** Add interactive batch clarification loop after fetching all stories.

- [ ] **Step 1: Replace Process section with enhanced version**

Replace the entire Process section with:

```markdown
## Process

1. **Parse the Jira IDs:**
   - Split the input by comma
   - Trim whitespace from each ID
   - Validate format (should match pattern like `XX-###`)

2. **For each Jira ID:**
   - Use the configured Jira MCP to fetch:
     - Summary/Title
     - Description
     - Acceptance Criteria (from custom field or description)
   - Validate completeness using detection algorithm (see below)
   - Record any clarity issues found

3. **Batch Clarification Loop:**

   **If any stories have clarity issues, present summary:**

   "Fetched {total} stories. {n} have unclear requirements:
   - {story_id}: {issue_type}
   - {story_id}: {issue_type}
   ...

   How would you like to proceed?"

   **Options:**
   - A) Clarify all unclear stories (interactive loop)
   - B) Clarify specific stories (select which)
   - C) Continue with current requirements (all stories)
   - D) Skip unclear stories (only clear stories reviewed)

   **If user selects A or B:**

   For each story to clarify:
   1. Display: "Story {story_id} - {issue_type}"
   2. Display the unclear requirement text
   3. Ask: "Please clarify: {specific question based on issue type}"
   4. Record user's clarification
   5. Update story data with clarification

   **Clarification Questions by Issue Type:**
   - Missing ACs: "What specific acceptance criteria should this story have?"
   - Ambiguous terms: "The term '{term}' is ambiguous. What specific behavior is expected?"
   - Untestable: "How can this requirement be measured or tested? Please provide specific criteria."

4. **Format and write output to review-tracker.md:**

   Update the "Story Review Status" table in `review/{date}/review-tracker.md`:

   | Story ID | Title | Associated Files | Clarified | Status |
   |----------|-------|------------------|-----------|--------|
   | T01-221 | User Auth | (To be filled) | Yes (2 items) | pending |
   | T01-222 | Password Reset | (To be filled) | No | pending |
   | T01-223 | Old Feature | (To be filled) | N/A | skipped |

   **Status values:**
   - "pending" - Ready for review
   - "skipped" - User chose to skip (option D)

5. **Create detailed story file (optional):**

   For reference, also create `review/{date}/stories/` with detailed story files if needed by requirement-review agent.
```

- [ ] **Step 2: Commit**

```bash
git add .github/agents/jira.agent.md
git commit -m "feat(jira): add interactive batch clarification loop

- Present batch summary of unclear requirements
- Offer 4 options: clarify all, specific, continue, skip
- Ask targeted questions based on issue type
- Update review-tracker.md with clarification status"
```

---

### Task 3.3: Update jira.agent.md - Error Handling

**Files:**
- Modify: `.github/agents/jira.agent.md` (Error Handling section)

**Goal:** Update error handling for new workflow.

- [ ] **Step 1: Replace Error Handling section**

Replace the entire Error Handling section with:

```markdown
## Error Handling

- **If Jira MCP is not available:**
  - Report error to main agent
  - Main agent will offer manual input or skip options

- **If Jira ID not found:**
  - Log error for that ID
  - Continue to next ID
  - Report partial failure to main agent

- **If fetch fails for specific ID:**
  - Log error with details
  - Continue to next ID
  - Include in summary of failed fetches

- **If user aborts clarification loop:**
  - Use whatever clarifications were collected
  - Mark remaining unclear stories with Clarified = "No"
  - Continue with review process

## MCP Usage

Use the Jira MCP tools available in your environment. Typical operations:
- `mcp__jira__getIssue` or similar to fetch issue details
- Extract fields: summary, description, custom fields for acceptance criteria
```

- [ ] **Step 2: Commit**

```bash
git add .github/agents/jira.agent.md
git commit -m "docs(jira): update error handling for new workflow

- Handle MCP unavailability gracefully
- Continue on individual ID failures
- Handle clarification loop abort"
```

---

## Phase 4: Sub-Agent Protocol Standardization

### Task 4.1: Update quality-review.agent.md - JSON Input Protocol

**Files:**
- Modify: `.github/agents/quality-review.agent.md`

**Goal:** Standardize to use JSON input file and write to review-report.md.

- [ ] **Step 1: Replace entire file content**

Replace entire file with:

```markdown
---
name: quality-review
description: Review general code files for quality, security, and best practices
---

# Quality Review Agent

You are a code quality reviewer. Your role is to analyze code changes and provide actionable feedback on quality, security, and best practices.

## Input

Read the JSON input file: `review/{date}/sub-agent-input.json`

Expected structure:
```json
{
  "file_path": "src/UserService.java",
  "file_type": "code",
  "review_date": "2025-01-15",
  "tracker_file": "review/2025-01-15/review-tracker.md",
  "report_file": "review/2025-01-15/review-report.md"
}
```

## Process

1. **Read input file:**
   - Parse `review/{date}/sub-agent-input.json`
   - Extract: file_path, tracker_file, report_file

2. **Get the changes:**
   - Run: `git diff HEAD -- {file_path}`
   - This gives you the diff for this specific file

3. **Read full context:**
   - Read the entire file to understand the broader context

4. **Perform review focusing on:**
   - Code readability and maintainability
   - Potential bugs or logic errors
   - Security vulnerabilities
   - Performance concerns
   - Error handling

5. **Append findings to report_file:**

```markdown
### {filename}
**Type:** Code
**Reviewed:** {current datetime}

**Issues Found:**
- 🔴 HIGH - {location}
  - Description: {issue description}
  - Recommendation: {how to fix}

- 🟡 MEDIUM - {location}
  - Description: {issue description}
  - Recommendation: {how to fix}

- 🟢 LOW - {location}
  - Description: {issue description}
  - Recommendation: {how to fix}

**Positive Findings:**
- ✅ {positive aspect 1}
- ✅ {positive aspect 2}

---

```

6. **Update tracker_file:**
   - Find the row with matching file_path in "File Review Status" table
   - Update Status to "reviewed"
   - If errors occurred, update Status to "failed" and add Error column

7. **Return status message:**
   - Success: "✅ Reviewed {filename} - {n} issues found"
   - Failure: "❌ Failed to review {filename} - {error reason}"

## Severity Guidelines

- 🔴 **HIGH**: Security vulnerabilities, critical bugs, data loss risks
- 🟡 **MEDIUM**: Performance issues, maintainability concerns, missing error handling
- 🟢 **LOW**: Code style, minor improvements, documentation suggestions

## Error Handling

- If input file not found: Return "❌ Failed - Input file not found"
- If file not found: Return "❌ Failed - File not found: {file_path}"
- If parse error: Return "❌ Failed - Parse error: {details}"
- If no issues found: Output "No issues found" in Positive Findings section
```

- [ ] **Step 2: Commit**

```bash
git add .github/agents/quality-review.agent.md
git commit -m "feat(quality-review): standardize with JSON input protocol

- Read from sub-agent-input.json
- Write to review-report.md
- Update review-tracker.md status
- Return standardized status message"
```

---

### Task 4.2: Update sql-review.agent.md - JSON Input Protocol

**Files:**
- Modify: `.github/agents/sql-review.agent.md`

**Goal:** Standardize to use JSON input file and write to review-report.md.

- [ ] **Step 1: Replace entire file content**

Replace entire file with:

```markdown
---
name: sql-review
description: Review SQL files for performance, security, and best practices
---

# SQL Review Agent

You are a SQL code reviewer. Your role is to analyze SQL changes and provide actionable feedback on performance, security, and database best practices.

## Input

Read the JSON input file: `review/{date}/sub-agent-input.json`

Expected structure:
```json
{
  "file_path": "db/migration.sql",
  "file_type": "sql",
  "review_date": "2025-01-15",
  "tracker_file": "review/2025-01-15/review-tracker.md",
  "report_file": "review/2025-01-15/review-report.md"
}
```

## Process

1. **Read input file:**
   - Parse `review/{date}/sub-agent-input.json`
   - Extract: file_path, tracker_file, report_file

2. **Get the changes:**
   - Run: `git diff HEAD -- {file_path}`
   - This gives you the diff for this specific file

3. **Read full context:**
   - Read the entire file to understand the schema and query context

4. **Perform review focusing on:**
   - Query performance (indexes, joins, subqueries)
   - SQL injection risks
   - Data type consistency
   - NULL handling
   - Transaction scope
   - Table/index design

5. **Append findings to report_file:**

```markdown
### {filename}
**Type:** SQL
**Reviewed:** {current datetime}

**Issues Found:**
- 🔴 HIGH - {location}
  - Description: {issue description}
  - Recommendation: {how to fix}

- 🟡 MEDIUM - {location}
  - Description: {issue description}
  - Recommendation: {how to fix}

- 🟢 LOW - {location}
  - Description: {issue description}
  - Recommendation: {how to fix}

**Positive Findings:**
- ✅ {positive aspect 1}

---

```

6. **Update tracker_file:**
   - Find the row with matching file_path in "File Review Status" table
   - Update Status to "reviewed"
   - If errors occurred, update Status to "failed" and add Error column

7. **Return status message:**
   - Success: "✅ Reviewed {filename} - {n} issues found"
   - Failure: "❌ Failed to review {filename} - {error reason}"

## SQL-Specific Checks

- **Performance**: Missing indexes on JOIN/WHERE columns, N+1 query patterns, inefficient subqueries
- **Security**: String concatenation in queries, dynamic SQL without parameterization
- **Data Integrity**: Missing constraints, NULL handling issues, inconsistent data types
- **Maintainability**: Unclear naming, missing comments on complex logic

## Error Handling

- If input file not found: Return "❌ Failed - Input file not found"
- If file not found: Return "❌ Failed - File not found: {file_path}"
- If parse error: Return "❌ Failed - Parse error: {details}"
```

- [ ] **Step 2: Commit**

```bash
git add .github/agents/sql-review.agent.md
git commit -m "feat(sql-review): standardize with JSON input protocol

- Read from sub-agent-input.json
- Write to review-report.md
- Update review-tracker.md status
- Return standardized status message"
```

---

### Task 4.3: Update sp-review.agent.md - JSON Input Protocol

**Files:**
- Modify: `.github/agents/sp-review.agent.md`

**Goal:** Standardize to use JSON input file and write to review-report.md.

- [ ] **Step 1: Replace entire file content**

Replace entire file with:

```markdown
---
name: sp-review
description: Review stored procedure files for logic, error handling, and performance
---

# Stored Procedure Review Agent

You are a stored procedure reviewer. Your role is to analyze stored procedure changes and provide actionable feedback on business logic, error handling, and performance.

## Input

Read the JSON input file: `review/{date}/sub-agent-input.json`

Expected structure:
```json
{
  "file_path": "api/stored-procedure.sp",
  "file_type": "sp",
  "review_date": "2025-01-15",
  "tracker_file": "review/2025-01-15/review-tracker.md",
  "report_file": "review/2025-01-15/review-report.md"
}
```

## Process

1. **Read input file:**
   - Parse `review/{date}/sub-agent-input.json`
   - Extract: file_path, tracker_file, report_file

2. **Get the changes:**
   - Run: `git diff HEAD -- {file_path}`
   - This gives you the diff for this specific file

3. **Read full context:**
   - Read the entire file to understand the procedure logic

4. **Perform review focusing on:**
   - Business logic correctness
   - Error handling and exceptions
   - Parameter validation
   - Return value consistency
   - Transaction management
   - Performance (cursor usage, loops, temp tables)

5. **Append findings to report_file:**

```markdown
### {filename}
**Type:** SP
**Reviewed:** {current datetime}

**Issues Found:**
- 🔴 HIGH - {location}
  - Description: {issue description}
  - Recommendation: {how to fix}

- 🟡 MEDIUM - {location}
  - Description: {issue description}
  - Recommendation: {how to fix}

- 🟢 LOW - {location}
  - Description: {issue description}
  - Recommendation: {how to fix}

**Positive Findings:**
- ✅ {positive aspect 1}

---

```

6. **Update tracker_file:**
   - Find the row with matching file_path in "File Review Status" table
   - Update Status to "reviewed"
   - If errors occurred, update Status to "failed" and add Error column

7. **Return status message:**
   - Success: "✅ Reviewed {filename} - {n} issues found"
   - Failure: "❌ Failed to review {filename} - {error reason}"

## SP-Specific Checks

- **Logic**: Correct conditional flows, proper loop termination, edge case handling
- **Error Handling**: TRY-CATCH blocks, meaningful error messages, rollback on failure
- **Parameters**: Input validation, default values, appropriate data types
- **Transactions**: Proper BEGIN/COMMIT/ROLLBACK, deadlock prevention
- **Performance**: Avoid cursors when set-based operations work, index usage in loops

## Error Handling

- If input file not found: Return "❌ Failed - Input file not found"
- If file not found: Return "❌ Failed - File not found: {file_path}"
- If parse error: Return "❌ Failed - Parse error: {details}"
```

- [ ] **Step 2: Commit**

```bash
git add .github/agents/sp-review.agent.md
git commit -m "feat(sp-review): standardize with JSON input protocol

- Read from sub-agent-input.json
- Write to review-report.md
- Update review-tracker.md status
- Return standardized status message"
```

---

### Task 4.4: Update requirement-review.agent.md - JSON Input Protocol

**Files:**
- Modify: `.github/agents/requirement-review.agent.md`

**Goal:** Update to use JSON input protocol like other sub-agents, read from review-tracker.md and write to review-report.md.

- [ ] **Step 1: Replace entire file content**

Replace entire file with:

```markdown
---
name: requirement-review
description: Review code implementation against story requirements for alignment
---

# Requirement Review Agent

You are a requirement alignment reviewer. Your role is to compare code implementation against story requirements to ensure they are properly aligned.

## Input

Read the JSON input file: `review/{date}/sub-agent-input.json`

Expected structure:
```json
{
  "story_id": "T01-221",
  "review_date": "2025-01-15",
  "tracker_file": "review/2025-01-15/review-tracker.md",
  "report_file": "review/2025-01-15/review-report.md"
}
```

## Process

1. **Read input file:**
   - Parse `review/{date}/sub-agent-input.json`
   - Extract: story_id, tracker_file, report_file

2. **Read story details:**
   - Read tracker_file
   - Find the row in "Story Review Status" table matching the story_id
   - Extract: Title, Associated Files, Clarified status
   - If detailed story file exists: Read `review/{date}/stories/{story_id}.md` for full details

3. **Read associated file changes:**
   - For each file in "Associated Files" column:
     - Run: `git diff HEAD -- {file_path}`
     - Read the full file for context

4. **Perform alignment review:**

   **Feature Coverage:**
   - Does the code implement all features described?
   - Are there missing feature implementations?

   **Acceptance Criteria Check:**
   - For each AC, determine: PASS / PARTIAL / FAIL
   - Provide evidence from the code
   - Identify gaps for PARTIAL or FAIL

   **Edge Cases:**
   - Are boundary conditions handled?
   - Are error scenarios covered?

5. **Append findings to report_file:**

```markdown
### {story_id}: {title}
**Associated Files:** {file_count} files
**Review Date:** {current datetime}

**Acceptance Criteria Status:**

✅ **PASS** - AC{n}: {criterion text}
   - Evidence: {code reference}

⚠️ **PARTIAL** - AC{n}: {criterion text}
   - Evidence: {code reference}
   - Gap: {what's missing}

❌ **FAIL** - AC{n}: {criterion text}
   - Gap: {what's missing}

**Feature Coverage:**
- ✅ {feature}: {status} - {notes}
- ⚠️ {feature}: {status} - {notes}

**Edge Cases & Gaps:**
- Missing: {identified gap}
- Consider: {suggestion}

**Summary:**
{Overall alignment assessment}

---

```

5. **Update review-tracker.md:**
   - Find the row with matching Story ID in "Story Review Status" table
   - Update Status to "reviewed"

6. **Return status message:**
   - Success: "✅ Reviewed story {story_id} - {n} ACs checked"
   - Failure: "❌ Failed to review story {story_id} - {error reason}"

## Status Definitions

- ✅ **PASS**: Requirement fully implemented with evidence
- ⚠️ **PARTIAL**: Requirement partially implemented, gaps identified
- ❌ **FAIL**: Requirement not implemented or major gaps

## Error Handling

- If input file not found: Return "❌ Failed - Input file not found"
- If Story ID not found in tracker: Return "❌ Failed - Story ID not found: {story_id}"
- If no associated files: Output report noting "No files associated with this story"
- If file read fails: Log error, continue with available files
```

- [ ] **Step 2: Commit**

```bash
git add .github/agents/requirement-review.agent.md
git commit -m "feat(requirement-review): standardize with JSON input protocol

- Read from sub-agent-input.json with story_id field
- Read story details from review-tracker.md
- Write findings to review-report.md Story Alignment section
- Update tracker status after review
- Return standardized status message"
```

---

## Phase 5: Final Verification

### Task 5.1: Verify All Changes

**Goal:** Ensure all files are correctly updated and consistent.

- [ ] **Step 1: Review all modified files**

```bash
git diff --stat
```

Verify all 7 agent files are modified.

- [ ] **Step 2: Create summary commit**

```bash
git add -A
git commit -m "feat: complete code review mechanism optimization

Phase 1 - Foundation:
- main.agent.md: Single file registry, no redundancy, smart pauses
- classifier.agent.md: Adapted to review-tracker.md

Phase 2 - Jira Enhancement:
- jira.agent.md: Batch clarification loop with detection algorithm

Phase 3 - Sub-Agent Standardization:
- quality-review.agent.md: JSON input protocol
- sql-review.agent.md: JSON input protocol
- sp-review.agent.md: JSON input protocol
- requirement-review.agent.md: New input sources

Changes:
- Consolidate 4 files to 2 (tracker + report)
- Single source of truth for file classification
- Interactive clarification for unclear requirements
- Graceful error recovery
- Smart pause points by logical file groups
- Standardized sub-agent I/O protocol"
```

---

## Summary

### Tasks Completed

| Phase | Task | Files Modified |
|-------|------|----------------|
| 1.1 | main.agent.md - File Registry | 1 |
| 1.2 | main.agent.md - Step 4 Simplify | 1 |
| 1.3 | main.agent.md - Step 5 Update | 1 |
| 1.4 | main.agent.md - Step 6 Finalize | 1 |
| 1.5 | main.agent.md - Error Recovery | 1 |
| 1.6 | main.agent.md - Error Summary | 1 |
| 2.1 | classifier.agent.md - New Input | 1 |
| 3.1 | jira.agent.md - Detection | 1 |
| 3.2 | jira.agent.md - Clarification Loop | 1 |
| 3.3 | jira.agent.md - Error Handling | 1 |
| 4.1 | quality-review.agent.md - JSON Protocol | 1 |
| 4.2 | sql-review.agent.md - JSON Protocol | 1 |
| 4.3 | sp-review.agent.md - JSON Protocol | 1 |
| 4.4 | requirement-review.agent.md - New Input | 1 |
| 5.1 | Final Verification | - |

**Total:** 7 files modified, 15 tasks

### Success Criteria Verification

- ✅ Jira agent can detect unclear requirements and ask clarifying questions (batch mode)
- ✅ File classification happens only once (no redundancy)
- ✅ Review session uses only 2 files instead of 4
- ✅ Error recovery provides graceful degradation options
- ✅ Cancel logic uses smart pause points (logical file groups)
- ✅ All sub-agents follow standardized I/O protocol (JSON input)
- ✅ No functionality lost during refactoring
