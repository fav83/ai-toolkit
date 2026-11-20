---
name: cb-env-update
description: Update Codex CLI and the Paseo daemon on dev-cloud-box, restart Paseo, and verify the running versions. Use when the user asks to update this coding environment.
---

# Update the coding environment

Update Codex first, update Paseo second, then restart the Paseo daemon once. Use latest stable releases unless the user specifies versions. A request to run this skill authorizes those updates, the restart, and one post-restart continuation message to the same Paseo conversation; do not ask for the same authorization again. Creating or editing this skill does not authorize running it.

## Inspect before updating

- Confirm the machine is `dev-cloud-box` with `hostname`. If running elsewhere, connect to that host using an established connection before making changes.
- Record `codex --version`, `paseo --version`, `paseo status`, and the resolved executable paths. Check the daemon's reported version, worker PID, home, and desktop-managed status. Do not expose pairing credentials or authentication files.
- Check installed command help and current installation paths; do not assume a service name or an undocumented update subcommand. See [Paseo's updates documentation](https://paseo.sh/docs/updates) and [CLI reference](https://paseo.sh/docs/cli).
- Announce that the update will briefly disconnect Paseo. Retain the current home, host identity, configuration, pairing, and workspace terminals. Do not use `--kill-terminals`.
- Resolve npm's `latest` tags once with `npm view @openai/codex version` and `npm view @getpaseo/cli version`. Install those exact versions. If already current, skip the corresponding install. Do not downgrade a newer installed version without a user request.

## Update the installed packages

The following paths were verified on this host when the skill was created; inspect them again before relying on them:

| Component | Installation |
| --- | --- |
| Codex | npm global prefix `/usr/local/lib/codex/npm` |
| System Codex command | `/usr/local/bin/codex` links to `/usr/local/lib/codex/npm/bin/codex` |
| Paseo CLI and bundled server | npm global prefix `/home/exedev/.local`, package `@getpaseo/cli` |
| Paseo command | `/home/exedev/.local/bin/paseo` |
| Paseo daemon home | `/home/exedev/.paseo` |

For these installations, substitute the resolved version strings:

```bash
sudo npm install --global --prefix /usr/local/lib/codex/npm @openai/codex@CODEX_VERSION
npm install --global --prefix /home/exedev/.local @getpaseo/cli@PASEO_VERSION
```

Use the execution tool's required elevation for system writes or access outside the sandbox. Preserve the system Codex link; only repair it if inspection shows it is missing or points at the old release. Verify the installed Codex executable and the updated Paseo CLI before restarting.

Updating `@getpaseo/cli` also updates its bundled `@getpaseo/server`; do not install an unrelated server package or update Node/npm as part of this task. If paths or installation methods changed, follow the actual installer and supported lifecycle instead. For a desktop-managed daemon, use its supported desktop update mechanism rather than replacing its managed files.

Stop after an installation error and report what changed and what failed. Do not restart a daemon against a partial or unverified installation. Retain previous version numbers for recovery; do not repeatedly reinstall or restart in a loop.

## Restart and verify across disconnection

The current agent runs through Paseo, so restarting can interrupt the tool call and the conversation. Before the restart, save the before/target versions and daemon identity to a private, durable log under `~/.local/state/cb-env-update/`. Include no credentials. Use a unique run directory or filename so earlier logs and continuation markers are preserved.

Before disconnecting, capture the full current conversation ID from `PASEO_AGENT_ID` and verify it with `paseo --home /home/exedev/.paseo inspect "$PASEO_AGENT_ID" --json`. Save this ID with the run metadata. Do not guess from the latest agent or select another conversation. If the variable is absent or the ID cannot be verified, disclose that automatic continuation is unavailable and retain the detached verification/log fallback.

Run the final restart and subsequent verification from a detached, one-shot process that survives the agent disconnecting (for example, a script launched with `nohup` and `setsid`, with stdin from `/dev/null` and stdout/stderr redirected to the private log). This process should:

1. Run the absolute Paseo command with the verified local home: `/home/exedev/.local/bin/paseo --home /home/exedev/.paseo restart --timeout 120`.
2. Record the restart exit status, then run `paseo --home /home/exedev/.paseo status` and both version commands.
3. Confirm the replacement worker PID changed and the live `daemonVersion` matches the target Paseo version. Use `paseo provider diagnostic codex --json` to check the daemon's resolved Codex path and version. If inspecting running Codex processes, verify their actual executable rather than relying on the command text alone. Allow bounded readiness retries; never issue repeated restarts merely because a status check timed out.
4. Write and flush a final success or failure result to the log.
5. Resume the original conversation automatically using the supported Paseo CLI, as described below. This step must run inside the same detached job; leaving it to the interrupted agent recreates the manual “continue” problem. Installed package versions alone do not establish that the running daemon or Codex processes were updated.

### Resume the original conversation

After verification finishes, have the detached job send one follow-up to the saved conversation ID. This works for an idle or interrupted agent and avoids creating a new conversation. See [Sending messages](https://paseo.sh/docs/cli#sending-messages).

Write a private UTF-8 prompt file alongside the log, including the actual absolute log path and this instruction:

> Automatic cb-env-update continuation. The detached update job has finished its restart and verification attempt. Read the saved log at LOG_PATH and check current daemon status, then report the actual outcome and before/after versions. Do not install packages, restart Paseo, launch another update job, or send another continuation message. If the result is incomplete or failed, report that accurately.

Use the verified home and full saved agent ID:

```bash
/home/exedev/.local/bin/paseo --home /home/exedev/.paseo send "$saved_agent_id" --no-wait --behavior steer --prompt-file "$prompt_path"
```

Run this command only after the daemon is reachable. If verification failed before readiness, allow bounded status retries for up to two minutes so a failure can still be reported; never restart again. If the daemon stays unavailable, record that automatic continuation could not be delivered and exit.

Record a per-run dispatch-attempt marker atomically before invoking `send`, and log its exit status. Make at most one send attempt per run; a timeout can mean the message was accepted, so do not retry ambiguous delivery. Apply a bounded command timeout and retain the log for manual recovery. Flush the verification result before dispatch so the resumed agent can read it immediately. Send on verification failure too when the daemon is reachable; the message must not claim success.

Save the detached job PID and log path before yielding. If the conversation resumes after interruption, inspect that job/log and current daemon status first; do not launch the update again. A UI update mechanism that already performs the restart counts as the restart step; do not restart twice.

If both packages were already current, restart only when running processes still use older versions or the user explicitly requested a restart. After reconnection, report before/after versions and verified daemon availability. Distinguish an update that was merely launched from one that finished successfully; include the log path if verification remains incomplete.
