# Inflow for ChatGPT

A private LinkedIn inbox with a ChatGPT sidebar entrypoint, built with **mcp-use 2.7.2**. OpenAI-specific entrypoint and fullscreen metadata use **@openai/mcp-extensions 0.1.0**. Standard tools, HTTP transport, OAuth bearer verification, Views, and React host interactions use mcp-use.

## Architecture

ChatGPT → authenticated HTTPS MCP server on Manufact → authenticated outbound long-poll bridge on your Mac → Inflow Chrome extension → your existing LinkedIn session.

The server does **not** log into LinkedIn or copy browser cookies. Inflow's database and sync engine stay in Chrome. Messages pass through the cloud server when requested, but this application does not persist them there. Disable gateway payload capture on your deployment. Keep **Chrome and the bridge running**. It cannot read your inbox while your Mac is asleep or offline. Inflow uses LinkedIn's undocumented APIs; see the upstream project's account-risk notice.

The current deployment is deliberately **single-owner, single-instance**. Relay requests and short-lived OAuth authorization codes are transient in-memory state. A deployment restart cancels in-flight requests and unfinished sign-ins; reconnect afterward. Horizontal scaling requires an external RPC broker and shared authorization-code storage. Never put multiple users behind this configuration. Rotate the signing key to revoke all connections.

## Local setup

1. Install Node.js 24+ and run `npm ci`.
2. Install [Inflow](https://github.com/grinich/inflow), sign into LinkedIn, and open Inflow.
3. Copy `.env.example` to `.env`. Generate three independent random secrets, 32+ characters each. Set the canonical server origin in `INFLOW_PUBLIC_URL`.
4. Run `npm run build && npm start`. MCP is at `/mcp`.
5. Create a private `.env.bridge` with **only** `INFLOW_PUBLIC_URL` and `INFLOW_BRIDGE_KEY`. Run `npm run bridge` on your Mac.
6. Open the pairing link printed by the bridge. Enable agent access in Inflow and save the pairing code. Write actions remain governed by Inflow's separate write toggle and send cap.

The bridge binds only `127.0.0.1:48632`, the port expected by unmodified Inflow. Stop another Inflow/Claude bridge if it already owns that port. Pairing state is saved in `.inflow-local/`, or `INFLOW_STATE_DIR` if set.

## Deploy on Manufact

Deploy from this directory with `npx mcp-use deploy --no-github --org YOUR_ORG_SLUG --json`, or use another Docker host. The deployment and credentials belong to each installer; this example does not connect to a shared service.

Keep private local files under `.env*` names or outside the source directory: managed uploads do not honor `.gitignore`. The Docker context also excludes private files.

Use the included Dockerfile, port 3000, and EU region. Set the four variables from `.env.example` in the destination server. Mark the three keys sensitive. Keep one running instance. Disable gateway request/response payload capture for this personal inbox.

`INFLOW_LOGIN_KEY` is the private connection key entered on the OAuth page. Do not put it in the repository, plugin, chat messages, screenshots, or command arguments. OAuth supports dynamic client registration, authorization-code + S256 PKCE, access tokens, and refresh tokens. It rejects unregistered redirects, wrong audiences, expired tokens, code replay, and bridge credentials presented to MCP.

## Install the plugin

The `plugin/` folder contains a portable Agent Plugins manifest, icon, workflow skill, and remote `mcp.json` connection. `.agents/plugins/marketplace.json` makes it discoverable locally.

In ChatGPT, register the deployed `/mcp` endpoint in **Plugins → +** with OAuth. Complete the connection using the private connection key. For OpenAI's registered-app plugin route, add the returned `plugin_asdk_app…` technical ID to `.app.json` and reference that mapping from `extensions.com.openai.apps` in `plugin.json`.

Local clients can instead load the remote MCP connection from `plugin/mcp.json`. Add the repository marketplace using `codex plugin marketplace add /absolute/path/to/inflow/examples/chatgpt`, then install `inflow@inflow-local`. Authenticate the remote server when prompted. Refresh/restart the app if the installed plugin is not visible. The `open_inbox` tool advertises global/sidebar and thread entrypoints. Pinning is a host UI choice.

Set the URL in `plugin/mcp.json` to your own deployed `/mcp` endpoint before packaging or installing. The checked-in example URL is intentionally nonfunctional. Actual host panel placement and sidebar pinning remain unverified.

## UI and session behavior

The UI reuses original Inflow conversation rows, grouped avatars, message bubbles,
emoji controls, helpers, and theme CSS. The header/layout/composer are adapted
from that source to MCP data rather than Chrome storage. See
`views/inbox/upstream/README.md` for provenance and differences.

There is one View (`inbox`) and one rendering tool (`open_inbox`). Its `state`
selects `inbox`, `thread`, `connection`, or `visualize` (visual inbox alias).
Reuse the returned UUID `widgetSessionId` in later calls in the same chat. It is
also returned as `_meta["openai/widgetSessionId"]`. New panels get different IDs;
the ID is a presentation correlation key, never an authorization credential.
Drafts remain in React memory and survive view switches, but not iframe teardown.

The resource's `_meta["openai/ui"]` declares `availableDisplayModes: ["fullscreen"]`
and `preferredDisplayMode: "fullscreen"`, validated by `@openai/mcp-extensions`.
The tool additionally prefers fullscreen and advertises global/thread entrypoints.
mcp-use 2.7.2 has no generated-resource metadata hook, so `src/openai-ui.ts` adds
only those OpenAI fields to the resource response (JSON or streaming SSE).
The portable React runtime requires inline in its capability list; it remains a
fallback and requests fullscreen once when the host supports it.

mcp-use's initial tool-context hook latches the first result. The small notification
observer in `use-inbox-updates.ts` accepts later state/results only from the parent
host and the same widget session. It never creates a competing MCP transport.

## What is implemented

- Focused, Other, Archived, and Spam folders; search and pagination.
- Thread reading, reply composition, draft retention while navigating, starring, archiving, and read/unread actions.
- Sender grouping, date separators, reactions, reply previews, and attachment labels. Use original Inflow for attachment uploads/downloads.
- All upstream MCP tool descriptors and handlers, forwarded to the extension's gated executor. No write is exercised during deployment verification.
- Offline/setup, loading, empty, failure, and sending states; responsive and dark presentation.
- Separate random credentials for OAuth signing, owner login, and the bridge. Secrets are excluded from Git and Docker.

## Verify

`npm test` checks OAuth PKCE, redirects, resource/audience binding, code replay, missing configuration, relay isolation, and single delivery. `npm run typecheck` and `npm run build` check the actual v2 API and View bundle. `scripts/verify-mcp.mjs` checks one rendering tool/resource, fullscreen metadata,
repeated session reuse, and input validation over the legacy HTTP protocol;
`scripts/verify-modern.mjs` checks current wire-protocol resource metadata directly.
The optional @mcp-use/client 2.4.0 subscription connection stalled subsequent
cloud resource reads in this Node runtime, while direct modern requests and
the installed plugin connection succeeded; that client path is not certified.
`scripts/preview-host.mjs` is a localhost-only synthetic host for interacting with
the actual UI bundle and testing repeated tool input/result notifications.
`scripts/capture-preview.mjs` captures the actual bound View with synthetic data.
Run these scripts with `node --env-file=.env scripts/<script>.mjs`. They read credentials from environment variables; none enters the browser.
Live LinkedIn sync, actual host placement, and pinning need the paired browser and
host UI and are separate from these fixture tests.

## Upstream and license

The vendored UI components, bridge protocol and tool catalog are copied from [grinich/inflow](https://github.com/grinich/inflow) under MIT; its notice is retained in `vendor/INFLOW-LICENSE`. Their upstream revision is recorded in `vendor/UPSTREAM.json`. This integration is not affiliated with LinkedIn.

## Dependency compatibility

OpenAI MCP Extensions 0.1.0 has an optional peer on MCP Apps 1.7.5. That peer is explicitly installed alongside mcp-use’s own nested MCP Apps 2.0.0 runtime. No peer checks are disabled. This project uses the OpenAI package for validated extension metadata and CSS; mcp-use owns the React bridge and MCP lifecycle.
