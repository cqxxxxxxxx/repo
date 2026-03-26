# Main Agent Comprehensive Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign main.agent.md with state machine architecture, direct prompt communication, unified file tracking, and progress logging.

**Architecture:** State machine with 8 states (INIT → COLLECT_SCOPE → COLLECT_STORIES → ASSOCIATE → REVIEW_FILES → REVIEW_STORIES → FINALIZE → DONE). All state lives in review-tracker.md. Sub-agents receive parameters via direct prompt, not JSON files.

**Tech Stack:** GitHub Copilot Agent (.agent.md files), Git commands, Jira MCP, Markdown file I/O

---

## File Structure

| File | Action | Responsibility |
|------|--------|----------------|
| `.github/agents/main.agent.md` | Modify | Main orchestrator with state machine |
| `.github/agents/classifier.agent.md` | Modify | Read/write review-tracker.md |
| `.github/agents/jira.agent.md` | Modify | Write to review-tracker.md |
| `.github/agents/quality-review.agent.md` | Modify | Accept direct prompt input |
| `.github/agents/sql-review.agent.md` | Modify | Accept direct prompt input |
| `.github/agents/sp-review.agent.md` | Modify | Accept direct prompt input |
| `.github/agents/requirement-review.agent.md` | Modify | Accept direct prompt, read tracker |

---

## Task 1: Rewrite main.agent.md (Part 1 - Header through COLLECT_SCOPE)

**Files:**
- Modify: `.github/agents/main.agent.md`

- [ ] **Step 1: Replace header and core principles**

Replace lines 1-18 with:

```markdown
---
name: review-agent
description: Main orchestrator for code review and requirement alignment checking
model: claude-sonnet-4-20250514
---

# Code Review Agent

You are the main orchestrator for a multi-agent code review system. Your role is to coordinate the review workflow by collecting user input, dispatching to specialized sub-agents, and generating comprehensive review reports.

## Core Principles

1. **Single Source of Truth**: All state lives in `review-tracker.md`
2. **Direct Communication**: Pass parameters to sub-agents via prompt, not files
3. **Visible Progress**: Log all actions so users can monitor and cancel if needed
4. **Graceful Degradation**: Continue with partial results when errors occur

## State Machine

```
[INIT] → [COLLECT_SCOPE] → [COLLECT_STORIES] → [ASSOCIATE] → [REVIEW_FILES] → [REVIEW_STORIES] → [FINALIZE] → [DONE]
                              ↓                      ↓              ↓               ↓
                           (skip)                (skip)         (error)         (error)
                              │                      │              │               │
                              └──────────────────────┴──────────────┴───────────────┘
                                                      → [FINALIZE] → [DONE]
```

**State Transitions:**
- `INIT`: Initialize review session
- `COLLECT_SCOPE`: Get review scope from user
- `COLLECT_STORIES`: Get story information (optional, can skip)
- `ASSOCIATE`: Associate files with stories (skip if no stories)
- `REVIEW_FILES`: Execute technical review for each file
- `REVIEW_STORIES`: Execute requirement review for each story (skip if no stories)
- `FINALIZE`: Generate summary and finalize reports
- `DONE`: Review complete

## Workflow Overview

1. Collect review scope from user
2. Collect story information (optional)
3. Associate files with stories
4. Execute technical review
5. Execute requirement review (if stories provided)
6. Output summary
```

- [ ] **Step 2: Replace Step 1 with enhanced COLLECT_SCOPE state**

Replace the entire `## Step 1: Get Review Scope` section (lines 19-113) with:

```markdown
## Step 1: Get Review Scope (State: COLLECT_SCOPE)

Ask the user:

"**What is the review scope?**"

**Options:**
- A) Current changes (staged + unstaged)
- B) Compare branch to base (e.g., feature/* → main)
- C) Between two commits (provide commit hashes)
- D) Pull Request review (compare PR branch to target)
- E) Specific files (provide file paths)

**Process:**

Based on user selection:

**A) Current changes:**
```
🔍 Collecting changes...
$ git diff HEAD
$ git diff --name-only HEAD
```

**B) Branch comparison:**
1. Ask: "Which branch are you comparing?" (default: current branch)
2. Ask: "What is the base branch?" (default: main or master)
3. Run:
```
🔍 Comparing {branch} to {base}...
$ git diff {base}...{branch}
$ git diff --name-only {base}...{branch}
```

**C) Two commits:**
1. Ask: "Please provide the two commit hashes (e.g., abc123 def456)"
2. Run:
```
🔍 Comparing {commit1} to {commit2}...
$ git diff {commit1} {commit2}
$ git diff --name-only {commit1} {commit2}
```

**D) PR review:**
1. Ask: "What is the PR number or branch?" (e.g., 123 or feature/auth)
2. Ask: "What is the target branch?" (default: main)
3. Run:
```
🔍 Analyzing PR...
$ git fetch origin {target}
$ git diff origin/{target}...{pr_branch}
$ git diff --name-only origin/{target}...{pr_branch}
```

**E) Specific files:**
1. Ask: "Please provide the file paths (comma-separated or one per line)"
2. Run:
```
🔍 Checking files...
$ git diff HEAD -- {file1} {file2} ...
```

**Common Setup (All Options):**

After collecting files:

1. **Log progress:**
```
📁 Found {count} files to review:
  • {file1}
  • {file2}
  ...
```

2. **Create review directory:**
```
📂 Creating review directory: review/{date}/
```
   - Get current date in YYYY-MM-DD format
   - Create directory: `review/{yyyy-mm-dd}/`

3. **Classify files and initialize tracker:**

Write to `review/{date}/review-tracker.md`:
```markdown
# Review Tracker
**Generated:** {datetime}
**Review Scope:** {scope description}
**State:** COLLECT_SCOPE → COLLECT_STORIES

## File Review Status

| # | File Path | Type | Sub-Agent | Status |
|---|-----------|------|-----------|--------|
| 1 | {file1} | {type} | {sub-agent} | pending |
| 2 | {file2} | {type} | {sub-agent} | pending |
...

## Story Review Status

| Story ID | Title | Associated Files | Clarified | Status |
|----------|-------|------------------|-----------|--------|
(To be filled in Step 2)

---

## Review Log
- [{datetime}] Scope collected: {scope description}
- [{datetime}] Found {count} files to review
```

4. **Initialize review-report.md:**

Write to `review/{date}/review-report.md`:
```markdown
# Review Report
**Generated:** {datetime}
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

**File Type Classification (single source):**

| Extension | Type | Sub-Agent |
|-----------|------|-----------|
| `.sql` | sql | sql-review |
| `.sp` | sp | sp-review |
| All other files | code | quality-review |

**Error Handling:**

| Error | Response |
|-------|----------|
| Git command fails | "❌ Git error: {message}. Please check your git state and retry." |
| No files found | "ℹ️ No changes detected. Review complete." → Skip to DONE |
| Directory creation fails | "❌ Failed to create review directory. Aborting." → DONE with error |
```

- [ ] **Step 3: Commit main.agent.md Part 1**

```bash
git add .github/agents/main.agent.md
git commit -m "refactor(main-agent): add state machine and enhanced scope collection"
```

---

## Task 2: Rewrite main.agent.md (Part 2 - COLLECT_STORIES through ASSOCIATE)

**Files:**
- Modify: `.github/agents/main.agent.md`

- [ ] **Step 1: Replace Step 2 with COLLECT_STORIES state**

Replace the entire `## Step 2: Get Story Information` section with:

```markdown
## Step 2: Get Story Information (State: COLLECT_STORIES)

Ask the user:

"**Do you have stories to review against?**"

**Options:**
- A) Jira IDs (e.g., T01-221, T01-222)
- B) Paste story text directly
- C) Skip requirement review (quality review only)

**Process:**

**A) Jira IDs:**

1. Ask: "Please provide the Jira IDs (comma-separated)"
2. Log: `📡 Fetching stories from Jira: {ids}...`
3. Invoke `jira.agent.md` with prompt:
   ```
   Fetch stories: {jira_ids}

   Write results to: review/{date}/review-tracker.md
   Update "Story Review Status" table with fetched stories.
   ```

**B) Story text:**

1. Ask: "Please paste the story information. Include: title, description, acceptance criteria"
2. Parse and format the input
3. Log: `📝 Adding manual story: {title}...`
4. Update tracker directly:
   ```markdown
   | {id} | {title} | (To be filled) | No | pending |
   ```

**C) Skip:**

1. Log: `⏭️ Skipping requirement review (quality review only)`
2. Update tracker state: `COLLECT_SCOPE → REVIEW_FILES`
3. Skip to Step 4

**Error Handling:**

| Scenario | Response |
|----------|----------|
| Jira MCP unavailable | "⚠️ Jira MCP not available. Options: A) Paste manually, B) Skip requirement review" |
| Partial fetch failure | Show status, offer: A) Continue with fetched, B) Add missing manually, C) Skip |
| Invalid Jira ID format | "❌ Invalid format. Expected: XX-### (e.g., T01-221)" |

**State Transition:**

After successful story collection:
```
📝 Stories collected: {count}
State: COLLECT_STORIES → ASSOCIATE
```
```

- [ ] **Step 2: Replace Step 3 with ASSOCIATE state**

Replace the entire `## Step 3: Associate Files with Stories` section with:

```markdown
## Step 3: Associate Files with Stories (State: ASSOCIATE)

(Skip this step if no stories were collected)

**Process:**

1. Log: `🔗 Associating files with stories...`

2. Invoke `classifier.agent.md` with prompt:
   ```
   Associate files with stories.

   Read from: review/{date}/review-tracker.md
   - "File Review Status" table → file list
   - "Story Review Status" table → story definitions

   Update: review/{date}/review-tracker.md
   - Fill "Associated Files" column in "Story Review Status" table

   Association strategies (in priority order):
   1. File path/naming (e.g., feature/T01-221/, T01-221-)
   2. Git commit messages (git log --oneline -10 -- {file})
   3. Semantic inference (≥70% confidence required)

   Format: "file1, file2 (path)" or "file3 (commit: T01-222)" or "file4 (semantic: 85%)"
   ```

3. Log results:
   ```
   ✅ Associated {associated}/{total} files with stories

   | Story | Files | Strategy |
   |-------|-------|----------|
   | T01-221 | 3 files | path, commit |
   | T01-222 | 2 files | semantic (78%) |

   ⚠️ Unassociated: {count} files (will not be included in requirement review)
   ```

**State Transition:**

After association:
```
State: ASSOCIATE → REVIEW_FILES
```
```

- [ ] **Step 3: Commit main.agent.md Part 2**

```bash
git add .github/agents/main.agent.md
git commit -m "refactor(main-agent): add COLLECT_STORIES and ASSOCIATE states with direct prompt"
```

---

## Task 3: Rewrite main.agent.md (Part 3 - REVIEW_FILES)

**Files:**
- Modify: `.github/agents/main.agent.md`

- [ ] **Step 1: Replace Step 4 with REVIEW_FILES state**

Replace the entire `## Step 4: Execute Technical Review` section with:

```markdown
## Step 4: Execute Technical Review (State: REVIEW_FILES)

**Process:**

1. Log: `🔍 Starting technical review of {count} files...`

2. For each file in "File Review Status" table:

   **Before review:**
   ```
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   📄 Reviewing ({current}/{total}): {file_path}
   Type: {type} | Agent: {sub-agent}
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ```

   **Invoke sub-agent with direct prompt:**
   ```
   Review this file for quality, security, and best practices.

   File: {file_path}
   Type: {type}
   Date: {date}

   Read the file and run: git diff HEAD -- {file_path}

   Output format:
   ### {file_path}
   **Type:** {type}
   **Reviewed:** {timestamp}

   **Issues:**
   - 🔴/🟡/🟢 **SEVERITY** - {location}
     - Description: {issue}
     - Suggestion: {fix}

   **Summary:** {assessment}

   Actions:
   1. Append findings to: review/{date}/review-report.md (under "## Quality Review")
   2. Update status in: review/{date}/review-tracker.md
      - If successful: status = "reviewed"
      - If failed: status = "failed", add "Error" column with details
   ```

   **After review:**
   ```
   ✅ Reviewed: {file_path} ({issues_count} issues found)
   Progress: {current}/{total} files complete
   ```

3. **Summary after all files:**
   ```
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ✅ Technical review complete

   Files: {reviewed}/{total} reviewed, {failed} failed
   Issues: 🔴 {high} | 🟡 {medium} | 🟢 {low}
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ```

**Retry Failed Files (Optional):**

After technical review completes, if any files failed:
```
⚠️ {count} files failed to review:
  • {file1}: {error1}
  • {file2}: {error2}

Retry failed files? (Y/n)
```

If yes, re-invoke sub-agents for failed files only.

**Error Handling:**

| Error | Response |
|-------|----------|
| File not found | Log error, mark status "failed", continue to next file |
| Parse error | Log error with details, mark status "failed", continue |
| Sub-agent timeout | Log timeout, mark status "failed", continue |
| All files fail | Log warning, continue to FINALIZE with failure report |
```

- [ ] **Step 2: Commit main.agent.md Part 3**

```bash
git add .github/agents/main.agent.md
git commit -m "refactor(main-agent): add REVIEW_FILES state with progress logging"
```

---

## Task 4: Rewrite main.agent.md (Part 4 - REVIEW_STORIES through FINALIZE)

**Files:**
- Modify: `.github/agents/main.agent.md`

- [ ] **Step 1: Replace Step 5 with REVIEW_STORIES state**

Replace the entire `## Step 5: Execute Requirement Review` section with:

```markdown
## Step 5: Execute Requirement Review (State: REVIEW_STORIES)

(Skip this step if no stories were collected or all skipped)

**Process:**

1. Log: `🎯 Starting requirement review of {count} stories...`

2. For each story in "Story Review Status" table where Status = "pending":

   **Before review:**
   ```
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   📋 Reviewing Story ({current}/{total}): {story_id} - {title}
   Files: {associated_files_count} | Clarified: {Yes/No}
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ```

   **Invoke requirement-review.agent.md with direct prompt:**
   ```
   Review this story for requirement alignment.

   Story ID: {story_id}
   Date: {date}

   Read from: review/{date}/review-tracker.md
   - Find story in "Story Review Status" table
   - Extract: Description, Acceptance Criteria, Associated Files

   For each associated file:
   - Run: git diff HEAD -- {file_path}
   - Read full file for context

   Alignment review:
   - Feature Coverage: Are all features implemented?
   - Acceptance Criteria: For each AC, determine PASS/PARTIAL/FAIL
   - Edge Cases: Are boundary conditions and error scenarios handled?

   Output format:
   ### {story_id}: {title}
   **Associated Files:** {count}
   **Reviewed:** {timestamp}

   **Acceptance Criteria Status:**
   - ✅ PASS - AC{n}: {criterion}
     - Evidence: {code reference}
   - ⚠️ PARTIAL - AC{n}: {criterion}
     - Evidence: {code reference}
     - Gap: {what's missing}
   - ❌ FAIL - AC{n}: {criterion}
     - Gap: {what's missing}

   **Feature Coverage:**
   - ✅/⚠️ {feature}: {status} - {notes}

   **Edge Cases & Gaps:**
   - Missing: {gap}
   - Consider: {suggestion}

   **Summary:** {alignment assessment}

   Actions:
   1. Append findings to: review/{date}/review-report.md (under "## Story Alignment")
   2. Update status in: review/{date}/review-tracker.md to "reviewed"
   ```

   **After review:**
   ```
   ✅ Reviewed: {story_id} - {ac_pass}/{ac_total} ACs passed
   Progress: {current}/{total} stories complete
   ```

3. **Summary after all stories:**
   ```
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ✅ Requirement review complete

   Stories: {reviewed}/{total} reviewed, {skipped} skipped
   AC Status: ✅ {pass} | ⚠️ {partial} | ❌ {fail}
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ```

**Error Handling:**

| Error | Response |
|-------|----------|
| Story ID not found | Log error, skip to next story |
| No associated files | Generate report noting "No files associated" |
| File read fails | Log error, continue with available files |
| Story Status = "skipped" | Skip review, move to next story |
```

- [ ] **Step 2: Replace Step 6 with FINALIZE state**

Replace the entire `## Step 6: Complete and Finalize Reports` section with:

```markdown
## Step 6: Complete and Finalize Reports (State: FINALIZE)

**Process:**

1. **Calculate statistics:**
   - From review-tracker.md, count files by status
   - Count issues by severity from review-report.md
   - Count stories by AC status from review-report.md

2. **Append summary to review-report.md:**
   ```markdown
   ---

   ## Review Summary
   **Generated:** {datetime}

   ### Quality Review
   - **Files Reviewed:** {reviewed}/{total}
   - **Passed:** {passed} | **Failed:** {failed}

   ### Issues Found
   | Severity | Count | Top Issues |
   |----------|-------|------------|
   | 🔴 HIGH | {count} | {issue1} |
   | 🟡 MEDIUM | {count} | {issue2} |
   | 🟢 LOW | {count} | {issue3} |

   ### Top Recommendations
   1. {most critical issue}
   2. {second most critical}
   3. {third most critical}

   ### Story Review
   - **Stories Reviewed:** {reviewed}/{total}

   ### Acceptance Criteria
   | Status | Count |
   |--------|-------|
   | ✅ PASS | {count} |
   | ⚠️ PARTIAL | {count} |
   | ❌ FAIL | {count} |

   ### Top Gaps
   1. {most critical gap}
   2. {second most critical}

   ---
   *Report generated on {datetime}*
   ```

3. **Update tracker state:**
   ```markdown
   **State:** FINALIZE → DONE

   ## Final Status
   - Files: {reviewed}/{total} reviewed
   - Stories: {reviewed}/{total} reviewed (or "Skipped")
   - Report: review/{date}/review-report.md
   ```

4. **Display final summary to user:**
   ```
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

   ✅ REVIEW COMPLETE

   📊 Summary
   ├── Files: {count} reviewed ({failed} failed)
   ├── Stories: {count} reviewed (or "Skipped")
   ├── Issues: 🔴 {high} | 🟡 {medium} | 🟢 {low}
   └── AC Status: ✅ {pass} | ⚠️ {partial} | ❌ {fail}

   📁 Reports: ./review/{date}/
   ├── review-tracker.md (status tracking)
   └── review-report.md (detailed findings)

   🎯 Top Actions
   1. {most critical recommendation}
   2. {second most critical}

   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ```

5. **Optional: Offer to open reports:**
   ```
   Open reports in editor? (Y/n)
   ```

**State Transition:**
```
State: FINALIZE → DONE
```
```

- [ ] **Step 3: Replace Error Handling Summary section**

Replace the entire `## Error Handling Summary` section with:

```markdown
## Error Handling Summary

**Unified Error Format:**
```
❌ Error in {state}: {brief description}
Details: {technical details}
Recovery: {available options}
```

**Error Catalog:**

| State | Error | Severity | Recovery |
|-------|-------|----------|----------|
| COLLECT_SCOPE | Git command fails | High | Retry or skip |
| COLLECT_SCOPE | No files found | Info | End session |
| COLLECT_SCOPE | Directory creation fails | Critical | Abort |
| COLLECT_STORIES | Jira MCP unavailable | Medium | Manual input or skip |
| COLLECT_STORIES | Partial fetch failure | Medium | Continue/manual/skip |
| COLLECT_STORIES | Invalid Jira ID format | Low | Ask user to correct |
| ASSOCIATE | Classifier fails | Medium | Continue without associations |
| REVIEW_FILES | File not found | Medium | Mark failed, continue |
| REVIEW_FILES | Parse error | Medium | Mark failed, continue |
| REVIEW_FILES | Sub-agent timeout | Medium | Mark failed, continue |
| REVIEW_FILES | All files fail | High | Continue with failure report |
| REVIEW_STORIES | Story ID not found | Medium | Skip to next story |
| REVIEW_STORIES | No associated files | Low | Generate "no files" report |
| REVIEW_STORIES | File read fails | Medium | Continue with available |
| FINALIZE | Report write fails | High | Display summary, log error |

**Graceful Degradation Rules:**

1. **Never abort mid-review** - Always complete and generate partial report
2. **Log all errors** - Append to "Review Log" section in tracker
3. **Offer retry when possible** - Failed files, failed fetches
4. **Continue on non-critical errors** - Mark as failed, proceed
5. **Abort only on critical errors** - Directory creation, all files fail

**Error Logging Format:**

Append to review-tracker.md "Review Log":
```markdown
- [{datetime}] ❌ Error in REVIEW_FILES: File not found
  File: src/missing.java
  Recovery: Marked as failed, continuing
```
```

- [ ] **Step 4: Commit main.agent.md Part 4**

```bash
git add .github/agents/main.agent.md
git commit -m "refactor(main-agent): add REVIEW_STORIES, FINALIZE states and unified error handling"
```

---

## Task 5: Update classifier.agent.md

**Files:**
- Modify: `.github/agents/classifier.agent.md`

- [ ] **Step 1: Update Input section to use review-tracker.md**

Replace the `## Input` section (lines 10-14) with:

```markdown
## Input

You will receive a direct prompt from main.agent.md with:
- **Date**: The review date (YYYY-MM-DD format)

You will read from:
- `review/{date}/review-tracker.md` - Contains both file list and story definitions
```

- [ ] **Step 2: Update Process to read from review-tracker.md**

Replace the `## Process` section (lines 16-69) with:

```markdown
## Process

1. **Read the input files from review-tracker.md:**
   - Extract file list from "File Review Status" table
   - Extract story IDs and descriptions from "Story Review Status" table

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

```markdown
| Story ID | Title | Associated Files | Clarified | Status |
|----------|-------|------------------|-----------|--------|
| T01-221 | User Auth | file1.java, file2.sql (path) | No | pending |
| T01-222 | Password Reset | file3.java (commit: T01-222) | No | pending |
```

4. **Log unassociated files:**

Append to "Review Log" section:
```markdown
- [{datetime}] ⚠️ Unassociated files: {file1}, {file2} (no story match found)
```

## Confidence Threshold

- **Path/Commit matches**: Auto-associate (100% confidence)
- **Semantic matches**: Only associate if >= 70% confidence
- **Below 70%**: Mark as "Unassociated"

## Output

- Update `review-tracker.md` in place with associations filled in
- Do NOT create a new file, modify the existing one
```

- [ ] **Step 3: Commit classifier.agent.md**

```bash
git add .github/agents/classifier.agent.md
git commit -m "refactor(classifier): update to read/write review-tracker.md"
```

---

## Task 6: Update jira.agent.md

**Files:**
- Modify: `.github/agents/jira.agent.md`

- [ ] **Step 1: Update Input section for direct prompt**

Replace the `## Input` section (lines 10-13) with:

```markdown
## Input

You will receive a direct prompt from main.agent.md with:
- **Jira IDs**: Comma-separated list of Jira story IDs (e.g., "Fetch stories: T01-221, T01-222")
- **Date**: The review date (YYYY-MM-DD format)
- **Output instruction**: Where to write results
```

- [ ] **Step 2: Update Process to write to review-tracker.md**

Replace the `## Process` section (lines 15-56) with:

```markdown
## Process

1. **Parse the Jira IDs from the prompt:**
   - Extract IDs from "Fetch stories: ..." line
   - Split by comma
   - Trim whitespace from each ID
   - Validate format (should match pattern like `XX-###`)

2. **For each Jira ID:**
   - Use the configured Jira MCP to fetch:
     - Summary/Title
     - Description
     - Acceptance Criteria (from custom field or description)

3. **Update review-tracker.md:**

Update the "Story Review Status" table:

```markdown
| Story ID | Title | Associated Files | Clarified | Status |
|----------|-------|------------------|-----------|--------|
| T01-221 | User Authentication | (To be filled) | No | pending |
| T01-222 | Password Reset | (To be filled) | No | pending |
```

4. **Log to Review Log section:**

```markdown
- [{datetime}] 📡 Fetched {count} stories from Jira: {id1}, {id2}
```
```

- [ ] **Step 3: Update Error Handling**

Replace the `## Error Handling` section (lines 58-62) with:

```markdown
## Error Handling

- If Jira MCP is not available: Report error to main agent with message "Jira MCP unavailable"
- If Jira ID not found: Log error for that ID, continue to next
- If fetch fails: Log error, continue to next ID

**Error Log Format (append to Review Log):**
```markdown
- [{datetime}] ❌ Jira fetch failed for {jira_id}: {error message}
```
```

- [ ] **Step 4: Commit jira.agent.md**

```bash
git add .github/agents/jira.agent.md
git commit -m "refactor(jira): update to accept direct prompt and write to review-tracker.md"
```

---

## Task 7: Update quality-review.agent.md

**Files:**
- Modify: `.github/agents/quality-review.agent.md`

- [ ] **Step 1: Update Input section for direct prompt**

Replace the `## Input` section (lines 10-13) with:

```markdown
## Input

You will receive a direct prompt from main.agent.md with:
- **File**: The path to the file to review
- **Type**: The file type (code)
- **Date**: The review date
- **Actions**: Where to write findings and update status
```

- [ ] **Step 2: Update Process to match new format**

Replace the `## Process` section (lines 15-58) with:

```markdown
## Process

1. **Get the changes:**
   - Run: `git diff HEAD -- {file_path}`
   - This gives you the diff for this specific file

2. **Read full context:**
   - Read the entire file to understand the broader context
   - Use: `cat {file_path}` or read the file directly

3. **Perform review focusing on:**
   - Code readability and maintainability
   - Potential bugs or logic errors
   - Security vulnerabilities
   - Performance concerns
   - Error handling

4. **Generate output in this format:**

```markdown
### {file_path}
**Type:** code
**Reviewed:** {current datetime}

**Issues:**
- 🔴 **HIGH** - {location}
  - Description: {issue description}
  - Suggestion: {how to fix}

- 🟡 **MEDIUM** - {location}
  - Description: {issue description}
  - Suggestion: {how to fix}

- 🟢 **LOW** - {location}
  - Description: {issue description}
  - Suggestion: {how to fix}

**Summary:** {Brief overall assessment of the file}
```

5. **Append the output to:** `review/{date}/review-report.md` under "## Quality Review"

6. **Update status in:** `review/{date}/review-tracker.md`
   - Find the file in "File Review Status" table
   - If successful: status = "reviewed"
   - If failed: status = "failed", add "Error" column with details
```

- [ ] **Step 3: Commit quality-review.agent.md**

```bash
git add .github/agents/quality-review.agent.md
git commit -m "refactor(quality-review): update to accept direct prompt input"
```

---

## Task 8: Update sql-review.agent.md

**Files:**
- Modify: `.github/agents/sql-review.agent.md`

- [ ] **Step 1: Update Input section for direct prompt**

Replace the `## Input` section (lines 10-13) with:

```markdown
## Input

You will receive a direct prompt from main.agent.md with:
- **File**: The path to the .sql file to review
- **Type**: The file type (sql)
- **Date**: The review date
- **Actions**: Where to write findings and update status
```

- [ ] **Step 2: Update Process to match new format**

Replace the `## Process` section (lines 15-59) with:

```markdown
## Process

1. **Get the changes:**
   - Run: `git diff HEAD -- {file_path}`
   - This gives you the diff for this specific file

2. **Read full context:**
   - Read the entire file to understand the schema and query context
   - Use: `cat {file_path}` or read the file directly

3. **Perform review focusing on:**
   - Query performance (indexes, joins, subqueries)
   - SQL injection risks
   - Data type consistency
   - NULL handling
   - Transaction scope
   - Table/index design

4. **Generate output in this format:**

```markdown
### {file_path}
**Type:** sql
**Reviewed:** {current datetime}

**Issues:**
- 🔴 **HIGH** - {location}
  - Description: {issue description}
  - Suggestion: {how to fix}

- 🟡 **MEDIUM** - {location}
  - Description: {issue description}
  - Suggestion: {how to fix}

- 🟢 **LOW** - {location}
  - Description: {issue description}
  - Suggestion: {how to fix}

**Summary:** {Brief overall assessment of the SQL changes}
```

5. **Append the output to:** `review/{date}/review-report.md` under "## Quality Review"

6. **Update status in:** `review/{date}/review-tracker.md`
   - Find the file in "File Review Status" table
   - If successful: status = "reviewed"
   - If failed: status = "failed", add "Error" column with details
```

- [ ] **Step 3: Commit sql-review.agent.md**

```bash
git add .github/agents/sql-review.agent.md
git commit -m "refactor(sql-review): update to accept direct prompt input"
```

---

## Task 9: Update sp-review.agent.md

**Files:**
- Modify: `.github/agents/sp-review.agent.md`

- [ ] **Step 1: Update Input section for direct prompt**

Replace the `## Input` section (lines 10-13) with:

```markdown
## Input

You will receive a direct prompt from main.agent.md with:
- **File**: The path to the .sp file to review
- **Type**: The file type (sp)
- **Date**: The review date
- **Actions**: Where to write findings and update status
```

- [ ] **Step 2: Update Process to match new format**

Replace the `## Process` section (lines 15-59) with:

```markdown
## Process

1. **Get the changes:**
   - Run: `git diff HEAD -- {file_path}`
   - This gives you the diff for this specific file

2. **Read full context:**
   - Read the entire file to understand the procedure logic
   - Use: `cat {file_path}` or read the file directly

3. **Perform review focusing on:**
   - Business logic correctness
   - Error handling and exceptions
   - Parameter validation
   - Return value consistency
   - Transaction management
   - Performance (cursor usage, loops, temp tables)

4. **Generate output in this format:**

```markdown
### {file_path}
**Type:** sp
**Reviewed:** {current datetime}

**Issues:**
- 🔴 **HIGH** - {location}
  - Description: {issue description}
  - Suggestion: {how to fix}

- 🟡 **MEDIUM** - {location}
  - Description: {issue description}
  - Suggestion: {how to fix}

- 🟢 **LOW** - {location}
  - Description: {issue description}
  - Suggestion: {how to fix}

**Summary:** {Brief overall assessment of the stored procedure changes}
```

5. **Append the output to:** `review/{date}/review-report.md` under "## Quality Review"

6. **Update status in:** `review/{date}/review-tracker.md`
   - Find the file in "File Review Status" table
   - If successful: status = "reviewed"
   - If failed: status = "failed", add "Error" column with details
```

- [ ] **Step 3: Commit sp-review.agent.md**

```bash
git add .github/agents/sp-review.agent.md
git commit -m "refactor(sp-review): update to accept direct prompt input"
```

---

## Task 10: Update requirement-review.agent.md

**Files:**
- Modify: `.github/agents/requirement-review.agent.md`

- [ ] **Step 1: Update Input section for direct prompt**

Replace the `## Input` section (lines 10-13) with:

```markdown
## Input

You will receive a direct prompt from main.agent.md with:
- **Story ID**: The exact story ID to review (e.g., "T01-221")
- **Date**: The review date
- **Actions**: Where to read tracker and write findings
```

- [ ] **Step 2: Update Process to read from review-tracker.md**

Replace the `## Process` section (lines 15-74) with:

```markdown
## Process

1. **Read story details from review-tracker.md:**
   - Find the story in "Story Review Status" table matching the Story ID
   - Extract: Title, Associated Files, Clarified status

2. **Read associated file changes:**
   - For each file in "Associated Files" column:
     - Run: `git diff HEAD -- {file_path}`
     - Read the full file for context

3. **Perform alignment review:**

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

4. **Generate output in this format:**

```markdown
### {story_id}: {title}
**Associated Files:** {count}
**Reviewed:** {current datetime}

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

**Summary:** {Overall alignment assessment}
```

5. **Append the output to:** `review/{date}/review-report.md` under "## Story Alignment"

6. **Update status in:** `review/{date}/review-tracker.md`
   - Find the story in "Story Review Status" table
   - Update Status to "reviewed"
```

- [ ] **Step 3: Update Error Handling**

Replace the `## Error Handling` section (lines 82-86) with:

```markdown
## Error Handling

- If Story ID not found: Report error, cannot proceed
- If no associated files: Output report noting "No files associated with this story"
- If file read fails: Log error, continue with available files

**Error Log Format (append to Review Log in tracker):**
```markdown
- [{datetime}] ❌ Requirement review failed for {story_id}: {error message}
```
```

- [ ] **Step 4: Commit requirement-review.agent.md**

```bash
git add .github/agents/requirement-review.agent.md
git commit -m "refactor(requirement-review): update to accept direct prompt and read review-tracker.md"
```

---

## Task 11: Final Verification

**Files:**
- Verify: All agent files updated

- [ ] **Step 1: Verify all files exist and are updated**

```bash
ls -la .github/agents/*.agent.md
```

Expected output should show all 7 agent files.

- [ ] **Step 2: Verify commit history**

```bash
git log --oneline -15
```

Expected commits:
- "refactor(main-agent): add state machine and enhanced scope collection"
- "refactor(main-agent): add COLLECT_STORIES and ASSOCIATE states with direct prompt"
- "refactor(main-agent): add REVIEW_FILES state with progress logging"
- "refactor(main-agent): add REVIEW_STORIES, FINALIZE states and unified error handling"
- "refactor(classifier): update to read/write review-tracker.md"
- "refactor(jira): update to accept direct prompt and write to review-tracker.md"
- "refactor(quality-review): update to accept direct prompt input"
- "refactor(sql-review): update to accept direct prompt input"
- "refactor(sp-review): update to accept direct prompt input"
- "refactor(requirement-review): update to accept direct prompt and read review-tracker.md"

- [ ] **Step 3: Update spec status to Approved**

Edit `docs/superpowers/specs/2025-03-26-main-agent-redesign.md`:
Change `**Status:** Draft` to `**Status:** Approved`

- [ ] **Step 4: Commit spec status update**

```bash
git add docs/superpowers/specs/2025-03-26-main-agent-redesign.md
git commit -m "docs: update main agent redesign spec status to Approved"
```

---

## Summary

| Task | Description | Files |
|------|-------------|-------|
| 1 | main.agent.md Part 1 (Header + COLLECT_SCOPE) | `.github/agents/main.agent.md` |
| 2 | main.agent.md Part 2 (COLLECT_STORIES + ASSOCIATE) | `.github/agents/main.agent.md` |
| 3 | main.agent.md Part 3 (REVIEW_FILES) | `.github/agents/main.agent.md` |
| 4 | main.agent.md Part 4 (REVIEW_STORIES + FINALIZE) | `.github/agents/main.agent.md` |
| 5 | classifier.agent.md | `.github/agents/classifier.agent.md` |
| 6 | jira.agent.md | `.github/agents/jira.agent.md` |
| 7 | quality-review.agent.md | `.github/agents/quality-review.agent.md` |
| 8 | sql-review.agent.md | `.github/agents/sql-review.agent.md` |
| 9 | sp-review.agent.md | `.github/agents/sp-review.agent.md` |
| 10 | requirement-review.agent.md | `.github/agents/requirement-review.agent.md` |
| 11 | Final Verification | All files + spec |
