# Rules of Engagement & authorization

The RoE is the contract that makes this a pentest and not an intrusion. It is written **before** any target is touched, saved to the engagement workspace, and confirmed with the operator. No RoE, no testing.

## RoE template

Fill this out with the operator and save it (e.g. `roe.md`) in the engagement folder.

```
# Rules of Engagement — <engagement name>

Authorized by:   <name / role of person who can grant this>
Date:            <YYYY-MM-DD>
Tester:          <who is running the assessment>
Authorization:   "I authorize security testing of the systems listed under
                  In-Scope below, which are owned/controlled by <org>."

## In-scope targets
- <IP / CIDR / hostname / URL>          <what it is>
- ...

## Explicitly out of scope
- <hosts, subdomains, third-party services, shared infra, anything you do NOT own>
- Any target not listed under In-scope.

## Constraints
- Test window(s):        <when active testing is allowed>
- Production in scope?:  <yes/no — if yes, note extra caution + change window>
- Rate limits:           <max scan intensity, any fragile services>
- Data handling:         no real customer data exfiltrated; benign markers only.

## Emergency stop
- Stop condition:        <e.g. any outage, any sign of real-attacker activity>
- Contact:               <name, phone/channel, reachable during the window>
```

### Scope hygiene
- **Only what the operator owns or controls.** Shared hosting, a CDN, a managed DB, an auth provider, a payment gateway — these are usually someone else's systems even if the operator's app uses them. They go out of scope unless the operator holds written authorization from that provider.
- **A domain is not the host.** Testing `app.example.com` may hit infrastructure the operator does not own (e.g. a third-party SaaS the CNAME points to). Resolve first, confirm ownership of the resolved address, then test.
- **Wildcards need care.** `*.example.com` can include forgotten or third-party-hosted subdomains. Enumerate, then confirm each is in scope before active testing.
- **Managed / serverless / BaaS targets.** When the app runs on Vercel, Cloudflare, Netlify, Supabase, Firebase, Neon, etc., the operator owns the **application, its configuration, data, and keys** — the provider owns the infrastructure. Scope the engagement to the app/data layer and put the platform itself **out of scope**: no port-scanning or fuzzing the provider edge/runtime/DB host, no `/cdn-cgi/*`, no volumetric/DoS tests — that typically breaches the provider's acceptable-use / security-testing policy as well as being someone else's system. Record the provider(s) and note their testing policy in the RoE. Playbook: [`serverless-and-baas.md`](serverless-and-baas.md).

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
