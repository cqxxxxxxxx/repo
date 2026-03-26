# Main Agent Comprehensive Redesign

**Created:** 2025-03-26
**Status:** Approved
**Author:** Claude (Brainstorming Session)

## Executive Summary

This document outlines a comprehensive redesign of `main.agent.md` to improve prompt quality, robustness, maintainability, and efficiency. The redesign introduces:

1. **State Machine Architecture** - Explicit state transitions for predictable behavior
2. **Direct Prompt Communication** - Remove JSON file overhead for sub-agent invocation
3. **Unified File Tracking** - Consolidate to `review-tracker.md` as single source of truth
4. **Enhanced Scope Options** - Support PR review, branch diff, specific files
5. **Progress Logging** - Replace pause points with visible progress indicators
6. **Structured Error Handling** - Unified error catalog with recovery options

## Design Goals

- **Correctness**: Single source of truth, eliminate inconsistencies
- **Clarity**: Explicit state machine, clear instructions
- **User Experience**: Visible progress, no interruptions, graceful degradation
- **Maintainability**: Modular sections, standardized patterns
- **Efficiency**: Direct communication, reduced file I/O

## Architecture

### State Machine

```
[INIT] → [COLLECT_SCOPE] → [COLLECT_STORIES] → [ASSOCIATE] → [REVIEW_FILES] → [REVIEW_STORIES] → [FINALIZE] → [DONE]
                              ↓                      ↓              ↓               ↓
                           (skip)                (skip)         (error)         (error)
                              │                      │              │               │
                              └──────────────────────┴──────────────┴───────────────┘
                                                      → [FINALIZE] → [DONE]
```

**States:**
- `INIT`: Initialize review session
- `COLLECT_SCOPE`: Get review scope from user
- `COLLECT_STORIES`: Get story information (optional, can skip)
- `ASSOCIATE`: Associate files with stories (skip if no stories)
- `REVIEW_FILES`: Execute technical review for each file
- `REVIEW_STORIES`: Execute requirement review for each story (skip if no stories)
- `FINALIZE`: Generate summary and finalize reports
- `DONE`: Review complete

### Core Principles

1. **Single Source of Truth**: All state lives in `review-tracker.md`
2. **Direct Communication**: Pass parameters to sub-agents via prompt, not files
3. **Visible Progress**: Log all actions so users can monitor and cancel if needed
4. **Graceful Degradation**: Continue with partial results when errors occur

---

## Implementation Specification

### Section 1: Header & Role Definition

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

[Include state diagram as shown above]

**State Transitions:**
- `INIT`: Initialize review session
- `COLLECT_SCOPE`: Get review scope from user
- `COLLECT_STORIES`: Get story information (optional, can skip)
- `ASSOCIATE`: Associate files with stories (skip if no stories)
- `REVIEW_FILES`: Execute technical review for each file
- `REVIEW_STORIES`: Execute requirement review for each story (skip if no stories)
- `FINALIZE`: Generate summary and finalize reports
- `DONE`: Review complete
```

### Section 2: COLLECT_SCOPE State

**User Prompt:**
```
**What is the review scope?**

Options:
- A) Current changes (staged + unstaged)
- B) Compare branch to base (e.g., feature/* → main)
- C) Between two commits (provide commit hashes)
- D) Pull Request review (compare PR branch to target)
- E) Specific files (provide file paths)
```

**Process by Option:**

| Option | Commands | Notes |
|--------|----------|-------|
| A | `git diff HEAD`, `git diff --name-only HEAD` | Staged + unstaged |
| B | `git diff {base}...{branch}` | Ask for branch names |
| C | `git diff {commit1} {commit2}` | Ask for commits |
| D | `git fetch origin {target}`, `git diff origin/{target}...{pr_branch}` | Ask for PR |
| E | `git diff HEAD -- {files}` | Ask for file paths |

**Output Files:**

Create `review/{date}/review-tracker.md`:
```markdown
# Review Tracker
**Generated:** {datetime}
**Review Scope:** {scope description}
**State:** COLLECT_SCOPE → COLLECT_STORIES

## File Review Status

| # | File Path | Type | Sub-Agent | Status |
|---|-----------|------|-----------|--------|
| 1 | {file1} | {type} | {sub-agent} | pending |
...

## Story Review Status

| Story ID | Title | Associated Files | Clarified | Status |
|----------|-------|------------------|-----------|--------|
(To be filled in COLLECT_STORIES state)

---

## Review Log
- [{datetime}] Scope collected: {scope description}
- [{datetime}] Found {count} files to review
```

**File Type Classification:**

| Extension | Type | Sub-Agent |
|-----------|------|-----------|
| `.sql` | sql | sql-review |
| `.sp` | sp | sp-review |
| `.java`, `.ts`, `.js`, `.py`, etc. | code | quality-review |
| Other | code | quality-review |

### Section 3: COLLECT_STORIES State

**User Prompt:**
```
**Do you have stories to review against?**

Options:
- A) Jira IDs (e.g., T01-221, T01-222)
- B) Paste story text directly
- C) Skip requirement review (quality review only)
```

**Sub-Agent Invocation (Jira):**
```
Invoke jira.agent.md with prompt:

Fetch stories: {jira_ids}

Write results to: review/{date}/review-tracker.md
Update "Story Review Status" table with fetched stories.
```

**Error Handling:**

| Scenario | Response |
|----------|----------|
| Jira MCP unavailable | "⚠️ Jira MCP not available. Options: A) Paste manually, B) Skip requirement review" |
| Partial fetch failure | Show status, offer: A) Continue with fetched, B) Add missing manually, C) Skip |
| Invalid Jira ID format | "❌ Invalid format. Expected: XX-### (e.g., T01-221)" |

### Section 4: ASSOCIATE State

**Sub-Agent Invocation:**
```
Invoke classifier.agent.md with prompt:

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

### Section 5: REVIEW_FILES State

**Progress Logging:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📄 Reviewing ({current}/{total}): {file_path}
Type: {type} | Agent: {sub-agent}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Sub-Agent Invocation:**
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

**Retry Failed Files:**
After all files reviewed, if any failed:
```
⚠️ {count} files failed to review:
  • {file1}: {error1}
  • {file2}: {error2}

Retry failed files? (Y/n)
```

### Section 6: REVIEW_STORIES State

**Sub-Agent Invocation:**
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

### Section 7: FINALIZE State

**Review Summary Template:**
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

**Final User Display:**
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

### Section 8: Error Handling

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
1. Never abort mid-review - always complete and generate partial report
2. Log all errors - append to "Review Log" section in tracker
3. Offer retry when possible - failed files, failed fetches
4. Continue on non-critical errors - mark as failed, proceed
5. Abort only on critical errors - directory creation, all files fail

---

## Migration Guide

### Files to Update

1. **`.github/agents/main.agent.md`** - Complete rewrite following this spec
2. **`.github/agents/classifier.agent.md`** - Update to read/write review-tracker.md
3. **`.github/agents/jira.agent.md`** - Update to write to review-tracker.md
4. **`.github/agents/quality-review.agent.md`** - Accept direct prompt input
5. **`.github/agents/sql-review.agent.md`** - Accept direct prompt input
6. **`.github/agents/sp-review.agent.md`** - Accept direct prompt input
7. **`.github/agents/requirement-review.agent.md`** - Accept direct prompt input, read from review-tracker.md

### Breaking Changes

1. **Removed**: `sub-agent-input.json` file (replaced with direct prompt)
2. **Removed**: `quality-todos.md` and `story-todos.md` (consolidated into review-tracker.md)
3. **Removed**: Pause points (replaced with progress logging)
4. **Changed**: Sub-agent invocation now uses direct prompt instead of JSON file

---

## Success Criteria

- ✅ All agents use review-tracker.md as single source of truth
- ✅ No JSON file I/O for sub-agent communication
- ✅ 5 scope options available (current, branch, commits, PR, files)
- ✅ Progress logging before/after each review action
- ✅ No interruption pauses during review
- ✅ Unified error handling with recovery options
- ✅ State machine with explicit transitions
- ✅ Retry mechanism for failed file reviews

## Risks and Mitigation

| Risk | Mitigation |
|------|------------|
| Breaking existing sub-agent invocations | Update all sub-agents in same PR |
| Direct prompt may hit token limits | Keep prompts concise, reference files |
| No pause points may frustrate users | Progress logging provides visibility |
| State machine complexity | Clear documentation, visual diagram |

## Future Enhancements

- Parallel file review for independent files
- Configurable progress log verbosity
- Progress persistence for resuming interrupted reviews
- Integration with GitHub Issues, Azure DevOps
