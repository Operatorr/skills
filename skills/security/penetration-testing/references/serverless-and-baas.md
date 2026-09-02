# Serverless / edge / managed-backend targets

Use this alongside the main phases when the target is hosted on a **managed platform** rather than a server you can log into — Vercel, Cloudflare Workers/Pages, Netlify, AWS Amplify/Lambda, and BaaS/DBaaS data layers like **Supabase**, **Firebase**, **Neon**, or **PlanetScale**. The phase order and the two rules (in-scope-only, least-impact) are unchanged; what changes is *where the boundary is* and *what the surface looks like*.

## The core reframe: you own the app, not the infrastructure

On a VPS you own the box, so the box is in scope. On a managed platform the provider owns the box, the edge, the runtime, and the database host — you own only your **application code, configuration, data, and keys**. That flips the default:

- **In scope:** your app's routes and serverless functions, your API's authn/authz logic, your client bundle and what it leaks, your platform configuration (headers, deployment protection, CORS, cache rules), your BaaS access rules (RLS policies, bucket ACLs, security rules), and your own keys/secrets.
- **Out of scope — always, without separate written provider authorization:** the provider's infrastructure. No `nmap`/`naabu`/`masscan` against the edge IPs, no `testssl` fuzzing of the shared TLS terminator, no volumetric/stress/DoS testing, no touching `/cdn-cgi/*` (Cloudflare-owned), no probing another tenant. Enumerating provider infra is not just noise here — it typically breaches the platform's Acceptable-Use / security-testing policy.

**Put this in the RoE explicitly.** Managed-platform engagements test the *application and its configuration*; the platform itself is excluded. Most providers permit self-service testing of your own app within their AUP (no infra scanning, no DoS); some ask for notice. Note the provider(s) and their policy in the RoE so the boundary is written down before anything is touched.

Because there is no host to port-scan, **Phase 2 (Map & scan) is re-pointed at the app**: instead of ports/services, you map routes, functions, the client bundle, the API surface, and the BaaS data endpoints.

## Recon & mapping for serverless (Phase 1–2)

The serverless equivalent of a port scan is **harvesting the client bundle and public endpoints** — all benign GETs of public assets:

- **Pull the JS bundle and source maps.** `curl` the page, extract `<script src>`/`/_next/static/**`, fetch them; if `*.js.map` is served, source is disclosed (finding). Grep the bundle for API base URLs, `NEXT_PUBLIC_*` / `VITE_*` / `PUBLIC_*` values, provider URLs (`*.supabase.co`, `*.firebaseio.com`, `*.workers.dev`, `*.vercel.app`), and any string that looks like a secret.
- **Enumerate function/API routes.** Vercel/Next: `/api/*`; Cloudflare Worker routes; Netlify `/.netlify/functions/*`. Use the bundle and any OpenAPI/schema endpoint before brute-forcing.
- **Find preview/staging deployments.** `*.vercel.app`, `*.pages.dev`, branch/preview URLs in the bundle, commit refs, `crt.sh`. Preview deployments frequently run with real data and weaker protection.
- **Check platform config surface.** Security headers, `vercel.json`/`_headers` behavior, CORS (`Access-Control-Allow-Origin` reflection + `-Allow-Credentials`), cache headers, and whether Deployment Protection / Access is enforced on non-prod.

## Platform-specific issue classes & non-destructive validation

### Vercel (Next.js / serverless / edge)
- **Secret leakage to client:** any real secret behind `NEXT_PUBLIC_` is in the bundle. `service_role`/DB URLs/private API keys in client JS = Critical. *Validate:* grep the fetched bundle (read-only).
- **Source maps exposed:** `/_next/static/**/*.js.map` → source disclosure. *Validate:* GET the `.js.map`.
- **Unprotected preview deployments:** preview URL serves the app (often with prod-like data) without auth. *Validate:* GET the preview URL; note if it returns the app vs. a Vercel auth wall. Don't exfiltrate data — one benign proof.
- **Function authz / SSRF / injection:** `/api/*` is just an API — apply the normal OWASP API tests (broken object/function-level auth, SSRF via server-side fetch of a user URL → point at a canary you control, injection). Same non-destructive rules as the main skill.
- **Middleware/edge auth bypass:** path-normalization or matcher gaps that skip `middleware.ts` auth. *Validate:* request a protected path via an equivalent form and observe if it bypasses.

### Cloudflare Workers / Pages
- **Secrets inlined in the Worker bundle:** `wrangler secret` values live server-side (good); anything hardcoded in source and shipped is exposed. *Validate:* read the Worker's served assets/bundle where reachable.
- **Bindings exposed without authz:** endpoints that read/write **KV / R2 / D1 / Durable Objects** with no auth gate → data exposure. *Validate:* one unauthenticated read (a COUNT / single object), never a write.
- **CORS / open proxy:** overly-permissive CORS or a Worker that fetches arbitrary URLs (SSRF/open-proxy). *Validate:* preflight with a foreign `Origin`; SSRF against your own canary only.
- Out of scope: Cloudflare edge, WAF internals, `/cdn-cgi/*`.

### Supabase (Postgres + PostgREST + GoTrue + Storage)
The **anon key is public by design** — it ships in the client. Never report "the anon key is exposed" as the finding; the finding is **what the anon key can reach**. Security rests entirely on **Row-Level Security (RLS)**.
- **RLS missing/misconfigured (the headline risk):** with the anon key (from the bundle), the auto-exposed REST API `GET https://<ref>.supabase.co/rest/v1/<table>?select=*&limit=1` (headers `apikey: <anon>` and `Authorization: Bearer <anon>`) returns rows it shouldn't. *Validate least-impact:* request a **count** (`Prefer: count=exact`, `Range: 0-0`) or a single row on one table; a non-empty/allowed response proves broken access control. Read only — never `POST/PATCH/DELETE` (those write). Also check `/graphql/v1`.
- **`service_role` key in the client:** catastrophic — it bypasses RLS entirely. *Validate:* grep the bundle for `service_role` / a second long JWT; if present, report Critical and stop (do not use it).
- **Public Storage buckets:** `GET /storage/v1/object/public/<bucket>/<path>` reachable, or bucket lists private files. *Validate:* GET one known/guessed public object.
- **Auth (GoTrue) at `/auth/v1/`:** open signup, user enumeration on login/reset, weak settings. *Validate:* benign, no account takeover.

### Neon / PlanetScale / other DBaaS
- Plain managed Postgres/MySQL — **no auto-exposed data API**, so the risk is **credential exposure + app-layer authz**, not RLS.
- **`DATABASE_URL` / connection-string leakage:** in the client bundle, a public repo, an exposed `.env`, or a verbose error. *Validate:* read-only grep/GET; **do not connect** to the DB with found creds (that's using real credentials against real data — out of least-impact scope; report the leak and recommend rotation).
- **Direct DB reachability / TLS:** confirm the app talks to the DB over TLS and the DB isn't meant to be publicly reachable — but the DB **host is provider infra**: do not port-scan it. Assess from config/leaked-creds evidence, not by hammering the endpoint.
- App-layer: the API in front of the DB gets the normal broken-access-control / injection tests.

## What stays exactly the same
- **Phase 0 gate, non-destructive validation, evidence discipline, and the report** (both `REPORT.md` and `REPORT.html`) are unchanged.
- **OWASP Web & API Top 10** are the backbone — serverless just changes *hosting*, not the app-logic bugs.
- **One benign proof, then stop.** A COUNT via the anon key, a single `.js.map`, a reflected CORS header, a leaked key found by grep — capture it and move on. No data dumping, no writes, no using leaked credentials, no pivoting to the provider.
