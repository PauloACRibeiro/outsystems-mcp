---
name: odc
description: "OutSystems Developer Cloud — remote MCP. Edit apps, publish, deploy, run impact analysis, search tenant elements, manage external libraries. Use for ANY OutSystems/ODC task."
---

# OutSystems Developer Cloud (ODC) — Remote MCP

You are connected to ODC over the remote MCP HTTP transport. ODC is a cloud-native low-code platform where apps are built from OML (OutSystems Model Language) — a binary format describing entities, screens, actions, and logic. Every tool call carries the harness's validated OAuth bearer; tenant + user identity are derived from the JWT, not from arguments.

## First use / setup

Before using any `odc` tool, the MCP server has to be registered against the user's tenant. If the tools aren't visible in your toolset, or if a call returns `tenant not configured` / connection errors, do this — once, and only once per machine:

1. **Ask the user for their OutSystems tenant hostname.** Format: `<short>-<region>-<index>.outsystems.dev` (e.g. `eng-stage-us-01.outsystems.dev`). Prompt verbatim:
   > "Which OutSystems tenant should I connect to? It's the host portion of your ODC URL — typically something like `eng-stage-us-01.outsystems.dev`."
2. **Validate** the answer matches `^[A-Za-z0-9]([A-Za-z0-9.-]*[A-Za-z0-9])?$`. If it doesn't, ask again with an example. Do NOT accept URLs, paths, or anything that looks like more than a bare hostname.
3. **Construct the MCP URL**: `https://datap-dev-us-east-1-01.dev-06.stamp.outsystemscloudrd.net/<TENANT>/mcp`.
4. **Detect the host and register the server.** Try Claude Code first; fall back to Kiro:
   - **Claude Code** (`claude` CLI is on PATH): run
     ```
     claude mcp add -s user --transport http --client-id service_studio --callback-port 7890 odc <URL>
     ```
     Then tell the user: "Run `/mcp` and pick `odc → Authenticate` — your browser will open the OutSystems login flow."
   - **Kiro** (`~/.kiro/settings/` exists, no `claude` CLI, OR explicitly running inside Kiro Chat): use your file Read+Write tools to patch two files. **Read first, preserve siblings, write back** — never clobber other entries.
     - `~/.kiro/powers/installed/odc-mcp/mcp.json`: set `mcpServers.odc` to
       ```json
       {"type": "http", "url": "<URL>", "timeout": 100000}
       ```
     - `~/.kiro/settings/mcp.json`: set `powers.mcpServers["power-odc-mcp-odc"]` to the same object. Always keep a top-level `mcpServers` key (empty object is fine if no other servers); Kiro's user-level loader rejects the file without it. If the existing file has comments (JSONC), preserve them — only modify the URL field.

     Then tell the user: "Restart Kiro or wait a few seconds for the MCP file watcher to reload — the OAuth flow opens automatically on the next `odc` tool call."

5. **Cache the tenant** for future reference: `printf '%s\n' '<TENANT>' > ~/.config/odc/tenant` (create the parent dir first if needed). This is informational only — Kiro and Claude both store the actual MCP config themselves; the cache file just lets a reinstall pick up the prior choice.

6. **Retry the user's original request** once they confirm authentication completed.

If the user already has an `odc` MCP server registered but it's pointing at the wrong tenant, follow the same flow — the patches are idempotent for the same tenant and update the URL for a new one.

## Rules
- This tool surface is agent-focused. Don't expose raw tool output to the user; extract the relevant fields and present them naturally.
- Go straight to the task. No setup checks, no auth pre-flight — identity comes from the bearer the harness already negotiated.
- OML stays server-side. There is no `app_download`. To inspect an app, use `app_refs` + `context_*` tools. To edit an app, use `mentor` (OML lives in the server-side mentor session, never crosses the wire as bytes).
- If `env_key`, `app_key`, an asset key, or a `mentor_session_*` token is missing and you can't reliably resolve it, ask the user. Never guess.
- Run independent tool calls in parallel when possible (e.g. once you have an app key, fetch info + per-type context lookups concurrently).
- Errors carry a structured category in `data.category` (one of `AuthError`, `ValidationError`, `UpstreamError`, `InternalError`); upstream errors also include `data.upstream_status`. Use these for retry decisions, not the message text.
- Long-running tools (`deploy_start`, `deploy_rollback`, `deploy_impact`, `publish_start`, `extlib_upload`, `extlib_publish`, `extlib_download_source`) return an operation/publication/analysis id immediately. Poll the matching `*_status` tool (`deploy_status`, `deploy_impact_status`, `publish_status`, `extlib_status`, `extlib_download_status`) until terminal — don't expect inline completion.
- `mentor` is the exception: it streams per-step events as `notifications/progress` while the turn runs, and returns the summary on the same call.

## Names

The platform exposes two name forms for every asset:
- `name` — the display name (human-readable, may contain spaces, e.g. `"AI Agent Feedback Portal"`). This is what `app_list` and `app_info` return.
- `assetName` — the internal identifier (no spaces, e.g. `"AIAgentFeedbackPortal"`). This is what `context_*` results carry, and it's what some platform-internal contexts use.

Both forms refer to the same asset. The `app:` parameter on `context_*` (and `app_list --search`) accepts either form: substring match runs case-insensitively against the display name AND a space-stripped variant of it, so `"feedback"`, `"Agent Feedback"`, and `"AgentFeedbackSystem"` all resolve the same asset.

The canonical identifier is the **asset key** (a UUID). Prefer it when storing references across calls, since names can be edited but the key is stable.

Each `context_*` row also carries `isReferenced` (true ⇒ inherited from a referenced library) and `producerAssetName` (e.g. `"OutSystemsUI"`) — see the `owned_only` semantics below.

## Mentor session round-trip

`mentor` is a multi-turn conversation backed by a server-side session that holds the loaded OML.

- **First turn** args: `{app_key, prompt}` (+ optional `mode`, `max_turn_time`, `verbose`). The server downloads the OML, runs the agent, and returns `{mentor_session_id, mentor_session_token, summary, events}`.
- **Resume turn** args: `{mentor_session_id, mentor_session_token, prompt}` (+ optional `mode`, `max_turn_time`, `verbose`). `app_key` is taken from the validated token and ignored if you pass it.
- The `mentor_session_token` is HMAC-signed and binds (tenant, user, session id, app, agent_resume_id). Echo it back **verbatim** on every resume turn — tampering yields `ValidationError`.
- Each turn returns a fresh token. Use the latest one on the next call.
- Sessions auto-GC after 30 min idle. Resuming after GC transparently re-downloads the OML (sticky-miss recovery) — same `mentor_session_id` and conversation continue.
- To call `publish_start` on the edited OML, you need both `mentor_session_id` and `mentor_session_token` from the most recent turn.

## Tools

### Identity
- `auth_status` — returns the validated JWT claims `{logged_in, tenant_id, user_id, claims}`. Useful for confirming the bearer the harness sent before doing anything else.

### Apps
- `app_list {search?, limit?, offset?}` — list apps; `search` is a case-insensitive substring.
- `app_info {key}` — app details.
- `app_revisions {key}` — revision history.
- `app_refs {key}` — what other modules this asset depends on (server fetches OML to a per-call temp file, computes refs, deletes the file before returning). Cheap dependency check before edit/deploy.

### Deployments

`deploy_start` and `deploy_rollback` always run with `no_wait=true` on this transport — they return an operation key immediately; poll `deploy_status` to terminal.

- `deploy_start {asset_key, env_key, build_key?, revision?, from_env?, operation?}` — promote a build to `env_key`. Cross-env: pass `from_env` to source the build from a different env (latest finished `Release` build wins). To pin an exact build, pass both `build_key` and `revision`. `operation` is `"Deploy"` (default), `"Undeploy"`, or `"ApplyConfigs"`. **HTTP-specific:** when `build_key`+`revision` are both omitted and `operation != "Undeploy"`, `from_env` is required (the target `env_key` is not used as a source).
- `deploy_rollback {asset_key, env_key}` — rollback to previous deployed version in `env_key`.
- `deploy_status {operation_key}` — poll a deploy operation until terminal.
- `deploy_list {asset_key?, env_key?}` — list deploy operations, optionally filtered.
- `deploy_messages {operation_key}` — messages for a deploy operation.
- `deploy_logs {env_key}` — recent deployments in `env_key`.
- `deploy_impact {key, delete?, env_key?}` — start an impact analysis. `delete=false` (default) needs `env_key`; `delete=true` runs deletion-impact and `env_key` is ignored. Returns an analysis id immediately; poll `deploy_impact_status`.
- `deploy_impact_status {analysis_id, kind}` — poll the analysis. `kind` must be `"deployment"` or `"deletion"` and must match the call you made.

### Publish

`publish_start` runs no_wait on this transport. There is no `oml` argument — the OML comes from the mentor session.

- `publish_start {mentor_session_id, mentor_session_token, env_key}` — upload the session's edited OML, build, and deploy to `env_key`. Returns `{publication_id}` immediately. The `app_key` is taken from the signed token, not from arguments.
- `publish_status {publication_id}` — poll until terminal. Same response shape as a polled-to-completion publish.
- `publish_logs {pub_key}` — publication messages. `pub_key` is the publication id, **not** the app key.

### Mentor (edits + deep questions)
- `mentor` — see "Mentor session round-trip" above for the args/response shapes.
- For info about an app, prefer `context_*` tools — lightweight, structured, no OML download.
- Only fall back to `mentor` for info queries when context genuinely can't answer (deep OML internals, logic flow traversal).
- For edits, `mentor` is the only path.
- Reuse the same session across follow-up turns within one task — the server-side OML and recent tool history stay loaded, which is cheaper and more coherent.
- Start a fresh session (call without `mentor_session_*`) when: (a) mentor hallucinates entities/actions that don't exist; (b) you switch to an unrelated task on the same app; or (c) prior turns left the OML in a bad state.
- If mentor refuses or returns an empty response, rephrase with concrete keys and a smaller scope before retrying.

### Tenant elements (search and browse)

Every `context_*` tool accepts:
- `app` — scope to one app (asset key UUID, or case-insensitive name substring; errors on 0 or >1 match). Omit for tenant-wide.
- `limit` — max results per page (server default: 20).
- `offset` — pagination start (default: 0). Responses include `pagination.nextPageOffset`.

The seven typed tools (`context_entities`, `_actions`, `_screens`, `_structures`, `_roles`, `_themes`, `_connections`) also accept `owned_only` — defaults to `true` when `app` is set, `false` when querying tenant-wide. The Context Service indexes by visibility, not ownership: an app-scoped query returns rows owned by the app **plus** rows inherited from referenced libraries (OutSystemsUI, Charts, OutSystemsMaps, etc.) — all carrying the visiting app's `assetKey`. The handler distinguishes them via `isReferenced` and surfaces `producerAssetKey` / `producerAssetName` for inherited rows. Set `owned_only: false` to keep the inherited rows in the response (they remain tagged so callers can group by producer).

- `context_search {query, search_type?, objects?, app?, limit?, offset?}` — search. `search_type` is `"semantic"` or omitted. `objects` is an array (e.g. `["Entities","Actions"]`).
- `context_entities {app?, owned_only?, limit?, offset?}` — entities (data model).
- `context_actions {app?, owned_only?, limit?, offset?}` — server/client actions.
- `context_screens {app?, owned_only?, limit?, offset?}` — screens (UI surface).
- `context_structures {app?, owned_only?, limit?, offset?}` — structures.
- `context_roles {app?, owned_only?, limit?, offset?}` — security roles.
- `context_themes {app?, owned_only?, limit?, offset?}` — themes.
- `context_connections {app?, owned_only?, limit?, offset?}` — AI model connections.

### Environments

There is no per-session "selected environment" on this transport — every environment-scoped tool takes `env_key` as a per-call argument.

- `env_list` — list environments in the tenant.
- `env_info {env_key}` — details of one environment.
- `env_apps {env_key, search?}` — deployed apps in `env_key`.
- `env_app {env_key, key}` — deployed-app details.

### External libraries (ELGS — generation, publish, lookup)

`extlib_upload`, `extlib_publish`, and `extlib_download_source` run no_wait on this transport.

- `extlib_upload {zip_b64, auto_publish?}` — base64-encoded zip bytes inline (decoded ≤ 50 MB; concurrent uploads gated per replica). `auto_publish=true` runs the operation through `Published` without a human review step. Returns an operation key immediately; poll `extlib_status`.
- `extlib_publish {operation_key}` — approve a `ReadyForReview` op → `Publishing` → `Published`. Returns immediately; poll `extlib_status`.
- `extlib_operations_list` — generation operations (publish jobs) in the tenant. Failed operations carry the upload `filename` even when the platform's `libraryName` is missing — useful for telling which library the failure was for.
- `extlib_status {operation_key}` — status of one operation.
- `extlib_contents {operation_key}` — extracted actions/structures (with GUID keys).
- `extlib_logs {operation_key}` — validation messages.
- `extlib_delete {operation_key}` — delete a generation operation.
- `extlib_download_source {asset_key, revision}` — start a source-zip download. Returns an operation key immediately.
- `extlib_download_status {operation_key, asset_key, revision}` — poll the download. When status is `ReadyForDownload`, the response carries a `fileUri` the harness fetches directly.

## Workflows

**Describe an existing app (no OML needed):**
1. If you only have a name, `app_list {search: "<name>"}` — or pass the name directly to `app` on the context tools and let it resolve.
2. Run per-type lookups in parallel:
   - `context_screens {app}` → UI surface
   - `context_entities {app}` → data model
   - `context_actions {app}` → logic
   - `context_roles {app}` → security
   - `app_refs {key}` → dependencies on other modules / external libraries
3. Synthesize into the user-facing explanation.

**Edit an existing app and ship it:**
1. First turn: `mentor {app_key, prompt: "Add a due date field to Task"}` → keep `mentor_session_id` + `mentor_session_token` from the response.
2. Optional follow-up turns: `mentor {mentor_session_id, mentor_session_token, prompt: "..."}` — each turn returns a fresh token; use the newest one next.
3. `publish_start {mentor_session_id, mentor_session_token, env_key}` → returns `publication_id`.
4. Poll `publish_status {publication_id}` until terminal. Use `publish_logs {pub_key: publication_id}` for messages on failure.

**Promote a build across environments:**
1. `deploy_start {asset_key, env_key: "<target>", from_env: "<source>"}` (or pin with `build_key`+`revision`) → returns operation key.
2. Poll `deploy_status {operation_key}` until terminal.
3. On failure: `deploy_messages {operation_key}` for diagnostic messages.

**Publish a new external library:**
1. Build a .NET 8 lib with `[OSInterface(Name = "<UniqueName>")]` (reusing a name produces a new revision, not a fresh asset). `dotnet publish -c Release`, zip the `.dll` + `.deps.json` at the zip root (no nested folder). Base64-encode the zip.
2. `extlib_upload {zip_b64: "<base64>", auto_publish: true}` → returns operation key.
3. Poll `extlib_status {operation_key}` until `Published`. On validation failure use `extlib_logs {operation_key}`.

**Reference an external library from an app:**
- Just ask mentor: `mentor {app_key, prompt: "Use the <ActionName> action from <LibraryName> in <screen edit>."}`.

**Run a deployment-impact analysis:**
1. `deploy_impact {key, env_key}` → returns analysis id.
2. Poll `deploy_impact_status {analysis_id, kind: "deployment"}` until terminal. Use `kind: "deletion"` instead when you started the analysis with `delete: true`.
