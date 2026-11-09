# workswarm-canva-plugin

A community Canva connector for WorkSwarm: lets the WorkSwarm agent search, create, edit, comment on and export your Canva designs through Canva's official MCP server.

Works with Canva. Not made, endorsed or certified by Canva.

## How it works

```
WorkSwarm agent ──► npx mcp-remote ──HTTPS + OAuth──► https://mcp.canva.com/mcp ──► Canva
        ▲
        └── skills/canva/SKILL.md (usage rules, installed by WorkSwarm on connect)
```

- `mcp-remote` (pinned to 0.15.2) is a local stdio bridge. It opens Canva's sign-in page on first connect, stores the login token in `~/.mcp-auth`, and relays tool calls to Canva.
- Canva's server provides the tools (search, create-design, edit transactions, export, comments, folders, brand templates, …).
- The bundled skill tells the agent how to use them safely: use the Canva tools for canva.com links (not the browser), poll async generation jobs, ask before permanent changes, and stop and report when results don't match.

## Package layout

```
canva/
├── manifest.json            package metadata, card text, examples
├── mcp.json                 launch command (npx mcp-remote@0.15.2 https://mcp.canva.com/mcp)
├── icon.png                 official Canva icon logo (unmodified)
└── skills/canva/SKILL.md    agent usage rules
```

## Install (local)

1. Copy the `canva/` folder to `~/.jiuwenswarm/agent/workspace/mcp/mcp_hub/canva`.
2. Restart WorkSwarm → 扩展 → 连接器 → 连接器广场 → **Canva 可画** → 连接.
3. Sign in to Canva in the browser window that opens. The first connect downloads `mcp-remote` and can take a minute or two.
4. In a chat, click **+ → 连接器** and select Canva before sending your request.

## Requirements and limits

- Node.js (bundled with WorkSwarm).
- Each user signs in with their own Canva account; AI generation uses that account's Canva credits and plan limits (some features need Canva Pro or Enterprise).
- Disconnecting in WorkSwarm does not sign out of Canva; delete `~/.mcp-auth` to remove the stored login.
- Use is subject to Canva's [MCP usage policy](https://www.canva.dev/docs/mcp/usage-policy/).
