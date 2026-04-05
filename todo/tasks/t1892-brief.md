<!-- SPDX-License-Identifier: MIT -->
<!-- SPDX-FileCopyrightText: 2025-2026 Marcus Quinn -->

# t1892: Add pulse default-on guidance when supervisor_pulse is first enabled

## Origin

- **Created:** 2026-04-05
- **Author:** ai-interactive
- **Source:** Review of [GH#17404](https://github.com/marcusquinn/aidevops/issues/17404) — closing comment identified documentation/onboarding gap

## What

When a user enables `orchestration.supervisor_pulse` (via `aidevops config set` or during onboarding), print a one-liner informing them that the pulse runs 24/7 by default and that `aidevops pulse stop` creates a persistent stop flag for manual start/stop mode.

## Why

Users enabling `supervisor_pulse: true` for the first time have no indication that the default is always-on. The only way to learn about the persistent stop flag is to already know about it. This creates a surprise 24/7 behaviour that could consume resources unexpectedly.

## How

1. **`scripts/config-helper.sh`** — In the `set` subcommand handler, after successfully writing `orchestration.supervisor_pulse` to `true`, echo guidance:
   ```
   [INFO] Pulse enabled — runs every ~2 minutes by default.
   [INFO] Run 'aidevops pulse stop' to switch to manual start/stop mode (persistent across reboots).
   ```
2. **`scripts/onboarding-helper.sh`** — If the onboarding flow enables `supervisor_pulse`, include the same guidance in the onboarding output.

Key files:
- `.agents/scripts/config-helper.sh:313` — dotpath mapping for `orchestration.supervisor_pulse`
- `.agents/scripts/onboarding-helper.sh:1201` — `_json_orchestration()` pulse detection
- `.agents/scripts/pulse-wrapper.sh` — pulse lifecycle (reference only)

## Acceptance Criteria

- When `aidevops config set orchestration.supervisor_pulse true` succeeds, guidance message is printed to stderr
- When `aidevops config set orchestration.supervisor_pulse false` succeeds, no guidance is printed
- Onboarding flow that enables pulse also prints guidance
- ShellCheck clean
- Existing tests pass

## Context

- The stop flag (`~/.aidevops/.pulse-stopped`) persists across reboots and is only cleared by `aidevops pulse start`
- No code/behaviour change — purely UX messaging improvement
- Low complexity: ~5-10 lines across 1-2 files
