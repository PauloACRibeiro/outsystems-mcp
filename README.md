# outsystems-mcp

Distribution repo for the OutSystems MCP: Claude Code plugin + Kiro Power. To install, paste the matching prompt below into your AI assistant.

## Install - Claude Code

Paste into Claude Code:

```
Install the OutSystems outsystems-mcp plugin from OutSystems/outsystems-mcp on GitHub.
Step 1: run `claude plugin marketplace add OutSystems/outsystems-mcp`.
Step 2: run `claude plugin install outsystems@outsystems`.
Step 3: ask me for my OutSystems tenant hostname (something like `eng-stage-us-01.outsystems.dev`).
Step 4: when I tell you, run `claude mcp add -s user --transport http --client-id service_studio --callback-port 7890 outsystems https://<my-tenant>/mcp` (substitute my actual tenant for `<my-tenant>`).
Step 5: tell me to restart Claude Code, then ask anything OutSystems-related; you'll drive the OAuth flow automatically via the `authenticate` tool. Do NOT tell me to run `/mcp -> outsystems -> Authenticate` manually.
```

## Install - Kiro Chat

Paste into Kiro Chat:

```
Install the OutSystems Power from https://github.com/OutSystems/outsystems-mcp.
Step 1: clone the repo to ~/git/outsystems-mcp if it isn't there yet: `git clone https://github.com/OutSystems/outsystems-mcp.git ~/git/outsystems-mcp`.
Step 2: base64-encode ~/git/outsystems-mcp/kiro/outsystems/icon.png with `base64 -w0` (Linux) or `base64 -i` (macOS). Then write ~/.kiro/powers/registries/outsystems.json with this content (substitute the literal value of $HOME, and inline the base64 string in place of <ICON_BASE64>):
{"name":"OutSystems","type":"local","powers":[{"name":"outsystems","displayName":"OutSystems - MCP","description":"Edit, publish, deploy OutSystems apps from your AI assistant.","iconUrl":"data:image/png;base64,<ICON_BASE64>","source":{"type":"local","path":"$HOME/git/outsystems-mcp/kiro/outsystems"},"autoInstall":true}]}
Step 3: tell me to restart Kiro so it auto-installs the Power.
Step 4: after the restart, ask me for my OutSystems tenant hostname.
Step 5: when I tell you, set the URL `https://<my-tenant>/mcp` in ~/.kiro/settings/mcp.json under top-level `mcpServers.outsystems` (read first, preserve every other entry): `{"type": "http", "url": "https://<my-tenant>/mcp", "timeout": 100000}`. If a stale `powers.mcpServers["power-outsystems-outsystems"]` entry exists from an earlier install pattern, remove it.
Step 6: tell me the OAuth flow opens automatically on the next OutSystems tool call; you'll drive auth via the `authenticate` tool, not by clicking in Kiro's UI.
```

## Upgrading from `odc-mcp`

If you previously installed this as `odc-mcp` (versions 0.4.x and earlier), the rename to `outsystems` requires a re-install:

- **Claude Code**: `claude plugin uninstall odc-mcp@odc-mcp` and `claude mcp remove odc`, then follow the install prompt above. You'll need to re-authenticate (OAuth state is keyed per server name).
- **Kiro**: delete `~/.kiro/powers/registries/outsystems-odc.json` and `~/.kiro/powers/installed/odc-mcp/`, drop the `odc-mcp` entry from `~/.kiro/powers/installed.json` (`installedPowers[]`), and remove the `power-odc-mcp-odc` entry from `~/.kiro/settings/mcp.json` `powers.mcpServers`. Then follow the install prompt above.

## Upgrading earlier Kiro installs (pre-`mcp.json` removal)

If you installed the Kiro Power before its per-install `mcp.json` was removed, the MCP server entry used to live at `powers.mcpServers["power-outsystems-outsystems"]` in `~/.kiro/settings/mcp.json`. The new pattern uses top-level `mcpServers.outsystems` instead, so updates can't wipe the tenant URL.

To migrate: update the Power (Kiro → Powers → Check for updates → Install updates). The Kiro update flow removes the old `power-outsystems-outsystems` namespaced entry when it uninstalls the previous version. The next OutSystems-related chat will trigger the steering flow and write the new top-level entry. You'll need to re-authenticate (OAuth state is keyed per server name; the rename `power-outsystems-outsystems` → `outsystems` doesn't carry it over).

