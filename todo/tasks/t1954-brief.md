---
mode: subagent
---

# t1954: fix: enforce tier checklist consistency in issue-sync — reject tier:simple when checklist has unchecked boxes

## Origin

- **Created:** 2026-04-10
- **Session:** claude-code:interactive
- **Created by:** ai-interactive (self-improvement from t1948 mis-tier)
- **Conversation context:** t1948 was labelled tier:simple despite its own tier checklist having 2 unchecked boxes. The model filled in the checklist honestly then contradicted it in the "Selected tier" line. Prompt-level enforcement failed — need deterministic validation.

## What

Add a validation function to `issue-sync-lib.sh` that parses the brief's tier checklist and rejects `tier:simple` when any checklist box is unchecked. When the validation fails, auto-correct the label to `tier:standard` and log a warning.

## Why

The tier checklist in the brief template exists to prevent mis-classification, but nothing enforces consistency between the checklist answers and the selected tier. Models (and humans) will fill in the checklist honestly and then ignore it under momentum. This wastes dispatch cycles: a `tier:simple` task that needs 3 files and has no code blocks will fail at Haiku, burn an escalation cycle, and only then reach Sonnet — wasting time and tokens.

Observed failure: t1948 had 2/6 checklist boxes unchecked (>2 files, no complete code blocks) but was labelled `tier:simple`. Caught by human review, not automation.

## Tier

### Tier checklist (verify before assigning)

- [x] **2 or fewer files to modify?** — 1 file (issue-sync-lib.sh) + 1 test
- [x] **Complete code blocks for every edit?** — yes, see below
- [x] **No judgment or design decisions?** — deterministic parsing logic
- [x] **No error handling or fallback logic to design?** — simple fallback: default to tier:standard
- [x] **Estimate 1h or less?** — ~30m
- [x] **4 or fewer acceptance criteria?** — 4

**Selected tier:** `tier:simple`

**Tier rationale:** All 6 checklist boxes checked. Single file edit with exact implementation logic. The validation is a grep + count — no judgment needed.

## How (Approach)

### Files to Modify

- `EDIT: .agents/scripts/issue-sync-lib.sh` — add `_validate_tier_checklist()` function
- `EDIT: .agents/scripts/issue-sync-helper.sh` — call the validator when applying tier labels

### Implementation Steps

1. Add `_validate_tier_checklist()` to `issue-sync-lib.sh`:

```bash
# _validate_tier_checklist: check that tier:simple briefs have all checklist boxes checked.
# If any box is unchecked, override to tier:standard and warn.
# Arguments:
#   $1 - brief file path
#   $2 - selected tier label (e.g., "tier:simple")
# Returns: the validated tier label on stdout
# Exit: 0 always (validation is advisory, not blocking)
_validate_tier_checklist() {
    local brief_path="$1"
    local selected_tier="$2"

    # Only validate tier:simple — standard and reasoning don't have hard checklist gates
    if [[ "$selected_tier" != "tier:simple" ]]; then
        printf '%s' "$selected_tier"
        return 0
    fi

    # Check if brief file exists
    if [[ ! -f "$brief_path" ]]; then
        printf '%s' "$selected_tier"
        return 0
    fi

    # Count unchecked boxes in the tier checklist section
    # Pattern: lines between "### Tier checklist" and "**Selected tier:**"
    local unchecked_count
    unchecked_count=$(sed -n '/^### Tier checklist/,/^\*\*Selected tier/p' "$brief_path" \
        | grep -c '^\- \[ \]' || true)

    if [[ "$unchecked_count" -gt 0 ]]; then
        echo "[WARN] tier:simple selected but $unchecked_count checklist box(es) unchecked in $brief_path — overriding to tier:standard" >&2
        printf '%s' "tier:standard"
        return 0
    fi

    printf '%s' "$selected_tier"
    return 0
}
```

2. In `issue-sync-helper.sh`, where tier labels are applied to issues, call `_validate_tier_checklist "$brief_path" "$tier_label"` and use the returned value instead of the raw label.

3. The function should also be callable standalone for testing:
   `source issue-sync-lib.sh && _validate_tier_checklist todo/tasks/t1948-brief.md "tier:simple"` should output `tier:standard`.

### Done When

- `source .agents/scripts/issue-sync-lib.sh && _validate_tier_checklist todo/tasks/t1948-brief.md "tier:simple"` outputs `tier:standard`
- `source .agents/scripts/issue-sync-lib.sh && _validate_tier_checklist todo/tasks/t1950-brief.md "tier:simple"` outputs `tier:simple` (t1950 has all boxes checked)
- `shellcheck .agents/scripts/issue-sync-lib.sh` exits 0

## Acceptance Criteria

- [ ] `_validate_tier_checklist` function exists in `issue-sync-lib.sh`
- [ ] tier:simple with unchecked boxes is auto-corrected to tier:standard with a warning
- [ ] tier:simple with all boxes checked passes through unchanged
- [ ] `shellcheck` passes on both modified files
