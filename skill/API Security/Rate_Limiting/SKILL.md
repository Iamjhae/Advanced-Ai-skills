# Rate Limiting

## 1. Objective

Identify weaknesses in application/API rate-limiting controls that allow an attacker to perform significantly more actions than intended.

Primary goals:

* Detect missing rate limits.
* Detect weak or bypassable rate limits.
* Identify inconsistent enforcement across endpoints.
* Determine the protected security boundary.
* Prove meaningful security or business impact.
* Avoid reporting harmless high request counts without impact.

---

## 2. Core Concept

Rate limiting should restrict sensitive operations based on an appropriate security identity or resource.

Common limiting dimensions:

* IP address
* authenticated user
* account
* API key
* session
* device
* phone number
* email address
* destination/resource
* endpoint/action
* tenant
* combination of multiple identifiers

A vulnerability exists when an attacker can circumvent the intended restriction or perform sensitive actions at an unreasonable rate.

---

## 3. High-Value Targets

Prioritize endpoints involving:

### Authentication

* Login
* Password reset
* OTP generation
* OTP verification
* MFA
* Account recovery
* Email verification
* Phone verification

### Account Actions

* Change email
* Change phone
* Change password
* Invite users
* Create accounts
* Delete accounts
* Generate API keys
* Create sessions

### Financial Actions

* Payment creation
* Coupon redemption
* Gift cards
* Withdrawals
* Transfers
* Refunds
* Wallet operations
* Checkout

### Resource Abuse

* Sending messages
* Sending emails/SMS
* File generation
* Expensive searches
* Export operations
* Report generation
* AI/model requests
* API-heavy operations

---

## 4. Reconnaissance

Before testing, identify:

* Endpoint
* HTTP method
* Authentication state
* User/session identity
* Parameters
* Response behavior
* Existing `429` responses
* `Retry-After`
* Rate-limit headers
* CAPTCHA/challenge behavior
* Lockout behavior
* Account/device identifiers

Useful headers to observe:

```text
Retry-After
X-RateLimit-Limit
X-RateLimit-Remaining
X-RateLimit-Reset
RateLimit-Limit
RateLimit-Remaining
RateLimit-Reset
```

Do not assume the presence of these headers means the endpoint is securely rate-limited.

---

## 5. Establish the Baseline

For each interesting endpoint:

1. Send a normal request.
2. Repeat the same request at a controlled low rate.
3. Observe whether enforcement appears.
4. Record the threshold if one exists.
5. Determine what identifier appears to control the limit.
6. Test whether the limit applies to the actual sensitive operation.

Record:

```text
Endpoint:
Method:
Authenticated:
Action:
Observed threshold:
Response after threshold:
Limit key:
Reset behavior:
Security impact:
```

---

# 6. Rate-Limit Testing Matrix

Test one variable at a time.

| Test                 | Variable                             |
| -------------------- | ------------------------------------ |
| Baseline             | Same request                         |
| IP variation         | Different source IP                  |
| Session variation    | Different session                    |
| Account variation    | Different accounts                   |
| User variation       | Different authenticated users        |
| API key variation    | Different keys                       |
| Header variation     | Client/IP-related headers            |
| Parameter variation  | Same action, different parameters    |
| Endpoint variation   | Alternate route                      |
| HTTP method          | GET/POST/PUT/PATCH/etc.              |
| Encoding             | Equivalent parameter representations |
| Content-Type         | Alternate accepted formats           |
| Path representation  | Equivalent URL representations       |
| Parallel requests    | Concurrent requests                  |
| Distributed requests | Multiple legitimate sources          |
| Retry behavior       | After receiving a limit response     |

Only perform tests that are authorized by the target's scope and rules.

---

# 7. Important Rate-Limit Classes

## 7.1 Missing Rate Limit

Sensitive endpoint accepts repeated operations without meaningful throttling.

Example:

```http
POST /api/auth/send-otp
```

Repeated requests continue succeeding without reasonable enforcement.

Impact may include:

* SMS abuse
* Email abuse
* OTP flooding
* resource exhaustion
* account recovery abuse

---

## 7.2 Per-IP Limiting Only

The server limits requests per IP but fails to protect the underlying account/resource.

Conceptually:

```text
IP A -> Account X
IP B -> Account X
IP C -> Account X
```

If the security boundary is supposed to be the account, IP-only enforcement may be insufficient.

---

## 7.3 Per-Session Limiting Only

A limit is attached to a session rather than the account or operation.

Test whether a new legitimate session resets the restriction.

---

## 7.4 Per-Account Limiting Failure

A sensitive operation is supposed to be limited per account, but the limit can be reset or avoided by changing an unrelated identifier.

Examples:

```text
account_id
user_id
session_id
device_id
API key
```

Determine which identifier actually controls enforcement.

---

## 7.5 Endpoint-Specific Limiting

One route is protected while an equivalent route is not.

Example:

```text
/api/v1/reset-password
/api/v2/reset-password
/api/auth/reset
```

Look for alternate legitimate application flows implementing the same security-sensitive operation.

---

# 8. Parameter-Level Rate Limits

Determine whether the limiter is tied to:

```text
user_id
email
phone
account_id
resource_id
transaction_id
recipient
destination
```

A dangerous design is:

```text
Rate limit = requester's IP
```

when the real security requirement is:

```text
Rate limit = target account / destination / action
```

Prioritize cases where an attacker can repeatedly target the same victim/resource while avoiding the intended restriction.

---

# 9. Distributed Rate-Limit Testing

A common weakness occurs when enforcement is inconsistent across infrastructure.

Conceptually test:

```text
Client A ─┐
Client B ─┼──> Application
Client C ─┤
Client D ─┘
```

Determine whether the backend maintains a shared security counter.

Do not generate uncontrolled traffic.

Use small, controlled experiments and stop once the enforcement model is understood.

---

# 10. Concurrency Testing

Some systems correctly limit sequential requests but fail when requests arrive concurrently.

Compare:

```text
Sequential:
Request 1
Request 2
Request 3
```

against:

```text
Concurrent:
Request 1 ─┐
Request 2 ─┼──> Backend
Request 3 ─┤
Request 4 ─┘
```

Look for:

* multiple requests succeeding beyond the intended threshold
* race conditions
* counters updated after processing
* inconsistent responses
* duplicate sensitive operations

This is particularly important for:

* OTP verification
* coupon redemption
* financial actions
* account recovery
* invitation systems

---

# 11. HTTP-Level Variations

Where authorized, compare semantically equivalent requests using:

* HTTP method variations
* accepted Content-Types
* parameter placement
* URL encoding
* JSON vs form encoding
* equivalent path representations
* trailing slash behavior
* versioned endpoints

The objective is to determine whether the rate limiter observes the same logical operation as the application.

Do not turn this into destructive parser-abuse testing unless that behavior is explicitly in scope.

---

# 12. Proxy/CDN/WAF vs Application Limits

Determine where throttling occurs.

Possible architecture:

```text
Client
  ↓
CDN
  ↓
WAF
  ↓
Load Balancer
  ↓
Application
  ↓
Database
```

A limit may exist at one layer but not another.

Questions:

* Is the limit global?
* Is it per edge node?
* Is it shared across application instances?
* Does the application enforce its own limit?
* Does authentication change the limiting key?
* Does a CDN cache interfere with enforcement?

---

# 13. Response Analysis

Interesting responses include:

```http
HTTP/1.1 429 Too Many Requests
```

But also watch for:

```text
200
201
202
400
401
403
409
```

A rate-limit vulnerability can exist even when no `429` behavior is present.

Measure the actual business action rather than relying solely on status codes.

For example:

```text
Request accepted
↓
Business operation executed
↓
Response returned
```

is more important than whether the response code is `200`.

---

# 14. False Positives

Do NOT report solely because:

* 100 requests succeeded.
* No `429` header exists.
* `Retry-After` is missing.
* A harmless endpoint accepts many requests.
* A public endpoint has no visible limit.
* Rate limits differ between accounts.
* A WAF behaves differently from the application.

Always connect the weakness to:

```text
Missing/weak control
        +
Sensitive operation
        +
Attacker capability
        +
Meaningful impact
```

---

# 15. Impact Classification

### Informational / Low

* Minor throttling weakness
* Non-sensitive endpoint
* Small increase in request volume
* No meaningful abuse demonstrated

### Medium

* Significant abuse of a protected feature
* OTP/email/SMS flooding
* Resource consumption
* Account action automation
* Meaningful business abuse

### High

Potentially:

* Account takeover assistance
* Unlimited OTP verification attempts
* Password-reset abuse
* Financial transaction abuse
* Coupon/gift-card abuse
* Mass account/resource manipulation
* Significant unauthorized resource consumption

Severity depends on the demonstrated impact and program policy.

---

# 16. Authentication and OTP Testing

Prioritize:

```text
Send OTP
Verify OTP
Resend OTP
Password reset
MFA verification
Recovery codes
```

Determine separately:

```text
Generation limit
Verification attempt limit
Resend limit
Per-account limit
Per-destination limit
Per-IP limit
```

A particularly important finding is when verification attempts are effectively unlimited.

Do not attempt large-scale guessing or brute force against real accounts. Use controlled test accounts and minimal attempts sufficient to demonstrate the control failure.

---

# 17. Password Reset Testing

Analyze the complete flow:

```text
Request reset
      ↓
Receive token/OTP
      ↓
Verify
      ↓
Set new password
```

Test rate limits independently at each stage.

A strong limit on `/forgot-password` does not compensate for unlimited attempts against `/verify-reset-code`.

---

# 18. Financial / Business Logic Testing

Rate-limit weaknesses become much more valuable when each successful request produces a business effect.

Look for:

```text
Coupon → discount
Redeem → balance
Withdraw → money
Transfer → funds
Create → resource
Invite → seat
Generate → credit consumption
```

The key question:

> Can the attacker repeatedly trigger a valuable business operation faster than the intended design allows?

---

# 19. API Testing

For APIs, map:

```text
Endpoint
Authentication
User identity
Resource identity
Action
Rate-limit key
Response
```

Compare equivalent API versions:

```text
/api/v1/*
/api/v2/*
/api/internal/*
```

Also inspect:

```text
GraphQL
REST
mobile APIs
web APIs
legacy APIs
```

Only test endpoints exposed to the authorized target.

---

# 20. Multi-Tenant Applications

For SaaS applications determine whether the limiter is scoped to:

```text
user
account
organization
tenant
resource
global system
```

Potential weakness:

```text
Tenant A
  ↓
limited correctly

Tenant B
  ↓
shares/avoids the same counter incorrectly
```

The impact depends on whether one tenant can consume another tenant's resources or bypass tenant-specific controls.

---

# 21. Evidence Collection

Capture:

* Original request
* Repeated request
* Relevant response
* Response status
* Business result
* Timing
* Number of successful operations
* Point where enforcement should have occurred
* Point where enforcement actually occurred

Prefer a minimal reproducible PoC.

Example evidence:

```text
Expected:
After N attempts → operation blocked

Observed:
Attempts N+1 through M → operation still succeeds

Impact:
Sensitive operation can be automated beyond intended threshold.
```

---

# 22. Burp Suite Workflow

Recommended workflow:

```text
Proxy
 ↓
Identify sensitive endpoint
 ↓
Send to Repeater
 ↓
Establish baseline
 ↓
Controlled repetition
 ↓
Compare responses
 ↓
Identify limiting key
 ↓
Test one variable
 ↓
Confirm bypass
 ↓
Document impact
```

For controlled automation, use Burp's tooling only within authorized scope and keep request volume minimal.

---

# 23. Agent Workflow

When operating as a bug-hunting agent:

### Phase 1 — Discover

Find:

* authentication endpoints
* recovery endpoints
* OTP endpoints
* transactional endpoints
* expensive operations
* APIs

### Phase 2 — Model

For each endpoint determine:

```text
Actor
Action
Target
Resource
Expected limit
Observed limit
```

### Phase 3 — Baseline

Perform low-volume controlled requests.

### Phase 4 — Identify Limiting Key

Determine whether enforcement follows:

```text
IP
session
account
user
device
API key
resource
tenant
global counter
```

### Phase 5 — Controlled Bypass Testing

Change exactly one relevant variable at a time.

### Phase 6 — Impact Validation

Determine whether bypass enables:

* authentication abuse
* resource abuse
* financial abuse
* account manipulation
* notification flooding
* meaningful business impact

### Phase 7 — Verification

Reproduce using the smallest number of requests necessary.

### Phase 8 — Report

Produce:

```text
Title
Endpoint
Prerequisites
Steps
Observed behavior
Expected behavior
Impact
Evidence
Severity rationale
Remediation
```

---

# 24. Decision Tree

```text
Sensitive endpoint?
      │
      ├── No → Low priority
      │
      └── Yes
           ↓
     Rate limit exists?
           │
      ┌────┴────┐
      No        Yes
      │          │
      ↓          ↓
 Assess      Identify
 impact      limiting key
                 │
                 ↓
          Can intended limit
             be bypassed?
             │
        ┌────┴────┐
       No        Yes
       │           │
       ↓           ↓
   Probably     Validate
   not bug      business impact
                    │
                    ↓
             Meaningful impact?
                │       │
               No      Yes
                │       │
                ↓       ↓
            Weak/Low   Report
```

---

# 25. Reporting Standard

A strong report should demonstrate:

```text
1. What operation is protected?
2. What rate limit should apply?
3. What identifier should enforce it?
4. What is actually enforced?
5. How is the restriction bypassed?
6. What happens after bypass?
7. Why does this matter to the target?
```

Avoid claims such as:

> "There is no rate limit."

Prefer:

> "The endpoint permits repeated execution of [sensitive operation] beyond the apparent intended threshold, and changing [limiting identifier] resets enforcement while the same target account/resource remains affected."

---

# 26. Remediation

Recommended defenses:

* Rate-limit sensitive operations server-side.
* Choose the correct security boundary.
* Use shared counters across application instances.
* Apply limits to authenticated identities where appropriate.
* Combine identity, resource, and network-level controls when necessary.
* Add progressive delays.
* Apply account lockouts carefully.
* Require CAPTCHA or additional verification where appropriate.
* Protect verification endpoints independently from token-generation endpoints.
* Make counters atomic to prevent race conditions.
* Monitor abnormal request patterns.
* Return appropriate throttling responses.
* Ensure alternate API versions enforce equivalent controls.

Never rely solely on client-side throttling.

---

# 27. Skill Output Requirements

When this skill is used by an autonomous bug-hunting agent, output:

```text
Rate-Limit Status:
[Protected / Weak / Bypassable / Missing / Unknown]

Endpoint:
<endpoint>

Operation:
<operation>

Limiting Key:
<IP / Account / Session / Resource / Tenant / Other>

Observed Behavior:
<behavior>

Bypass:
<minimal reproduction>

Impact:
<security/business impact>

Evidence:
<request/response summary>

Confidence:
[Low / Medium / High]

Recommendation:
<next action>
```

---

# 28. Safety and Scope

Only perform rate-limit testing against systems for which testing is explicitly authorized.

Use:

* test accounts
* controlled request volumes
* minimal concurrency
* reversible actions
* non-destructive resources

Do not conduct uncontrolled flooding, denial-of-service testing, mass OTP/SMS/email generation, credential attacks, or financial abuse merely to demonstrate a rate-limit weakness.

The objective is to prove the vulnerability with the **minimum necessary traffic and impact**.
