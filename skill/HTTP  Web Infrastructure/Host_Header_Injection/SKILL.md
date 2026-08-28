---

name: host-header-injection
description: Detect, validate, and chain HTTP Host-header trust flaws during authorized web application and bug-bounty testing. Use this skill whenever Host, :authority, X-Forwarded-Host, X-Host, Forwarded, virtual-host routing, absolute URL generation, redirects, password-reset links, cache behavior, internal host access, or host-based access controls are encountered. Prioritize exploitable impact over reflection alone.
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Host Header Injection

## 1. Mission

Find cases where attacker-controlled host information crosses a trust boundary and changes security-sensitive behavior.

Treat the Host value as **untrusted input**.

Do not report:

* simple acceptance of an arbitrary Host
* harmless reflection
* a generic redirect with no meaningful security impact
* a response difference caused only by normal virtual-host routing

Prioritize:

1. Password-reset / account-recovery poisoning
2. Cache poisoning
3. Routing-based SSRF / internal routing
4. Authentication or access-control bypass
5. Exposure of unintended virtual hosts
6. Security-sensitive absolute URL generation
7. Host-controlled application logic
8. Host reaching dangerous backend sinks such as SQL or template processing

HTTP/2 and HTTP/3 may represent authority through `:authority`; reason about Host and authority consistently rather than assuming HTTP/1.1 only.

---

## 2. Preconditions

Before active testing, establish:

* Target is authorized/in scope.
* Current destination IP can be kept separate from the supplied Host value.
* Baseline request and response are known.
* Application architecture is understood enough to distinguish:

  * CDN/WAF
  * reverse proxy
  * load balancer
  * application server
  * cache
  * internal services

Prefer Burp Repeater or an equivalent tool that allows the connection destination and HTTP Host value to be manipulated independently.

---

## 3. Core Workflow

```text
Baseline
  ↓
Can Host/authority be modified while reaching the same target?
  ↓
Identify which component consumes it
  ↓
Find security-sensitive sinks
  ↓
Run low-cost differential tests
  ↓
Classify the behavior
  ↓
Pivot to the highest-value sink
  ↓
Validate impact safely
  ↓
Check cache / cross-user / internal effects
  ↓
Collect minimal reproducible evidence
```

Do not spray payloads before identifying a hypothesis.

---

## 4. Attack-Surface Mapping

Search first for functionality that naturally needs the application's canonical hostname.

### Highest-signal endpoints

* `/forgot-password`
* `/password-reset`
* `/reset-password`
* email verification
* invitation flows
* magic links
* login links
* account activation
* OAuth/OIDC callback construction
* redirects
* canonical URL generation
* share links
* PDF/export links
* API-generated absolute URLs
* webhook/callback generation
* image/script/resource URLs
* tenant switching
* admin/internal panels

### Infrastructure signals

Prioritize targets with:

* CDN + origin separation
* reverse proxies
* multiple domains/subdomains
* staging/admin/internal-looking hosts
* wildcard DNS
* Kubernetes ingress
* cloud load balancers
* multi-tenant routing
* unusual redirects
* `X-Forwarded-*` headers
* different behavior across HTTP/1.1 and HTTP/2

---

## 5. Baseline First

Record:

```text
Target IP / destination
Original Host
Original :authority if applicable
Status
Location
Absolute URLs
Cookies
Cache headers
Response body
Redirect chain
Timing
```

Then change **one variable**.

Example:

```text
Original:
Host: target.example

Test:
Host: unique-controlled.example
```

Keep constant:

* method
* path
* query
* body
* authentication
* cookies
* destination IP
* other headers

The purpose is attribution: determine whether the changed behavior came from Host manipulation rather than another variable.

---

## 6. First-Pass Decision Tree

```text
Can arbitrary Host reach the same application?
│
├─ NO → determine whether rejection is consistent.
│       Do not keep fuzzing blindly.
│
└─ YES
   │
   ├─ Host reflected only?
   │    └─ Search for a security-sensitive sink.
   │
   ├─ Host changes Location?
   │    └─ Test whether the redirect becomes attacker-controlled.
   │
   ├─ Host changes generated absolute URLs?
   │    └─ Test reset/invitation/verification flows.
   │
   ├─ Host changes cacheable content?
   │    └─ Investigate cache poisoning.
   │
   ├─ Host changes backend routing?
   │    └─ Investigate virtual hosts/internal routing.
   │
   └─ Host changes authorization/security decisions?
        └─ Test host-based access-control bypass.
```

---

## 7. Priority Testing

### P0 — Low cost / high signal

Test:

1. arbitrary Host
2. Host reflected into `Location`
3. Host reflected into absolute URLs
4. Host affecting canonical URLs
5. Host affecting password-reset/recovery links
6. `X-Forwarded-Host`
7. other supported host-override mechanisms

### P1 — High-value pivots

If Host is consumed:

* reset/account recovery
* email verification
* invitation
* magic links
* OAuth redirects
* cacheable responses
* internal/admin routing
* tenant selection
* authentication decisions

### P2 — Deep infrastructure testing

Only after evidence justifies it:

* duplicate Host handling
* Host vs `:authority` inconsistencies
* proxy/backend disagreement
* connection-state behavior
* virtual-host discovery
* routing-based SSRF
* ambiguous request handling

---

## 8. Host-Override Headers

If direct Host validation blocks the test, determine whether an intermediary rewrites or trusts another header.

High-signal candidates:

```text
X-Forwarded-Host
X-Host
X-Forwarded-Server
X-HTTP-Host-Override
Forwarded
```

Do not assume every header works.

Test one header at a time and compare:

```text
Host only
Host + candidate override
candidate override only
```

If a candidate changes application-generated URLs while the direct Host remains valid, investigate which infrastructure layer is trusting it.

Use header discovery only when there is evidence of proxy/framework behavior; avoid broad blind fuzzing.

---

## 9. Password-Reset Poisoning

This is the highest-priority Host-header sink when recovery links are generated from request-derived host information.

### Hypothesis

```text
Recovery request
      ↓
Server builds absolute reset URL
      ↓
URL hostname comes from request Host
      ↓
Attacker controls hostname
      ↓
Victim receives attacker-hosted reset link
```

### Test

1. Use a controlled test account where possible.
2. Trigger the normal recovery flow.
3. Modify only Host/authority.
4. Inspect the generated email/link.
5. Confirm whether the hostname is attacker-controlled.
6. Determine whether the secret token is exposed to the attacker-controlled origin.
7. Do not reset or take over a third-party account.

### Strong evidence

* valid recovery token generated by the application
* attacker-controlled hostname in the real recovery link
* token delivered to the attacker-controlled destination
* demonstrated ability to use the token on a test account

### Do not overclaim

A malicious hostname appearing in an email is not automatically account takeover.

The impact depends on:

* whether the link is actually delivered
* whether the token reaches the attacker
* whether the token remains valid
* whether additional protections exist
* whether the victim must interact with the link

---

## 10. Absolute-URL Sink Hunting

When Host changes generated URLs, enumerate every feature that can generate an absolute URL.

Prioritize:

```text
reset
verify
invite
magic login
unsubscribe
share
OAuth
webhooks
API callbacks
asset URLs
canonical URLs
redirects
```

For each:

```text
Does Host influence URL?
        ↓
Does the URL leave the server?
        ↓
Does another user/system consume it?
        ↓
Does it contain a secret or privileged action?
        ↓
Potential security impact
```

The last two conditions usually matter more than reflection itself.

---

## 11. Redirect Testing

If:

```text
Host: controlled.example
```

causes:

```text
Location: https://controlled.example/...
```

classify it first as **host-controlled redirect behavior**.

Then ask:

* Is it exploitable without user interaction?
* Is it used in authentication?
* Is it used after login/logout?
* Is it used in recovery?
* Does it bypass an allowlist?
* Can it affect OAuth/OIDC?
* Can it poison a cache?
* Does it expose a token?

Do not report "open redirect" solely because Host changes a normal canonical redirect unless the program accepts that class and meaningful impact is demonstrated.

---

## 12. Cache Poisoning Pivot

If Host changes response content or URLs:

```text
Host mutation
   ↓
Response differs
   ↓
Is response cacheable?
   ↓
YES
   ↓
Does cache key ignore the manipulated input?
   ↓
YES
   ↓
Can a clean request receive the poisoned representation?
   ↓
Potential cache poisoning
```

Compare:

* `Cache-Control`
* `Age`
* `X-Cache`
* CDN cache headers
* `ETag`
* `Vary`
* cache status indicators
* response body
* generated resource URLs

Use a unique harmless marker.

Validate with:

```text
poisoning request
→ cacheable response
→ independent clean request
→ same poisoned representation
```

Do not rely on your own immediate response as proof of poisoning.

---

## 13. Virtual-Host / Routing Pivot

If changing Host changes the backend application:

```text
Host A → application A
Host B → application B
```

determine whether B is intentionally public.

High-signal cases:

* admin interface
* staging application
* internal dashboard
* debug endpoint
* development API
* alternate tenant
* origin-only application
* management interface

Validation requires showing that the alternate service is not merely another documented public hostname.

Avoid brute forcing large hostname lists until the target's architecture justifies it.

---

## 14. Host-Based Access-Control Bypass

Look for security decisions such as:

```text
if Host == internal.example:
    allow_admin()
```

or equivalent proxy/application behavior.

Test:

```text
normal public Host
        vs
trusted-looking alternate Host
```

Then compare authorization.

Strong evidence requires:

* privileged functionality becomes accessible
* access is denied under the legitimate public Host
* the manipulated Host alone changes authorization
* the behavior is reproducible

Do not treat a different homepage or banner as authorization bypass.

---

## 15. Routing-Based SSRF Pivot

Consider routing-based SSRF only when Host/authority affects upstream selection.

Look for:

```text
client
 ↓
reverse proxy
 ↓
Host-derived upstream
 ↓
internal service
```

Signals:

* different backend responses
* internal service banners
* internal-only status codes
* routing to private infrastructure
* host-based upstream configuration

Validate against an authorized controlled endpoint whenever possible.

Do not perform broad internal-network scanning.

---

## 16. Host vs :authority

For HTTP/2/HTTP/3, determine whether the stack treats:

```text
:authority
Host
X-Forwarded-Host
```

consistently.

Test only one authority source at a time.

Interesting result:

```text
proxy validates A
backend consumes B
```

This indicates a parser/trust-boundary mismatch.

Prioritize it when the mismatch changes:

* routing
* cache behavior
* authentication
* generated URLs
* internal access

HTTP semantics explicitly distinguish Host and `:authority`; do not assume HTTP/1.1 behavior maps perfectly to HTTP/2/3.

---

## 17. Duplicate / Ambiguous Host Handling

Use only when the stack appears to parse duplicate headers differently.

Compare:

```text
Host: legitimate.example
Host: controlled.example
```

and equivalent proxy/backend variants where safely supported.

Record:

* first value wins
* last value wins
* proxy uses one value
* backend uses another
* request rejected

A parsing discrepancy is not automatically a vulnerability.

It becomes interesting when the disagreement crosses a security boundary:

```text
proxy validates legitimate
backend consumes attacker-controlled
```

---

## 18. Differential Testing Matrix

Use the smallest useful comparison.

| Comparison                                    | Question                         |
| --------------------------------------------- | -------------------------------- |
| baseline vs modified Host                     | Does Host influence behavior?    |
| Host vs X-Forwarded-Host                      | Which value is trusted?          |
| public Host vs alternate Host                 | Does routing change?             |
| clean vs cacheable response                   | Is manipulated data stored?      |
| recovery request with normal vs modified Host | Is a secret URL poisoned?        |
| anonymous vs authenticated                    | Does Host affect security state? |
| HTTP/1.1 vs HTTP/2                            | Is authority parsed differently? |
| first vs subsequent request                   | Is connection state relevant?    |

---

## 19. Failure Classification

When a test fails, classify the failure before pivoting.

### Rejected Host

Likely proper validation.

Pivot:

* identify proxy-added host headers
* check alternate host override mechanisms
* inspect other in-scope subdomains

### Application unreachable

Likely virtual-host routing rather than injection.

Pivot:

* separate destination IP from Host
* determine expected virtual hosts
* compare known aliases

### Host reflected but no impact

Low signal.

Pivot:

* search absolute URLs
* redirects
* recovery
* cache
* authorization

### Host changes response but not security state

Investigate whether the changed response is cacheable or exposes another virtual host.

### Direct Host blocked, override works

Investigate proxy/application trust mismatch.

### Different backend reached

Stop generic injection testing and switch to routing/virtual-host analysis.

---

## 20. False-Positive Control

Reject or downgrade findings when:

* Host is only reflected in harmless HTML
* application intentionally supports multiple public domains
* redirect is normal canonicalization
* cache is explicitly keyed by Host
* alternate virtual host is documented/public
* modified Host causes an error without security impact
* recovery URL still uses a configured canonical domain
* token never reaches attacker-controlled infrastructure
* behavior exists only in the researcher's local proxy
* the finding depends on unsupported browser/protocol behavior

Always prove the security boundary that changed.

---

## 21. Chaining

Think in chains:

```text
Host trust
  +
absolute URL generation
  ↓
password-reset poisoning
  ↓
token exposure
  ↓
account compromise
```

```text
Host trust
  +
cache ignores Host
  ↓
cache poisoning
  ↓
persistent attacker-controlled content
```

```text
Host trust
  +
proxy/backend disagreement
  ↓
routing manipulation
  ↓
internal service access
```

```text
Host trust
  +
host-based authorization
  ↓
security-control bypass
  ↓
privileged functionality
```

Do not claim the final chain unless every link is demonstrated.

---

## 22. Evidence Requirements

A high-quality finding should contain:

1. baseline request
2. modified request
3. exact changed variable
4. response difference
5. security-sensitive sink
6. reproducible impact
7. affected user/context
8. required attacker capabilities
9. minimal safe PoC
10. explanation of why validation failed

For recovery poisoning, preserve evidence showing:

```text
attacker-controlled Host
→ server-generated recovery message
→ attacker-controlled URL
→ test token exposure / safe validation
```

Redact real credentials and unrelated user data.

---

## 23. Stop / Continue Rules

### Stop

Stop Host fuzzing when:

* strict allowlisting is consistently enforced
* all relevant host override paths are rejected
* no Host-dependent sink exists
* remaining tests would be blind low-signal fuzzing

### Continue

Continue when:

* Host affects generated URLs
* Host affects redirects
* Host affects cacheable content
* Host changes routing
* Host changes authorization
* an override header is trusted
* HTTP/1.1 and HTTP/2 disagree
* multiple proxy layers appear to parse Host differently

---

## 24. Agent Execution Rules

For every test, maintain:

```text
Hypothesis
→ Input changed
→ Expected signal
→ Actual result
→ Classification
→ Next test
```

Use one variable at a time whenever practical.

Prefer targeted hypotheses over payload spraying.

Before repeating a test, ask:

```text
What new information will this request provide?
```

If the answer is "none", do not send it.

Before declaring a finding:

```text
Input controllable?
        ↓
Security-sensitive sink?
        ↓
Trust boundary crossed?
        ↓
Observable security impact?
        ↓
Reproducible?
        ↓
Reportable
```

---

## 25. References

Load `references/advanced.md` only when:

* direct testing reveals proxy/parser complexity
* HTTP/2/HTTP/3 behavior matters
* duplicate/ambiguous Host behavior appears
* routing-based SSRF is suspected
* deeper cache analysis is required
