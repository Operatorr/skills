# Asset inventory — <org name>

Copy this file into the workspace **before** invoking the skill, as `asset-inventory.md` (or `security/asset-inventory.md` / `.security/asset-inventory.md`). One row per host, URL, or CIDR the security team may assess. Hosts not listed here stay out of a run until you add them.

Standing authorization: see `org-authorization.md`.

| Target | Env | Platform | App / owner | Notes |
|---|---|---|---|---|
| <hostname, URL, or CIDR> | prod / staging / dev | vercel / cloudflare / netlify / vps / other | <name> | |
| | | | | |

`Platform` is how the app is hosted, not an invitation to scan the provider. For vercel / cloudflare / netlify / similar, the **application** is in scope; the provider edge and runtime are not.
