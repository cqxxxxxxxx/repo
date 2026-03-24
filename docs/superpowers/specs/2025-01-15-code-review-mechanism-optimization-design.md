# Code Review Mechanism Optimization Design

**Created:** 2025-01-15
**Status:** Draft
**Author:** Claude (Superpowers Brainstorming)

## Executive Summary

This document outlines optimizations for the multi-agent code review system, focusing on:
1. Adding interactive clarification to the Jira agent
2. Eliminating redundancy in the main orchestrator
3. Simplifying file structure
4. Improving error recovery and user experience
5. Standardizing sub-agent communication

## Problem Statement

### Current Issues

1. **Redundant File Processing**: File type determination happens twice in main.agent.md (Step 1 lines 44-48 and Step 4 lines 183-186)
2. **Missing Jira Clarification**: jira.agent.md lacks the ability to ask users for clarification when requirements are unclear
3. **File Organization Overhead**: 4 files per review session creates cognitive load (quality-todos.md, quality-report.md, story-todos.md, story-report.md)
4. **Limited Error Recovery**: Jira fetch failures and sub-agent failures have limited recovery options
5. **Disruptive Cancel Logic**: Cancel check every 5 files interrupts natural workflow
6. **Inconsistent Sub-Agent Protocol**: Each sub-agent has different input/output patterns

## Design Goals

- **Correctness**: Eliminate redundant logic, ensure single source of truth
- **Simplicity**: Reduce file count, standardize patterns
- **User Experience**: Add clarification loops, improve error recovery, smart pause points
- **Maintainability**: Standardized sub-agent contracts for easier extension

## Proposed Solution

### 1. Jira Agent Enhancement - Interactive Clarification

**Location:** `.github/agents/jira.agent.md`

**Current Behavior:**
- Fetches story data from Jira
- Writes to file without validation
- No user interaction

**Enhanced Process:**

```markdown
## Process (Enhanced)

1. **Parse the Jira IDs** (unchanged)
   - Split input by comma
   - Trim whitespace
   - Validate format (XX-### pattern)

2. **For each Jira ID:**
   - Fetch story data from Jira MCP
   - **Validate Completeness:**
     - Check for missing/incomplete Acceptance Criteria
     - Identify ambiguous descriptions (vague terms, missing specifics)
     - Detect requirements lacking testability

3. **Interactive Clarification Loop:**

   If clarity issues found, present to user:

   "I found unclear requirements in {jira_id}. Would you like to clarify before review?"

   **Options:**
   - A) Provide clarifications now
   - B) Continue with current requirements
   - C) Skip this story

   **If user selects A:**
   For each unclear requirement:
   - Display: "Requirement: {requirement text}"
   - Ask specific question: "Please clarify: {specific question about ambiguity}"
   - Update story data with user's clarification

4. **Format and write output** (unchanged)
```

**Clarification Triggers:**
- Acceptance Criteria missing or contains only 1-2 vague items
- Description uses ambiguous terms: "appropriate", "reasonable", "as needed", "properly"
- No specific test cases or examples provided
- Requirements that could be interpreted multiple ways

**Example Clarification Flow:**
```
Agent: "I found unclear requirements in T01-221. Would you like to clarify before review?"
User: "A"

Agent: "Requirement: 'System should handle user authentication appropriately'
       Please clarify: What specific authentication methods should be supported?"
User: "Support email/password and OAuth (Google, GitHub)"

Agent: "Requirement: 'User experience should be responsive'
       Please clarify: What is the target response time and for which operations?"
User: "Page load < 2 seconds, API responses < 500ms"

Agent: "Clarifications recorded. Proceeding with review."
```

### 2. Main Agent - Eliminate Redundancy

**Location:** `.github/agents/main.agent.md`

**Current Issue:**
- Step 1 classifies files by type (lines 44-48)
- Step 4 re-classifies files to determine sub-agent (lines 183-186)
- Creates duplicate logic and potential inconsistency

**Solution: Single File Registry**

**Step 1 Enhanced:**
```markdown
5. Create file registry (replaces quality-todos.md):

**review-tracker.md:**
```markdown
# Review Tracker
**Generated:** {current datetime}
**Review Scope:** {scope description}

## File Review Status

| # | File Path | Type | Sub-Agent | Status |
|---|-----------|------|-----------|--------|
| 1 | src/UserService.java | code | quality-review | pending |
| 2 | db/migration.sql | sql | sql-review | pending |
| 3 | api/stored-procedure.sp | sp | sp-review | pending |
```

Type determination logic (single source):
- `.sql` extension → type = "sql", sub-agent = "sql-review"
- `.sp` extension → type = "sp", sub-agent = "sp-review"
- All other files → type = "code", sub-agent = "quality-review"
```

**Step 4 Simplified:**
```markdown
## Step 4: Execute Technical Review

For each file in review-tracker.md (process serially):

1. Read sub-agent from "Sub-Agent" column (no re-classification)
2. Invoke sub-agent with standardized input:
   - file_path
   - tracker_file: review/{date}/review-tracker.md
   - report_file: review/{date}/review-report.md
3. Wait for completion
4. Update status in tracker
5. Show progress feedback
6. Check for cancel at smart pause points
```

**Benefits:**
- Single classification logic
- No redundant file scanning
- Faster execution
- Consistent metadata

### 3. Simplified File Structure

**Current:** 4 files
- `quality-todos.md` - File tracking
- `quality-report.md` - Technical findings
- `story-todos.md` - Story tracking
- `story-report.md` - Requirement findings

**Proposed:** 2 files

**review-tracker.md:**
```markdown
# Review Tracker
**Generated:** 2025-01-15 10:30:00
**Review Scope:** Between commits abc123 and def456

## File Review Status
| # | File Path | Type | Sub-Agent | Status |
|---|-----------|------|-----------|--------|
| 1 | src/UserService.java | code | quality-review | reviewed |
| 2 | db/migration.sql | sql | sql-review | pending |

## Story Review Status
| Story ID | Title | Associated Files | Clarified | Status |
|----------|-------|------------------|-----------|--------|
| T01-221 | User Auth | 3 files | Yes | pending |
| T01-222 | Password Reset | 2 files | No | pending |
```

**review-report.md:**
```markdown
# Review Report
**Generated:** 2025-01-15 10:30:00
**Review Scope:** Between commits abc123 and def456

---

## Quality Review

### src/UserService.java
**Type:** Code
**Reviewed:** 2025-01-15 10:35:00

**Issues Found:**
- 🟡 MEDIUM: Method complexity too high (cyclomatic complexity: 15)
  - Location: authenticate() method, line 45
  - Recommendation: Extract validation logic to separate methods

**Positive Findings:**
- ✅ Proper error handling implemented
- ✅ Logging coverage is adequate

---

### db/migration.sql
**Type:** SQL
**Status:** pending

---

## Story Alignment

### T01-221: User Authentication
**Associated Files:** 3 files
**Review Date:** pending

**Acceptance Criteria Status:**
(To be filled by requirement-review agent)

---

## Review Summary

**Quality Review:**
- Files Reviewed: 1/2
- Issues: 🔴 0 | 🟡 1 | 🟢 0

**Story Review:**
- Stories: 0/2 reviewed

*Report last updated: 2025-01-15 10:35:00*
```

**Benefits:**
- 50% fewer files to manage
- Clear separation: tracking vs findings
- Single source of truth for status
- Easier navigation

### 4. Error Recovery and UX Improvements

**Jira Fetch Failure:**

```markdown
## Step 2: Get Story Information (Enhanced Error Handling)

If Jira MCP fetch fails:

1. **Graceful Degradation:**

   "Jira fetch failed for {jira_id}. How would you like to proceed?"

   **Options:**
   - A) Paste story details manually
   - B) Skip this story, continue with others
   - C) Abort requirement review (proceed with quality review only)

2. **Partial Success Handling:**

   If some stories fetched successfully:

   "Successfully fetched {n}/{total} stories:
   - ✅ T01-221: User Authentication
   - ❌ T01-222: Password Reset (fetch failed)

   Continue with available stories? (Y/n)"
```

**Smart Pause Points (Replaces "every 5 files"):**

```markdown
## Step 4: Execute Technical Review (Smart Pauses)

Pause and check for cancel at:
- After completing each logical file group (all SQL, all Java, etc.)
- After 10 files reviewed (whichever comes first)
- Before starting Step 5 (story review)

Prompt:
"Reviewed {group_name} files ({count} total). {remaining} files remaining.

Continue? (Y/n)"
```

**Sub-Agent Failure Recovery:**

```markdown
If sub-agent fails during review:

1. Log error in tracker with details
2. Mark file status as "failed"
3. Continue to next file
4. After completing all files, offer retry:

   "Review completed with {n} failures:
   - ❌ db/migration.sql: File not found
   - ❌ src/OldService.java: Parse error

   Would you like to retry these files?"
   - A) Retry failed files
   - B) Continue without retry
   - C) Show detailed error logs first
```

### 5. Standardized Sub-Agent Protocol

**Unified Input Contract:**

All sub-agents receive standardized input from orchestrator:

```json
{
  "file_path": "src/UserService.java",
  "file_type": "code",
  "review_date": "2025-01-15",
  "tracker_file": "review/2025-01-15/review-tracker.md",
  "report_file": "review/2025-01-15/review-report.md"
}
```

**Unified Output Contract:**

All sub-agents must:
1. Read only their assigned file + tracker
2. Perform specialized review
3. Append findings to report_file under section `### {filename}`
4. Update status in tracker_file
5. Return simple status message to orchestrator

**Return Message Format:**
```
"✅ Reviewed {filename} - {n} issues found"
"❌ Failed to review {filename} - {error reason}"
```

**Standardized Agent Template:**

```markdown
---
name: {agent-name}
description: {agent-purpose}
---

# {Agent Name} Agent

## Input
- File path (from orchestrator)
- File type
- Tracker file location
- Report file location

## Process

1. Read assigned file content
2. Perform specialized analysis
3. Generate findings in standard format
4. Append to report file
5. Update tracker status
6. Return status message

## Output Format

Append to review-report.md:

### {filename}
**Type:** {file_type}
**Reviewed:** {timestamp}

**Issues Found:**
- {severity} {issue_type}: {description}
  - Location: {specific location}
  - Recommendation: {fix suggestion}

**Positive Findings:**
- ✅ {positive_aspect}

---

## Error Handling
- If file not found: Return "❌ Failed - File not found"
- If parse error: Return "❌ Failed - Parse error: {details}"
- If analysis error: Log error, continue with partial results
```

## Implementation Plan

### Phase 1: Critical Enhancements
1. **Enhance jira.agent.md** - Add interactive clarification loop
2. **Refactor main.agent.md** - Eliminate redundancy, implement file registry

### Phase 2: Structural Improvements
3. **Consolidate file structure** - Implement 2-file model (tracker + report)
4. **Update classifier.agent.md** - Adapt to new tracker format

### Phase 3: UX Enhancements
5. **Implement error recovery** - Graceful degradation paths
6. **Smart pause points** - Replace arbitrary cancel logic

### Phase 4: Standardization
7. **Standardize all sub-agents** - Apply unified protocol to:
   - quality-review.agent.md
   - sql-review.agent.md
   - sp-review.agent.md
   - requirement-review.agent.md

## Files to Modify

1. `.github/agents/jira.agent.md` - Add clarification feature
2. `.github/agents/main.agent.md` - Major refactoring
3. `.github/agents/classifier.agent.md` - Adapt to new tracker
4. `.github/agents/quality-review.agent.md` - Standardize protocol
5. `.github/agents/sql-review.agent.md` - Standardize protocol
6. `.github/agents/sp-review.agent.md` - Standardize protocol
7. `.github/agents/requirement-review.agent.md` - Standardize protocol

## Success Criteria

- ✅ Jira agent can detect unclear requirements and ask clarifying questions
- ✅ File classification happens only once (no redundancy)
- ✅ Review session uses only 2 files instead of 4
- ✅ Error recovery provides graceful degradation options
- ✅ Cancel logic uses smart pause points
- ✅ All sub-agents follow standardized I/O protocol
- ✅ No functionality lost during refactoring
- ✅ Backward compatibility maintained for existing workflows

## Risks and Mitigation

**Risk:** Breaking existing sub-agent invocations
**Mitigation:** Implement adapter layer during transition, test thoroughly

**Risk:** Clarification loop adds too much friction
**Mitigation:** Make clarification optional (user can choose to skip)

**Risk:** File consolidation loses information
**Mitigation:** Ensure all tracker/report data preserved in new format

**Risk:** Smart pause points still feel disruptive
**Mitigation:** Make pause frequency configurable in future iteration

## Future Enhancements

- Parallel processing for independent files/stories (currently serial)
- Configurable pause frequency
- Progress persistence for resuming interrupted reviews
- Integration with other project management tools (GitHub Issues, Azure DevOps)
