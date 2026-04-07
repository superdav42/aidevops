---
mode: subagent
---

<!-- SPDX-License-Identifier: MIT -->
<!-- SPDX-FileCopyrightText: 2025-2026 Marcus Quinn -->
# t1904: fix stat stdout pollution and WAL pragma leak in opencode-db-archive.sh

## Origin

- **Created:** 2026-04-07
- **Session:** Claude Code (claude-sonnet-4-6)
- **Created by:** ai-interactive
- **Parent task:** none
- **Conversation context:** Reviewed issue #17681 which reported two Linux-incompatible bugs in opencode-db-archive.sh. Both bugs confirmed in codebase. This brief captures the approved fixes.

## What

Two single-line fixes to `.agents/scripts/opencode-db-archive.sh`:

1. Replace `||`-chained `stat -f '%z'` with platform-conditional block (line 141–149) to prevent stdout pollution on Linux.
2. Move `PRAGMA journal_mode=WAL;` out of the schema heredoc and run it as a separate suppressed sqlite3 call (line 255).

## Why

Bug 1: On Linux, `stat -f '%z'` outputs filesystem metadata to stdout before failing. `2>/dev/null` suppresses only stderr. The metadata leaks into callers that capture output (e.g., `file_size=$(file_size_bytes "$file")`), causing arithmetic failures under `set -u`. Regression-by-omission — `opencode-db-archive.sh` was written after the framework-wide fix in #1491 (v2.115.0).

Bug 2: SQLite writes `wal` to stdout when `PRAGMA journal_mode=WAL` executes inside a heredoc. Leaks to caller on every `create_archive_schema()` invocation.

## Tier

`tier:simple`

**Tier rationale:** Two single-file edits. Exact code blocks provided. Worker copies and verifies with shellcheck, no design judgment required.

## How (Approach)

### Files to Modify

- `EDIT: .agents/scripts/opencode-db-archive.sh:141-149` — replace `||`-chained stat with platform-conditional block
- `EDIT: .agents/scripts/opencode-db-archive.sh:254-255` — remove PRAGMA from heredoc; run separately with stdout suppressed

### Implementation Steps

1. Replace `file_size_bytes()` function (lines 141–149) with platform-conditional pattern from `document-creation-helper.sh:72-79`:

```bash
file_size_bytes() {
	local filepath="$1"
	if [[ ! -f "$filepath" ]]; then
		echo "0"
		return 0
	fi
	if [[ "$(uname)" == "Darwin" ]]; then
		stat -f '%z' "$filepath" 2>/dev/null || echo "0"
	else
		stat -c '%s' "$filepath" 2>/dev/null || echo "0"
	fi
	return 0
}
```

2. In `create_archive_schema()` (lines 254–256), remove `PRAGMA journal_mode=WAL;` from the heredoc and add a separate suppressed call after it closes:

```bash
# Remove from inside SCHEMA_SQL heredoc (line 254-255):
-- WAL mode for the archive too (better read concurrency)
PRAGMA journal_mode=WAL;

# Replace the closing lines (256-258) with:
SCHEMA_SQL
	sqlite3 "$archive_db" "PRAGMA journal_mode=WAL;" >/dev/null 2>&1 || true
	return 0
}
```

### Verification

```bash
shellcheck .agents/scripts/opencode-db-archive.sh
# Confirm stat uses uname conditional
grep -n "uname" .agents/scripts/opencode-db-archive.sh
# Confirm PRAGMA not in heredoc
grep -n "PRAGMA" .agents/scripts/opencode-db-archive.sh
```

## Acceptance Criteria

1. `shellcheck .agents/scripts/opencode-db-archive.sh` exits 0 with no new violations.
2. `file_size_bytes()` uses a `uname`-conditional block, not `||` chain.
3. `PRAGMA journal_mode=WAL;` does not appear inside the `SCHEMA_SQL` heredoc.
4. A separate `sqlite3 ... "PRAGMA journal_mode=WAL;" >/dev/null` call exists after the heredoc.

## References

- GH#17683 (this task's issue)
- GH#17681 (original bug report reviewed)
- GH#1491 (v2.115.0 framework-wide stat fix)
- Reference pattern: `.agents/scripts/document-creation-helper.sh:72-79`
