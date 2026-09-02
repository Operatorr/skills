# Reporting

The report is the product. It is read by two audiences: an owner/manager who needs to know *how bad and what to do*, and an engineer who needs enough detail to reproduce and fix. Serve both — executive summary up top, per-finding detail below, ranked worst-first.

## Deliverables — always produce both formats

Every engagement ships **two** files in the workspace, same content, ranked worst-first:

1. **`REPORT.md`** — the canonical Markdown report (source of truth), following the structure and finding template below.
2. **`REPORT.html`** — a rendered, self-contained HTML version built from [`report-template.html`](report-template.html) (in this folder). It carries the design system — severity color-coding, count tiles, a findings-at-a-glance table, per-finding cards with CVSS/CWE/OWASP metadata, a grouped remediation checklist, light/dark themes, and a print stylesheet — so the owner can read and share it without a Markdown viewer.

**Build order:** write `REPORT.md` first (it forces the analysis), then produce `REPORT.html` by copying the template and filling it from the Markdown — do not hand-roll a different design per engagement, only swap content. Keep the two in sync; if a finding changes, update both. Deliver the HTML to the operator (e.g. via SendUserFile) and, when useful, publish it as an Artifact for a shareable private link.

### Filling `report-template.html`
- Replace every `{{PLACEHOLDER}}`. Set the four/five severity count tiles and the glance table to the real finding set.
- Duplicate the `<article class="finding …">` block once per finding, ranked worst-first, with `id="f1"`, `id="f2"`, … matching the glance-table anchors. Severity classes: `crit | high | med | low | info` — keep the card class and its header chip the same level.
- Reference redacted evidence files from `raw/` in the Evidence blocks. **Never paste raw secrets or PII into the HTML** — the same discipline as the Markdown; flag any sensitive evidence file in the Appendix.
- Keep the `<title>` a short product-style name (`"<App> Security Assessment"`), not a sentence.

## Structure

```
1. Executive summary      — scope tested, dates, headline: N findings by severity,
                            the 1–3 things to fix now, overall posture in plain words.
2. Scope & method         — targets tested, what was and wasn't covered, RoE reference.
3. Findings               — one entry per confirmed finding, ranked by severity.
4. Remediation checklist  — every fix as a checkable action, ordered by severity.
5. Appendix               — engagement log, raw tool output references, retest notes.
```

## Finding template

```
### <FINDING-ID> — <short title>

Severity:     Critical | High | Medium | Low | Info   (CVSS <score> — <vector>)
CWE:          CWE-<n> <name>
OWASP:        A0X:2021 <category>   (or API/other framework as relevant)
Affected:     <asset(s) / endpoint(s)>
Status:       Confirmed

Summary
  One or two sentences: what the flaw is.

Evidence
  The proof from validation — request/response, command + output, screenshot,
  timestamp. Reference saved files rather than pasting everything.

Impact
  What an attacker can actually do, and the blast radius: what data/function is
  exposed, what else it reaches, and the preconditions to exploit it.

Remediation
  Specific, actionable fix — the config change, patch version, code pattern, or
  control to add. Name versions and settings, not "harden the server".

Retest
  How to confirm it's fixed (which phase/command to re-run).
```

## Severity

Use CVSS v3.1 for the vector/score, but let real impact and exploitability decide the label — a "medium" CVSS that exposes all customer data on an internet-facing box is reported as the business risk it is, with the reasoning shown.

| Severity | Rule of thumb |
|---|---|
| Critical | Direct path to full compromise or bulk sensitive-data exposure, low effort, internet-facing. |
| High | Serious exposure or strong foothold; some precondition or effort required. |
| Medium | Real weakness, limited impact or meaningful preconditions. |
| Low | Minor exposure / defense-in-depth gap. |
| Info | No direct risk; hardening opportunity or noted observation. |

CVSS calculator reference: use the FIRST.org v3.1 calculator and record the full vector string, not just the number.

## Framework mapping

Tag every finding so the operator can group, track, and communicate it:
- **CWE** — the weakness class (e.g. CWE-89 SQL Injection, CWE-79 XSS, CWE-287 Improper Authentication, CWE-16 Configuration). Precise root cause.
- **OWASP Top 10 (2021)** for web, or **OWASP API Top 10** for APIs — the category, for prioritization and shared language with stakeholders.
- Optionally **NIST CSF** functions (Identify/Protect/Detect/…) when reporting posture to non-technical leadership.

## Remediation checklist

Distill the findings into an ordered, checkable list the operator works top-down:

```
- [ ] [Critical] <fix> — <affected asset> (FINDING-ID)
- [ ] [High]     <fix> — ...
- [ ] [Medium]   ...
```

Group quick wins (patch, header, config toggle) separately from structural fixes (auth redesign, dependency upgrade) so the operator can knock out the cheap high-impact items immediately.

## Retest

After fixes land, re-run the phases that produced each finding and update its status to `Resolved` / `Partially resolved` / `Still present` with fresh evidence. A finding isn't closed until a retest confirms it.
