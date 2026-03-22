# Review Agent System Design

**Date:** 2025-03-22
**Status:** Draft
**Author:** User + Claude

## Overview

A multi-agent code review system running in VS Code GitHub Copilot. The system performs:
1. Code quality review (general code, SQL, stored procedures)
2. Requirement alignment review (code implementation vs. story requirements)

## File Structure

```
.github/
└── agents/
    ├── main.agent.md           # Main entry point, orchestration
    ├── quality-review.agent.md # General code quality review
    ├── sql-review.agent.md     # SQL file review
    ├── sp-review.agent.md      # Stored procedure review
    ├── requirement-review.agent.md  # Requirement alignment review
    ├── jira.agent.md           # Jira MCP integration
    └── classifier.agent.md     # File-Story association

review/
└── {yyyy-mm-dd}/
    ├── quality-todos.md        # Files pending review
    ├── story-todos.md          # Story info + file associations
    ├── quality-report.md       # Technical review report
    └── story-report.md         # Requirement alignment report
```

## Main Agent Flow

### Step 1: Get Review Scope

```
Ask: "What is the review scope?"
Options:
  A) Current changes (staged + unstaged)
  B) Between two commits (provide commit hashes)
```

→ Write file list to `./review/{date}/quality-todos.md`

### Step 2: Get Story Information

```
Ask: "Do you have stories to review against?"
Options:
  A) Jira IDs (e.g., T01-221, T01-222)
  B) Paste story text directly
  C) Skip requirement review
```

- **If A**: Call `jira.agent.md` → Write to `story-todos.md`
- **If B**: Write directly to `story-todos.md`
- **If C**: Skip to Step 4

### Step 3: Associate Files with Stories

Call `classifier.agent.md`
- Input: `quality-todos.md` + `story-todos.md`
- Output: Append associations to `story-todos.md`

### Step 4: Execute Technical Review

For each file in `quality-todos.md` (serial execution):
- `.sql` → `sql-review.agent.md`
- `.sp` → `sp-review.agent.md`
- other → `quality-review.agent.md`

Append results to `quality-report.md`

### Step 5: Execute Requirement Review

(If Step 2 was not Skip)

For each story in `story-todos.md`:
- Call `requirement-review.agent.md`
- Append results to `story-report.md`

### Step 6: Complete

Output summary:
- Files reviewed: X
- Stories reviewed: Y
- Reports location: `./review/{date}/`

## Sub Agent Specifications

### jira.agent.md

**Input:**
- Jira IDs (comma-separated, e.g., "T01-221, T01-222")

**Process:**
For each Jira ID:
1. Call Jira MCP to fetch: Summary/Title, Description, Acceptance Criteria
2. Format and append to `story-todos.md`

**Error Handling:**
- If Jira MCP call fails, log error and continue to next ID

### classifier.agent.md

**Input:**
- `quality-todos.md` (file list with changes)
- `story-todos.md` (story definitions)

**Process:**
For each file, determine association using:
1. File path/naming (e.g., `feature/T01-221/`)
2. Git commit message (extract story IDs)
3. LLM semantic inference (compare changes with story)

**Output:**
- Update `story-todos.md`, fill "Associated Files" section
- If no match found, mark as "Unassociated"

### quality-review.agent.md

**Input:**
- File path

**Process:**
1. Execute git diff to get changes for this file
2. Read full file content for context
3. Perform review based on diff + full context

**Review Focus:**
- Code readability and maintainability
- Potential bugs or logic errors
- Security vulnerabilities
- Performance concerns
- Error handling

**Error Handling:**
- If file read fails, log error and skip to next file

### sql-review.agent.md

**Input:**
- `.sql` file path

**Process:**
1. Execute git diff to get changes
2. Read full file content for context
3. Perform review

**Review Focus:**
- Query performance (indexes, joins, subqueries)
- SQL injection risks
- Data type consistency
- NULL handling
- Transaction scope

**Error Handling:**
- If file read fails, log error and skip to next file

### sp-review.agent.md

**Input:**
- `.sp` file path

**Process:**
1. Execute git diff to get changes
2. Read full file content for context
3. Perform review

**Review Focus:**
- Business logic correctness
- Error handling and exceptions
- Parameter validation
- Return value consistency
- Transaction management
- Performance (cursor usage, loops)

**Error Handling:**
- If file read fails, log error and skip to next file

### requirement-review.agent.md

**Input:**
- Story ID or Title (e.g., "T01-221")

**Process:**
1. Read `story-todos.md` to get story details
2. Get associated files from `story-todos.md`
3. Execute git diff for each associated file
4. Read full file content for context
5. Compare implementation against story requirements

**Review Focus:**
- Feature coverage: Does code implement all features?
- Acceptance Criteria: Does code satisfy each AC?
- Edge cases: Are boundary conditions handled?
- Missing implementations

**Error Handling:**
- If story not found, log error and skip
- If file read fails, log error and continue with available files

## Intermediate File Formats

### quality-todos.md

```markdown
# Quality Review Todos
**Generated:** {yyyy-mm-dd HH:mm:ss}
**Review Scope:** {Current changes / Commits: abc123..def456}

## Files to Review

| # | File Path | Type | Status |
|---|-----------|------|--------|
| 1 | src/services/UserService.java | code | pending |
| 2 | db/migrations/add_user_table.sql | sql | pending |
| 3 | sp/calculate_user_score.sp | sp | pending |

---
*Status: pending → reviewed → failed*
```

### story-todos.md

```markdown
# Story Review Todos
**Generated:** {yyyy-mm-dd HH:mm:ss}

---

## Story: T01-221 - User Authentication Enhancement

**Description:**
Implement OAuth2 authentication with Google and GitHub providers.

**Acceptance Criteria:**
- AC1: User can login with Google account
- AC2: User can login with GitHub account
- AC3: Failed login attempts are logged
- AC4: Session expires after 30 minutes of inactivity

**Associated Files:**
- src/services/AuthService.java (path: feature/T01-221/)
- src/controllers/AuthController.java (commit: T01-221)
- db/migrations/add_oauth_tables.sql (semantic match)

**Review Status:** pending

---

## Unassociated Files
- src/utils/Helper.java (no story match found)
```

## Report Output Formats

### quality-report.md

```markdown
# Quality Review Report
**Generated:** {yyyy-mm-dd HH:mm:ss}
**Review Scope:** {Current changes / Commits: abc123..def456}

---

## File: src/services/UserService.java
**Review Date:** 2024-01-15 10:30:00

### Issues

🔴 **HIGH** - createUser() L45
   - Description: SQL injection risk in username parameter
   - Suggestion: Use parameterized query

🟡 **MEDIUM** - updateUser() L78
   - Description: Missing null check for email field
   - Suggestion: Add validation before assignment

🟢 **LOW** - deleteUser() L102
   - Description: Log message missing user context
   - Suggestion: Include userId in log entry

### Summary
The UserService has a critical security vulnerability in the createUser method.
Minor improvements needed for null handling and logging. Overall code structure is clean.

---

## File: db/migrations/add_user_table.sql
**Review Date:** 2024-01-15 10:32:00

### Issues

🟡 **MEDIUM** - L12
   - Description: Missing index on email column
   - Suggestion: Add CREATE INDEX for query performance

🟢 **LOW** - L5
   - Description: Column name inconsistency (user_name vs userName)
   - Suggestion: Follow naming convention

### Summary
Migration script is functional but would benefit from an index on the email column
for better query performance.

---

## Review Summary

**Total Files Reviewed:** 2
**Issues Found:** 5
- 🔴 HIGH: 1
- 🟡 MEDIUM: 2
- 🟢 LOW: 2

**Recommendations:**
1. Fix 🔴 HIGH severity SQL injection issue before merge
2. Review 🟡 MEDIUM issues for potential inclusion
```

### story-report.md

```markdown
# Story Review Report
**Generated:** {yyyy-mm-dd HH:mm:ss}

---

## Story: T01-221 - User Authentication Enhancement
**Review Date:** 2024-01-15 10:35:00

### Acceptance Criteria Status

✅ **PASS** - User can login with Google account
   - Evidence: AuthService.loginGoogle() L23-45

✅ **PASS** - User can login with GitHub account
   - Evidence: AuthService.loginGitHub() L47-69

⚠️ **PARTIAL** - Failed login attempts are logged
   - Evidence: AuditLogger.logFailedAttempt() L80
   - Gap: Missing IP address in log

❌ **FAIL** - Session expires after 30 min inactivity
   - Gap: No session timeout implementation found

### Feature Coverage
- ✅ OAuth2 integration: Covered - Google and GitHub providers implemented
- ⚠️ Error handling: Partial - Missing timeout handling
- ⚠️ Logging: Partial - Failed attempts logged but incomplete data

### Edge Cases & Gaps
- Missing: Session timeout mechanism
- Missing: IP address in failed login logs
- Consider: Rate limiting for login attempts

### Summary
Core OAuth2 functionality is implemented and working. Two gaps identified:
session timeout and incomplete logging. Recommend addressing before merge.

---

## Review Summary

**Total Stories Reviewed:** 1
**Acceptance Criteria:** 4
- ✅ PASS: 2
- ⚠️ PARTIAL: 1
- ❌ FAIL: 1

**Recommendations:**
1. Implement session timeout mechanism (AC4)
2. Enhance failed login logging with IP address
```

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Architecture | Simple pipeline (Plan A) | Clear flow, single entry point, easy to maintain |
| File organization | Flat in `.github/agents/` | Simple, follows GitHub Copilot convention |
| Execution mode | Serial | Predictable, easier to debug |
| Error handling | Skip and log | Non-blocking, continues review process |
| Review strategy | Simple guidelines + LLM flexibility | Leverage LLM capability, ensure stable output format |
| Report language | English | Standard for international teams |
| Interaction mode | Ask one by one | Clear user guidance at each step |

## Dependencies

- **Jira MCP**: Pre-configured in VS Code for Jira integration
- **Git**: For diff extraction and commit message parsing
- **File system**: For reading source files and writing reports
