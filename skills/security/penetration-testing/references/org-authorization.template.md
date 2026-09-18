# Org authorization — <org name>

Copy this file into the workspace **before** invoking the skill, as `org-authorization.md` (or `security/org-authorization.md` / `.security/org-authorization.md`). Fill every field. The skill treats a dated statement here as standing authorization for inventory hosts — it will not quiz ownership per site.

Authorized by:   <name / role of the person who can grant this>
Team:            <e.g. Security>
Date:            <YYYY-MM-DD>
Expires:         <YYYY-MM-DD, or "until revoked">
Inventory file:  <asset-inventory.md | security/asset-inventory.md | .security/asset-inventory.md>

Authorization:   "I authorize the Team named above to perform security testing
                  of the systems listed in the Inventory file, which are
                  owned/controlled by <org>."

## Default constraints

- Production in scope?:  <yes/no>
- Test window(s):        <e.g. any time / weekdays 18:00–08:00 local>
- Rate limits:           <max scan intensity, any fragile services>
- Data handling:         no real customer data exfiltrated; benign markers only.

## Emergency stop

- Stop condition:        <e.g. any outage, any sign of real-attacker activity>
- Contact:               <name, phone/channel, reachable during the window>
