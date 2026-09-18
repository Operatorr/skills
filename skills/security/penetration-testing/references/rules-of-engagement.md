# Rules of Engagement & authorization

The RoE is the contract that makes this a pentest and not an intrusion. No RoE, no testing.

Authorization is **standing**: the operator drops `org-authorization.md` and `asset-inventory.md` in the workspace (including before the skill is invoked). This run then writes a short `roe.md` that names the inventory slice being tested. Do not quiz ownership per host, and do not require registrar or platform-dashboard login to start.

Templates to copy: [`org-authorization.template.md`](org-authorization.template.md), [`asset-inventory.template.md`](asset-inventory.template.md).

## Search paths

From the workspace working directory, read the **first file that exists**. Do not treat files inside this skill’s `references/` directory as filled standing auth — those are templates.

**Org authorization**

1. `org-authorization.md`
2. `security/org-authorization.md`
3. `.security/org-authorization.md`

**Asset inventory**

1. `asset-inventory.md`
2. `security/asset-inventory.md`
3. `.security/asset-inventory.md`

A file counts as present when it is readable and filled (authorization has a dated statement and a named grantor; inventory has at least one target row). A still-templated copy (`<org>`, `YYYY-MM-DD`, empty tables) is missing.

## Phase 0 procedure

1. Search the paths above.
2. **Both present.** Intersect the operator’s requested hosts with the inventory (hostname, URL, or CIDR match, including inventory rows covered by an explicit inventory wildcard). Write `roe.md` for this run. Continue to Phase 1. Do not ask whether the operator owns inventory hosts. Do not ask them to log into DNS, Vercel, Cloudflare, Netlify, Wix, or any other dashboard to prove control.
3. **Either missing.** Copy the templates into the workspace as `org-authorization.md` and `asset-inventory.md` (use `security/` instead if that folder already exists). Stop. Tell the operator to fill both files and re-invoke. Do not start probing. Do not replace the files with a multiple-choice ownership questionnaire.
4. **Requested host not on the inventory.** Omit it from `roe.md`. Name the omitted hosts and say they become in-scope when added to the inventory file. Do not probe them.

Dashboard access (DNS, Vercel, Cloudflare, etc.) is optional later, when the operator wants private config or source the public surface does not show. It is not a Phase 0 requirement.

## Run RoE template

Save as `roe.md` in the engagement workspace.

```
# Rules of Engagement — <engagement name>

Standing authorization: <path to org-authorization.md>
Authorized by:          <copied from standing file>
Standing date:          <copied from standing file>
This run date:          <YYYY-MM-DD>
Tester:                 <who is running this assessment>

## In-scope this run
- <inventory hostname / URL / CIDR>    <env>    <platform>    <what it is>
- ...

## Explicitly out of scope
- Any target not listed under In-scope this run.
- Provider infrastructure for managed platforms in this run (edge, runtime hosts, shared TLS, `/cdn-cgi/*`, other tenants).
- <any extra exclusions for this run>

## Constraints
- Test window(s):        <when active testing is allowed; inherit standing defaults unless this run overrides>
- Production in scope?:  <yes/no per standing file and the env column of in-scope rows>
- Rate limits:           <inherit standing defaults unless this run overrides>
- Data handling:         no real customer data exfiltrated; benign markers only.

## Emergency stop
- Stop condition:        <inherit from standing file>
- Contact:               <inherit from standing file>
```

For managed / serverless / BaaS rows, the in-scope line is the **application** (routes, functions, client bundle, config, data, keys). The platform itself stays in Explicitly out of scope. Playbook: [`serverless-and-baas.md`](serverless-and-baas.md).

## Scope hygiene

- **Inventory is the ownership record.** A host is in scope when it is on `asset-inventory.md` (and on this run’s list). Do not re-prove ownership via WHOIS, registrar screenshots, or dashboard sessions.
- **A domain is not the host.** Testing `app.example.com` may hit infrastructure the inventory does not list (a third-party SaaS the CNAME points to). Resolve in Phase 1. If the resolved destination is not on the inventory, it stays out of active testing until the operator adds it.
- **Wildcards need care.** An inventory row `*.example.com` still requires listing or confirming each name before active testing — forgotten or third-party-hosted subdomains are common. Enumerate, then only test names that are in-scope for this run.
- **Managed / serverless / BaaS targets.** The operator owns the application, its configuration, data, and keys; the provider owns the infrastructure. No port-scanning or fuzzing the provider edge/runtime/DB host, no `/cdn-cgi/*`, no volumetric/DoS tests. Record the provider in `roe.md` and keep the platform out of scope.

## Branch — believed-compromised host (triage first)

A host that is *currently* compromised is not a clean pentest target. Handle it as an incident before you handle it as an assessment.

Why not just pentest it:
- **Collision** — you may run into the attacker's activity, or your scans may look like theirs and confuse the picture.
- **Evidence loss** — active scanning, logins, and tooling write to the box and can overwrite logs, timestamps, and artifacts that show *how the attacker got in* — the single most valuable thing to learn.
- **Unreliable results** — while an intruder is changing the system, findings are a moving target.

Triage sequence (preserve before you probe):
1. **Decide isolate vs. keep-live.** Isolating (network-off / firewall to admin only) stops ongoing damage but may tip the attacker and lose live state. Keeping it live preserves volatile state for forensics but continues exposure. This is the operator's risk call — surface the tradeoff, let them decide.
2. **Preserve evidence.** Snapshot the disk / image the volume; copy logs, web/app logs, auth logs, and any suspicious files *off the box* to read-only storage before changing anything. Note times and what was collected in the engagement log.
3. **Look, from the copies.** Triage the preserved data for indicators — unexpected accounts, cron/systemd persistence, modified binaries, outbound connections, web shells, changed timestamps. This is reading collected evidence, not live probing.
4. **Rebuild or clean, then assess.** Treat the compromise as proof the exposed surface has an exploitable hole. Rebuild the host (or fully clean and patch), *then* run the pentest phases against the rebuilt system to confirm the entry path is closed and nothing else is open. Rotate all credentials and keys the host could have touched.

If the operator wants to understand *how* the box was breached, that is incident response / forensics on the preserved evidence — a complementary track to the pentest, run on the copies, not the live host.
