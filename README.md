# Agent 365 — MCP App for Microsoft 365 Copilot

> **The first authenticated MCP App sample** — calls Microsoft Graph API via the On-Behalf-Of (OBO) flow and renders interactive Fluent UI widgets inside M365 Copilot Chat.
>
> ⚠️ **This is a reference implementation / experiment** — intended to demonstrate patterns and best practices for building authenticated MCP Apps with rich UI. Not intended for production use as-is.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

<p align="center">
  <img src="media/Agent365-MCPApps.gif" alt="Agent 365 MCP App demo" width="800" />
</p>

Surface the full **Microsoft Agent 365** governance experience inside M365 Copilot Chat: browse the agent registry, visualize the agent landscape, monitor risky agents, and take admin actions — all through natural language with rich interactive widgets.

---

## What This Agent Can Do

### Rich UI Tools (render interactive widgets in chat)

| Tool | Widget | Description |
|------|--------|-------------|
| `show_agent_registry` | Agent Registry | Full inventory of all AI agents with search, filters, and detail drawer |
| `show_agent_map` | Agent Map | Circle-packing bubble visualization grouped by publisher type |
| `show_risky_agents` | Risky Agents | Agents with active identity protection risk signals |

### Admin Action Tools (callable from widgets)

| Tool | Description |
|------|-------------|
| `block_agent` | Block a compromised agent, preventing organization-wide use |
| `unblock_agent` | Restore a previously blocked agent to active status |
| `reassign_agent` | Transfer ownership of an agent to a different user |
| `search_users` | Search organization directory to find new agent owners |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     M365 Copilot Chat                            │
│                                                                  │
│  User: "Show me the agent registry"                              │
│                                                                  │
│  ┌─────────────────────┐                                         │
│  │ Declarative Agent    │  (declarativeAgent.json)                │
│  │ "Agent 365"          │                                         │
│  └────────┬────────────┘                                         │
│           │ invokes tool via MCP plugin                          │
│  ┌────────▼────────────┐                                         │
│  │ MCP Plugin           │  (agent365-plugin.json)                 │
│  │ OAuthPluginVault     │  → triggers OAuth sign-in (first use)  │
│  └────────┬────────────┘                                         │
└───────────┼──────────────────────────────────────────────────────┘
            │ HTTPS + Bearer Token
            │ (via devtunnel in dev)
┌───────────▼──────────────────────────────────────────────────────┐
│  MCP Server (Express + StreamableHTTP)         localhost:3001    │
│                                                                  │
│  1. Extract Bearer token from Authorization header               │
│  2. OBO exchange → Graph token (via MSAL)                        │
│  3. Call Graph API (beta endpoints)                              │
│  4. Return structuredContent + text                              │
│  5. Copilot renders widget from registered UI resource           │
└──────────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────┐     ┌──────────────────────────────────────┐
│  Entra ID (Azure AD)│     │  Microsoft Graph API (beta)           │
│  OBO Token Exchange │────▶│  /copilot/admin/catalog/packages      │
│  MSAL Node          │     │  /identityProtection/riskyServiceP..  │
└─────────────────────┘     │  /users                               │
                            └──────────────────────────────────────┘
```

---

## What Makes This Unique

- **Full OAuth + OBO authentication** — demonstrates the complete token exchange flow from Copilot → your app → Microsoft Graph
- **Real Microsoft Graph API calls** — delegated user-context calls, not mock data
- **Interactive admin actions** — block/unblock/reassign agents directly from widgets
- **Mock data fallback** — toggle `USE_MOCK_DATA=true` for demos without Graph access
- **Production-ready patterns** — error handling, caching, accessibility, keyboard navigation

---

## Prerequisites

- [Node.js](https://nodejs.org/) 18+
- [Microsoft 365 Agents Toolkit (ATK)](https://aka.ms/teams-toolkit) CLI or VS Code extension
- [Dev Tunnels](https://learn.microsoft.com/azure/developer/dev-tunnels/) CLI (`devtunnel`)
- An Azure/Entra ID app registration with Graph API permissions
- M365 Copilot Chat access (included with M365 E5 or M365 Copilot license)

---

## Quick Start

### 1. Clone & Install

```bash
git clone https://github.com/<your-username>/agent365-mcpapp.git
cd agent365-mcpapp
npm install
```

### 2. Create Entra App Registration

In [Azure Portal](https://portal.azure.com) → Entra ID → App registrations:

1. **Register a new app** (single-tenant or multi-tenant)
2. **Platform:** Web
3. **Redirect URI:** `https://teams.microsoft.com/api/platform/v1.0/oAuthRedirect`
4. **API Permissions** (Delegated — all require admin consent):
   - `User.Read` (Microsoft Graph) — basic sign-in
   - `User.ReadBasic.All` (Microsoft Graph) — search users for reassignment
   - `CopilotSettings.ReadWrite.All` (Microsoft Graph) — read/block/unblock agents in the catalog
   - `IdentityRiskyServicePrincipal.Read.All` (Microsoft Graph) — read risky agent signals
5. **Expose an API:**
   - Set Application ID URI: `api://<your-client-id>`
   - Add scope: `access_as_user` (Admins and users can consent)
6. **Authorized client applications:**
   - Add `ab3be6b7-baf2-4ad0-ae4c-e0209abb4820` (M365 Copilot) for scope `access_as_user`
7. **Create a client secret** — copy the value
8. **Set `accessTokenAcceptedVersion` to `2`** in the app manifest (Entra portal → Manifest tab). Without this, OBO will fail with `AADSTS50013`.

> **Important:** The admin-scoped permissions require a Global Administrator or Privileged Role Administrator to grant consent. The *calling user* must also hold an admin role (Global Reader, Security Reader, or Copilot Admin) for `/copilot/admin/...` endpoints to work — app-level consent alone is not sufficient.

### 3. Configure Environment

```bash
# Copy sample files
cp .env.sample .env
cp env/.env.local.user.sample env/.env.local.user
```

Edit `.env`:
```env
PORT=3001
USE_MOCK_DATA=false
ENTRA_CLIENT_ID=<your-client-id>
ENTRA_CLIENT_SECRET=<your-client-secret>
ENTRA_TENANT_ID=<your-tenant-id>
```

Edit `env/.env.local.user`:
```env
AGENT365_MCP_CLIENT_ID=<your-client-id>
AGENT365_MCP_CLIENT_SECRET=<your-client-secret>
```

### 4. Build

```bash
npm run build
```

### 5. Start Dev Tunnel

```bash
devtunnel create agent365-mcp --allow-anonymous
devtunnel port create agent365-mcp --port-number 3001
devtunnel host agent365-mcp
```

Copy the tunnel URL (e.g., `https://<id>-3001.inc1.devtunnels.ms`) and update `env/.env.local`:

```env
MCP_SERVER_URL=https://<your-tunnel-id>-3001.inc1.devtunnels.ms
MCP_SERVER_DOMAIN=<your-tunnel-id>-3001.inc1.devtunnels.ms
ENTRA_TENANT_ID=<your-tenant-id>
```

### 6. Start MCP Server

```bash
npm run serve
```

### 7. Provision & Deploy

```bash
npx -y --package @microsoft/m365agentstoolkit-cli atk provision --env local
```

This will:
- Register the Teams app
- Create OAuth configuration in Developer Portal
- Build and validate the app package
- Deploy to M365 Copilot

### 8. Test in M365 Copilot

Open the URL from the provision output:
```
https://m365.cloud.microsoft/chat/?titleId=<your-M365-TITLE-ID>
```

Try: *"Show me the agent registry"* or *"Show the agent map"*

---

## Project Structure

```
agent365-mcpapp/
├── appPackage/                    # Declarative Agent manifest
│   ├── manifest.json              # Teams app manifest (v1.26)
│   ├── declarativeAgent.json      # Agent config with conversation starters
│   ├── agent365-plugin.json       # MCP plugin — tools, auth, runtime URL
│   ├── instruction.txt            # Agent behavioral instructions
│   └── color.png / outline.png    # App icons
├── env/                           # ATK environment files
│   ├── .env.local                 # Non-secret config (committed)
│   ├── .env.local.user            # Secrets (gitignored)
│   └── .env.local.user.sample     # Template for secrets
├── src/
│   ├── tools/                     # MCP tool handlers
│   │   ├── show-registry.ts       # Agent registry tool
│   │   ├── show-agent-map.ts      # Agent map tool
│   │   ├── show-risky.ts          # Risky agents tool
│   │   ├── block-agent.ts         # Block action
│   │   ├── unblock-agent.ts       # Unblock action
│   │   └── reassign-agent.ts      # Reassign action
│   ├── graph/                     # Microsoft Graph API clients
│   │   ├── client.ts              # Graph fetch abstraction + mock switch
│   │   ├── packages.ts            # /copilot/admin/catalog/packages
│   │   ├── risk.ts                # /identityProtection/riskyServicePrincipals
│   │   └── users.ts               # /users (search)
│   ├── mock/                      # Mock data for demo/testing
│   │   ├── packages.ts            # 50 sample agents
│   │   ├── risk.ts                # Sample risk signals
│   │   └── users.ts               # Sample users
│   ├── widgets/                   # React + Fluent UI widget source
│   │   ├── agent-registry/        # Registry table with filters & detail drawer
│   │   ├── agent-map/             # D3 circle-packing visualization
│   │   ├── risky-agents/          # Risk signal cards
│   │   └── shared/                # Theme, providers, shared components
│   └── types.ts                   # Shared TypeScript types
├── ui/                            # Vite HTML entry points for widgets
├── dist/ui/                       # Built single-file HTML widgets (generated)
├── auth.ts                        # MSAL OBO token exchange
├── server.ts                      # MCP server — tool + resource registration
├── main.ts                        # Express entry point
├── build-ui.mjs                   # Widget build script (Vite)
├── m365agents.yml                 # ATK provisioning lifecycle
├── .env.sample                    # Server env template
├── package.json
├── tsconfig.json                  # Client TypeScript config
├── tsconfig.server.json           # Server TypeScript config
└── vite.config.ts                 # Vite config for widget builds
```

---

## Authentication Deep-Dive

### Why OBO?

M365 Copilot obtains a token scoped to *your app* (`api://<client-id>/access_as_user`) — it cannot call Microsoft Graph directly on behalf of the user. Your MCP server uses the On-Behalf-Of (OBO) flow to exchange that incoming token for a Graph-scoped token representing the same user. This is the core pattern this sample demonstrates: **Copilot token → OBO → Graph token → API call → rich widget response**.

### How the OBO Flow Works

1. User opens Copilot Chat → asks "Show me agents"
2. Copilot invokes the MCP plugin → triggers OAuth sign-in (first time only)
3. User signs in → Copilot gets a token scoped to `api://<client-id>/access_as_user`
4. Copilot sends the request to the MCP server with `Authorization: Bearer <token>`
5. MCP server extracts the token → MSAL OBO exchange → gets a Graph-scoped token
6. Server calls Graph API with the Graph token → returns data + widget

### The `.default` Scope

The server requests `https://graph.microsoft.com/.default` (see [`auth.ts:43`](auth.ts#L43)). This means the permissions granted are controlled entirely by the **app registration's configured API permissions** — not by per-call scope strings. Add or remove permissions in Entra, grant admin consent, and the OBO token automatically reflects the change.

### Environment Variables Explained

| Variable | Used By | Purpose |
|----------|---------|---------|
| `ENTRA_CLIENT_ID` | MSAL in [`auth.ts`](auth.ts) | Identifies the app during OBO token exchange |
| `ENTRA_CLIENT_SECRET` | MSAL in [`auth.ts`](auth.ts) | Proves app identity to Entra ID |
| `ENTRA_TENANT_ID` | MSAL in [`auth.ts`](auth.ts) | Directs auth to your tenant |
| `AGENT365_MCP_CLIENT_ID` | ATK provisioning ([`m365agents.yml`](m365agents.yml)) | Same Entra app — used by ATK to register OAuth in Teams Developer Portal |
| `AGENT365_MCP_AUTH_ID` | ATK provisioning | OAuth registration ID created by ATK during `atk provision`. Clear this and re-provision if you change client ID or tunnel URL. |

> **`ENTRA_CLIENT_ID` and `AGENT365_MCP_CLIENT_ID` are the same app.** Two env vars exist because ATK provisioning and the MSAL runtime each consume the value independently.

### Why a Teams Redirect URI?

The redirect URI `https://teams.microsoft.com/api/platform/v1.0/oAuthRedirect` is required because ATK uses **Teams Developer Portal** as the OAuth registration broker for Copilot extensibility. Copilot's sign-in flow routes through this endpoint to complete the OAuth handshake.

### Entra App Registration — Required Configuration

| Setting | Value | Why |
|---------|-------|-----|
| Platform | Web | Required for server-side OBO |
| Redirect URI | `https://teams.microsoft.com/api/platform/v1.0/oAuthRedirect` | Teams Developer Portal OAuth broker |
| `accessTokenAcceptedVersion` | **2** | Must be set in the app manifest JSON. Without this, MSAL Node OBO fails with `AADSTS50013` or `AADSTS500011`. This is the #1 silent-failure cause for OBO samples. |
| Expose an API | `api://<client-id>/access_as_user` | The scope Copilot requests on sign-in |
| Authorized client | `ab3be6b7-baf2-4ad0-ae4c-e0209abb4820` (M365 Copilot) | Pre-consents Copilot to request your app's scope |

### Required API Permissions (Delegated)

All require **admin consent**. The test account must also hold an admin role (Global Reader, Security Reader, or Copilot Admin) — the `/copilot/admin/...` endpoints require admin privilege on the *calling user*, not just the app.

| Permission | Required For | Code Reference |
|------------|-------------|----------------|
| `User.Read` | Sign-in baseline | Always required |
| `User.ReadBasic.All` | `search_users` directory lookups | [`src/graph/users.ts:16`](src/graph/users.ts#L16) |
| `CopilotSettings.ReadWrite.All` | `/copilot/admin/catalog/packages` — read, block, unblock | [`src/graph/packages.ts`](src/graph/packages.ts) |
| `IdentityRiskyServicePrincipal.Read.All` | `/identityProtection/riskyServicePrincipals` | [`src/graph/risk.ts:16`](src/graph/risk.ts#L16) |

---

## Mock Data Mode

Set `USE_MOCK_DATA=true` in `.env` to run without Graph API access. The server will return realistic sample data (50 agents, risk signals, users) — perfect for UI development or demos.

---

## Widget Lifecycle in Copilot

When Copilot renders an MCP App widget, it mounts the widget iframe in **multiple render slots** against a single `tools/call` response. This means your widget's `ontoolresult` callback (see [`McpAppProvider.tsx`](src/widgets/shared/McpAppProvider.tsx)) fires independently in each iframe instance. Widgets must be idempotent — they receive the same `structuredContent` payload each time and should render identically regardless of which slot they occupy. If you see your widget "mount 4 times" during debugging, this is expected behavior, not a bug. Action tools invoked via `callServerTool()` from any slot will trigger a fresh `tools/call` to the server.

---

## Development

```bash
# Start in dev mode (watch server + widgets)
npm run dev

# Build widgets only
node build-ui.mjs

# Inspect MCP server with MCP Inspector
npm run inspector

# Type-check without build
tsc --noEmit
```

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| **AADSTS500011** — resource principal not found | Wrong client ID in `env/.env.local.user` | Ensure `AGENT365_MCP_CLIENT_ID` matches your Entra app registration |
| **No sign-in prompt** in Copilot | Stale OAuth registration | Clear `AGENT365_MCP_AUTH_ID` in `env/.env.local` and re-provision |
| **Widget not rendering** | Tool returns error or no `structuredContent` | Check server logs for Graph API errors |
| **"We couldn't find this agent"** | Stale M365 title ID | Re-provision to get a fresh `M365_TITLE_ID` |
| **Auth token MISSING on server** | Copilot not sending bearer token | Verify OAuth is registered correctly (re-provision with empty AUTH_ID) |

---

## Graph API Endpoints Used

| Endpoint | Permission | Purpose |
|----------|-----------|---------|
| `GET /beta/copilot/admin/catalog/packages` | `CopilotSettings.ReadWrite.All` | List all registered agents |
| `GET /beta/copilot/admin/catalog/packages/{id}` | `CopilotSettings.ReadWrite.All` | Get agent detail (instructions, capabilities) |
| `POST /beta/copilot/admin/catalog/packages/{id}/block` | `CopilotSettings.ReadWrite.All` | Block an agent |
| `POST /beta/copilot/admin/catalog/packages/{id}/unblock` | `CopilotSettings.ReadWrite.All` | Unblock an agent |
| `PATCH /beta/copilot/admin/catalog/packages/{id}` | `CopilotSettings.ReadWrite.All` | Reassign agent ownership |
| `GET /beta/identityProtection/riskyServicePrincipals` | `IdentityRiskyServicePrincipal.Read.All` | Risk signals for service principals |
| `GET /v1.0/users` | `User.ReadBasic.All` | Search users for reassignment |

---

## Deploying to Azure (Production)

For production, replace the dev tunnel with an Azure-hosted endpoint:

| Component | Recommended Service | Notes |
|-----------|-------------------|-------|
| MCP Server | **Azure Container Apps** or App Service | Scales to zero, built-in HTTPS |
| Secrets | **Azure Key Vault** | Referenced via App Settings |
| Identity | **Managed Identity** | No credentials needed to access Key Vault |

**Steps:**
1. Deploy the Express server to Container Apps (or App Service)
2. Store `ENTRA_CLIENT_SECRET` in Key Vault; reference it via `@Microsoft.KeyVault(SecretUri=...)`
3. Set remaining env vars (`ENTRA_CLIENT_ID`, `ENTRA_TENANT_ID`, `PORT`) as App Settings
4. Update `MCP_SERVER_URL` / `MCP_SERVER_DOMAIN` in `env/.env.local` to the Azure URL
5. Re-provision with ATK: `atk provision --env local`

No code changes required — the server reads `process.env` identically whether values come from `.env` or Azure App Settings.

---

## Related Resources

- [MCP Apps Specification](https://modelcontextprotocol.github.io/ext-apps/api/documents/Overview.html)
- [M365 Agents Toolkit](https://aka.ms/teams-toolkit)
- [Declarative Agents Documentation](https://learn.microsoft.com/microsoft-365-copilot/extensibility/overview-declarative-agent)
- [Microsoft Graph API](https://learn.microsoft.com/graph/overview)
- [MSAL Node — On-Behalf-Of Flow](https://learn.microsoft.com/entra/msal/node/how-to/on-behalf-of)

---

## License

[MIT](LICENSE)
