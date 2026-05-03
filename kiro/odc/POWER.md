---
name: "odc-mcp"
displayName: "OutSystems Developer Cloud (ODC) — remote MCP"
description: "Drive OutSystems Developer Cloud from Kiro over the remote MCP HTTP transport: edit apps, publish, deploy, run impact analysis, search tenant elements, manage external libraries."
keywords: [outsystems, odc, low-code, oml, deployment, mcp, remote-mcp]
author: "OutSystems AI Platform"
---

# OutSystems Developer Cloud (ODC) — remote MCP

## Overview

ODC is a cloud-native low-code platform where apps are built from OML (OutSystems Model Language) — a binary format describing entities, screens, actions, and logic. This Power connects Kiro to the **ODC remote MCP server**: a hosted, multi-tenant HTTP transport that exposes the full ODC tool surface (`mentor`, `app_*`, `context_*`, `deploy_*`, `publish_*`, `extlib_*`, `env_*`).

There is no CLI to install. There is no OML on disk. OML stays server-side; you edit through the `mentor` session and ship via `publish_start`.

## Onboarding

### Prerequisites

- Kiro 0.11.133 or newer.
- A web browser on the same machine as Kiro (Kiro picks an ephemeral local port for the OAuth callback after Dynamic Client Registration).
- An OutSystems tenant hostname (e.g. `eng-stage-us-01.outsystems.dev`).
- Network access to `*.outsystemscloudrd.net` (the proxy host).

### Installation

This Power ships with a **sentinel** URL — `https://mcp-test.../UNCONFIGURED-TENANT/mcp`. The first time you ask Kiro Chat to do something with ODC, the agent will notice no working MCP connection, ask you for your tenant hostname, and patch the URL into Kiro's MCP configuration. No script, no shell command. Just install the Power and start chatting.

Two ways to install the Power into Kiro (both work for an internal/private repo, assuming you have GitHub access):

**Option A — clone locally, point Kiro at the local copy.** Most deterministic; doesn't depend on Kiro shelling out to git.

```bash
# 1. Clone the toolkit somewhere (keep this clone — Kiro reads from it)
git clone https://github.com/OutSystems/ase-mcp ~/git/ase-mcp

# 2. Drop a registry pointer that references the local clone
mkdir -p ~/.kiro/powers/registries
cat > ~/.kiro/powers/registries/outsystems-odc.json <<EOF
{
  "name": "OutSystems Developer Cloud",
  "type": "local",
  "powers": [{
    "name": "odc-mcp",
    "displayName": "OutSystems Developer Cloud (ODC) — remote MCP",
    "description": "Edit, publish, deploy OutSystems apps from your AI assistant.",
    "source": {"type": "local", "path": "$HOME/git/ase-mcp/kiro/odc"},
    "autoInstall": true
  }]
}
EOF
```

**Option B — let Kiro clone the repo itself.** Simpler if your `git` credentials are configured (HTTPS via `gh auth setup-git` or SSH agent).

```bash
mkdir -p ~/.kiro/powers/registries
cat > ~/.kiro/powers/registries/outsystems-odc.json <<EOF
{
  "name": "OutSystems Developer Cloud",
  "type": "local",
  "powers": [{
    "name": "odc-mcp",
    "displayName": "OutSystems Developer Cloud (ODC) — remote MCP",
    "description": "Edit, publish, deploy OutSystems apps from your AI assistant.",
    "source": {
      "type": "repo",
      "repositoryCloneUrl": "https://github.com/OutSystems/ase-mcp",
      "pathInRepo": "kiro/odc",
      "repositoryBranch": "main"
    },
    "autoInstall": true
  }]
}
EOF
```

Restart Kiro after dropping the registry file. On startup Kiro auto-installs the Power: it copies `POWER.md`, `mcp.json`, and `steering/skill.md` into `~/.kiro/powers/installed/odc-mcp/` and adds an entry to `~/.kiro/settings/mcp.json` under `powers.mcpServers["power-odc-mcp-odc"]`. The Power appears in Kiro's Powers UI.

Then open Kiro Chat and ask it for anything OutSystems-related — the steering content takes over and the agent walks you through the tenant prompt + OAuth on first use.

### Switching tenants

Just tell Kiro Chat "switch ODC to a different tenant" (or similar). The agent will repeat the tenant prompt and update the URL in both `~/.kiro/powers/installed/odc-mcp/mcp.json` and `~/.kiro/settings/mcp.json`. Kiro's file watcher reloads MCP automatically.

## Common Workflows

Workflows below show MCP tool form. Identity (tenant + user) is derived from the validated bearer JWT — none of the tools take a `tenant` argument.

### Workflow 1: Describe an existing app

1. Resolve the app: `app_list { search: "<name>" }` → app key.
2. In parallel, gather context (independent calls):
   - `context_screens { app: "<key>" }` → UI surface
   - `context_entities { app: "<key>" }` → data model
   - `context_actions { app: "<key>" }` → logic
   - `context_roles { app: "<key>" }` → security
   - `app_refs { key: "<key>" }` → dependencies on libraries / other modules
3. Synthesize for the user.

### Workflow 2: Edit an app and ship it

1. `mentor { app_key: "<key>", prompt: "Add a due date field to Task" }` → returns `mentor_session_id`, `mentor_session_token`, `summary`, `events`.
2. (Optional) Follow-up turns: `mentor { mentor_session_id, mentor_session_token, prompt: "..." }`. Each turn returns a fresh token; use the latest.
3. `publish_start { mentor_session_id, mentor_session_token, env_key: "<env>" }` → returns `publication_id`.
4. Poll `publish_status { publication_id }` until terminal. On failure, `publish_logs { pub_key: publication_id }`.

### Workflow 3: Promote a build across environments

1. `deploy_start { asset_key: "<key>", env_key: "<target>", from_env: "<source>" }` (or pin with `build_key` + `revision`) → returns operation key.
2. Poll `deploy_status { operation_key }` until terminal.
3. On failure: `deploy_messages { operation_key }`.

### Workflow 4: Publish a new external library

1. Build a .NET 8 lib with `[OSInterface(Name = "<UniqueName>")]`. `dotnet publish -c Release`, zip `.dll` + `.deps.json` at the zip root, base64-encode it.
2. `extlib_upload { zip_b64: "<base64>", auto_publish: true }` → returns operation key.
3. Poll `extlib_status { operation_key }` until `Published`. Use `extlib_logs` on validation failure.

## Conventions

- **OML is server-side.** No `app_download`. Use `app_refs` + `context_*` for inspection; `mentor` for edits.
- **No selected environment.** Every environment-scoped tool takes `env_key` per call.
- **Operations return immediately.** `deploy_start`, `deploy_rollback`, `deploy_impact`, `publish_start`, `extlib_upload`, `extlib_publish`, `extlib_download_source` all return an operation/publication/analysis id; poll the matching `*_status` tool.
- **Never invent IDs.** App keys, env keys, build keys, operation keys are opaque. Resolve them via `app_list`, `env_list`, etc., or ask the user.

## Troubleshooting

### MCP server unreachable

**Symptom:** Kiro reports the `odc` server as not configured / not connected, or tools return errors mentioning `UNCONFIGURED-TENANT`.

**Solutions:**
1. Have you completed the first-use tenant prompt? Open Kiro Chat and ask anything ODC-related — the agent will notice the unconfigured state and walk you through it.
2. Verify the install: `jq '.installedPowers[] | select(.name=="odc-mcp")' ~/.kiro/powers/installed.json` should return the entry, and `jq '.powers.mcpServers."power-odc-mcp-odc"' ~/.kiro/settings/mcp.json` should return an object with a `https://mcp-test.<region>.<cluster>.outsystemscloudrd.net/<your-tenant>/mcp` URL.
3. If the URL still says `UNCONFIGURED-TENANT`, the tenant prompt was skipped or interrupted. Tell Kiro Chat "set up ODC again" and complete the flow.
4. Verify network reachability: `curl -I "<URL-from-settings>"` should return an HTTP response (likely 401 without a bearer — that's expected and means routing works).

### OAuth doesn't open / callback fails

**Symptom:** Kiro hangs on first tool call; no browser tab opens, or the browser shows a redirect-uri mismatch.

**Solutions:**
1. Kiro discovers OAuth from the server's `WWW-Authenticate` 401 challenge and runs Dynamic Client Registration (RFC 7591) against the AS proxy on first connect; it picks an ephemeral local callback port internally. If DCR fails, the `as-proxy` pod logs will show the registration attempt — start there.
2. Wipe any stale Kiro-side OAuth state and retry: remove the `odc` server from Kiro's MCP servers list (Kiro UI → MCP Servers → odc → Remove), restart Kiro, re-add the power. Kiro will re-register with a fresh `client_id` and re-issue tokens.

### Tool errors

Errors carry a structured category in `data.category` (`AuthError`, `ValidationError`, `UpstreamError`, `InternalError`); upstream errors also include `data.upstream_status`. Use these for retry decisions, not the message text.

## Limitations

- **No `app_download` on this transport.** OML stays in the server-side mentor session and never crosses the wire as bytes.
- **No per-session "selected environment".** Every environment-scoped tool takes `env_key` per call.
- **Long-running tools return immediately.** `deploy_start`, `deploy_rollback`, `deploy_impact`, `publish_start`, `extlib_upload`, `extlib_publish`, `extlib_download_source` all return an operation/publication/analysis id; you must poll the matching `*_status` tool.
- **Mentor session GC after 30 min idle.** Resuming after GC transparently re-downloads OML (sticky-miss recovery), but the first turn after that pause is slower.

## Configuration files

The Power's installed state (created by Kiro's auto-install when it processes the registry file):

| Path | Purpose |
|---|---|
| `~/.kiro/powers/registries/outsystems-odc.json` | LocalRegistrySchema — points at the Power source. |
| `~/.kiro/powers/installed.json` | Lists `odc-mcp` as installed. |
| `~/.kiro/powers/installed/odc-mcp/POWER.md` | This file (copied from source). |
| `~/.kiro/powers/installed/odc-mcp/mcp.json` | Per-power MCP config. The agent updates the URL on first use; Kiro then namespaces the entry into `settings/mcp.json` as `power-odc-mcp-odc`. |
| `~/.kiro/powers/installed/odc-mcp/steering/skill.md` | Agent-facing skill content; loads into the chat agent's context whenever the Power is active. Drives the tenant prompt + MCP wiring. |
| `~/.kiro/settings/mcp.json` | Kiro's MCP loader file. The Power's namespaced entry lives at `powers.mcpServers["power-odc-mcp-odc"]`. The agent updates the URL here to match `installed/odc-mcp/mcp.json`. Top-level `mcpServers` is preserved (even empty) — Kiro's user-level loader requires it. |

Schemas verified against `/opt/kiro/resources/app/extensions/kiro.kiro-agent/dist/extension.js` (Kiro 0.11.133, search for `PowerDefinitionV2Schema`, `LocalRegistrySchema`, `InstalledPowersFileSchema`, `MCPOptionsSchema`). Kiro's `MCPOptionsSchema` accepts `command/args/env/cwd/url/headers/type/timeout/disabled/autoApprove/disabledTools` — **no `oauth` block**. OAuth is discovered from the 401 challenge and Kiro handles DCR internally with an ephemeral callback port.
