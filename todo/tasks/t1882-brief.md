# t1882: Passphrase-less SSH Signing for Headless Workers + ssh-agent Integration

ref:GH#17322

## Session Origin

Interactive session (t1880/t1881 attribution protection work). Discovered that SSH commit signing with a passphrase-protected key blocks headless worker commits — the passphrase prompt hangs indefinitely in non-interactive contexts.

## What

Make commit signing work seamlessly for all contexts:

1. **Headless workers**: Commits should be signed automatically when possible, skipped silently when not — never hang on a passphrase prompt.
2. **Interactive users**: `aidevops signing setup` should guide users through ssh-agent configuration so signing "just works" after initial setup.
3. **CI/GitHub Actions**: Squash merges are signed by GitHub's GPG key (already works). Direct pushes from CI need a deploy key or passphrase-less key.

## Why

The current `signing-setup.sh` configures `commit.gpgsign = true` globally. If the SSH key has a passphrase and `ssh-agent` isn't running, every `git commit` blocks on a TTY prompt. In headless worker sessions (pulse dispatch, full-loop), this causes the commit to fail with "incorrect passphrase supplied to decrypt private key" and the task stalls.

This is a usability gap: users who enable signing shouldn't have their automation break silently.

## How

### Phase 1: Graceful fallback in headless contexts

- **`signing-setup.sh`**: Add a `can-sign` subcommand that tests whether signing will succeed without blocking. Logic: attempt `ssh-add -l` to check if the key is loaded in ssh-agent. If not, check if the key is passphrase-free (`ssh-keygen -y -P "" -f <key>`). Returns 0/1.
- **`full-loop-helper.sh`** and **`headless-runtime-helper.sh`**: Before committing, call `signing-setup.sh can-sign`. If it returns 1, use `git -c commit.gpgsign=false commit` for that session. Log a notice: "Signing skipped — key not available in ssh-agent."
- **`aidevops-update-check.sh`**: If signing is enabled but `can-sign` fails, add to the startup check output: "Signing enabled but key not loaded in ssh-agent. Run: ssh-add"

### Phase 2: ssh-agent integration in signing setup

- **`signing-setup.sh setup`**: After configuring git, check if `ssh-agent` is running. If not, offer to:
  - Add `eval "$(ssh-agent -s)"` to `~/.zshrc` / `~/.bashrc`
  - Run `ssh-add ~/.ssh/id_ed25519` (prompts for passphrase once)
  - Explain: "After this, signing works automatically for the rest of the session"
- **macOS Keychain integration**: On macOS, suggest `ssh-add --apple-use-keychain ~/.ssh/id_ed25519` which stores the passphrase in Keychain and loads it automatically on login. This is the zero-friction path.
- **launchd agent**: Optionally create `sh.aidevops.ssh-agent.plist` that ensures ssh-agent is running and key is loaded on login.

### Phase 3: Documentation

- Update `signing-setup.sh help` to explain the passphrase/ssh-agent relationship
- Add a troubleshooting section to the signing setup flow
- Document the macOS Keychain path as the recommended approach

### Key files

- `.agents/scripts/signing-setup.sh` — new `can-sign` subcommand, ssh-agent setup
- `.agents/scripts/full-loop-helper.sh` — graceful fallback
- `.agents/scripts/headless-runtime-helper.sh` — graceful fallback
- `.agents/scripts/aidevops-update-check.sh` — startup warning

## Acceptance Criteria

1. `aidevops signing can-sign` returns 0 when key is loaded in ssh-agent, 1 when not
2. Headless workers commit without signing when key is not available — no hangs, no errors
3. `aidevops signing setup` on macOS offers Keychain integration
4. `aidevops signing setup` on Linux offers ssh-agent + .bashrc integration
5. Startup check warns when signing is enabled but key isn't loaded
6. ShellCheck clean on all modified scripts

## Context

- Current key: `~/.ssh/id_ed25519` (passphrase-protected, ED25519)
- Signing configured globally: `gpg.format=ssh`, `commit.gpgsign=true`
- Headless workers run via `headless-runtime-helper.sh run` which spawns claude/opencode CLI
- macOS `ssh-add --apple-use-keychain` persists across reboots via Keychain
- Linux needs explicit `ssh-agent` setup in shell profile
