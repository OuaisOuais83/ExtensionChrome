# API Coverage — Phase 1 (connecteur MCP Claude, serveur OAuth, plateforme Cloudflare)

> Full coverage by default. Opt-outs are explicit, reasoned decisions.
> Surfaces couvertes : SDK serveur MCP v2 (`@modelcontextprotocol/server` 2.4.0) tel que le consomme le client MCP de Claude, serveur d'autorisation `@cloudflare/workers-oauth-provider` 1.4.0, liaisons Cloudflare du Worker.

## MCP server surface (as consumed by Claude)

| capability | decision | reason |
|---|---|---|
| tools/list + tools/call | INTEGRATE | |
| tool annotations (title + read-only, destructive, idempotent, open-world hints) | INTEGRATE | |
| legacy 2025 era (initialize, stateless fallback) | INTEGRATE | |
| modern 2026-07-28 era (server/discover, per-request envelope) | INTEGRATE | |
| structuredContent in tool results | INTEGRATE | |
| resources (resources/list, resources/read, templates) | OPT-OUT | not needed yet — presentation of connector resources in claude.ai unconfirmed (ARCHITECTURE.md, LOW); revisit with the read tools in phase 4 |
| resource subscriptions / subscriptions/listen | OPT-OUT | explicitly out of scope — Claude's client does not support subscriptions (STACK.md, platform facts) |
| prompts | OPT-OUT | not needed yet — no prompt surface in the spike scope |
| sampling | OPT-OUT | explicitly out of scope — unsupported by Claude's client (STACK.md) |
| elicitation (server-initiated form) | OPT-OUT | not needed yet — phase 1 only records whether each surface declares the capability (ping + metadata log); acceptance stays out-of-model on the web (phases 3-4) |
| logging notifications (logging/setLevel, notifications/message) | OPT-OUT | not needed — server keeps a metadata-only log; nothing is streamed to the client |
| progress notifications | OPT-OUT | not needed yet — spike tools return in well under 240 s |
| completions | OPT-OUT | not needed — no argument completion in spike tools |
| tasks (task-augmented requests) | OPT-OUT | not needed yet — no long-running tool in phase 1 |
| file transfers extension (io.modelcontextprotocol/files) | OPT-OUT | explicitly out of scope — directory-listed connectors only (STACK.md), milestone 2 |
| MCP Apps (@modelcontextprotocol/ext-apps) | OPT-OUT | explicitly out of scope — STACK.md marks it "phase ultérieure, demande une recherche dédiée" |
| skills extension (./ext/skills) | OPT-OUT | not needed — no skill distribution in this product |

## OAuth 2.1 authorization server (workers-oauth-provider, single-Worker OAuthProvider)

| capability | decision | reason |
|---|---|---|
| Client ID Metadata Documents (CIMD) | INTEGRATE | |
| Dynamic Client Registration (RFC 7591) fallback | INTEGRATE | |
| PKCE S256 (plain refused) | INTEGRATE | |
| authorization code grant + refresh token rotation | INTEGRATE | |
| RFC 9728 protected resource metadata + 401 Bearer challenge | INTEGRATE | |
| RFC 8414 authorization server metadata | INTEGRATE | |
| RFC 8707 resource indicators / audience binding | INTEGRATE | |
| RFC 8252 loopback redirects (any port, localhost and 127.0.0.1) | INTEGRATE | |
| RFC 7009 token revocation (served by the library at the token endpoint) | INTEGRATE | |
| RFC 9207 iss in authorization responses | INTEGRATE | |
| private_key_jwt for CIMD clients (advertised automatically with CIMD) | INTEGRATE | |
| consent helpers (describeConsent, beginConsent, approveConsent, denyConsent) | INTEGRATE | |
| onError hook (routed to the metadata-only log) | INTEGRATE | |
| remembered consent (isConsentRemembered) | OPT-OUT | not needed yet — consent design belongs to AUTH-03 (phase 2) |
| upstream sign-in (beginUpstream / finishUpstream, Google) | OPT-OUT | explicitly out of scope — real identity is phase 2 (AUTH-01, Better Auth) |
| listUserGrants / revokeGrant | OPT-OUT | not needed yet — connected-devices page is AUTH-05 (phase 2) |
| purgeExpiredData cron | OPT-OUT | not needed yet — KV TTLs cover spike volume; scheduled purge arrives with phase 2 |
| token exchange (RFC 8693) | OPT-OUT | not needed — no downstream API |
| Enterprise-Managed Authorization (ID-JAG) | OPT-OUT | not needed — experimental, no enterprise customer |
| split-role OAuthAuthorizationServer + OAuthResourceServer | OPT-OUT | not needed — STACK.md prescribes the single-Worker OAuthProvider shape with apiRoute /mcp |

## Cloudflare platform bindings

| capability | decision | reason |
|---|---|---|
| Workers runtime (nodejs_compat, global_fetch_strictly_public) | INTEGRATE | |
| D1 database created with EU jurisdiction | INTEGRATE | |
| D1 migrations (wrangler d1 migrations apply) | INTEGRATE | |
| R2 bucket with EU jurisdiction (binding CONTENT) | INTEGRATE | |
| KV namespace OAUTH_KV | INTEGRATE | |
| Worker secrets (SPIKE_ACCESS_CODE) | INTEGRATE | |
| Custom Domains (single production host) | INTEGRATE | |
| wrangler tail (live, non-persisted logs) | INTEGRATE | |
| Workers Logs (persisted, observability.enabled) | OPT-OUT | explicitly out of scope — storage follows the Customer Metadata Boundary and is global without it (Cloudflare docs, read 2026-11-27); persisting would put logs outside the EU (PLAT-04) |
| Logpush to an EU R2 bucket | OPT-OUT | not needed yet — requires Workers Paid; recorded as the upgrade path in SKELETON.md |
| Cron Triggers | OPT-OUT | not needed yet — no purge job in phase 1 |
| Queues, Durable Objects, Workflows, Workers AI | OPT-OUT | not needed — STACK.md defers queues; sessions do not exist in the stateless spec |
| R2 presigned URLs (aws4fetch) | OPT-OUT | not needed yet — no browser uploads before milestone 2 |
