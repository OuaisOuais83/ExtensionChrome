# Walking Skeleton — Relay (nom provisoire)

**Phase:** 1
**Generated:** 2026-11-27

## Capability Proven End-to-End

Depuis n'importe quelle surface Claude, le porteur ajoute le connecteur, se connecte par OAuth sur une page de consentement qui nomme le compte en clair, et l'outil `ping` lui répond « pong » avec le compte relié — le tout servi par un Worker Cloudflare unique, sur l'URL définitive, avec base et stockage créés en juridiction Union européenne.

## Architectural Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Runtime / hosting | One Cloudflare Worker (`apps/server`, wrangler 4.168.0), flags `nodejs_compat` + `global_fetch_strictly_public`, `compatibility_date` 2026-11-20 | STACK.md: no cold start before `/token` (10 s budget), stateless MCP fits Workers, one vendor for compute + D1 + R2 + KV |
| HTTP routing and server pages | Hono 4.13.23 as the OAuthProvider `defaultHandler`; pages rendered with `hono/jsx` (auto-escaping) | STACK.md standard; JSX escaping protects the consent page from attacker-chosen client names |
| MCP server | `@modelcontextprotocol/server` 2.4.0, one module-level `createMcpHandler(factory)` with the SDK default stateless legacy posture (both 2025 and 2026-07-28 eras served); fresh `McpServer` per request; principal passed through `authInfo` | STACK.md: never serve modern-only; SDK v2 is required later for file transfers |
| Authorization server | `@cloudflare/workers-oauth-provider` 1.4.0, single-Worker `OAuthProvider`: `apiRoute: '/mcp'`, `authorizeEndpoint: '/authorize'`, `tokenEndpoint: '/token'`, `clientRegistrationEndpoint: '/register'`, `clientIdMetadataDocumentEnabled: true`, PKCE S256, refresh rotation, `scopesSupported: ['mcp:tools', 'offline_access']`, `requiredScopes: ['mcp:tools']` | Only library verified to cover CIMD + DCR + loopback + RFC 8414/9728; phase 1 proves it against real Claude (STATE.md blocker) |
| Identity (phase 1 only) | Fixed spike account `SPIKE_ACCOUNT_EMAIL` (the placeholder `porteur@relay.invalid`, not a person's address, unchanged in production) named in plain text on `/authorize`, approval gated by the `SPIKE_ACCESS_CODE` secret (>= 16 chars, constant-time compare, fail-closed); grant `props` are exactly `{ userId, authorizedAt }` (`SpikePropsSchema`, strict at write and read), KV grant `userId` is an opaque hash, and the label shown by `/authorize` and `ping` is resolved at request time by `resolveSpikeAccount(env, userId)` — no e-mail and no other personal datum is stored in KV | Real identity (Better Auth) is phase 2 and plugs into the same `/authorize` route and the same resolve seam (reading the EU D1 users table instead of configuration). KV is global and cannot be restricted to the EU (PLAT-04), so it stays free of personal data. From phase 2 the KV account id belongs to a real person (pseudonymous): phase 2 planning keeps KV free of anything linkable or puts the exception to the owner before the first external user |
| Data layer | Cloudflare D1 `relay-db` created with `--jurisdiction eu`, raw SQL migrations in `apps/server/migrations/` applied by `wrangler d1 migrations apply`; R2 `relay-content` bound as `CONTENT` with `jurisdiction: "eu"`; KV `OAUTH_KV` reserved to OAuth | PLAT-04; D1 jurisdiction cannot be changed after creation (Cloudflare docs). Drizzle ORM is decided in phase 2 with Better Auth (its adapter needs it) and writes into the same migrations folder |
| Logging | One seam `apps/server/src/log.ts`: strict whitelist schema of metadata fields (route template, status, duration, era, JSON-RPC method, tool name, argument byte size, client name/version, declared capability keys); the route enum is built from one `ROUTE_TEMPLATES` array that `routeTemplate()` also returns from, so every logged route is accepted; `console.*` forbidden elsewhere. Logs are not persisted by the service: Workers Logs disabled (`observability.enabled: false`) from the first commit, no Logpush job, no tail consumer; `wrangler tail` is a live stream the service does not store; retention inside the host beyond these features is not assessed and never claimed — this « journaux non persistés » reading of PLAT-04 / ROADMAP criterion 5 is validated by the owner at the plan 07 Task 1 checkpoint | Directory policy 1.E, encryption claim, GDPR minimisation; Workers Logs storage is global without a Customer Metadata Boundary. Upgrade path: Logpush to an EU R2 bucket (Workers Paid) |
| Configuration | Vars `PUBLIC_BASE_URL`, `PRODUCT_NAME`, `SPIKE_ACCOUNT_EMAIL`; secret `SPIKE_ACCESS_CODE`; local overrides in `apps/server/.dev.vars` (gitignored) and in vitest miniflare bindings | The connector URL is the one-way door: code never hardcodes a host, so the final domain is a config change |
| Public trust surface | `/securite` renders the single statement `CONFIDENTIALITY_STATEMENT_FR` from `packages/shared/src/confidentiality.ts`, written in the future tense (« sera chiffré … ») because phase 1 stores no message; every other sentence of the page is either a delivered mechanism (present) or future | PLAT-03: write the claim once, quote it everywhere; phase 3 switches the constant to the present tense when per-message keys and expiry ship |
| Monorepo / tooling | pnpm 10.28.0 workspaces (`apps/*`, `packages/*`), exact pins + committed lockfile, `minimumReleaseAge: 1440`; TypeScript 7.0.2 (fallback 6.0.3); Biome 2.6.9; Node 22.23 type-stripping for repo scripts | STACK.md supply-chain guidance; pnpm 10.28.0 is the version installed in the build container |
| Testing | vitest 5.1.2 + `@cloudflare/vitest-plugin` 1.9.2 (tests run inside workerd with isolated KV/D1/R2); Node smoke script `apps/server/scripts/smoke.ts` performing the real OAuth dance over HTTP (legacy JSON-RPC + official `@modelcontextprotocol/client` 2.4.0 for the modern era), runnable against `wrangler dev` or the deployed host | Inspector 2.17.0 dropped: published hours before planning and ships a postinstall script |
| Deployment target | `wrangler deploy` to a Cloudflare Custom Domain on the single production host chosen in plan 07 (`workers_dev: false`); no provisional host (workers.dev or other) is ever deployed or written into `PUBLIC_BASE_URL` | No redirect between hosts (Authorization header is dropped), IPv4 A record and port 443 provided by Cloudflare, no WAF rule in front of Anthropic's range |
| Directory layout | `apps/server/src/{index.ts, oauth.ts, app.ts, authorize.tsx, log.ts, spike/, mcp/{handler.ts, server.ts, tools/}, pages/}`, `apps/server/{migrations, test, scripts}`, `packages/shared/src`, `scripts/spike`, `fixtures/spike` | ARCHITECTURE.md structure (auth, mcp, pages) at spike size; MCP tools one file each, read and write tools separate |

## Stack Touched in Phase 1

- [ ] Project scaffold (framework, build, lint, test runner) and the log seam — plan 01
- [ ] Routing — `/` (plan 01), `/authorize`, `/mcp`, `/token`, `/register`, `/.well-known/*` (plan 02), `/p/:file`, `/spike/sonde/:id` (plan 04), `/spike/corpus/:size` (plan 05), `/securite` (plan 06)
- [ ] Database — real write and read: `markdown_image_probe` inserts a probe row, the image endpoint updates its counters, `/spike/sonde/:id` reads them (plan 04)
- [ ] UI — the `/authorize` consent form (access code + Autoriser / Refuser) wired to the OAuth API (plan 02)
- [ ] Deployment — local full stack: `pnpm --filter @relay/server run dev` then `bash apps/server/scripts/smoke-local.sh` (plan 02); production host: `wrangler deploy` + `pnpm --filter @relay/server run smoke -- --from-config` (plan 07)

## Run Locally

1. `pnpm install`
2. `cp apps/server/.dev.vars.example apps/server/.dev.vars`
3. `pnpm --filter @relay/server run dev` (Worker on http://localhost:8787)
4. In another shell: `bash apps/server/scripts/smoke-local.sh` (starts its own `wrangler dev` when none is running, prints `SMOKE OK`)
5. Tests: `pnpm -r run test`; types: `pnpm -r run typecheck`; lint: `pnpm exec biome check .`

## Out of Scope (Deferred to Later Slices)

- Real accounts, Google sign-in, e-mail magic codes, Better Auth, Drizzle ORM (phase 2)
- Install page with screenshots, consent remembered, connected-devices page and revocation (phase 2)
- Contacts, briefs, inbox, Vault encryption, credits ledger, e-mail notifications (phases 3-5)
- Untrusted-content envelope, egress confirmation, send caps (phase 4)
- Magic-link invitations (phase 6)
- Chrome extension, web SPA, file pipeline, Stripe, Resend, MCP Apps, directory listing (milestone 2)
- Persisted logs (Logpush to EU R2) — upgrade path only

## Subsequent Slice Plan

Each later phase adds one vertical slice on top of this skeleton without altering its architectural decisions:

- Phase 2: un founder crée son compte sans mot de passe et relie son Claude via la même page `/authorize` (Better Auth remplace l'identité de spike)
- Phase 3: « envoie ce brief à Paul » — contacts mutuels, envoi, page web de message, contenu chiffré au repos
- Phase 4: « lis mes messages » dans Claude avec porte d'acceptation hors modèle et enveloppe non fiable
- Phase 5: statuts, rappel, expiration, blocage
- Phase 6: invitation par lien magique
