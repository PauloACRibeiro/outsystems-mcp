# ase-mcp

Distribution repo for the OutSystems MCP: Claude Code plugin + Kiro Power. The MCP server itself lives in `OutSystems/mcp-server-remote`. To install, paste the matching prompt below into your AI assistant.

## Install - Claude Code

Paste into Claude Code:

```
Install the OutSystems ase-mcp plugin from OutSystems/ase-mcp on GitHub.
Step 1: run `claude plugin marketplace add OutSystems/ase-mcp`.
Step 2: run `claude plugin install outsystems@outsystems`.
Step 3: ask me for my OutSystems tenant hostname (something like `eng-stage-us-01.outsystems.dev`).
Step 4: when I tell you, run `claude mcp add -s user --transport http --client-id service_studio --callback-port 7890 outsystems https://datap-stage-us-east-1-01.stage-07.stamp.outsystemscloudrd.net/<my-tenant>/mcp` (substitute my actual tenant for `<my-tenant>`).
Step 5: tell me to restart Claude Code, then ask anything OutSystems-related; you'll drive the OAuth flow automatically via the `authenticate` tool. Do NOT tell me to run `/mcp -> outsystems -> Authenticate` manually.
```

## Install - Kiro Chat

Paste into Kiro Chat:

```
Install the OutSystems Power from https://github.com/OutSystems/ase-mcp.
Step 1: clone the repo to ~/git/ase-mcp if it isn't there yet: `git clone https://github.com/OutSystems/ase-mcp.git ~/git/ase-mcp`.
Step 2: write ~/.kiro/powers/registries/outsystems.json with this exact content (substitute the literal value of $HOME):
{"name":"OutSystems","type":"local","powers":[{"name":"outsystems","displayName":"OutSystems - MCP","description":"Edit, publish, deploy OutSystems apps from your AI assistant.","iconUrl":"file://$HOME/git/ase-mcp/kiro/outsystems/icon.png","source":{"type":"local","path":"$HOME/git/ase-mcp/kiro/outsystems"},"autoInstall":true}]}
Step 3: tell me to restart Kiro so it auto-installs the Power.
Step 4: after the restart, ask me for my OutSystems tenant hostname.
Step 5: when I tell you, set the URL `https://datap-stage-us-east-1-01.stage-07.stamp.outsystemscloudrd.net/<my-tenant>/mcp` in two files (read first, preserve every other entry):
  - ~/.kiro/powers/installed/outsystems/mcp.json: set mcpServers.outsystems.url
  - ~/.kiro/settings/mcp.json: set powers.mcpServers["power-outsystems-outsystems"].url
Step 6: tell me the OAuth flow opens automatically on the next OutSystems tool call; you'll drive auth via the `authenticate` tool, not by clicking in Kiro's UI.
```

## Upgrading from `odc-mcp`

If you previously installed this as `odc-mcp` (versions 0.4.x and earlier), the rename to `outsystems` requires a re-install:

- **Claude Code**: `claude plugin uninstall odc-mcp@odc-mcp` and `claude mcp remove odc`, then follow the install prompt above. You'll need to re-authenticate (OAuth state is keyed per server name).
- **Kiro**: delete `~/.kiro/powers/registries/outsystems-odc.json` and `~/.kiro/powers/installed/odc-mcp/`, drop the `odc-mcp` entry from `~/.kiro/powers/installed.json` (`installedPowers[]`), and remove the `power-odc-mcp-odc` entry from `~/.kiro/settings/mcp.json` `powers.mcpServers`. Then follow the install prompt above.

