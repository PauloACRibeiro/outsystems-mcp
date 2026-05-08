# ase-mcp

Distribution repo for the OutSystems Developer Cloud (ODC) remote MCP — Claude Code plugin + Kiro Power. The MCP server itself lives in `OutSystems/mcp-server-remote`. To install, paste the matching prompt below into your AI assistant.

## Install — Claude Code

Paste into Claude Code:

```
Install the OutSystems ase-mcp plugin from OutSystems/ase-mcp on GitHub.
Step 1: run `claude plugin marketplace add OutSystems/ase-mcp`.
Step 2: run `claude plugin install odc-mcp@odc-mcp`.
Step 3: ask me for my OutSystems tenant hostname (something like `eng-stage-us-01.outsystems.dev`).
Step 4: when I tell you, run `claude mcp add -s user --transport http --client-id service_studio --callback-port 7890 odc https://datap-dev-us-east-1-01.dev-06.stamp.outsystemscloudrd.net/<my-tenant>/mcp` (substitute my actual tenant for `<my-tenant>`).
Step 5: tell me to restart Claude Code and run `/mcp → odc → Authenticate` to complete the OAuth flow.
```

## Install — Kiro Chat

Paste into Kiro Chat:

```
Install the OutSystems ODC Power from https://github.com/OutSystems/ase-mcp.
Step 1: clone the repo to ~/git/ase-mcp if it isn't there yet: `git clone https://github.com/OutSystems/ase-mcp.git ~/git/ase-mcp`.
Step 2: write ~/.kiro/powers/registries/outsystems-odc.json with this exact content (substitute the literal value of $HOME):
{"name":"OutSystems Developer Cloud","type":"local","powers":[{"name":"odc-mcp","displayName":"OutSystems Developer Cloud (ODC) — remote MCP","description":"Edit, publish, deploy OutSystems apps from your AI assistant.","source":{"type":"local","path":"$HOME/git/ase-mcp/kiro/odc"},"autoInstall":true}]}
Step 3: tell me to restart Kiro so it auto-installs the Power.
Step 4: after the restart, ask me for my OutSystems tenant hostname.
Step 5: when I tell you, set the URL `https://datap-dev-us-east-1-01.dev-06.stamp.outsystemscloudrd.net/<my-tenant>/mcp` in two files (read first, preserve every other entry):
  - ~/.kiro/powers/installed/odc-mcp/mcp.json: set mcpServers.odc.url
  - ~/.kiro/settings/mcp.json: set powers.mcpServers["power-odc-mcp-odc"].url
Step 6: tell me the OAuth flow opens automatically on the next ODC tool call.
```

