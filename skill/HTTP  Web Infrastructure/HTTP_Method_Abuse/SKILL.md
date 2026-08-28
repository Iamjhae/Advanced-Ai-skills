# HTTP Method Abuse — Bug-Hunting Skill

## 1. Skill Identity

**Skill Name:** `HTTP_Method_Abuse`

**Category:** HTTP / Access Control / Business Logic

**Primary Objective:**
Identify vulnerabilities caused by applications, APIs, reverse proxies, WAFs, or authorization layers handling HTTP methods inconsistently or trusting attacker-controlled method semantics.

The agent should focus on **real exploitable security impact**, not merely discovering that an endpoint accepts unusual HTTP methods.

---

# 2. Core Concept

HTTP Method Abuse occurs when an application exposes functionality through HTTP methods that are:

* undocumented
* insufficiently restricted
* inconsistently authorized
* interpreted differently by different infrastructure layers
* accidentally enabled
* normalized differently by proxies/frameworks
* protected for one method but accessible through another

Typical methods:

```text
GET
POST
PUT
PATCH
DELETE
HEAD
OPTIONS
TRACE
CONNECT
```

Also test method-related variants such as:

```text
GET → POST
POST → PUT
POST → PATCH
POST → DELETE
PUT → PATCH
DELETE → GET
HEAD → GET
OPTIONS → actual method
```

The key question is:

> **Can changing the HTTP method bypass an intended security control or access restriction?**

---

# 3. What Makes It Vulnerable?

Do not report method differences by themselves.

A valid finding normally requires:

```text
Unexpected Method
        ↓
Different Application Behavior
        ↓
Security Control Bypass
        ↓
Unauthorized Action / Data Access
        ↓
Demonstrable Impact
```

Examples:

* `POST /admin/users/delete` requires admin privileges
* `DELETE /admin/users/123` does not enforce the same authorization
* `GET /api/user/delete?id=123` unexpectedly performs deletion
* `PUT` bypasses validation applied to `POST`
* `PATCH` changes fields unavailable through the normal endpoint
* `HEAD` exposes information protected on `GET`
* proxy allows a method that backend authorization does not expect

---

# 4. Primary Hunting Targets

Prioritize endpoints performing state-changing or privileged operations.

## High-Value Targets

```text
/admin/*
/api/admin/*
/users/*
/accounts/*
/account/*
/settings/*
/profile/*
/billing/*
/payment/*
/orders/*
/subscriptions/*
/roles/*
/permissions/*
/members/*
/organization/*
/team/*
/projects/*
/files/*
/documents/*
/api/*
/internal/*
```

Especially interesting:

```text
/delete
/remove
/update
/edit
/change
/approve
/reject
/disable
/enable
/reset
/restore
/invite
/promote
/demote
/export
/import
```

---

# 5. Reconnaissance

Before testing methods, build an endpoint inventory.

Collect:

```text
URL
HTTP Method
Parameters
Authentication State
User Role
Response Code
Response Size
Content-Type
Authorization Requirements
State-Changing Behavior
```

Prioritize endpoints where:

```text
POST /something
PUT /something
PATCH /something
DELETE /something
```

are present.

Also identify endpoints where frontend JavaScript reveals methods not obvious from normal browsing.

Search JavaScript for:

```text
fetch(
axios.
axios.get
axios.post
axios.put
axios.patch
axios.delete
XMLHttpRequest
method:
HTTP_METHOD
```

---

# 6. Baseline Method Mapping

For every interesting endpoint, establish the normal request first.

Example:

```http
POST /api/users/123/role HTTP/1.1
Host: target.example
Authorization: Bearer USER_TOKEN
Content-Type: application/json

{"role":"admin"}
```

Record:

```text
Status
Response body
Response length
Headers
Side effects
Authorization result
```

Then test alternative methods individually.

---

# 7. Method Mutation Matrix

For an endpoint using:

```text
POST
```

test:

```text
GET
PUT
PATCH
DELETE
HEAD
OPTIONS
```

For:

```text
PUT
```

test:

```text
POST
PATCH
DELETE
GET
```

For:

```text
PATCH
```

test:

```text
POST
PUT
DELETE
GET
```

For:

```text
DELETE
```

test:

```text
POST
PUT
PATCH
GET
```

Do not blindly brute-force every endpoint.

Prioritize methods capable of changing state.

---

# 8. Authorization Differential Testing

The most important test is comparing behavior between users.

Create at least:

```text
Session A = authorized / privileged user
Session B = unauthorized / lower-privileged user
```

For sensitive endpoint:

```text
Normal Method + Low Privilege
Alternative Method + Low Privilege
Normal Method + Privilege
Alternative Method + Privilege
```

Compare results.

Example:

```text
POST + normal user → 403
PATCH + normal user → 200
```

This is highly interesting if the PATCH request performs the protected action.

---

# 9. IDOR/BOLA Combination

HTTP Method Abuse frequently becomes more serious when combined with object-level authorization flaws.

Example:

```http
POST /api/users/1001/settings
```

returns:

```text
403 Forbidden
```

Then:

```http
PATCH /api/users/1001/settings
```

returns:

```text
200 OK
```

If the attacker can modify another user's object, classify the underlying impact as an authorization bypass / BOLA-style issue rather than reporting "PATCH is allowed" alone.

Test:

```text
Own Object
Other User Object
Privileged Object
Organization Object
Admin Object
```

---

# 10. Authentication Bypass Testing

Check whether authentication requirements are method-dependent.

Example:

```text
GET /api/admin/data → 401
POST /api/admin/data → 401
PUT /api/admin/data → 200
```

Investigate whether the alternative method genuinely reaches protected functionality.

Also test:

```text
Authenticated request
Unauthenticated request
Expired session
Low-privilege session
Different role
```

---

# 11. Method Override Testing

Applications sometimes support method override mechanisms.

Potential headers:

```http
X-HTTP-Method-Override: PUT
X-HTTP-Method-Override: DELETE
X-Method-Override: PUT
X-HTTP-Method: DELETE
```

Potential parameters:

```text
_method=PUT
_method=DELETE
method=PUT
method=DELETE
```

Test whether:

```text
POST + override
```

is interpreted differently by:

```text
Proxy
WAF
Web Server
Framework
Application
Authorization Middleware
```

The important condition is a **security-relevant interpretation mismatch**.

---

# 12. Content-Type + Method Testing

Changing method may also change parser behavior.

Test combinations such as:

```text
POST + application/json
POST + application/x-www-form-urlencoded

PUT + application/json
PUT + application/x-www-form-urlencoded

PATCH + application/json
PATCH + application/x-www-form-urlencoded
```

Look for:

```text
Authorization bypass
Validation bypass
Mass assignment
Parameter pollution
Unexpected field acceptance
State-changing behavior
```

---

# 13. GET-Based State Changes

One particularly dangerous pattern is state-changing functionality exposed through GET.

Examples:

```http
GET /api/user/delete?id=123
GET /account/disable
GET /admin/promote?id=123
GET /unsubscribe?user=123
```

Determine whether the GET request actually performs the action.

If yes, investigate:

```text
CSRF
Cross-origin triggering
Cache interaction
Crawler triggering
Link-based execution
Authorization
```

A state-changing GET endpoint can become significantly more severe when exploitable cross-origin.

---

# 14. HEAD Testing

`HEAD` should generally behave like `GET` without a response body.

Test sensitive GET endpoints:

```http
HEAD /admin/export
HEAD /api/private/data
HEAD /user/123
```

Look for:

```text
Different authorization
Different status code
Sensitive headers
Content-Length disclosure
Redirect differences
Existence oracle
Cache behavior
```

Do not report HEAD behavior unless there is meaningful security impact.

---

# 15. OPTIONS Testing

`OPTIONS` can reveal:

```text
Allow: GET, POST, PUT, DELETE
```

Use it primarily for reconnaissance.

Example:

```http
OPTIONS /api/resource
```

Potential output:

```http
Allow: GET, POST, PUT, PATCH, DELETE
```

Then verify each method manually.

**Important:**

An overly broad `Allow` header alone is generally not enough for a vulnerability.

---

# 16. TRACE Testing

Test:

```http
TRACE / HTTP/1.1
Host: target.example
```

Look for:

```text
TRACE enabled
Request reflection
Sensitive header reflection
Proxy behavior
```

Do not automatically classify enabled TRACE as high severity.

Require demonstrable security impact.

---

# 17. Reverse Proxy / Backend Mismatch

A high-value scenario is:

```text
Client
  ↓
CDN / WAF / Reverse Proxy
  ↓
Web Server
  ↓
Framework
  ↓
Application
```

Different layers may support different methods.

Example:

```text
WAF validates POST
Backend accepts PUT
Authorization middleware checks only POST
```

Test whether:

```text
Proxy behavior != backend behavior
```

This can produce authorization bypasses or unexpected access to protected functionality.

---

# 18. Method Normalization

Test unusual method representations where appropriate:

```text
lowercase method
uppercase method
mixed case
duplicate method-related headers
```

Example conceptual variations:

```text
get
GET
Get
```

The goal is to identify parser discrepancies between infrastructure layers.

Do not generate excessive traffic.

---

# 19. Method Smuggling / Parser Differential

Investigate situations where different layers interpret the method differently.

Look for:

```text
Proxy method parsing
Backend method parsing
Method override
HTTP/1.1 vs HTTP/2 translation
CDN behavior
WAF behavior
Framework routing
```

A finding becomes interesting when a discrepancy allows:

```text
Security control bypass
Authorization bypass
Protected route access
State-changing operation
Cache/security boundary bypass
```

---

# 20. Route + Method Confusion

Some frameworks map routes differently based on method.

Test:

```text
/api/resource
/api/resource/
/api/resource?id=123
```

with:

```text
GET
POST
PUT
PATCH
DELETE
```

Look for:

```text
Route exists only for one method
Authorization middleware attached only to one route
Fallback route accepting another method
Different controller
Different validation
Different authentication
```

---

# 21. Method + Parameter Pollution

Combine method changes with parameter locations.

Example:

```text
GET ?id=123
POST body id=123
POST query id=123
PUT body id=123
PATCH body id=123
```

Investigate whether different methods cause different parameter precedence.

Potential impact:

```text
Authorization bypass
IDOR
Mass assignment
Validation bypass
Business logic abuse
```

---

# 22. Method + HTTP Status Differential

Record status codes for every mutation.

Example:

| Method | Status | Behavior               |
| ------ | -----: | ---------------------- |
| GET    |    405 | rejected               |
| POST   |    403 | authorization enforced |
| PUT    |    200 | investigate            |
| PATCH  |    405 | rejected               |
| DELETE |    405 | rejected               |

A `200` response is **not automatically a vulnerability**.

Confirm the backend actually performed the sensitive operation.

---

# 23. Response Differential Analysis

Compare:

```text
Status code
Response body
Response length
Headers
Location
Set-Cookie
Content-Type
Content-Length
Timing
Side effects
```

Prioritize differences involving:

```text
200 vs 401
200 vs 403
200 vs 404
302 vs 403
500 vs 403
```

But always verify whether the difference is security relevant.

---

# 24. Burp Suite Workflow

Recommended workflow:

```text
1. Proxy normal request
2. Send to Repeater
3. Establish baseline
4. Change HTTP method
5. Compare response
6. Repeat with second account
7. Verify side effect
8. Test object ownership
9. Test authentication boundary
10. Document reproducible impact
```

Useful Burp features:

```text
Proxy
HTTP history
Repeater
Intruder
Comparer
Logger
```

Use Repeater for precise manual validation.

---

# 25. Browser / Frontend Analysis

Inspect:

```text
Network requests
Fetch calls
XHR
JavaScript bundles
API clients
OpenAPI specifications
Swagger
GraphQL clients
```

Look for explicit method definitions:

```javascript
method: "PUT"
method: "PATCH"
method: "DELETE"
```

and API definitions such as:

```text
DELETE /api/users/{id}
PATCH /api/account
PUT /api/settings
```

These reveal high-value attack surfaces.

---

# 26. API Specification Mining

Search for:

```text
swagger.json
openapi.json
api-docs
swagger-ui
openapi
```

Extract:

```text
paths
methods
parameters
authentication
roles
request bodies
responses
```

Compare documented behavior against actual behavior.

Interesting discrepancy:

```text
OpenAPI:
DELETE /users/{id} → admin only

Reality:
PATCH /users/{id} → accepted by low-privilege user
```

---

# 27. Automation Strategy

Automation should be **differential**, not blind.

Pseudo-workflow:

```text
FOR each interesting endpoint:

    establish baseline

    FOR each alternative HTTP method:
        send request
        compare response

        IF behavior differs:
            test authenticated session
            test low privilege session
            test object ownership

            IF state changes or protected data becomes accessible:
                flag candidate
```

Prioritize:

```text
DELETE
PUT
PATCH
POST
GET
```

Use:

```text
HEAD
OPTIONS
TRACE
```

primarily for discovery and specialized testing.

---

# 28. False Positives

Do NOT report merely because:

```text
OPTIONS is enabled
TRACE returns 200
PUT returns 200
PATCH returns 200
DELETE returns 405
Allow header contains DELETE
Unknown method returns 200
Endpoint accepts multiple methods
```

unless you demonstrate:

```text
Unauthorized access
Unauthorized modification
Unauthorized deletion
Authentication bypass
Authorization bypass
Sensitive information exposure
Business logic abuse
```

---

# 29. Impact Classification

## Informational

Examples:

```text
OPTIONS exposes supported methods
TRACE enabled without demonstrated impact
Extra methods return harmless responses
```

## Low

Examples:

```text
Minor information disclosure
Unexpected method behavior without meaningful privilege impact
```

## Medium

Examples:

```text
Authorization bypass for non-sensitive action
Unauthorized modification of limited data
CSRF-enabling GET state change with meaningful but limited impact
```

## High

Examples:

```text
Privilege escalation
Unauthorized modification of sensitive objects
Unauthorized deletion
Authentication bypass
Cross-tenant modification
Admin functionality accessible through alternate method
```

## Critical

Only when the method abuse enables a severe systemic compromise, such as:

```text
Unauthenticated administrative control
Mass cross-tenant destructive actions
Account takeover
Highly sensitive data exposure
Remote code execution
```

Severity must be based on **actual demonstrated impact**, not the HTTP method itself.

---

# 30. Multi-Tenant Testing

For SaaS applications, explicitly test:

```text
Tenant A user
Tenant B user
Tenant A resource
Tenant B resource
```

Test:

```text
POST
PUT
PATCH
DELETE
GET
```

A powerful finding is:

```text
Normal method:
Tenant B → 403

Alternative method:
Tenant B → 200
```

followed by successful modification of Tenant A's resource.

This should be treated primarily as a **cross-tenant authorization vulnerability**.

---

# 31. Business Logic Testing

HTTP Method Abuse becomes particularly valuable when endpoints control:

```text
Money
Orders
Subscriptions
Roles
Permissions
Invitations
Approvals
Refunds
Account status
Security settings
API keys
Integrations
Files
```

Test whether method changes bypass workflow restrictions.

Example:

```text
POST /refund
```

requires:

```text
approved=true
```

but:

```text
PATCH /refund
```

allows:

```json
{"status":"approved"}
```

The important vulnerability is the **business logic / authorization bypass**.

---

# 32. Evidence Requirements

Before reporting, capture:

```text
Original request
Original response
Modified request
Modified response
Account/role used
Target object
Expected behavior
Actual behavior
Security impact
```

For state-changing vulnerabilities, prove the side effect safely.

Prefer:

```text
test account
test object
non-destructive modification
```

Avoid destructive testing against real user data.

---

# 33. Reporting Structure

Use:

```text
Title
Summary
Affected Endpoint
HTTP Method
Prerequisites
Steps to Reproduce
Expected Result
Actual Result
Security Impact
Evidence
Suggested Remediation
```

Good title:

```text
Authorization Bypass via PATCH Method on User Role Modification Endpoint
```

Bad title:

```text
PATCH Method Allowed
```

The title should describe the **security consequence**.

---

# 34. Remediation

Recommend:

```text
Explicitly allow only required HTTP methods.
Apply authorization consistently across every method.
Do not rely solely on frontend method restrictions.
Disable unnecessary methods.
Apply authentication middleware before method-specific routing.
Apply authorization consistently to all state-changing routes.
Validate method override headers/parameters.
Ensure proxy and backend interpret methods consistently.
Avoid state-changing GET endpoints.
Test method behavior across CDN/WAF/proxy/backend layers.
```

---

# 35. Agent Decision Tree

```text
START
  │
  ├── Discover endpoint
  │
  ├── Identify normal HTTP method
  │
  ├── Identify authentication requirements
  │
  ├── Identify authorization requirements
  │
  ├── Test alternative methods
  │
  ├── Behavior changed?
  │       │
  │       ├── NO → deprioritize
  │       │
  │       └── YES
  │
  ├── Protected functionality reached?
  │       │
  │       ├── NO → likely benign
  │       │
  │       └── YES
  │
  ├── Test lower privilege
  │
  ├── Test another object / tenant
  │
  ├── Verify actual side effect
  │
  ├── Determine root vulnerability
  │
  └── Report with demonstrated impact
```

---

# 36. Priority Matrix

| Test                                | Priority |
| ----------------------------------- | -------- |
| Method swap on admin endpoint       | Critical |
| Method swap on privileged API       | Critical |
| Method swap + IDOR/BOLA             | Critical |
| Method swap + privilege escalation  | Critical |
| Method swap + cross-tenant access   | Critical |
| Method swap + account modification  | High     |
| Method swap + deletion              | High     |
| Method swap + sensitive data access | High     |
| Method override                     | High     |
| GET state-changing endpoint         | High     |
| Proxy/backend method mismatch       | High     |
| Method + parameter pollution        | Medium   |
| HEAD differential                   | Medium   |
| OPTIONS discovery                   | Low      |
| TRACE enabled alone                 | Low      |

---

# 37. Final Validation Rules

The agent MUST NOT classify:

```text
"unusual HTTP method"
```

as a vulnerability by itself.

A candidate should normally satisfy:

```text
Method Manipulation
        +
Security-Control Difference
        +
Unauthorized Capability
        +
Verified Impact
```

Before finalizing a finding, answer:

```text
1. What method was expected?
2. What alternative method worked?
3. What security control was bypassed?
4. Which user/role could exploit it?
5. What resource was affected?
6. Was the action actually performed?
7. Can the impact be reproduced?
8. Is there a more accurate root-cause classification?
```

If these questions cannot be answered, keep the result as a **candidate**, not a confirmed vulnerability.

---

# 38. Safe Operating Rules

Only test assets that are explicitly authorized.

Prefer:

```text
controlled accounts
test objects
non-destructive actions
low request volume
manual verification
```

Avoid:

```text
mass deletion
mass modification
production disruption
testing unrelated third-party infrastructure
```

The goal is to establish exploitability with the minimum necessary interaction.

---

# 39. Skill Output Contract

For every candidate, return:

```text
{
  "endpoint": "...",
  "normal_method": "...",
  "tested_method": "...",
  "authentication_state": "...",
  "authorization_state": "...",
  "behavior_change": "...",
  "security_control_bypassed": "...",
  "verified_impact": "...",
  "root_cause": "...",
  "severity": "...",
  "confidence": "...",
  "evidence": "...",
  "next_test": "..."
}
```

Final principle:

> **Do not hunt for methods that are unusual. Hunt for method inconsistencies that cross a security boundary.**
