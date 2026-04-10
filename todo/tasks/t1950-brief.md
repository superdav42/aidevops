---
mode: subagent
---

# t1950: simplification: reduce returns in git_safety_guard.py + extract-urls.py

## Origin

- **Created:** 2026-04-10
- **Session:** claude-code:interactive
- **Created by:** ai-interactive
- **Conversation context:** Qlty maintainability C rating recovery — systematic complexity reduction across all flagged files

## What

Fix function-level smells: `_is_main_allowlisted` (7 returns), `_check_main_branch_allowlist` (6 returns) in git_safety_guard.py; `is_valid_hostname` (7 returns) in extract-urls.py.

## Why

Part of Qlty maintainability recovery (C to A). Three function-level smell findings.

## Tier

### Tier checklist (verify before assigning)

- [x] **2 or fewer files to modify?**
- [ ] **Complete code blocks for every edit?**
- [x] **No judgment or design decisions?** — Straightforward return consolidation
- [x] **No error handling or fallback logic to design?**
- [x] **Estimate 1h or less?**
- [x] **4 or fewer acceptance criteria?**

**Selected tier:** `tier:standard`

**Tier rationale:** "Complete code blocks" box unchecked — the brief describes the approach (early-exit pattern) but doesn't provide exact oldString/newString. Worker needs to read the functions and design the consolidation.

## How (Approach)

### Files to Modify

- `EDIT: .agents/hooks/git_safety_guard.py:219-260` — refactor `_is_main_allowlisted` to consolidate return paths with early-exit pattern
- `EDIT: .agents/hooks/git_safety_guard.py:264-310` — refactor `_check_main_branch_allowlist` similarly
- `EDIT: .agents/scripts/extract-urls.py:40-70` — refactor `is_valid_hostname` to use validation chain

### Implementation Steps

1. Consolidate multiple `return False` paths into a single validation check
2. Use early-exit for the `return True` happy path
3. Verify: `qlty smells --all --no-snippets 2>&1 | grep -E 'git_safety_guard|extract-urls'` shows no findings

## Acceptance Criteria

- [ ] No function has more than 5 returns in either file
- [ ] `qlty smells` reports zero findings for both files
- [ ] git_safety_guard tests still pass
- [ ] `python3 extract-urls.py --help` still works
