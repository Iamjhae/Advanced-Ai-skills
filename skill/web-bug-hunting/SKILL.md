---
name: web-bug-hunting
description: Use when bug hunting, pentesting, or security testing web applications - recon, XSS, SQLi, SSRF, IDOR, auth bypass, API testing. Triggers on "pentest", "bug bounty", "hackerone", "burp", "nuclei", "vulnerability".
---

# Web Bug Hunting / Pentest

## Rules of engagement

- Only test targets the user is authorized to test (their own apps, bug bounty scope, or lab environments like DVWA/Juice Shop/HackTheBox).
- Always ask for and record explicit scope (domains, IPs, rate limits) before running anything.
- Never run destructive payloads (data deletion, DoS, resource exhaustion) unless explicitly requested.

## Methodology

1. **Recon**
   - Subdomain enum: `subfinder -d <domain>`, `amass enum -passive -d <domain>`
   - Probing alive hosts: `httpx -l subs.txt -title -tech-detect -status-code`
   - Tech fingerprint: `whatweb <url>`, check headers, cookies, favicon hashes
   - Wayback/JS recon: `katana`, `waybackurls`, extract endpoints from JS files
2. **Content discovery**
   - `ffuf -w wordlist -u <url>/FUZZ -mc all -fc 404` for dirs/files/params
   - Enumerate APIs: `/api/v1..vN`, OpenAPI specs (`/swagger.json`, `/openapi.json`)
3. **Vulnerability testing** (prioritize by impact)
   - **Auth**: weak passwords, JWT flaws (alg=none, kid injection), session fixation, OAuth misconfig (redirect_uri, state)
   - **IDOR/BOLA**: iterate object IDs across users; compare authorized vs unauthorized responses
   - **Injection**: SQLi (`sqlmap -u <url> --batch --risk=1 --level=1`, manual `' OR 1=1--`), SSTI (`{{7*7}}`), command injection (`; id`), XXE
   - **XSS**: reflected → stored → DOM (check sinks in JS); polyglots for filters
   - **SSRF**: internal IPs, cloud metadata (`169.254.169.254`), DNS rebinding
   - **Access control**: forced browsing, role tampering, mass assignment
4. **Automation**
   - Nuclei templates: `nuclei -l urls.txt -t nuclei-templates/ -severity medium,high,critical`
   - Burp Suite proxy for manual testing; save requests/responses to files for analysis

## Reporting

For every finding produce:
- Title + severity (CVSS)
- Affected URL(s)/parameter(s) with reproducible request (raw HTTP)
- Step-by-step PoC (curl command or payload)
- Impact explanation and remediation advice

## Useful one-liners

```bash
# Alive subdomains with titles
cat subs.txt | httpx -title -tech-detect -cdn

# Quick XSS reflection check
cat params.txt | gf xss | dalfox url --output xss.txt

# Find secrets in JS
katana -u <url> -d 3 | grep '\.js$' | while read u; do curl -s $u | tr ',' '\n' | grep -Ei 'api[_-]?key|token|secret|password'; done
```

Always explain what each tool/payload does before running it, and save findings to a markdown report file.
