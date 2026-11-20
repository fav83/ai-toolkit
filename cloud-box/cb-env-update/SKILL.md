---
name: cb-env-update
description: Update Codex CLI and Paseo on dev-cloud-box, restart if needed, and verify running versions.
---

# Update the coding environment

Update Codex, then Paseo, then restart once if needed. Invocation authorizes these actions and one continuation message to the same conversation; editing this skill does not authorize running it.

## Inspect and install

- Confirm `hostname` is `dev-cloud-box`; otherwise use an established connection to it.
- Record executable paths, `codex --version`, `paseo --version`, and `paseo status`: daemon version, worker PID, home, identity, and desktop-managed status. Keep credentials private.
- Check installed help and actual installation paths. Use supported commands: [updates](https://paseo.sh/docs/updates), [CLI reference](https://paseo.sh/docs/cli).
- Resolve stable targets once with `npm view @openai/codex version` and `npm view @getpaseo/cli version`, unless specific versions were requested. Install exact versions; skip current packages and never downgrade without a request.

Expected paths (recheck): Codex prefix `/usr/local/lib/codex/npm`, `/usr/local/bin/codex` → that prefix's `bin/codex`; Paseo prefix `/home/exedev/.local`, command `/home/exedev/.local/bin/paseo`, home `/home/exedev/.paseo`.

```bash
sudo npm install --global --prefix /usr/local/lib/codex/npm @openai/codex@CODEX_VERSION
npm install --global --prefix /home/exedev/.local @getpaseo/cli@PASEO_VERSION
```

Use required execution-tool elevation. Preserve the Codex link unless broken or stale. Paseo CLI includes its server; do not install a separate server or update Node/npm. Follow the actual installer if paths differ; use supported desktop updates for desktop-managed installations.

Verify both installed commands before restarting. On installation error, stop and report changes/failure; retain previous versions for recovery. Never restart a partial or unverified installation or loop reinstalls/restarts.

If packages and running versions are current, report that and stop. Otherwise restart as needed; when both packages were already current, restart only for older running versions or an explicit restart request. Announce the brief disconnection. Preserve home, identity, configuration, pairing, and terminals; never use `--kill-terminals`. A desktop update's restart counts.

## Survive restart and resume

Before restarting:

1. Create a unique private run directory under `~/.local/state/cb-env-update/`. Save before/target versions and daemon identity, without credentials.
2. Save the full `PASEO_AGENT_ID`, verified with `paseo --home /home/exedev/.paseo inspect "$PASEO_AGENT_ID" --json`. Never guess another conversation. If absent/unverifiable, disclose that automatic continuation is unavailable; keep detached verification and logs.
3. Prepare a private UTF-8 prompt file containing the absolute log path and:

   > Automatic cb-env-update continuation. Read the completed detached job's log at LOG_PATH and check current daemon status. Report the actual outcome and before/after versions, including failures or incomplete verification. Do not install packages, restart Paseo, launch another update job, or send another continuation.

Launch a detached one-shot job (e.g. `nohup` + `setsid`, stdin `/dev/null`, stdout/stderr to the private log). Save its PID and log path before yielding. The **same job** must:

1. Run `/home/exedev/.local/bin/paseo --home /home/exedev/.paseo restart --timeout 120` once and record its exit status.
2. Check status and both version commands. With bounded readiness retries, confirm the worker PID changed and live `daemonVersion` matches the target. Run `paseo --home /home/exedev/.paseo provider diagnostic codex --json` to verify the daemon's resolved Codex path/version. Package versions alone are insufficient; if examining processes, verify actual executables.
3. Write and flush the success/failure result before dispatch. If unreachable, retry status for at most two minutes, without restarting again. If still unavailable, log that continuation could not be delivered and exit.
4. When reachable, send the saved conversation one continuation even if verification failed. Atomically create a per-run dispatch-attempt marker **before** invoking the command; use a bounded command timeout, log its exit status, and never retry ambiguous delivery:

   ```bash
   /home/exedev/.local/bin/paseo --home /home/exedev/.paseo send "$saved_agent_id" --no-wait --behavior steer --prompt-file "$prompt_path"
   ```

On resumption, inspect the existing job/log and current daemon status; do not rerun the update. Report before/after versions and verified availability. Distinguish launched from completed work; include the log path if incomplete.
