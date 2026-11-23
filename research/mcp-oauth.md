# Remote MCP server with OAuth login on Nhost + Hasura

Research for [#8](https://github.com/MLP-Software-Development/stammtisch-app/issues/8), as of 2026-11-23. Sources are
primary docs, specs and code; where something could not be confirmed it says so. Busflow is described from its source
code, without hostnames or secrets.

**Short answer.** Nhost Auth can now be the OAuth authorization server itself (CIMD, PKCE, RS256 tokens carrying
Hasura claims), which is what Busflow runs. The MCP server is a small stateless HTTP service that verifies that token
and calls Hasura **as the member**, so Hasura's permissions stay the only gate. Claude works today; ChatGPT works on
Business/Enterprise plans for write tools.

## 1. What the MCP authorization spec requires

The current spec version is **2026-07-28**; older ones are 2025-11-25, 2025-06-18 and 2025-03-26
([versioning](https://modelcontextprotocol.io/specification/versioning)). 2026-07-28 makes the protocol stateless (no
`initialize` handshake, no session id, a new `server/discover`;
[changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog.md)). Clients still speak 2025-11-25,
so a server should answer both.

For a remote server
([authorization](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/index.md),
[discovery](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/authorization-server-discovery.md),
[security](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations.md)):

- **The MCP server is an OAuth resource server.** It serves Protected Resource Metadata (RFC 9728) naming at least
  one authorization server, and points to it from a `401` with `WWW-Authenticate: Bearer resource_metadata=…` and/or
  at `/.well-known/oauth-protected-resource/<path>`.
- **The authorization server** speaks OAuth 2.1 and publishes RFC 8414 or OIDC discovery metadata. PKCE with S256 is
  mandatory; clients refuse an AS that does not advertise `code_challenge_methods_supported`. New in 2026-07-28: the
  AS should return the RFC 9207 `iss` parameter. Public clients get rotated refresh tokens.
- **Audience binding.** Clients send the RFC 8707 `resource` parameter; the server must accept only tokens meant for
  it and must not pass a client's token on to another service ("token passthrough").
- **Client registration.** Preferred is a **Client ID Metadata Document (CIMD)**: the client id is an HTTPS URL to a
  JSON document. Dynamic Client Registration (RFC 7591) is now **deprecated**, kept for compatibility. Clients try
  pre-registered, then CIMD, then DCR
  ([client registration](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/client-registration.md)).
- **Scopes.** The `401` challenge can name the scope; a valid token lacking one gets `403 insufficient_scope`, and the
  client re-authorises with the union (incremental consent).
- Enterprise-managed authorization and client-credentials are stable extensions
  ([ext-auth](https://github.com/modelcontextprotocol/ext-auth)); neither matters for a small group app.

## 2. What clients support today

**Claude** (web, Desktop, mobile, Claude Code share one auth stack;
[authentication](https://claude.com/docs/connectors/building/authentication.md),
[building](https://claude.com/docs/connectors/building.md)):

- Registration by CIMD, DCR, or a client id/secret the user enters. Claude picks CIMD only when the AS metadata has
  `client_id_metadata_document_supported: true` and `none` among the token endpoint auth methods.
- Web/Desktop/mobile redirect to one fixed claude.ai callback; Claude Code uses a loopback redirect on any port and
  its own CIMD document. The AS must therefore accept loopback redirects regardless of port.
- Sign-in starts only from a `401`; only the first `authorization_servers` entry is used; discovery and token calls
  time out after 10 s; the AS must be reachable from Anthropic's published egress range.
- Supported auth specs listed: 2025-03-26, 2025-06-18, 2025-11-25 (2026-07-28 not yet listed).
- Custom connectors on Free (one), Pro, Max, Team and Enterprise; on Team/Enterprise an owner adds them
  ([remote MCP](https://www.claude.com/docs/connectors/custom/remote-mcp)).

**ChatGPT** (Apps SDK, developer mode;
[auth](https://developers.openai.com/apps-sdk/build/auth),
[help](https://help.openai.com/en/articles/12584461-developer-mode-and-full-mcp-connectors-in-chatgpt)):

- Prefers CIMD (with its own client document); DCR and a predefined client also work. S256 required, `resource` sent.
- Uses a fixed redirect when the AS supports RFC 9207 `iss`, otherwise a per-connector one.
- Full MCP with write tools on Business and Enterprise/Edu; Pro is read/fetch only; web only. Plus not confirmed.

**Others.** VS Code reportedly tries DCR first and has a CIMD document; Cursor uses DCR with a custom-scheme redirect
that some providers reject (forum reports, not official docs). Not verified: MCP Inspector.

Consequence: an AS with **CIMD + PKCE + `iss`**, and loopback redirects on any port, covers Claude (all apps) and
ChatGPT without DCR.

## 3. Nhost and Hasura capabilities

**Nhost Auth is an OAuth2/OIDC provider** since auth 0.46.0 (Feb 2026); later releases added PKCE and localhost
redirects for CIMD clients, client scoping by role, and SSRF hardening for CIMD fetches
([OAuth2 provider docs](https://docs.nhost.io/products/auth/oauth2-provider/),
[auth OpenAPI](https://docs.nhost.io/openapi/auth.json), release tags `auth@0.4x` in
[nhost/nhost](https://github.com/nhost/nhost)).

- Enabled via `[auth.oauth2Provider]` in `nhost.toml` (or the dashboard), with `loginURL`,
  token lifetimes (access 15 min, refresh 30 days by default) and `clientIdMetadataDocument.enabled`.
- Serves RFC 8414 and OIDC discovery, `/oauth2/authorize`, `/token`, `/jwks`, `/revoke`, `/introspect` and a
  `/oauth2/login` endpoint. Advertises S256 only, public and confidential clients, `iss` support and CIMD. **No DCR
  endpoint**, which no longer matters (section 2).
- **You build the login/consent page** (`loginURL`): Nhost redirects there with a request id, the page signs the
  member in with the normal Nhost session and POSTs to `/oauth2/login`, and Nhost redirects back with a code.
- **Requires an RSA signing key** (RS256/384/512); Nhost projects default to HS256, so the project's JWT key changes.
  Whether that logs out existing app sessions was not confirmed.
- Scopes: `openid profile email phone offline_access graphql`. **`graphql` puts the Hasura claims into the access
  token**, so it is directly usable at Hasura. A role-narrowing `graphql:role:<role>` scope is in the release notes,
  not on the docs page.
- **Gap:** the access token's `aud` is the client id, not the `resource`. RFC 8707 audience binding on `aud` is
  therefore not possible; Busflow works around it with a client allowlist (section 4).

Nhost also ships an open-source **Backend MCP server** (Go, deployable on Nhost Run) that serves the metadata,
verifies tokens against JWKS and exposes generic `graphql-query`/`graphql-mutation` tools
([MCP authentication](https://docs.nhost.io/products/ai/mcp/authentication)). It forwards the user's token to
Hasura and exposes raw GraphQL: useful as a reference, too generic as the product surface.

**Hasura** ([multiple JWT secrets](https://hasura.io/docs/2.0/auth/authentication/multiple-jwt-secrets/),
[admin secret access](https://hasura.io/docs/2.0/auth/authentication/admin-secret-access/)): a JWT request may pick
a role from `x-hasura-allowed-roles` with the `x-hasura-role` header; other `x-hasura-*` headers are ignored unless
the admin secret is sent. With the admin secret plus `x-hasura-role`/`x-hasura-user-id`, a request runs under that
role's permissions. Nhost exposes extra JWT secrets via `[[hasura.jwtSecrets]]`, but an external signer disables Nhost
Auth.

**Nhost Functions** ([limits](https://docs.nhost.io/products/functions/limits)): Node 22/24, request timeout 10 s on
Starter, 180 s on Pro, 600 s on Teams, 6 MB payloads. Streaming and cold starts are not documented.

## 4. How Busflow does it

Busflow self-hosts Nhost Auth and Hasura next to a NestJS API. Its MCP lives in the API's `agents` module
(`mcp.controller.ts`, `mcp-server.ts`, `auth/`, `session/`, `tools/`), with a small Go token service ("mint")
beside it.

1. **Authorization server = Nhost Auth's OAuth2 provider** with CIMD on and RS256 keys. The edge exposes only the
   discovery, authorize, token, JWKS and login routes of auth.
2. **Consent page in the existing web app** (`agent-consent` feature, `oauthLogin.ts`): the user is signed in with
   the normal session, picks which tenant the connection works in, and the page completes `/oauth2/login`. The choice
   is stored as a row in an `agent_connection` table (user, client, tenant); the same page lists and removes
   connections.
3. **MCP endpoint** `POST /mcp`, Streamable HTTP **without sessions**, JSON responses, one server instance per request;
   `GET`/`DELETE` answer 405.
4. **Protected Resource Metadata** (`protected-resource.controller.ts`) names Nhost Auth as the only AS and `graphql`
   as the only scope. The guard (`mcp-token.guard.ts`) answers `401` with the `resource_metadata` challenge, and
   `403 insufficient_scope` for a token without `graphql`.
5. **Token check** (`mcp-token.ts`): RS256 against auth's JWKS, issuer match, `sub` and `exp` required, `graphql` in
   `scope`, and **`aud` must be one of the allowlisted client ids** (`claude-clients.ts`, the Claude app's and
   Claude Code's CIMD URLs). The allowlist stands in for the missing resource audience.
6. **No passthrough.** The OAuth token is exchanged at the mint's internal-only listener for a short-lived
   (≤ 15 min, never longer than the source session) Hasura token scoped to the connected tenant, with a claim marking
   the actor channel as MCP for the audit trail. Busflow needs this because its Nhost tokens carry identity only and
   tenant claims are derived per request.
7. **Tools call Hasura as the user, never with the admin secret** (`hasura-as-user.ts`): each request names its
   Hasura role and may only send documents from an allow-list (`request-as.ts`). Write tools first check that the
   caller is a tenant administrator, then run; a permission denial comes back as a plain sentence the model explains
   (`tool-answer.ts`, `tenant-tool.ts`).
8. **Guard rails:** an in-memory rate limit per user (calls and writes per minute, via `readOnlyHint`); tool
   arguments and answers stay out of logs; tools return a short sentence plus structured facts. A later concept adds
   two-step writes (preview, then commit after the user's yes in the chat).

Only the two Claude clients are allowlisted; ChatGPT would need its CIMD URL and redirect added.

**What carries over to Stammtisch:** items 1–5, 7 and 8 almost as they are. Item 6 (the mint) mostly does not:
Stammtisch's Hasura permissions can be written against group membership (`exists` over the member table with
`X-Hasura-User-Id`), so a token with only the user id is enough, and Nhost Cloud offers no place for a custom token
service beside Auth anyway.

## 5. Mapping a tool call onto the member's permissions

- Hasura roles stay coarse (`user`); **member vs. admin is group data**, enforced by row permissions that join the
  member table (`is admin of this group`). The MCP never holds a role the app does not.
- Each tool is a fixed GraphQL document from an allow-list (Busflow's `request-as.ts` pattern), sent with the
  member's credential and `x-hasura-role: user`. No generic "run any GraphQL" tool.
- Two ways to send "as the member", the main open choice:
  - **(A) Forward the verified Nhost OAuth token to Hasura.** Simple, no secret in the MCP service; Hasura validates it
    with the project's JWKS. Formally token passthrough, acceptable only by treating MCP and Hasura as one resource.
    Audit "via MCP" is not visible to Hasura, since the token has no channel claim.
  - **(B) Admin secret plus `x-hasura-role: user` and `x-hasura-user-id: <sub>`.** Same permissions, no passthrough,
    and the service can add an `x-hasura-actor-channel: mcp` session variable for audit. But the MCP service holds the
    admin secret, so a bug that skips the role header runs as admin.
- Writes that move money (cash ledger) or many rows (bulk events) use preview-then-commit: the first call returns
  what would change and a short-lived confirmation id; only the second call writes.
- Refusals are tool results (`isError: true`) in plain words the model can relay, not protocol errors.

## 6. Hosting options

| Option | Limits that matter | Fit |
|:--|:--|:--|
| Nhost Functions | 10 s (Starter) / 180 s (Pro) / 600 s (Teams) per request, 6 MB, streaming undocumented | Fine for stateless JSON tool calls on Pro; 10 s is tight on Starter for bulk writes |
| Nhost Run (container) | a small always-on service next to the project; documented host of Nhost's Backend MCP | Closest to Busflow; no cold start, no per-request timeout documented |
| Cloudflare Workers | CPU 30 s default (up to 5 min) on Paid, I/O wait not counted, streaming fine ([limits](https://developers.cloudflare.com/workers/platform/limits/)); `createMcpHandler` is the stateless path, `McpAgent` deprecated ([transport](https://developers.cloudflare.com/agents/model-context-protocol/transport/)) | Good runtime; its `workers-oauth-provider` would duplicate the AS Nhost now provides |
| Vercel functions | max duration 300 s (Hobby), 800 s (Pro) incl. streaming ([limits](https://vercel.com/docs/functions/limitations)) | Works; one more vendor |

Long tool calls are avoided by design: bulk event creation is one Hasura mutation with many rows, which finishes in
well under a second.

## 7. Recommended architecture

```
Claude / ChatGPT ──(OAuth 2.1, CIMD, PKCE)──▶ Nhost Auth OAuth2 provider (RS256)
      │                                          ▲
      │                                          └─ consent page in the Stammtisch web app
      │                                             (signs in with the normal Nhost session, POSTs /oauth2/login,
      │                                              lists and removes connections)
      ▼
MCP service  POST /mcp (Streamable HTTP, stateless, JSON)
  ├─ /.well-known/oauth-protected-resource/mcp  → names Nhost Auth, scope "graphql"
  ├─ verify: RS256 via Nhost JWKS, iss, exp, scope ⊇ graphql, aud ∈ {Claude, Claude Code, ChatGPT CIMD URLs}
  ├─ rate limit per user; read/write by readOnlyHint
  └─ tools: fixed GraphQL documents, x-hasura-role: user, preview-then-commit for money and bulk writes
      ▼
Hasura  (row permissions on group membership decide member vs. admin, exactly as for the app)
```

- Host the MCP service on Nhost Run (or Functions on Pro) in the same project; one TypeScript codebase with the
  official MCP SDK, as Busflow does.
- Enable the OAuth2 provider with CIMD; switch the project to an RSA JWT key before the app has users, which sidesteps
  the session question.
- Start with Claude (all plans have custom connectors); add ChatGPT's client id once the group's members use a
  Business plan or read-only tools suffice.

## 8. Open trade-offs

1. **Token forwarding (A) vs. admin-secret impersonation (B)** for the Hasura call: spec purity and an audit channel
   against holding the admin secret. A Hasura Action or a DB trigger reading the token's client could give (A) an
   audit mark; not researched.
2. **Audience binding:** Nhost puts the client id into `aud`, so the server relies on a client allowlist. Any other
   service accepting Nhost OAuth tokens for the same clients would accept the same token.
3. **Which group a connection acts in:** Busflow pins one tenant per connection on the consent page; Stammtisch members
   belong to several groups, so tools could instead take a `group` argument checked by Hasura. Simpler, but a member
   could not limit a connection to one group.
4. **ChatGPT reach:** write tools need a Business/Enterprise plan; private members on Plus may get nothing.
5. **RS256 switch** on an existing Nhost project and its effect on live sessions: unconfirmed.
6. **Spec drift:** 2026-07-28 (stateless, `server/discover`) is newer than what Claude lists; the server should accept
   2025-11-25 clients until clients move.
7. **Nhost Functions vs. Nhost Run:** cost and cold start against a per-request timeout; streaming on Functions is
   undocumented.
