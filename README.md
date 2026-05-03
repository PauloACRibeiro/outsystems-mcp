# ase-mcp

Plugin and Kiro Power for the OutSystems Developer Cloud (ODC) remote MCP. Install this into Claude Code or Kiro and the agent gains direct access to your ODC tenant: list/edit/publish/deploy apps, run impact analysis, search tenant elements, manage external libraries.

The MCP server itself (Rust, Helm, Dockerfile, AS proxy) lives in a separate repo: **`OutSystems/mcp-server-remote`**. This repo carries only the user-facing distribution: plugin manifests, Power assets, and the agent skill that drives setup + tool use.

## What's in this repo

```
.claude-plugin/        # Claude Code plugin + marketplace manifests
skills/odc/SKILL.md    # Canonical agent skill — recipe + tool reference
kiro/odc/
  POWER.md             # Kiro Power doc + frontmatter
  mcp.json             # per-power MCP config template (sentinel URL)
  steering/skill.md    # symlink → ../../../skills/odc/SKILL.md (single source of truth)
```

The skill text lives in `skills/odc/SKILL.md`. Both Claude Code and Kiro consume it — Claude reads it directly via the plugin's `skills:` field; Kiro picks it up via the steering symlink when it auto-installs the Power. Edit the canonical file, both platforms get the change.

## Install — Claude Code

You need GitHub access to the OutSystems org (`gh auth login` or git credential helper).

```bash
claude plugin marketplace add OutSystems/ase-mcp
claude plugin install odc-mcp@odc-mcp
```

Open a chat. The agent detects no `odc` MCP server is configured, asks you for your tenant hostname (e.g. `eng-stage-us-01.outsystems.dev`), and runs:

```
claude mcp add -s user --transport http \
  --client-id service_studio --callback-port 7890 \
  odc https://mcp-test.<region>.<cluster>.outsystemscloudrd.net/<tenant>/mcp
```

Restart your Claude session, then `/mcp → odc → Authenticate`. OAuth opens in your browser; the proxy injects `kc_idp_hint=os-default-idp` and handles the PKCE callback.

## Install — Kiro

Kiro 0.11.133 or newer. Two paths; pick whichever matches your setup.

### Path A — clone the repo locally, point Kiro at the local copy

```bash
git clone https://github.com/OutSystems/ase-mcp ~/git/ase-mcp

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

### Path B — let Kiro clone the repo itself

Requires `git` credentials configured (HTTPS via `gh auth setup-git`, or SSH agent loaded).

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

Restart Kiro. The Power appears in Kiro's Powers UI; `odc` shows up in the MCP servers list at `~/.kiro/settings/mcp.json` `powers.mcpServers["power-odc-mcp-odc"]` with the sentinel URL. Open Kiro Chat and ask anything OutSystems-related — the agent walks through the tenant prompt and patches the URL into both `~/.kiro/powers/installed/odc-mcp/mcp.json` and `~/.kiro/settings/mcp.json`. Kiro's file watcher picks up the change and OAuth fires on the next tool call.

## Switching tenants

- **Claude Code:** ask the agent to "switch ODC to a different tenant" — it'll re-run `claude mcp add` with the new URL (overwriting the existing entry).
- **Kiro:** same — the agent re-prompts and patches the two config files. No restart needed; Kiro reloads MCP automatically on file change.

## Architecture notes

- **One canonical skill file.** `skills/odc/SKILL.md` is the source of truth for what the agent knows. `kiro/odc/steering/skill.md` is a symlink to it (Linux/macOS); Windows users may need to copy manually.
- **No setup script.** The first-time tenant prompt happens in the chat itself — the agent reads the skill, asks the user, and uses platform-native tools (`claude mcp add` for Claude, file Read/Write for Kiro) to wire the MCP server. No shell script ships in this repo.
- **Sentinel URL.** The Power ships with `https://mcp-test.../UNCONFIGURED-TENANT/mcp` so MCP fails fast and visibly until the tenant prompt completes. The agent recognizes this string and triggers the prompt on first use.
- **Server-side tenant routing.** The proxy at `mcp-test.<region>.<cluster>.outsystemscloudrd.net/<tenant>/mcp` discriminates tenants by URL path; each tenant has its own Keycloak realm. There's no shared OutSystems IdP today, which is why the user has to identify their tenant once at install time.

## Versioning

`.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` carry the version number. Bump both together when shipping changes.
