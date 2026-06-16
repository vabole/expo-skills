# Telemetry internals (maintainer reference)

Background detail for the `skill-feedback` scripts. The day-to-day surface — what's
sent, how to submit feedback, and how to turn it off — lives in `../SKILL.md`. This
file is for people changing or auditing the telemetry, not for the agent at runtime.

## Configuring the PostHog project

The project key lives in `scripts/telemetry_common.js`. It is a **write-only, public**
ingestion key (`phc_...`) — the same kind embedded in browser snippets — so it is safe
to commit, and it is what makes telemetry **on by default**. Override it per environment
(e.g. a staging project) without editing the file:

```bash
export EXPO_SKILLS_POSTHOG_KEY="phc_your_project_key"
```

Only ever use a *project* key (`phc_...`) here. Never a *personal* API key (`phx_...`) —
those are secret and used only for reading/querying (e.g. the PostHog MCP). If the key is
stripped to empty or `phc_REPLACE_ME` (e.g. a fork), `telemetryConfigured()` returns false
and the scripts go inert — no installation ID, no network.

## Event shape

- `event`: `skill_invoked` | `skill_feedback`
- `distinct_id`: `expo-skills-installation:<installation_id_hash>`
- `properties.source`: `expo-skills`
- `properties.$process_person_profile`: `false`
- `properties.skill`: skill folder name (e.g. `expo-deployment`)
- `properties.agent_harness`: auto-detected (`claude-code`, `codex`), else `unknown`; override with `--agent-harness`
- `properties.initiator` (on `skill_invoked`): `ai` (invoked via the `Skill` tool) or `user` (a `/slash` command)
- `properties.os` / `properties.arch`: platform, e.g. `macos` / `arm64` (non-PII)
- `properties.installation_id_hash`: anonymous hash of the local random installation ID
- `properties.session_id_hash`: short hash only; raw session IDs are never sent
- `skill_feedback` adds `properties.rating` and `properties.feedback_text`

## Why prompts / chat history are never captured

Richer context (the prompt that triggered a skill, surrounding chat) would help study how
to improve a skill, but capturing it would break the anonymity guarantee that justifies
on-by-default collection. So the automatic `skill_invoked` event carries **only the skill
name** and the anonymous properties above — never prompt text, code, file paths, or tool
inputs. If a maintainer wants qualitative context, the path is **explicit and
user-approved**: a human (or an agent with the user's consent) puts a short, scrubbed note
in the `--text` field of `skill_feedback`. Automatic prompt/history capture is intentionally
off the table.

## How the auto path stays off the critical path

`skill-event.sh` is launched from a Claude Code hook, which waits for the command to exit.
To avoid adding the POST's latency to every turn, the wrapper detaches the send into its
own session (`setsid`, falling back to `perl POSIX::setsid`, then `nohup`) and returns
immediately; the child completes the POST in the background. `skill-event.js` still enforces
a hard 3s request timeout so a hung network call can't linger.

The hook payload (which skill was invoked) arrives on the wrapper's **stdin** — but a
backgrounded process's stdin is `/dev/null` (POSIX), so the detached child can't read it
directly. So the wrapper reads stdin **synchronously** (a fast local pipe read, not the
network) into a `mktemp` temp file and passes its path via `--hook-input-file`;
`skill-event.js` reads that file and then unlinks it. It reads the file **before** the
telemetry-off guards, so the temp file is cleaned up even when nothing is sent. The
detached child has all three of stdin/stdout/stderr redirected to `/dev/null` (stdout and
stderr to keep a stray write from raising SIGPIPE and killing the POST mid-flight). If
`mktemp` is unavailable the wrapper exits without sending, rather than writing the payload
to a predictable path.

The explicit `skill-feedback.sh` path does **not** detach — it runs in the foreground and
awaits the send, because it reports success/failure back to the user.

## Harness support, in depth

- **Claude Code** — fully wired via `hooks/hooks.json`: `PostToolUse` matcher `Skill`
  (the AI's `Skill` tool → `initiator: ai`) and `UserPromptExpansion` (user `/slash`
  commands → `initiator: user`).

- **Codex** — `skill_invoked` cannot fire from the plugin today (verified against
  codex-cli 0.138.0):
  1. **Plugin hooks are removed.** `codex features list` shows `plugin_hooks` = stage
     `removed`, state `false`. The general hook engine (`hooks`) is stable, but it only
     loads hooks from the user / project / managed layers (a `hooks.json`/`hooks.toml` in
     `CODEX_HOME` or `config.toml [hooks]`), not from a plugin. A `hooks` key in
     `.codex-plugin/plugin.json` is accepted by the installer but never registers a hook —
     and Codex's own plugin scaffolder explicitly says to omit unsupported manifest fields
     including `hooks`. So we do **not** ship a Codex plugin hooks file.
  2. **Skills aren't tool-invoked.** Codex injects the skill catalog into context and the
     model reads a skill's `SKILL.md` with its normal file/shell tools — there is no
     `Skill` tool. So a `PostToolUse` matcher on `tool_name == "Skill"` would never fire,
     and matching the generic `shell`/read tool would be noisy and brittle.
  3. **No `UserPromptExpansion`.** Codex has `UserPromptSubmit` (fires on every prompt,
     not skill-scoped), so the Claude user-path event key is ignored.

  Good news for the future: Codex **aliases** `CLAUDE_PLUGIN_ROOT` / `CLAUDE_PLUGIN_DATA`
  (confirmed in the hook engine's env table), so if both gaps close — `plugin_hooks`
  returns *and* skill use becomes observable via a hook event — the wiring is small. Until
  then, manual `skill_feedback` is the Codex signal.

- **Cursor / others** — no plugin-hook system at all; manual `skill_feedback` (run the
  bundled script directly) is the only signal.

## Windows

The hook command and footer commands start with `sh`, which stock Windows lacks (it exists
under Git Bash / WSL, where many Expo devs already work). The design is fail-open: if `sh`
isn't found the hook simply no-ops, so nothing breaks — but automatic `skill_invoked`
tracking won't fire on a bare Windows shell, so Windows usage is under-counted there.
Manual `skill_feedback` still works by running the JS directly with `node`/`bun`. A future
cross-platform fix is to do the detach inside Node (`spawn(..., { detached: true,
stdio: "ignore" }).unref()`) instead of in `sh`.
