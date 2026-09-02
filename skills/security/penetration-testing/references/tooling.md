# Tooling by phase

Standard, open-source assessment tools, organized by the phase that uses them. All are for use **against in-scope targets only**. Prefer the least-intrusive option that answers the question; escalate intensity only within the RoE window and rate limits.

Placeholders: `TARGET` = a host/IP in scope, `URL` = an in-scope base URL, `outdir/` = the engagement evidence folder. Save output to files under `outdir/` and reference it from the report.

> Install what's missing rather than improvising a weaker check. Most of these are one `apt`/`brew`/`go install`/`pipx install` away; the ProjectDiscovery suite (`subfinder`, `dnsx`, `httpx`, `naabu`, `nuclei`) shares a consistent interface and is a good baseline.

## Recon (passive-leaning)

| Tool | Purpose | Example |
|---|---|---|
| `dig` / `host` | Resolve names, confirm you're testing the right IP | `dig +short A app.TARGET` |
| `whois` | Ownership / registrar of a domain or IP block | `whois TARGET` |
| `subfinder` | Passive subdomain enumeration | `subfinder -d TARGET -silent -o outdir/subs.txt` |
| `dnsx` | Resolve a subdomain list, drop dead ones | `dnsx -l outdir/subs.txt -a -resp -o outdir/resolved.txt` |
| `httpx` | Which resolved hosts serve HTTP(S), with title/tech/status | `httpx -l outdir/resolved.txt -title -tech-detect -status-code -o outdir/web.txt` |
| `whatweb` | Technology fingerprint of a web target | `whatweb -a 1 URL` |
| Cert transparency | Find subdomains/hosts via issued certs | `curl -s "https://crt.sh/?q=%25.TARGET&output=json"` |

Goal: a resolved, ownership-confirmed asset inventory. Cross-check every discovered host against the RoE before it moves to active phases.

## Map & scan

| Tool | Purpose | Example |
|---|---|---|
| `nmap` | Port + service + version discovery | `nmap -sV -sC -p- --open -oA outdir/nmap_TARGET TARGET` |
| `nmap` (lighter) | Fast top-ports pass when `-p-` is too heavy | `nmap -sV --top-ports 1000 -oA outdir/nmap_top TARGET` |
| `naabu` | Fast port sweep to feed nmap | `naabu -host TARGET -o outdir/ports.txt` |
| `testssl.sh` | TLS/cipher/cert posture per service | `testssl.sh --quiet URL` |
| `ffuf` / `gobuster` | Content & directory discovery on web | `ffuf -u URL/FUZZ -w wordlist.txt -mc 200,301,302,401,403 -o outdir/ffuf.json` |
| `katana` / crawler | Endpoint + parameter map of an app | `katana -u URL -jc -o outdir/endpoints.txt` |

Respect rate limits: throttle `ffuf`/`nmap` (e.g. `nmap -T2`, `ffuf -rate 50`) on production or fragile targets, and stop if a service degrades.

## Serverless / edge / BaaS (managed platforms)

No host to port-scan — map the app and its data layer instead. See [`serverless-and-baas.md`](serverless-and-baas.md) for the full playbook. All benign GETs of public assets / your own keys; **read-only, never write, never use leaked DB creds**.

| Tool | Purpose | Example |
|---|---|---|
| `curl` + JS pull | Fetch page, then its bundle/source maps | `curl -s URL \| grep -oE '/_next/static/[^" ]+\.js'` then fetch each |
| Source-map check | Source disclosure via served `.js.map` | `curl -sI URL/_next/static/chunks/main.js.map` |
| Secret grep of bundle | Leaked keys / `NEXT_PUBLIC_*` / URLs | `grep -aoE '(service_role\|eyJ[A-Za-z0-9_-]{20,}\|https://[a-z0-9]+\.supabase\.co\|postgres(ql)?://[^" ]+)' bundle.js` |
| `katana` / `hakrawler` | Crawl app routes & JS endpoints | `katana -u URL -jc -o outdir/endpoints.txt` |
| Preview-deploy enum | Unprotected `*.vercel.app` / `*.pages.dev` | `crt.sh` + bundle refs; GET each, note auth wall vs. app |
| CORS check | Reflected origin + credentials | `curl -sI -H 'Origin: https://evil.example' URL/api/... \| grep -i access-control` |
| Supabase RLS probe | Anon key reaches data it shouldn't | `curl -s 'https://<ref>.supabase.co/rest/v1/<table>?select=*&limit=1' -H "apikey: <ANON>" -H "Authorization: Bearer <ANON>" -H 'Range: 0-0' -H 'Prefer: count=exact'` — read a COUNT/one row only |
| Supabase Storage | Public bucket / private file exposure | `curl -sI 'https://<ref>.supabase.co/storage/v1/object/public/<bucket>/<path>'` |
| `nuclei` (http only) | Exposures/misconfigs on your app routes | `nuclei -u URL -tags exposure,misconfig,cors -o outdir/nuclei.txt` |

Out of scope on these platforms: `nmap`/`naabu`/`testssl` against the provider edge, DB host, or `/cdn-cgi/*`; any volumetric/DoS test.

## Vulnerability analysis

| Tool | Purpose | Example |
|---|---|---|
| `nuclei` | Templated checks for known CVEs, misconfigs, exposures | `nuclei -l outdir/web.txt -severity low,medium,high,critical -o outdir/nuclei.txt` |
| `nikto` | Web server misconfig & known-issue scan | `nikto -h URL -o outdir/nikto.txt` |
| OWASP ZAP | Full web-app scanner (passive + active), spider, API scan | `zap.sh -cmd -quickurl URL -quickout outdir/zap.html` |
| Burp Suite | Interactive proxy for manual web testing | (GUI — proxy the browser, map requests) |
| `nmap --script vuln` | Service-level known-vuln scripts | `nmap --script vuln -p <ports> TARGET` |
| Version → CVE | Map discovered versions to known CVEs | check service/version against a CVE source; note CVE IDs |
| Dependency scan | Vulnerable components in code you can read | `npm audit`, `pip-audit`, `osv-scanner -r .`, `trivy fs .` |
| Config review | Read configs/headers directly when available | `curl -sI URL` (headers), review nginx/apache/app config |

Prefer **passive** ZAP and read-only config review first; run **active** scans only inside the window. Record each candidate with affected asset, suspected class (CWE/OWASP), and the observation.

## Validation (least-impact)

Confirm real-vs-noise with the smallest benign proof. Use the safe flags.

| Finding class | Confirm with | Safety notes |
|---|---|---|
| Known-CVE exposure | `nuclei` template match + version banner | The template match *is* the proof; don't run public exploit PoCs against a live box. |
| SQL injection | `sqlmap -u 'URL?param=1' --batch --technique=B --level=1` | Boolean-based detection only; **do not** use `--dump`, `--os-shell`, `--sql-shell`, or destructive tampers. A confirmed boolean/error diff is enough. |
| XSS | Reflect a unique benign marker (e.g. `canary1234`) and observe it execute in a test browser | Use your own marker; don't hook real users or steal sessions. |
| Auth / access control | Request a resource you shouldn't reach with a low-priv or no session; observe it returns | Read one object / a `COUNT`, not the dataset. Use test accounts you created. |
| Misconfiguration / exposure | `curl` the exposed path, capture the response | Read-only. |
| SSRF / open redirect | Point at a benign collaborator/canary you control; observe the callback | Your own listener; never target third-party or internal-sensitive endpoints beyond proving the class. |
| TLS/crypto | `testssl.sh` output | Passive. |

Hard limits during validation: non-destructive, no real data pulled, no persistence, no pivot to other hosts unless the RoE explicitly authorizes chained/assumed-breach testing. Capture request/response + command + output + timestamp at the moment of proof.

## Evidence capture

- `nmap -oA`, `nuclei -o`, `ffuf -o`, `zap ... -quickout` all write files — keep them.
- For manual proofs: save the raw HTTP request and response, and a screenshot where visual.
- Append to the engagement log: `timestamp | target | exact command | output file`.
