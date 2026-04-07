---
mode: subagent
---

<!-- SPDX-License-Identifier: MIT -->
<!-- SPDX-FileCopyrightText: 2025-2026 Marcus Quinn -->
# t1905: Add conversation-end loop scan rule to build.txt

## Origin

- **Created:** 2026-04-07
- **Session:** claude-code:interactive
- **Created by:** ai-interactive
- **Parent task:** none
- **Conversation context:** During GH#17677 review session, three separate loop-closing failures occurred: (1) gave wrong crypto approve command without verifying, (2) documented a workflow fix but didn't execute the action it described, (3) created internal task GH#17682 but never commented on or closed source issue GH#17677. All three are the same root cause — no scan for open threads before moving forward. build.txt has completion discipline rules (line 64-71) but nothing about scanning back over conversation for unresolved commitments and affected third parties.

## What

Add a "Conversation-end loop scan" bullet to build.txt's "Completion and quality discipline" section. The rule applies generically to all session types — not just external issue workflows.

## Why

Without this, the model completes the immediate deliverable but drops: (1) commitments displaced by corrections, (2) external parties not notified, (3) user requests that got lost mid-troubleshooting. The existing "Finding-to-task completeness" rule (line 71) only covers audit findings becoming tasks — not conversation-level thread tracking.

## Tier

`tier:simple`

**Tier rationale:** Single-line addition to a prompt file. Exact text provided. No design judgment needed.

## How (Approach)

### Files to Modify

- `EDIT: .agents/prompts/build.txt:71` — add new bullet after "Finding-to-task completeness"

### Implementation Steps

1. After line 71 in `.agents/prompts/build.txt`, add:

```text
- Conversation-end loop scan: before declaring a task complete or moving to the next task, scan back over the full conversation for: (1) commitments made to the user that weren't fulfilled, (2) external parties affected but not notified, (3) requests displaced by troubleshooting or corrections. Internal task creation does not close external loops.
```

2. Run `markdownlint-cli2 .agents/prompts/build.txt` or verify formatting manually (build.txt is plain text with markdown-style comments, not strict markdown — just verify the bullet aligns with siblings).

### Verification

```bash
# Confirm the line exists
grep -c "Conversation-end loop scan" .agents/prompts/build.txt
# Expected: 1

# Confirm it's in the right section
grep -B2 "Conversation-end loop scan" .agents/prompts/build.txt | head -3
# Expected: nearby lines should include "Finding-to-task completeness" or "Completion and quality discipline"
```

## Acceptance Criteria

- [ ] New bullet present in "Completion and quality discipline" section of `.agents/prompts/build.txt`
  ```yaml
  verify:
    method: bash
    run: "grep -q 'Conversation-end loop scan' .agents/prompts/build.txt"
  ```
- [ ] Bullet is generic (not specific to external issues — covers user commitments, third parties, displaced requests)
  ```yaml
  verify:
    method: bash
    run: "grep 'Conversation-end loop scan' .agents/prompts/build.txt | grep -q 'commitments made to the user'"
  ```
- [ ] No other lines in the section were modified
  ```yaml
  verify:
    method: bash
    run: "git diff .agents/prompts/build.txt | grep '^[-+]' | grep -v '[-+][-+][-+]' | wc -l | grep -q '^1$'"
  ```

## Context & Decisions

- Scoped generically after user feedback that the original external-issue-only wording was too narrow
- Placed in "Completion and quality discipline" (not a new section) because it's a completion gate, not a new concern
- Single bullet, not a subsection — proportional to the fix

## Relevant Files

- `.agents/prompts/build.txt:64-76` — Completion and quality discipline + Claim discipline sections

## Dependencies

- **Blocked by:** none
- **Blocks:** nothing
- **External:** none

## Estimate Breakdown

| Phase | Time | Notes |
|-------|------|-------|
| Research/read | 2m | Confirm line numbers |
| Implementation | 2m | Single line addition |
| Testing | 2m | grep verification |
| **Total** | **~6m** | |
