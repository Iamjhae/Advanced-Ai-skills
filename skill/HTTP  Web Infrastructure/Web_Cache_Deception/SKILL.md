# Cache Deception Bug-Hunting Skill

## Skill Identity

**Name:** Cache Deception
**Category:** Web Cache / Information Disclosure
**Primary Goal:** Identify and validate cache deception vulnerabilities where a cache stores a dynamic, user-specific response because the requested URL is incorrectly classified as cacheable/static.

---

# 1. Mission

You are an expert web security researcher specializing in:

* Web Cache Deception
* Web Cache Poisoning
* CDN/cache behavior analysis
* Cache-key analysis
* HTTP request/response behavior
* Dynamic vs static resource classification
* Sensitive-data exposure
* Burp Suite based vulnerability research

Your objective is **real-world vulnerability discovery**, not generic education.

When testing Cache Deception, prioritize:

1. Sensitive authenticated endpoints
2. Cache-rule discovery
3. URL parsing discrepancies
4. Cache-key discrepancies
5. Origin-vs-cache normalization differences
6. Cacheability verification
7. Cross-user data exposure
8. Reliable impact validation

Only treat a result as a confirmed vulnerability when the cache behavior and cross-user exposure are demonstrated.

---

# 2. Core Concept

Cache Deception occurs when:

```text
Attacker-controlled URL
        ↓
Cache interprets URL as static/cacheable
        ↓
Origin interprets URL as dynamic/private endpoint
        ↓
Victim's authenticated response is generated
        ↓
Cache stores the response
        ↓
Attacker retrieves the cached victim response
```

The critical condition is a **parser/normalization disagreement** between the cache layer and the origin application.

Example:

```text
/my-account/anything.css
```

The cache may interpret the request as a static `.css` resource while the origin may route it to the dynamic `/my-account` handler.

---

# 3. Important Distinction

## Web Cache Deception

Goal:

```text
Victim's private response
        ↓
Stored by cache
        ↓
Retrieved by attacker
```

Typical impact:

* Sensitive information disclosure
* Account information exposure
* API key disclosure
* Personal information exposure
* Potential session/account compromise depending on leaked data

---

## Web Cache Poisoning

Goal:

```text
Attacker-controlled response
        ↓
Stored by cache
        ↓
Served to other users
```

Potential impacts:

* Stored XSS-like behavior
* Open redirect
* Malicious content injection
* Response manipulation
* Security-control bypasses

Do not automatically classify every cache issue as Cache Deception.

Determine whether the attacker is:

* **extracting another user's cached response**, or
* **inserting a malicious response into the cache**.

---

# 4. Reconnaissance

Before testing payloads, map the target.

Identify:

### Sensitive authenticated endpoints

Look for:

```text
/my-account
/profile
/account
/settings
/preferences
/dashboard
/admin
/panel
/billing
/invoices
/payment-methods
/orders
/history
/transactions
/messages
/inbox
/notifications
/documents
/files
/downloads
/saved
/favorites
/watchlist
```

Also inspect API endpoints:

```text
/api/user
/api/user/info
/api/account
/api/account/balance
/api/profile
/api/settings
/api/orders
/api/transactions
```

Do not assume these exact paths exist.

Extract actual endpoints from:

* Burp HTTP history
* JavaScript files
* API documentation
* Browser traffic
* Crawling
* Application navigation
* GraphQL requests
* Mobile/API traffic

---

# 5. Sensitive Data Discovery

First authenticate with an account controlled by the researcher.

Search responses for:

```text
username
email
phone
address
user_id
account_id
API keys
tokens
JWTs
session identifiers
billing information
order information
private messages
personal documents
internal identifiers
```

Use Burp search/filtering to identify endpoints returning user-specific information.

The preferred target is a response that is clearly:

```text
User-specific
+
Authenticated
+
Not intended to be public
+
Potentially cacheable
```

---

# 6. Baseline Request

For every candidate endpoint:

1. Send the normal authenticated request.
2. Record the complete request.
3. Record the complete response.
4. Record cache-related headers.
5. Determine whether the normal endpoint is cached.
6. Record response differences between authenticated and unauthenticated requests.

Example baseline:

```http
GET /my-account HTTP/1.1
Host: target.example
Cookie: session=RESEARCHER_SESSION
```

Record:

```text
Status
Content-Type
Cache-Control
Age
ETag
Vary
X-Cache
X-Cache-Hit
CDN-specific headers
Content-Length
Location
```

Do not assume a particular cache header exists.

---

# 7. Cache Detection

Useful indicators include:

```text
X-Cache
X-Cache-Hit
X-Cache-Miss
Age
CF-Cache-Status
Via
ETag
Cache-Control
CDN-specific cache headers
```

Headers are implementation-dependent.

The absence of cache headers does **not** prove that no caching occurs.

Use behavioral testing.

---

# 8. Cache Behavior Verification

A typical sequence:

```text
Request A
    ↓
Observe response

Request A again
    ↓
Compare response

Request A with another session
    ↓
Compare response
```

Look for:

```text
MISS → HIT
Age appears/increases
Same body returned across sessions
Origin no longer contacted
Response remains unchanged after session changes
```

A header such as:

```text
X-Cache: HIT
```

is useful evidence, but the strongest evidence is **cross-user response reuse**.

---

# 9. URL Manipulation Matrix

Test URL parsing differences systematically.

## A. Fake Extension

```text
/my-account/test.css
/my-account/test.js
/my-account/test.jpg
/my-account/test.png
/my-account/test.gif
/my-account/test.svg
```

Also test realistic extensions accepted by the target's cache rules.

---

## B. Semicolon Delimiter

```text
/my-account;foo=.css
/my-account;foo=.js
/my-account;anything
```

Test whether:

```text
Cache interpretation
        ≠
Origin interpretation
```

---

## C. Query-Based Variations

```text
/my-account?foo=.css
/my-account?test=.js
/my-account?cache=.css
```

Do not assume query parameters are included in the cache key.

Determine experimentally whether they are:

```text
keyed
unkeyed
ignored
normalized
```

---

## D. Path Traversal / Normalization

Where permitted by the application's routing behavior:

```text
/resources/../my-account
/resources/..%2fmy-account
/resources/%2e%2e/my-account
```

Also investigate encoded path separators and dot segments.

The important condition is:

```text
Cache sees:
/resources/...

Origin resolves:
/my-account
```

---

## E. Encoded Delimiters

Test relevant URL encodings:

```text
%3b
%3f
%2f
%2e
```

Examples:

```text
/my-account%3b.css
/resources/..%2fmy-account
```

---

## F. Double Encoding

Where the application/proxy performs multiple decoding stages:

```text
%253b
%252f
%252e
```

Example:

```text
/my-account%253b.css
```

Only use encoding combinations that are meaningful for the target's parser chain.

---

# 10. Parser Differential Analysis

This is one of the most important parts of the skill.

For every interesting payload determine:

```text
WHAT DOES THE CACHE THINK THE PATH IS?
WHAT DOES THE ORIGIN THINK THE PATH IS?
```

Build a mental/request-level model:

```text
Client
 ↓
CDN / Reverse Proxy
 ↓
WAF
 ↓
Load Balancer
 ↓
Web Server
 ↓
Framework Router
 ↓
Application
```

Different layers may normalize:

* `/`
* `\`
* `%2f`
* `%2e`
* `..`
* `;`
* `?`
* duplicate slashes
* encoded delimiters

A vulnerability becomes interesting when normalization differs between layers.

---

# 11. Cache-Key Analysis

Determine what components participate in the cache key.

Potential components:

```text
Scheme
Host
Port
Path
Query string
Selected query parameters
Headers
Cookies
HTTP method
Content negotiation
```

A simplified cache key may look like:

```text
Host + Path
```

while the application response may depend on:

```text
Host + Path + Cookie
```

This mismatch can create cross-user exposure.

---

# 12. Unkeyed Parameters

Investigate parameters that influence the response but may not influence the cache key.

Examples:

```text
?timestamp=
?t=
?cb=
?debug=
?lang=
?format=
```

Do not assume these are unkeyed.

Test:

```text
Request A:
GET /endpoint?x=AAA

Request B:
GET /endpoint?x=BBB
```

Compare:

```text
Response body
Cache status
Age
Headers
```

If:

```text
Response changes with x
+
Cache key does not change with x
```

the parameter may be cache-poisoning relevant.

---

# 13. Unkeyed Headers

Investigate headers that influence origin behavior but may not participate in the cache key.

Potential examples:

```text
X-Forwarded-Host
X-Forwarded-Proto
X-Forwarded-Port
X-Original-URL
X-Rewrite-URL
Host
```

Do not blindly send large header lists.

First establish:

```text
Header changes origin response
```

then establish:

```text
Header does not change cache key
```

This is the foundation of cache poisoning testing.

---

# 14. Victim Validation Model

A confirmed Cache Deception scenario should demonstrate:

```text
1. Researcher identifies private endpoint
2. Researcher creates cacheable-looking URL
3. Victim requests URL while authenticated
4. Origin returns victim-specific content
5. Cache stores response
6. Researcher requests same URL
7. Researcher receives victim-specific content
```

The strongest proof is:

```text
Victim marker
        ↓
appears in cached response
        ↓
when researcher is unauthenticated / using another session
```

Use harmless test accounts whenever possible.

---

# 15. Two-Account Validation

Prefer two controlled accounts.

### Account A

```text
Unique username:
CACHE_TEST_A

Unique email:
cache-a@example.test
```

### Account B

```text
Unique username:
CACHE_TEST_B

Unique email:
cache-b@example.test
```

Workflow:

```text
A requests candidate URL
        ↓
Cache stores A response
        ↓
B requests same URL
        ↓
If B receives A data
        ↓
Confirmed cross-user cache exposure
```

This is substantially stronger than relying only on cache headers.

---

# 16. Cache Busters

Use unique cache-busting values when testing to avoid contaminating existing cache entries.

Examples:

```text
?cb=unique123
?cache_test=abc123
?x=random-value
```

However, determine whether the parameter is part of the cache key.

A cache buster can accidentally create a new cache entry instead of bypassing the cache.

Use unique values during controlled testing.

---

# 17. 404 / Error Response Testing

Do not ignore error responses.

Test whether nonexistent resources are handled consistently:

```text
/random-nonexistent.css
/anything/random.js
/unknown/path.png
```

Investigate:

```text
404 vs 200
Cache-Control
Age
X-Cache
Response body
```

A cached error response is not automatically a vulnerability.

It becomes security-relevant if:

```text
Sensitive dynamic information
+
incorrect cacheability
+
cross-user exposure
```

is demonstrated.

---

# 18. Framework / CDN Awareness

Pay special attention to applications using:

* Next.js
* React-based applications
* SSR frameworks
* CDN-backed applications
* Reverse proxies
* API gateways
* Cloud-hosted applications
* E-commerce platforms
* Private dashboards

Do not assume that a framework is vulnerable simply because it uses a particular routing mechanism.

The actual cache and origin behavior must be demonstrated.

---

# 19. Cache Deception Test Workflow

Use this workflow for every promising target.

```text
STEP 1
Map authenticated endpoints

STEP 2
Identify sensitive dynamic responses

STEP 3
Capture baseline request/response

STEP 4
Determine normal cache behavior

STEP 5
Identify cacheable path patterns

STEP 6
Generate URL normalization variants

STEP 7
Test fake extensions

STEP 8
Test delimiters

STEP 9
Test encoded delimiters

STEP 10
Test path normalization

STEP 11
Compare cache interpretation vs origin interpretation

STEP 12
Check cache indicators

STEP 13
Repeat request

STEP 14
Test from second session

STEP 15
Test unauthenticated access where appropriate

STEP 16
Confirm victim-specific data leakage

STEP 17
Determine exact impact

STEP 18
Stop testing once reliable proof is obtained
```

---

# 20. Priority Test Order

Use this order to maximize efficiency:

```text
1. /account
2. /profile
3. /settings
4. /dashboard
5. /billing
6. /orders
7. /messages
8. /notifications
9. /documents
10. Sensitive API endpoints
```

Then investigate application-specific endpoints discovered during recon.

---

# 21. Payload Generation Strategy

Do not blindly fuzz thousands of URLs.

Start with:

```text
/endpoint/test.css
/endpoint/test.js
/endpoint/test.jpg
```

Then:

```text
/endpoint;test=.css
```

Then:

```text
/static/../endpoint
/resources/../endpoint
```

Then encoded forms:

```text
/resources/..%2fendpoint
/resources/%2e%2e/endpoint
```

Then encoding combinations:

```text
%3b
%3f
%2f
%2e
%25
```

Prioritize payloads based on observed routing/cache behavior.

---

# 22. Burp Suite Methodology

Use:

### Proxy

Capture authenticated traffic.

### HTTP History

Find:

* sensitive endpoints
* dynamic responses
* cache headers
* repeated requests

### Repeater

Primary tool for manual cache analysis.

### Intruder

Use only after identifying a useful payload dimension.

Potential payload positions:

```text
/path§PAYLOAD§
```

Extensions:

```text
.css
.js
.jpg
.png
.gif
.svg
```

Delimiters:

```text
;
?
```

Encoding variants:

```text
%3b
%3f
%2f
%2e
```

### Param Miner

Use for discovering:

* unkeyed parameters
* unkeyed headers
* cache behavior differences

Do not treat Param Miner findings as automatically exploitable.

Every finding requires manual validation.

---

# 23. Automation Logic

A scanner should follow:

```text
FOR each authenticated sensitive endpoint:

    Capture baseline response

    Generate URL variants

    FOR each variant:

        Send request with researcher session

        Record:
            status
            content-type
            cache headers
            age
            response hash
            response length

        Repeat request

        Compare cache behavior

        IF suspicious cache behavior:

            Test second controlled session

            Test unauthenticated request

            Compare response body

            IF private user data crosses sessions:

                Mark as confirmed
```

---

# 24. Automation Signals

Useful detection signals:

```text
MISS → HIT
Age > 0
Stable cached response
Response body identical across sessions
Authenticated content returned without authentication
Dynamic endpoint classified as static
Different URL representations return same origin resource
```

Strongest signal:

```text
Authenticated User A data
        ↓
cached
        ↓
Unauthenticated / User B request
        ↓
User A data returned
```

---

# 25. False Positive Prevention

Do NOT report solely because:

```text
X-Cache: HIT
```

Do NOT report solely because:

```text
Age: 100
```

Do NOT report solely because:

```text
Dynamic endpoint appears cacheable
```

Do NOT report solely because:

```text
A `.css` suffix returns HTTP 200
```

Require evidence of actual security impact.

Possible legitimate behavior:

* Public cacheable pages
* Intentionally cached API responses
* Public profile information
* Shared application configuration
* Static resources
* Generic 404 responses
* Cache headers generated by intermediary systems

---

# 26. Impact Analysis

Evaluate:

### Confidentiality

Potentially:

```text
Low
Medium
High
Critical
```

depending on leaked data.

Examples of higher-value data:

```text
Authentication tokens
API credentials
Session material
Private financial information
Private documents
Sensitive personal information
Internal secrets
```

### Integrity

Usually:

```text
Low
```

for pure Cache Deception.

Integrity can become significant if cache poisoning is also possible.

### Availability

Usually:

```text
Low
```

unless cache manipulation causes meaningful service disruption.

---

# 27. Account Takeover Potential

If cached information includes:

```text
password reset token
session token
JWT
API credential
OAuth credential
authentication secret
```

investigate whether the leaked material can lead to account compromise.

Do not claim Account Takeover unless the credential/token is actually demonstrated to provide unauthorized access in the authorized testing environment.

---

# 28. Cache Poisoning Workflow

If testing Cache Poisoning:

```text
1. Identify cacheable endpoint
2. Identify input reflected/used by origin
3. Determine whether input affects response
4. Determine whether input is excluded from cache key
5. Inject harmless proof payload
6. Prime cache
7. Request from a separate session
8. Verify attacker-controlled content is served
```

Potential classes:

```text
Reflected XSS
Open Redirect
Header-based content manipulation
Content injection
Origin manipulation
```

Always use harmless proof payloads where possible.

---

# 29. Cache Poisoning vs Cache Deception Decision Tree

```text
Does victim-specific private data become cached?
        |
       YES
        ↓
Cache Deception

Does attacker-controlled content become cached?
        |
       YES
        ↓
Cache Poisoning

Can both happen?
        |
       YES
        ↓
Report/assess both attack paths separately
```

---

# 30. Evidence Collection

For every confirmed issue save:

```text
Original authenticated request
Manipulated URL
Victim-session request
Cache response
Unauthenticated request
Second-account request
Relevant headers
Response body
Timestamp
Cache status
Age
```

Highlight:

```text
Victim-specific identifier
Sensitive information
Cache HIT
Cross-session response
```

Avoid collecting unnecessary personal information.

---

# 31. Reporting Structure

A professional report should contain:

## Title

Example:

```text
Web Cache Deception exposes authenticated account information
```

## Summary

Explain:

```text
A cache/origin parsing discrepancy causes a private authenticated response to be stored under a cacheable URL.
```

## Affected Endpoint

```text
GET /target-endpoint
```

## Attack URL

```text
GET /target-endpoint/test.css
```

## Steps to Reproduce

```text
1. Authenticate as test account A.
2. Request the vulnerable endpoint.
3. Request the cache-deception URL.
4. Observe the response.
5. Request the same URL from account B / unauthenticated context.
6. Observe account A's data.
```

## Evidence

Show:

```text
Cache headers
Response body
Account identifiers
Before/after comparison
```

## Impact

Explain exactly what information crosses the security boundary.

## Remediation

Recommend:

```text
Correct cache rules
Prevent dynamic responses from being cached
Use appropriate Cache-Control directives
Ensure cache/origin URL normalization is consistent
Ensure private responses are not stored in shared caches
Review cache-key configuration
```

---

# 32. Remediation Guidance

Recommended controls:

```http
Cache-Control: private, no-store
```

for responses containing highly sensitive user-specific information where appropriate.

Additional controls:

* Cache only known-static resources.
* Avoid broad extension-based cache rules.
* Normalize URLs consistently across all proxy layers.
* Ensure CDN and origin use compatible path parsing.
* Review cache-key composition.
* Ensure authentication state is respected.
* Avoid caching personalized responses in shared caches.
* Correctly configure `Vary` where applicable.
* Ensure error responses use appropriate status codes.
* Test CDN/proxy behavior separately from origin behavior.

Do not blindly add `Vary: Cookie` as a universal solution; cache design should be reviewed according to the application's architecture.

---

# 33. Quick Checklist

```text
- [ ] Identify sensitive authenticated endpoints
- [ ] Capture baseline responses
- [ ] Search responses for sensitive user data
- [ ] Determine normal cache behavior
- [ ] Check cache/CDN headers
- [ ] Test fake extensions
- [ ] Test .css
- [ ] Test .js
- [ ] Test image extensions
- [ ] Test semicolon delimiter
- [ ] Test query variations
- [ ] Test path normalization
- [ ] Test encoded delimiters
- [ ] Test encoded path traversal
- [ ] Test double encoding where relevant
- [ ] Identify cacheable directories
- [ ] Analyze cache-key behavior
- [ ] Test unkeyed parameters
- [ ] Test relevant unkeyed headers
- [ ] Compare authenticated sessions
- [ ] Test unauthenticated retrieval
- [ ] Confirm cross-user exposure
- [ ] Check for token/credential leakage
- [ ] Assess account-takeover potential
- [ ] Test cache poisoning separately
- [ ] Capture reproducible evidence
- [ ] Stop after reliable proof
- [ ] Prepare impact-focused report
```

---

# 34. Agent Operating Rules

When this Skill is active:

### Rule 1 — Think in layers

Always reason about:

```text
Client
→ CDN
→ Reverse Proxy
→ WAF
→ Origin
→ Framework
→ Application
```

### Rule 2 — Find parser discrepancies

The highest-value question is:

```text
Does the cache interpret this URL differently from the origin?
```

### Rule 3 — Prioritize private data

Do not waste excessive time on public/static content.

Prioritize:

```text
Accounts
Sessions
Tokens
Financial data
Private documents
Messages
Personal data
Administrative data
```

### Rule 4 — Verify across sessions

Whenever possible use:

```text
Account A
Account B
Unauthenticated session
```

### Rule 5 — Never trust cache headers alone

Headers are supporting evidence.

Behavioral cross-user leakage is stronger evidence.

### Rule 6 — Separate Deception from Poisoning

Determine whether the attack is:

```text
Data extraction
```

or:

```text
Attacker-controlled cache manipulation
```

### Rule 7 — Avoid destructive testing

Use:

* Controlled accounts
* Harmless markers
* Unique cache keys
* Minimal requests
* Non-destructive payloads

### Rule 8 — Report only validated vulnerabilities

A suspicious cache response is not enough.

The final conclusion should be based on reproducible evidence.

---

# 35. Final Skill Objective

The skill succeeds when it can transform:

```text
Raw Burp traffic
        ↓
Sensitive endpoint discovery
        ↓
Cache behavior analysis
        ↓
Cache/origin parser differential
        ↓
Cache-key analysis
        ↓
Controlled exploitation
        ↓
Cross-user verification
        ↓
Impact assessment
        ↓
Professional vulnerability report
```

The primary objective is not to generate the largest number of payloads.

The objective is to identify **real cache security boundaries that can be crossed**, prove the impact safely, and produce a reproducible bug-bounty-quality finding.
