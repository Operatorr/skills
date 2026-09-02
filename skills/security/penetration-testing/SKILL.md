---
name: penetration-testing
description: >-
  Run an authorized security assessment / penetration test against servers,
  websites, or APIs you own or have written permission to test — including
  serverless/edge apps and managed backends (Vercel, Cloudflare Workers,
  Netlify) and BaaS/DBaaS data layers (Supabase, Firebase, Neon, PlanetScale),
  not only VPS/traditional hosts. Use when the user asks to pentest,
  security-scan, vuln-assess, audit exposure, or harden their own
  infrastructure — or to triage a host they believe is compromised. Gates hard
  on written scope, validates non-destructively, and produces a
  remediation-focused report mapped to OWASP/CWE/CVSS.
---

# Authorized Penetration Testing

You are running a **security assessment of infrastructure the operator owns or is contractually authorized to test.** The deliverable is a prioritized, evidence-backed **remediation report** — not access, not persistence, not data. Every phase below feeds that report.

Two rules govern everything and are never waived:

1. **In scope or it does not happen.** No packet leaves for a host that is not on the confirmed target list in the current Rules of Engagement. When unsure whether something is in scope, stop and ask — do not test to find out.
2. **Least-impact.** Prefer passive over active, read over write, one proof over a hundred. You confirm a vulnerability is *real* with the smallest possible action, capture evidence, and move on. You never destroy data, degrade service, pivot to persistence, or exfiltrate real records.

If a request pushes past authorized assessment — attacking a third party, a target the operator cannot show authorization for, weaponizing a finding for stealth/persistence against a live system, or mass-targeting — decline that piece in one sentence, state why (out of scope / unauthorized), and offer the in-scope alternative.

## Target types — traditional host vs. managed / serverless

Two shapes of target, same phases and same two rules:

- **Traditional host** (VPS, dedicated server, on-prem, a box with an OS you can reach): the host and its services are in scope. Ports/services/versions matter; the main phases below apply as written.
- **Managed / serverless / BaaS** (Vercel, Cloudflare Workers/Pages, Netlify, AWS Amplify/Lambda; data layers Supabase, Firebase, Neon, PlanetScale): you own the **app, config, data, and keys** — the provider owns the infrastructure. Do **not** port-scan or fuzz the provider edge/runtime/DB host (out of scope, and usually against the platform's acceptable-use policy). Re-point the assessment at the app/data layer: the client bundle and what it leaks, function/route authz, preview-deployment exposure, CORS/edge-auth, and BaaS access rules (Supabase **RLS**, Storage ACLs, Neon/`DATABASE_URL` credential leakage). Read [`references/serverless-and-baas.md`](references/serverless-and-baas.md) for the platform playbooks and non-destructive validation, and reflect the app-only boundary in the RoE.

When in doubt which shape you have, resolve the target first (Phase 1): if it lands on a provider's shared infrastructure, treat it as managed and keep the platform itself out of scope.

## Phase 0 — Rules of Engagement (gate)

**Do not scan, probe, or connect to any target until this phase is complete.** Read [`references/rules-of-engagement.md`](references/rules-of-engagement.md) and produce a written RoE with the operator covering: authorized targets (IPs, hostnames, CIDRs, URLs), explicit exclusions, allowed test windows, whether production is in scope, an emergency stop/contact, and a dated statement of authorization from someone who can grant it.

Write the RoE to a file in the engagement workspace and read the target list back to the operator for confirmation.

**Branch — believed-compromised host.** If the target is a machine the operator thinks is *currently* hacked, do not pentest it as step one. An active intrusion means (a) you may collide with the attacker, (b) active scanning can overwrite the forensic trail of the initial entry, and (c) findings are unreliable while someone else is changing the box. Route to **triage first**: preserve evidence (snapshot/image, copy logs off-box), determine whether to isolate or rebuild, and treat the *rebuilt or isolated* system as the pentest target. See the compromise-triage section in [`references/rules-of-engagement.md`](references/rules-of-engagement.md).

**Completion criterion:** a written RoE file exists listing authorized targets and a dated authorization statement, and the operator has confirmed the target list. Only then continue.

## Phase 1 — Recon (passive)

Build a picture of the exposed surface *without touching the target where possible* — DNS, certificates, public metadata, technology fingerprint, exposed subdomains and ports as seen from outside. The goal is a complete **asset inventory**: every host, service, domain, and endpoint in scope, so nothing tested is a surprise and nothing in scope is missed.

Tools and commands: [`references/tooling.md`](references/tooling.md) → Recon.

**Completion criterion:** an asset inventory listing every in-scope host with its resolved addresses, and — for web targets — discovered subdomains and a first-pass technology fingerprint.

## Phase 2 — Map & scan

For each in-scope asset, enumerate open ports, running services, and versions; for web targets, map the content and endpoint surface (directories, parameters, auth points, APIs). This is where the inventory becomes a list of concrete things that can hold vulnerabilities.

Respect the RoE test window and rate limits. Note anything that looks like it would break under load and flag it rather than hammering it. Tools and commands: [`references/tooling.md`](references/tooling.md) → Map & Scan.

**Managed / serverless targets:** there is no host to port-scan — do not scan the provider infra. Re-point this phase at the app: harvest the client JS bundle and source maps, enumerate function/API routes and preview deployments, and map the BaaS data endpoints (Supabase REST/Storage, etc.). See [`references/serverless-and-baas.md`](references/serverless-and-baas.md).

**Completion criterion:** per asset, a service/version table and (for web) an endpoint map, each service tagged with what it is and how exposed it is. For managed targets, an app-surface map instead: routes/functions, client-bundle leaks, preview deployments, and reachable BaaS data endpoints.

## Phase 3 — Vulnerability analysis

Turn the map into candidate findings. Run vulnerability scanners, check service versions against known CVEs, review TLS/crypto posture, inspect configuration and headers, and enumerate the common issue classes (OWASP Top 10 for web: injection, broken auth, access control, misconfiguration, vulnerable components, SSRF, etc.). Read whatever source, configs, or dependency manifests the operator can give you — a config you can read beats a probe you have to guess with.

Every candidate gets recorded with *where it was observed* and *why it is suspected*. Do not confirm yet. Tools and commands: [`references/tooling.md`](references/tooling.md) → Vulnerability Analysis.

**Completion criterion:** a candidate-findings list where every item names the affected asset/endpoint, the suspected issue class (with CWE/OWASP tag), and the observation that raised it.

## Phase 4 — Validation (least-impact)

Confirm which candidates are **real** and which are noise, using the smallest action that proves it. A single benign proof — a read-only marker returned, a version banner, a redirect that fires, a boolean-based response difference — is enough. Capture evidence at the moment of proof: request/response, command + output, screenshot, timestamp.

Non-negotiable during validation:
- **Non-destructive only.** No dropping tables, no deleting/altering records, no filling disks, no shutting services down. Use read-only or clearly-benign markers (e.g. a canary string you inserted into your own test data).
- **No real data.** If a flaw would expose real records, prove access with one innocuous field or a `COUNT`, not by pulling the dataset.
- **No persistence, no pivot.** Confirming a foothold is the finding; installing anything, adding accounts, or moving to the next host is out of scope unless the RoE explicitly authorizes chained/assumed-breach testing.

Tools and commands, including the safe/non-destructive flags for each: [`references/tooling.md`](references/tooling.md) → Validation.

**Completion criterion:** each candidate marked confirmed / not-exploitable / needs-operator-input, and every confirmed finding has captured evidence attached.

## Phase 5 — Impact & blast radius

For each confirmed finding, describe realistically what it lets an attacker do and how far it reaches: what data or function is exposed, whether it enables reaching other in-scope systems, and what has to be true for it to be exploited. This is analysis and description, not further action — you are estimating **blast radius** to drive severity, not detonating it.

**Completion criterion:** every confirmed finding has an impact statement and a preliminary severity.

## Phase 6 — Report

Produce the deliverable: an executive summary a non-security stakeholder can act on, plus per-finding detail. Read [`references/reporting.md`](references/reporting.md) for the finding template and the severity/CWE/OWASP/CVSS mapping.

Each finding carries: title, affected asset(s), severity (with CVSS vector), CWE + OWASP category, the evidence from Phase 4, impact from Phase 5, concrete remediation, and a retest note. Rank the report by severity so the operator fixes the worst thing first.

Close with a **remediation checklist** the operator can work top-down, and offer to re-run the relevant phases as a retest once fixes land.

**Deliver in two formats — always both:**
1. **`REPORT.md`** — the canonical Markdown report (write this first; it forces the analysis).
2. **`REPORT.html`** — a rendered, self-contained version built by copying [`references/report-template.html`](references/report-template.html) into the workspace and filling it from the Markdown. It carries the design system (severity colors, count tiles, findings-at-a-glance, per-finding cards, grouped checklist, light/dark + print). Swap content only — do not redesign per engagement, and never paste raw secrets/PII into it (reference redacted `raw/` files, same as the Markdown).

Hand the HTML to the operator (e.g. SendUserFile) and, when useful, publish it as an Artifact for a shareable private link. Keep the two files in sync.

**Completion criterion:** both `REPORT.md` and `REPORT.html` exist, ranked by severity, where every finding has evidence and specific remediation, and nothing in the candidate list is left in an unresolved state.

## Evidence discipline (all phases)

Keep an append-only engagement log: timestamp, target, tool + exact command, and a pointer to saved output. It is what makes a finding credible, lets you retest, and — if the operator ever needs it — shows exactly what was and was not done. Save raw tool output to files; reference them from the report rather than pasting walls of output.
