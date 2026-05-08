---
name: outsystems
description: "OutSystems Developer Cloud (ODC) over remote MCP. Edit apps, publish, deploy, run impact analysis, search tenant elements, manage external libraries. Use for ANY OutSystems task."
---

# OutSystems Developer Cloud (ODC) - Remote MCP

You are connected to ODC over the remote MCP HTTP transport. ODC is a cloud-native low-code platform where apps are built from OML (OutSystems Model Language), a binary format describing entities, screens, actions, and logic. Every tool call carries the harness's validated OAuth bearer; tenant + user identity are derived from the JWT, not from arguments.

## First use / setup

The Power ships with a sentinel URL containing `UNCONFIGURED-TENANT`. The first time the user asks for anything OutSystems-related, finish the wiring on their behalf. Do this once; the edits are idempotent for re-runs against a different tenant.

Trigger conditions:
- The user asks for anything OutSystems-related, AND
- The MCP server URL still contains `UNCONFIGURED-TENANT`, OR a call returns `tenant not configured` / connection errors.

Steps:

1. **Ask the user for their OutSystems tenant hostname.** Format: `<short>-<region>-<index>.outsystems.dev` (e.g. `eng-stage-us-01.outsystems.dev`). Prompt verbatim:
   > "Which OutSystems tenant should I connect to? It's the host portion of your ODC URL, typically something like `eng-stage-us-01.outsystems.dev`."
2. **Validate** against `^[A-Za-z0-9]([A-Za-z0-9.-]*[A-Za-z0-9])?$`. Reject URLs, paths, or anything that isn't a bare hostname; ask again with an example.
3. **Construct the URL**: `https://mcp-test.datap-dev-us-east-1-01.dev-06.stamp.outsystemscloudrd.net/<TENANT>/mcp`.
4. **Patch two files.** Use Read + Write: read first, preserve every other entry, write back. Never clobber siblings. If the existing file has comments (JSONC), preserve them; only modify the URL field.
   - `~/.kiro/powers/installed/outsystems/mcp.json`: set `mcpServers.outsystems` to
     ```
     {"type": "http", "url": "<URL>", "timeout": 100000}
     ```
   - `~/.kiro/settings/mcp.json`: set `powers.mcpServers["power-outsystems-outsystems"]` to the same object. ALWAYS keep a top-level `mcpServers` key (empty object is fine if no other servers); Kiro's user-level loader rejects the file without it.
5. **Cache the tenant** for future reference: `printf '%s\n' '<TENANT>' > ~/.config/outsystems/tenant` (create the parent dir first if needed). Informational only.
6. **Tell the user** Kiro's MCP file watcher will reload within a few seconds. Then proceed to the "Authenticating" section below; the agent drives the OAuth flow on the first OutSystems tool call.
7. **Retry the user's original request** once authentication completes.

If the user later asks to switch tenants, repeat this flow; the patches overwrite the URL idempotently.

## Authenticating

The server is OAuth-protected. The harness exposes two deferred tools when the user isn't yet authenticated; the agent drives the flow, the user does NOT click anywhere in Kiro's UI.

- `mcp__outsystems__authenticate`: start the OAuth flow; returns an authorization URL.
- `mcp__outsystems__complete_authentication`: finalize remote-session auth (see below).

**Lazy on first use.** Before the first OutSystems tool call in a session, call `mcp__outsystems__authenticate`. Share the returned URL with the user. Then either:

- **Local session** (browser can reach `http://localhost:<port>/callback`): the server's real tools become available automatically. Wait for the user to confirm the browser redirected successfully, then proceed with their original request.
- **Remote session** (callback page fails to load, e.g. SSH or devcontainer): ask the user to copy the full URL from their browser's address bar (it'll look like `http://localhost:<port>/callback?code=...&state=...`) and call `mcp__outsystems__complete_authentication { callback_url: "<that URL>" }`.

**Reactive on auth error.** If a tool call mid-session fails with `data.category: "AuthError"` (token expired, refresh denied, etc.), call `mcp__outsystems__authenticate` again with the same flow, then retry the original call ONCE.

**Never** instruct the user to remove and re-add the server in Kiro's MCP UI as a first response. The agent always has the tool pair available; UI removal is the host's emergency fallback.

**If `authenticate` itself errors** (proxy unreachable, DCR fails): surface the error verbatim to the user. Don't speculate about proxy internals; that's the signal to file an issue against `OutSystems/mcp-server-remote`.

## Rules
- This tool surface is agent-focused. Don't expose raw tool output to the user; extract the relevant fields and present them naturally.
- Go straight to the task. No setup checks, no auth pre-flight beyond the lazy `authenticate` call described above. Identity comes from the bearer the harness already negotiated.
- OML stays server-side. There is no `app_download`. To inspect an app, use `app_refs` + `context_*` tools. To edit an app, use `mentor` (OML lives in the server-side mentor session, never crosses the wire as bytes).
- If `env_key`, `app_key`, an asset key, or a `mentor_session_*` token is missing and you can't reliably resolve it, ask the user. Never guess.
- Run independent tool calls in parallel when possible (e.g. once you have an app key, fetch info + per-type context lookups concurrently).
- Errors carry a structured category in `data.category` (one of `AuthError`, `ValidationError`, `UpstreamError`, `InternalError`); upstream errors also include `data.upstream_status`. Use these for retry decisions, not the message text.
- Long-running tools (`deploy_start`, `deploy_rollback`, `deploy_impact`, `publish_start`, `extlib_upload`, `extlib_publish`, `extlib_download_source`) return an operation/publication/analysis id immediately. Poll the matching `*_status` tool (`deploy_status`, `deploy_impact_status`, `publish_status`, `extlib_status`, `extlib_download_status`) until terminal; don't expect inline completion.
- `mentor` is the exception: it streams per-step events as `notifications/progress` while the turn runs, and returns the summary on the same call.

## Names

The platform exposes two name forms for every asset:
- `name`: the display name (human-readable, may contain spaces, e.g. `"AI Agent Feedback Portal"`). This is what `app_list` and `app_info` return.
- `assetName`: the internal identifier (no spaces, e.g. `"AIAgentFeedbackPortal"`). This is what `context_*` results carry, and it's what some platform-internal contexts use.

Both forms refer to the same asset. The `app:` parameter on `context_*` (and `app_list --search`) accepts either form: substring match runs case-insensitively against the display name AND a space-stripped variant of it, so `"feedback"`, `"Agent Feedback"`, and `"AgentFeedbackSystem"` all resolve the same asset.

The canonical identifier is the **asset key** (a UUID). Prefer it when storing references across calls, since names can be edited but the key is stable.

Each `context_*` row also carries `isReferenced` (true means inherited from a referenced library) and `producerAssetName` (e.g. `"OutSystemsUI"`). See the `owned_only` semantics under "Tools at a glance".

## Mentor session round-trip

`mentor` is a multi-turn conversation backed by a server-side session that holds the loaded OML.

- **First turn** args: `{app_key, prompt}` (+ optional `mode`, `max_turn_time`, `verbose`). The server downloads the OML, runs the agent, and returns `{mentor_session_id, mentor_session_token, summary, events}`.
- **Resume turn** args: `{mentor_session_id, mentor_session_token, prompt}` (+ optional `mode`, `max_turn_time`, `verbose`). `app_key` is taken from the validated token and ignored if you pass it.
- The `mentor_session_token` is HMAC-signed and binds (tenant, user, session id, app, agent_resume_id). Echo it back **verbatim** on every resume turn; tampering yields `ValidationError`.
- Each turn returns a fresh token. Use the latest one on the next call.
- Sessions auto-GC after 30 min idle. Resuming after GC transparently re-downloads the OML (sticky-miss recovery); same `mentor_session_id` and conversation continue.
- To call `publish_start` on the edited OML, you need both `mentor_session_id` and `mentor_session_token` from the most recent turn.

## Tools at a glance

Tool names, descriptions, and arg schemas come from your tool catalog (the server's `tools/list`). The content here covers cross-tool behavior the per-tool descriptions can't carry.

The surface groups into:

- **Identity**: `auth_status`
- **Apps**: `app_list`, `app_info`, `app_revisions`, `app_refs`
- **Deployments**: `deploy_start`, `deploy_rollback`, `deploy_status`, `deploy_list`, `deploy_messages`, `deploy_logs`, `deploy_impact`, `deploy_impact_status`
- **Publish**: `publish_start`, `publish_status`, `publish_logs` (always sources OML from a mentor session; there is no `oml` argument)
- **Mentor**: `mentor` (multi-turn edits and deep OML questions; see "Mentor session round-trip")
- **Context (search and browse)**: `context_search`, `context_entities`, `context_actions`, `context_screens`, `context_structures`, `context_roles`, `context_themes`, `context_connections`
- **Environments**: `env_list`, `env_info`, `env_apps`, `env_app`
- **External libraries**: `extlib_upload`, `extlib_publish`, `extlib_status`, `extlib_operations_list`, `extlib_contents`, `extlib_logs`, `extlib_delete`, `extlib_download_source`, `extlib_download_status`

Long-running tools (`deploy_start`, `deploy_rollback`, `deploy_impact`, `publish_start`, `extlib_upload`, `extlib_publish`, `extlib_download_source`) return an operation/publication/analysis id immediately and require polling the matching `*_status` tool. `mentor` is the only exception; it streams progress and returns inline.

### `context_*` semantics

Every `context_*` tool accepts `app` (asset key UUID, or case-insensitive name substring; errors on 0 or >1 match), `limit`, and `offset`. The seven typed tools (`context_entities`, `_actions`, `_screens`, `_structures`, `_roles`, `_themes`, `_connections`) also accept `owned_only`, which defaults to `true` when `app` is set, `false` when querying tenant-wide.

The Context Service indexes by visibility, not ownership: an app-scoped query returns rows owned by the app **plus** rows inherited from referenced libraries (OutSystemsUI, Charts, OutSystemsMaps, etc.), all carrying the visiting app's `assetKey`. The handler distinguishes them via `isReferenced` and surfaces `producerAssetKey` / `producerAssetName` for inherited rows. Set `owned_only: false` to keep the inherited rows in the response (they remain tagged so callers can group by producer).

### Caveats

A handful of behaviors that the per-tool descriptions in your catalog don't spell out:

- `app_refs` is cheap. Server roundtrips OML through a temp file and deletes it before returning. Use it as the "what does this depend on" check before edits.
- `deploy_start`: when neither `build_key`+`revision` nor `from_env` is supplied and `operation != "Undeploy"`, the call fails. The target `env_key` is never used as the build source. Pass `from_env` for cross-env promotion.
- `deploy_impact`: `delete=false` (default) requires `env_key`; `delete=true` runs deletion-impact and `env_key` is ignored.
- `deploy_impact_status`: `kind` (`"deployment"` or `"deletion"`) must match the call you made; they hit different endpoints.
- `publish_start`: `app_key` comes from the signed `mentor_session_token`, not from arguments. Always pass `mentor_session_id` + `mentor_session_token` from the most recent mentor turn.
- `publish_logs`: `pub_key` is the publication id (returned by `publish_start` / `publish_status`), NOT the app key. Easy to confuse.
- `extlib_upload`: `zip_b64` decoded must be <= 50 MB; concurrent uploads are gated per replica. `auto_publish=true` advances through `Published` without a human review step.
- `extlib_publish`: approves a `ReadyForReview` op to advance through `Publishing` to `Published`. State machine; won't accept ops in other states.
- `extlib_operations_list`: failed operations expose the upload `filename` even when the platform's `libraryName` is missing. Useful for matching a failure back to its source upload.

## Workflows

**Describe an existing app (no OML needed):**
1. If you only have a name, `app_list {search: "<name>"}`, or pass the name directly to `app` on the context tools and let it resolve.
2. Run per-type lookups in parallel:
   - `context_screens {app}` for UI surface
   - `context_entities {app}` for data model
   - `context_actions {app}` for logic
   - `context_roles {app}` for security
   - `app_refs {key}` for dependencies on other modules / external libraries
3. Synthesize into the user-facing explanation.

**Edit an existing app and ship it:**
1. First turn: `mentor {app_key, prompt: "Add a due date field to Task"}`. Keep `mentor_session_id` + `mentor_session_token` from the response.
2. Optional follow-up turns: `mentor {mentor_session_id, mentor_session_token, prompt: "..."}`. Each turn returns a fresh token; use the newest one next.
3. `publish_start {mentor_session_id, mentor_session_token, env_key}` returns `publication_id`.
4. Poll `publish_status {publication_id}` until terminal. Use `publish_logs {pub_key: publication_id}` for messages on failure.

**Promote a build across environments:**
1. `deploy_start {asset_key, env_key: "<target>", from_env: "<source>"}` (or pin with `build_key`+`revision`) returns operation key.
2. Poll `deploy_status {operation_key}` until terminal.
3. On failure: `deploy_messages {operation_key}` for diagnostic messages.

**Publish a new external library:**
1. Build a .NET 8 lib with `[OSInterface(Name = "<UniqueName>")]` (reusing a name produces a new revision, not a fresh asset). `dotnet publish -c Release`, zip the `.dll` + `.deps.json` at the zip root (no nested folder). Base64-encode the zip.
2. `extlib_upload {zip_b64: "<base64>", auto_publish: true}` returns operation key.
3. Poll `extlib_status {operation_key}` until `Published`. On validation failure use `extlib_logs {operation_key}`.

**Reference an external library from an app:**
- Just ask mentor: `mentor {app_key, prompt: "Use the <ActionName> action from <LibraryName> in <screen edit>."}`.

**Run a deployment-impact analysis:**
1. `deploy_impact {key, env_key}` returns analysis id.
2. Poll `deploy_impact_status {analysis_id, kind: "deployment"}` until terminal. Use `kind: "deletion"` instead when you started the analysis with `delete: true`.
