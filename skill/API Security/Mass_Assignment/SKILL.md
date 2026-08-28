# Mass Assignment Bug-Hunting Skill

## 1. Skill Identity

**Skill Name:** `Mass_Assignment`

**Category:** API Security / Broken Access Control / Input Validation

**Primary Objective:**
Identify cases where an application automatically binds attacker-controlled request parameters to internal object/model properties without enforcing a strict allowlist, allowing unauthorized modification of sensitive attributes.

**Core Principle:**

> Never assume that a field is protected because it is hidden from the frontend. Test whether the backend accepts and persists additional properties that the client was never intended to control.

---

# 2. What This Skill Hunts

Mass Assignment occurs when application frameworks automatically map request parameters into an object/model.

Typical vulnerable flow:

```text
HTTP Request
    ↓
JSON / Form Parameters
    ↓
Automatic Object Binding
    ↓
Database Model
    ↓
Sensitive Attribute Modified
```

Example concept:

```json
{
  "name": "Alice",
  "email": "alice@example.com",
  "role": "admin"
}
```

The UI may only expose:

```json
{
  "name": "Alice",
  "email": "alice@example.com"
}
```

If the backend automatically binds `role`, the attacker may gain control over a server-side property that should have been immutable or privileged.

---

# 3. High-Value Targets

Prioritize endpoints that create or update persistent objects.

## 3.1 User / Account Objects

Look for:

```text
role
is_admin
admin
permissions
privileges
verified
email_verified
phone_verified
status
account_status
disabled
active
staff
moderator
organization_id
tenant_id
owner_id
user_id
```

## 3.2 Financial Objects

Look for:

```text
balance
credit
credits
discount
price
amount
currency
payment_status
subscription
plan
billing_status
```

## 3.3 Organization / Tenant Objects

Look for:

```text
organization_id
tenant_id
workspace_id
team_id
owner_id
membership_role
access_level
```

## 3.4 Object Ownership

High priority:

```text
user_id
owner_id
created_by
account_id
customer_id
project_id
organization_id
tenant_id
```

## 3.5 Workflow / State Fields

Test properties such as:

```text
status
state
approved
approval_status
published
verified
reviewed
locked
completed
activated
```

---

# 4. Reconnaissance

Before testing, map the application's object model.

Identify:

* Account registration
* Profile update
* User update
* Organization management
* Team management
* Project creation
* Project update
* Subscription management
* Payment-related APIs
* Administrative APIs
* Object creation endpoints
* Object update endpoints
* GraphQL mutations
* REST `POST`
* REST `PUT`
* REST `PATCH`

Prioritize requests containing:

```text
POST
PUT
PATCH
```

especially when they contain JSON objects.

---

# 5. Baseline Request

Capture a legitimate request first.

Example:

```http
PATCH /api/users/me HTTP/1.1
Content-Type: application/json

{
  "name": "Alice",
  "bio": "Hello"
}
```

Determine:

1. Which fields are accepted?
2. Which fields are returned?
3. Which fields persist?
4. Which fields are server-generated?
5. Which fields are visible only in responses?
6. Which fields appear in other API responses?
7. Which properties appear to represent authorization or ownership?

---

# 6. Property Discovery

Mass Assignment testing depends heavily on discovering hidden model properties.

Use several sources.

## 6.1 API Responses

Inspect complete JSON responses.

Example:

```json
{
  "id": 123,
  "name": "Alice",
  "email": "alice@example.com",
  "role": "user",
  "verified": true,
  "organization_id": 42
}
```

Fields such as `role`, `verified`, and `organization_id` become candidates.

---

## 6.2 Frontend JavaScript

Search frontend bundles for:

```text
role
permissions
isAdmin
organizationId
tenantId
status
verified
```

Also search for:

```text
updateUser
updateProfile
createUser
PATCH
PUT
POST
mutation
```

---

## 6.3 API Documentation

Inspect:

```text
OpenAPI
Swagger
GraphQL schema
API documentation
SDK models
TypeScript interfaces
```

Pay particular attention to fields documented in response objects but absent from normal update forms.

---

## 6.4 GraphQL Schema

For GraphQL, compare:

```text
User
UserInput
UpdateUserInput
AdminUserInput
```

A field existing in the object model does not automatically mean it should be writable.

Test whether sensitive properties are accepted inside mutation inputs.

---

# 7. Core Testing Methodology

## Phase 1 — Establish Baseline

Send the normal request.

Record:

```text
Status code
Response body
Changed fields
Database-visible effects
Authorization context
```

---

## Phase 2 — Add One Candidate Property

Do not immediately modify many fields.

Example:

```json
{
  "name": "Alice",
  "role": "admin"
}
```

Compare against the baseline.

This makes it easier to determine which property caused the behavior.

---

## Phase 3 — Verify Persistence

A successful HTTP response alone is insufficient.

Check whether the property actually changed.

Possible verification:

```text
GET /api/users/me
```

or another legitimate endpoint that exposes the object.

Look for:

```text
role changed
status changed
permissions changed
ownership changed
organization changed
verification state changed
```

---

# 8. Candidate Property Classes

Test candidates by impact.

## Authorization

```text
role
permissions
is_admin
admin
access_level
privilege
```

## Ownership

```text
owner_id
user_id
organization_id
tenant_id
account_id
```

## Security State

```text
verified
email_verified
phone_verified
mfa_enabled
security_level
```

## Business State

```text
approved
status
published
active
subscription
plan
```

## Financial

```text
balance
credit
discount
amount
price
```

---

# 9. Differential Testing

The agent should compare:

```text
Baseline
      ↓
Modified request
      ↓
Response difference
      ↓
State difference
      ↓
Authorization difference
```

Useful comparison dimensions:

| Dimension    | Baseline  | Modified  |
| ------------ | --------- | --------- |
| Status       | normal    | changed?  |
| Response     | normal    | changed?  |
| Object state | unchanged | changed?  |
| Privileges   | normal    | elevated? |
| Ownership    | original  | changed?  |
| Workflow     | normal    | bypassed? |

A changed response without a persistent state change should not automatically be reported as a vulnerability.

---

# 10. Type Confusion Testing

Do not test only obvious values.

Where appropriate, test whether the backend treats the property differently based on type.

Examples of candidate representations:

```json
{
  "is_admin": true
}
```

```json
{
  "is_admin": false
}
```

```json
{
  "is_admin": 1
}
```

```json
{
  "is_admin": "true"
}
```

The objective is to determine whether inconsistent type handling allows an unauthorized property modification.

Keep testing controlled and avoid destructive values.

---

# 11. Nested Object Testing

Mass Assignment frequently appears in nested objects.

Example:

```json
{
  "profile": {
    "name": "Alice"
  }
}
```

Potential model structure:

```json
{
  "profile": {
    "name": "Alice",
    "role": "admin"
  }
}
```

Also investigate nested structures such as:

```text
user
profile
account
organization
settings
permissions
billing
metadata
```

---

# 12. Array / Collection Properties

Test whether server-side collections are writable.

Potential candidates:

```text
roles
permissions
groups
members
admins
owners
scopes
```

Example:

```json
{
  "permissions": ["read", "write"]
}
```

The key question is whether the application intended the current user to control the entire collection.

---

# 13. Metadata / Generic JSON Fields

Pay special attention to fields such as:

```text
metadata
attributes
properties
settings
options
config
data
extra
custom_fields
```

A generic object may be passed directly into an internal model or downstream service.

Investigate whether sensitive server-side properties can be injected through these structures.

---

# 14. HTTP Method Coverage

Do not limit testing to one method.

Compare:

```text
POST
PUT
PATCH
```

Some applications correctly restrict one update path but accidentally expose another.

Example:

```text
PATCH /api/profile
PUT /api/profile
POST /api/profile/update
```

The same object may have different binding behavior across endpoints.

---

# 15. Content-Type Coverage

Where legitimately supported by the target, compare:

```text
application/json
application/x-www-form-urlencoded
multipart/form-data
```

Different parsers or binding layers may process fields differently.

The goal is to identify inconsistent server-side property filtering.

---

# 16. REST API Testing

Prioritize:

```text
POST /users
POST /projects
POST /organizations
PATCH /users/{id}
PATCH /projects/{id}
PUT /profile
PUT /settings
```

For each endpoint:

```text
1. Capture baseline.
2. Identify object properties.
3. Add one candidate property.
4. Send request.
5. Observe response.
6. Verify persistence.
7. Determine security impact.
```

---

# 17. GraphQL Testing

Identify mutations such as:

```graphql
updateUser
updateProfile
createUser
updateOrganization
updateProject
```

Inspect input types:

```graphql
UpdateUserInput
UpdateProfileInput
UpdateOrganizationInput
```

Compare input fields with output fields.

High-risk mismatch:

```text
User object:
    id
    email
    role
    verified

UpdateUserInput:
    name
    email
```

If the server unexpectedly accepts and processes:

```text
role
verified
```

investigate further.

---

# 18. Multi-Account Testing

Use separate authorized accounts when the application permits it.

Recommended model:

```text
Account A = normal user
Account B = another normal user
Account C = privileged test account, if available
```

Compare:

```text
What A can modify about A
What A can modify about B
What fields A can influence
What fields only privileged users should control
```

This helps distinguish:

```text
Mass Assignment
```

from:

```text
IDOR/BOLA
Broken Access Control
Privilege Escalation
```

A finding may involve multiple vulnerability classes simultaneously.

---

# 19. Chained Impact Analysis

Mass Assignment is often a primitive rather than the final impact.

Investigate whether modifying a property leads to:

```text
Privilege escalation
Account takeover
Authorization bypass
Tenant escape
Ownership takeover
Payment manipulation
Workflow bypass
Security-control bypass
```

Example chain:

```text
Mass Assignment
      ↓
role=user → role=admin
      ↓
Admin endpoint becomes accessible
      ↓
Unauthorized administrative actions
```

Or:

```text
Mass Assignment
      ↓
organization_id modified
      ↓
Object associated with another tenant
      ↓
Cross-tenant access
```

Do not claim the chain unless each step is actually demonstrated.

---

# 20. False Positives

Do not report merely because:

```text
1. An undocumented field is accepted.
2. A property appears in the response.
3. The server echoes an injected parameter.
4. The request returns HTTP 200.
5. The field appears temporarily in memory.
```

A meaningful finding normally requires evidence that an unauthorized property is:

```text
accepted
AND
processed
AND/OR
persisted
AND
outside the intended authorization boundary
```

---

# 21. Safe Verification

Prefer non-destructive properties first.

Good candidates:

```text
test metadata
non-sensitive state
controlled test profile
benign role/state where the program explicitly permits testing
```

Avoid:

```text
destructive account changes
real financial manipulation
deleting data
changing another user's credentials
production-impacting administrative operations
```

Only perform impact validation within the program's authorized testing boundaries.

---

# 22. Framework Fingerprinting

The agent may use application fingerprints to prioritize hypotheses.

Potential ecosystems include:

```text
Ruby on Rails
Laravel
Django
Spring
ASP.NET
Node.js / Express
NestJS
Mongoose
Sequelize
Prisma
ORM-backed REST APIs
GraphQL resolvers
```

Framework knowledge should guide investigation, **not be treated as proof of vulnerability**.

The actual behavior of the endpoint is authoritative.

---

# 23. Automation Strategy

The agent should maintain a candidate-property dictionary.

Example categories:

```text
AUTH_PROPERTIES
OWNERSHIP_PROPERTIES
SECURITY_PROPERTIES
WORKFLOW_PROPERTIES
FINANCIAL_PROPERTIES
TENANT_PROPERTIES
```

For each discovered object:

```text
1. Extract visible properties.
2. Identify sensitive-looking properties.
3. Compare request vs response schemas.
4. Generate controlled candidates.
5. Test one property at a time.
6. Compare response/state.
7. Record evidence.
8. Escalate only confirmed behavior.
```

Avoid blindly spraying hundreds of parameters against production systems.

---

# 24. Evidence Requirements

For every suspected vulnerability collect:

```text
Endpoint
HTTP method
Authentication context
Baseline request
Modified request
Baseline response
Modified response
State verification
Affected property
Expected authorization
Actual behavior
Security impact
```

The strongest evidence demonstrates:

```text
Unauthorized actor
        ↓
controls hidden property
        ↓
server accepts property
        ↓
property persists
        ↓
security boundary changes
```

---

# 25. Severity Assessment

Severity depends on the property and resulting impact.

### Informational / Low

```text
Non-security-sensitive property
No meaningful authorization impact
No persistence
```

### Medium

```text
Unauthorized modification of meaningful account state
Sensitive workflow manipulation
Security-relevant configuration change
```

### High

```text
Privilege escalation
Unauthorized ownership modification
Security-control bypass
Cross-user impact
Cross-tenant impact
```

### Critical

Potentially:

```text
Full administrative takeover
Cross-tenant compromise at scale
Account takeover affecting arbitrary users
Major financial/business compromise
```

Never assign severity based solely on the existence of mass assignment. Rate the demonstrated impact.

---

# 26. Distinguishing Related Vulnerabilities

## Mass Assignment vs IDOR/BOLA

**Mass Assignment:**

```text
Attacker controls an unintended property
```

**IDOR/BOLA:**

```text
Attacker accesses/modifies an object they are not authorized to access
```

They can occur together.

---

## Mass Assignment vs Privilege Escalation

Mass Assignment describes the vulnerable input-binding mechanism.

Privilege escalation describes the resulting security impact.

Example:

```text
Mass Assignment → role modification → Privilege Escalation
```

---

## Mass Assignment vs Parameter Pollution

Parameter pollution concerns duplicate/conflicting parameters.

Mass Assignment concerns unintended mapping of request properties into server-side objects.

---

# 27. Agent Decision Tree

```text
START
  |
  v
Find object creation/update endpoint
  |
  v
Capture baseline request
  |
  v
Discover object properties
  |
  v
Identify security-sensitive candidates
  |
  v
Inject ONE candidate
  |
  v
Request accepted?
  |
  +-- NO --> Record negative result
  |
  +-- YES
       |
       v
Was property processed?
       |
       +-- NO --> Possible parser/echo behavior
       |
       +-- YES
             |
             v
Was state persisted?
             |
             +-- NO --> Do not overclaim
             |
             +-- YES
                   |
                   v
Is modification unauthorized?
                   |
                   +-- NO --> Not a vulnerability
                   |
                   +-- YES
                         |
                         v
Determine security impact
                         |
                         v
Test safe impact escalation
                         |
                         v
Generate evidence
```

---

# 28. Burp Workflow

When Burp is available:

```text
Proxy → HTTP history
        ↓
Filter POST / PUT / PATCH
        ↓
Identify JSON/object bodies
        ↓
Send candidate request to Repeater
        ↓
Create baseline
        ↓
Modify one property
        ↓
Compare responses
        ↓
Verify with GET
```

Use Burp extensions or automation only when they remain within the target's testing rules and rate limits.

---

# 29. Browser / Application Workflow

When browser automation is available:

```text
Login
 ↓
Perform legitimate profile/object update
 ↓
Capture network request
 ↓
Identify object schema
 ↓
Inspect response
 ↓
Find hidden properties
 ↓
Reproduce request in controlled API client
 ↓
Test candidate property
 ↓
Verify resulting state
```

The browser UI should not be considered the authoritative list of writable properties.

The backend is authoritative.

---

# 30. Required Notes

Maintain structured notes:

```text
Target:
Endpoint:
Method:
Object:
Account:
Baseline:
Candidate property:
Injected value:
Response:
Persistence:
Authorization:
Impact:
Evidence:
```

For every confirmed finding, preserve the minimum reproducible request/response pair.

---

# 31. Reporting Template

## Title

```text
Mass Assignment Allows Unauthorized Modification of [Property]
```

If there is a demonstrated impact:

```text
Mass Assignment Leads to [Privilege Escalation / Ownership Takeover / etc.]
```

## Summary

Explain:

```text
The endpoint accepts an attacker-controlled property that should be
server-controlled, allowing an unauthorized user to modify [property].
```

## Steps

Include:

1. Authentication context.
2. Baseline request.
3. Modified request.
4. Server response.
5. State verification.
6. Demonstrated impact.

## Impact

Explain the actual security consequence rather than simply stating:

```text
Mass Assignment exists.
```

---

# 32. Agent Rules

The agent MUST:

* Start from real application behavior.
* Establish a baseline before modification.
* Test one candidate property at a time.
* Prefer non-destructive verification.
* Verify persistence whenever possible.
* Distinguish reflection from actual processing.
* Distinguish accepted input from authorized input.
* Use multiple accounts when authorization boundaries matter.
* Check both REST and GraphQL when present.
* Analyze nested objects.
* Analyze ownership and tenant properties.
* Analyze authorization-sensitive properties.
* Stop when testing would become destructive.
* Respect program scope and rate limits.
* Preserve reproducible evidence.
* Avoid claiming impact that was not demonstrated.

The agent MUST NOT:

* Treat every hidden parameter as vulnerable.
* Treat HTTP 200 as proof.
* Blindly inject arbitrary sensitive values.
* Modify real financial balances.
* Perform destructive actions to prove impact.
* Access unrelated users or tenants without authorization.
* Claim account takeover without demonstrating the chain.
* Claim critical severity solely because a sensitive-looking parameter exists.

---

# 33. Priority Matrix

| Target                          | Priority |
| ------------------------------- | -------: |
| `role` / `is_admin`             | Critical |
| `permissions`                   | Critical |
| `tenant_id` / `organization_id` | Critical |
| `owner_id`                      | Critical |
| `verified` / security state     |     High |
| `status` / approval state       |     High |
| `subscription` / plan           |     High |
| `account_id`                    |     High |
| `price` / discount / credit     |     High |
| Generic metadata                |   Medium |
| Cosmetic profile fields         |      Low |

Priority is for **testing order**, not automatic severity.

---

# 34. Final Validation Checklist

* [ ] Identified an object creation/update endpoint.
* [ ] Captured a legitimate baseline.
* [ ] Discovered candidate server-side properties.
* [ ] Selected a security-relevant property.
* [ ] Added only one candidate property.
* [ ] Confirmed the server processed it.
* [ ] Confirmed persistence where applicable.
* [ ] Confirmed the property was not intended to be user-controlled.
* [ ] Confirmed the attacker lacked authorization to modify it.
* [ ] Verified security impact safely.
* [ ] Distinguished Mass Assignment from IDOR/BOLA.
* [ ] Preserved baseline and modified requests.
* [ ] Preserved response/state evidence.
* [ ] Checked program scope and rate limits.
* [ ] Assigned severity based on demonstrated impact.

---

# 35. Core Hunting Philosophy

```text
Do not ask:

"Can I add a hidden parameter?"

Ask:

"What server-side properties exist,
which ones does the application actually bind,
which ones should the current user control,
and what security boundary changes when one is modified?"
```

The strongest Mass Assignment findings are not merely undocumented fields.

They demonstrate:

```text
Hidden Property
      +
Automatic Binding
      +
Unauthorized Control
      +
Persistent State Change
      +
Security Impact
```

That combination should drive the final finding.
