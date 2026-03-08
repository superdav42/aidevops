# t1419: Standalone Worker Watchdog

## Session Origin

Interactive session. User manually dispatched 6 workers (pulse was disabled). 4 crashed/hung within 5 minutes (thundering herd on MCP cold boot). 1 buffered (API rate limiting). 1 kept working. A second interactive session later hung indefinitely. No automated cleanup existed outside the pulse supervisor.

## What

Create `worker-watchdog.sh` — a standalone launchd service that automatically detects and kills hung/idle headless AI workers, then re-queues their GitHub issues for re-dispatch.

## Why

The pulse supervisor (`pulse-wrapper.sh`) has a sophisticated 3-layer watchdog (wall-clock, CPU idle, progress stall) but ONLY for the pulse process itself. No monitoring exists for independently dispatched workers. When workers crash, hang, or enter the OpenCode idle-state bug, they sit indefinitely consuming resources and blocking issue re-dispatch. This is a systemic gap — the pulse's "Kill stuck workers" section is LLM guidance, not automated enforcement, and only fires when pulse is enabled.

## How

Three deliverables:

1. **`worker-lifecycle-common.sh`** — Extract shared process lifecycle functions from `pulse-wrapper.sh` (`_kill_tree`, `_force_kill_tree`, `_get_process_age`, `_get_pid_cpu`, `_get_process_tree_cpu`, `_sanitize_log_field`, `_sanitize_markdown`, `_validate_int`, `_compute_struggle_ratio`) into a shared library. Update `pulse-wrapper.sh` to source it.

2. **`worker-watchdog.sh`** — Standalone watchdog with three detection signals:
   - CPU idle: tree CPU < 5% for 5 minutes (catches OpenCode idle-state bug)
   - Progress stall: no session messages for 10 minutes (catches API rate limiting, stuck workers)
   - Runtime ceiling: 3-hour hard kill (prevents infinite loops)
   On kill: posts GitHub issue comment, swaps labels (`status:in-progress` -> `status:available`), logs action.
   CLI: `--check` (single scan), `--status` (show workers), `--install`/`--uninstall` (launchd plist).

3. **Stagger protection guidance** in `headless-dispatch.md` — Document the thundering herd problem and recommend 30-60s stagger for manual multi-worker dispatch.

## Acceptance Criteria

- [ ] `worker-watchdog.sh --check` scans all headless opencode workers and applies three detection signals
- [ ] `worker-watchdog.sh --status` shows active workers with CPU, runtime, idle/stall tracking, and struggle ratio
- [ ] `worker-watchdog.sh --install` creates and loads a launchd plist at `sh.aidevops.worker-watchdog`
- [ ] `worker-watchdog.sh --uninstall` removes the plist and cleans up state files
- [ ] On kill: GitHub issue gets a comment explaining the reason and labels are swapped for re-dispatch
- [ ] `pulse-wrapper.sh` sources `worker-lifecycle-common.sh` and no longer defines the extracted functions inline
- [ ] ShellCheck passes on all new and modified scripts
- [ ] `headless-dispatch.md` includes stagger protection guidance with correct/incorrect examples

## Context

- GitHub issue: GH#3918
- Branch: `feature/t1419-worker-watchdog`
- Key reference files: `pulse-wrapper.sh` (lines 206-350, 714-792), `memory-pressure-monitor.sh` (launchd CLI pattern), `shared-constants.sh`
