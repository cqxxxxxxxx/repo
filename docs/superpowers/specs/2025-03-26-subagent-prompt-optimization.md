# Subagent Prompt Optimization Design

**Created:** 2025-03-26
**Status:** Draft
**Author:** Claude (Brainstorming Session)

## Executive Summary

This document outlines an optimization to reduce duplication between `main.agent.md` prompts and subagent content. The goal is to **minimize content in main.agent.md** that is already defined in subagent files, reducing context interference and improving maintainability.

## Problem Statement

### Current Issue

When `main.agent.md` invokes subagents, it includes detailed instructions that are **already present in the subagent files themselves**. This causes:

1. **Context Pollution** - Main agent's context gets cluttered with details that subagent will re-read
2. **Maintenance Burden** - Same content maintained in two places (main.agent.md + subagent)
3. **Inconsistency Risk** - If one is updated but not the other, behavior diverges
4. **Token Waste** - Main agent carries unnecessary context through entire session

### Affected Areas

| Location | Lines | Issue |
|----------|-------|-------|
| Step 4 (REVIEW_FILES) | 362-436 | Full output format template duplicated |
| Step 5 (REVIEW_STORIES) | 457-561 | Full alignment review instructions duplicated |

### Duplication Examples

**In main.agent.md Step 4:**
```markdown
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
```

**Already in quality-review.agent.md:**
- Process step 1: `git diff HEAD -- {file_path}` instruction
- Process step 4: Full output format template
- Process step 5: `Append to review-report.md` instruction
- Process step 6: `Update status in review-tracker.md` instruction

## Design Goals

1. **Single Source of Truth** - Each instruction exists in only one place
2. **Minimal Prompts** - Main agent passes only essential parameters
3. **Self-Contained Subagents** - Subagents have complete instructions
4. **Easy Maintenance** - Update subagent without touching main agent

## Proposed Solution

### Principle: Parameter-Passing Only

Main.agent.md should only pass:
- **What** to work on (file path, story ID)
- **Where** to read/write (tracker file, report file)
- **When** (date for file organization)

Subagent contains:
- **How** to do the work (process steps)
- **Output format** (templates)
- **Error handling** (recovery actions)

### Optimized Prompt Templates

#### Step 4 (REVIEW_FILES) - Optimized

**Before (75 lines):**
```markdown
**Invoke sub-agent with direct prompt:**
```
Review this file for quality, security, and best practices.

File: {file_path}
Type: {type}
Date: {date}

Read the file and run: git diff HEAD -- {file_path}

Output format:
### {file_path}
... (15 lines of format template) ...

Actions:
1. Append findings to: review/{date}/review-report.md (under "## Quality Review")
2. Update status in: review/{date}/review-tracker.md
   - If successful: status = "reviewed"
   - If failed: status = "failed", add "Error" column with details
```
```

**After (5 lines):**
```markdown
Invoke {sub-agent}.agent.md with:

```
Review: {file_path}
Type: {type}
Date: {date}
Report: review/{date}/review-report.md
Tracker: review/{date}/review-tracker.md
```
```

**Reduction:** 75 lines → 5 lines (93% reduction)

#### Step 5 (REVIEW_STORIES) - Optimized

**Before (105 lines):**
```markdown
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
... (30 lines of format template) ...

Actions:
1. Append findings to: review/{date}/review-report.md (under "## Story Alignment")
2. Update status in: review/{date}/review-tracker.md to "reviewed"
```
```

**After (5 lines):**
```markdown
Invoke requirement-review.agent.md with:

```
Story: {story_id}
Date: {date}
Tracker: review/{date}/review-tracker.md
Report: review/{date}/review-report.md
```
```

**Reduction:** 105 lines → 5 lines (95% reduction)

### Subagent Updates Required

Subagents need minimal updates to accept the simplified parameter format:

#### quality-review.agent.md Update

**Current Input section:**
```markdown
## Input

You will receive a direct prompt containing:
- **File**: The path to the file to review
- **Type**: The file type/category (e.g., typescript, python, config)
- **Date**: The review date (format: YYYY-MM-DD)
- **Actions**: List of actions to perform on the file
```

**Updated Input section:**
```markdown
## Input

You will receive a prompt in this format:
```
Review: {file_path}
Type: {type}
Date: {date}
Report: review/{date}/review-report.md
Tracker: review/{date}/review-tracker.md
```

Parse the prompt to extract parameters.
```

#### requirement-review.agent.md Update

**Current Input section:**
```markdown
## Input

You will receive a direct prompt from main.agent.md with:
- **Story ID**: The exact story ID to review (e.g., "T01-221")
- **Date**: The review date
- **Actions**: Where to read tracker and write findings
```

**Updated Input section:**
```markdown
## Input

You will receive a prompt in this format:
```
Story: {story_id}
Date: {date}
Tracker: review/{date}/review-tracker.md
Report: review/{date}/review-report.md
```

Parse the prompt to extract parameters.
```

### Complete Prompt Format Reference

All subagents should use this consistent format:

```markdown
Invoke {agent-name}.agent.md with:

```
{Key1}: {value1}
{Key2}: {value2}
Date: {date}
Tracker: review/{date}/review-tracker.md
Report: review/{date}/review-report.md
```
```

**Key Benefits:**
1. Consistent format across all subagents
2. Easy to parse (key: value pairs)
3. Minimal token usage
4. Self-documenting

## Files to Modify

| File | Change | Lines Changed |
|------|--------|----------------|
| `.github/agents/main.agent.md` | Simplify Step 4 prompt | -70 lines |
| `.github/agents/main.agent.md` | Simplify Step 5 prompt | -100 lines |
| `.github/agents/quality-review.agent.md` | Update Input section | +5 lines |
| `.github/agents/sql-review.agent.md` | Update Input section | +5 lines |
| `.github/agents/sp-review.agent.md` | Update Input section | +5 lines |
| `.github/agents/requirement-review.agent.md` | Update Input section | +5 lines |

**Net Reduction:** ~165 lines in main.agent.md, +25 lines across subagents

## Implementation Tasks

### Task 1: Update main.agent.md Step 4

Replace the REVIEW_FILES invocation section (lines 362-436) with:

```markdown
2. For each file in "File Review Status" table:

   **Before review:**
   ```
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   📄 Reviewing ({current}/{total}): {file_path}
   Type: {type} | Agent: {sub-agent}
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ```

   **Invoke sub-agent:**
   ```
   Invoke {sub-agent}.agent.md with:

   Review: {file_path}
   Type: {type}
   Date: {date}
   Report: review/{date}/review-report.md
   Tracker: review/{date}/review-tracker.md
   ```

   **After review:**
   ```
   ✅ Reviewed: {file_path} ({issues_count} issues found)
   Progress: {current}/{total} files complete
   ```
```

### Task 2: Update main.agent.md Step 5

Replace the REVIEW_STORIES invocation section (lines 457-561) with:

```markdown
2. For each story in "Story Review Status" table where Status = "pending":

   **Before review:**
   ```
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   📋 Reviewing Story ({current}/{total}): {story_id} - {title}
   Files: {associated_files_count} | Clarified: {Yes/No}
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ```

   **Invoke requirement-review.agent.md:**
   ```
   Story: {story_id}
   Date: {date}
   Tracker: review/{date}/review-tracker.md
   Report: review/{date}/review-report.md
   ```

   **After review:**
   ```
   ✅ Reviewed: {story_id} - {ac_pass}/{ac_total} ACs passed
   Progress: {current}/{total} stories complete
   ```
```

### Task 3: Update Subagent Input Sections

For each subagent (quality-review, sql-review, sp-review, requirement-review):

1. Update `## Input` section with new format
2. Add parsing instruction
3. No changes to Process, Output, or Error Handling sections

## Success Criteria

- ✅ main.agent.md reduced by ~170 lines
- ✅ All subagent prompts use consistent key:value format
- ✅ No duplication between main.agent.md and subagents
- ✅ Subagents remain self-contained (can run independently)
- ✅ Process steps remain in subagent files only
- ✅ Output formats remain in subagent files only

## Risks and Mitigation

| Risk | Mitigation |
|------|------------|
| Subagent can't parse prompt | Use simple key:value format, add parsing example |
| Missing context in subagent | Ensure subagent reads tracker for full context |
| Inconsistent parameter names | Standardize: Review/Story, Type, Date, Tracker, Report |

## Future Enhancements

- Consider JSON format for even more structured parsing
- Add parameter validation in subagents
- Create shared "parameter parsing" instruction snippet
