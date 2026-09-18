# Security Skills

- **[penetration-testing](./penetration-testing/SKILL.md)** — authorized security assessments of owned infrastructure (traditional hosts and managed/serverless/BaaS), gated on standing org authorization plus an asset inventory, validated non-destructively, producing an OWASP/CWE/CVSS-mapped remediation report.

  Before invoking, copy `penetration-testing/references/org-authorization.template.md` and `asset-inventory.template.md` into the workspace as `org-authorization.md` and `asset-inventory.md` (or under `security/` / `.security/`), and fill them. Setup detail: top-level [README](../../README.md#security).

  Usage:

  ```text
  /penetration-testing assess checkout.example.com api.example.com
  ```
