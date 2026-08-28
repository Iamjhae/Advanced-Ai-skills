# API Parameter Manipulation

## 1. Skill Identity

**Skill Name:** `API_Parameter_Manipulation`

**Category:** API Security / Input Manipulation / Business Logic

**Primary Objective:**
Identify vulnerabilities caused by manipulating API parameters, parameter names, values, types, structures, duplication, ordering, encoding, or unexpected fields in order to bypass validation, alter application behavior, access unintended functionality, modify protected attributes, or create security-impacting business-logic conditions.

**Core Principle:**

> Never assume an API parameter is trusted merely because the frontend normally sends it in a specific format.

The goal is not simply to change parameters and observe errors. The goal is to determine whether parameter manipulation creates a **security boundary violation, unauthorized state change, privilege impact, data exposure, integrity impact, or meaningful business-logic bypass**.

---

# 2. Scope

Test parameter manipulation across:

* REST APIs
* JSON APIs
* GraphQL where applicable
* RPC-style APIs
* Mobile APIs
* Internal APIs exposed through frontend applications
* Admin APIs
* Partner/integration APIs
* Webhooks
* File/API metadata
* Search/filter/sort endpoints
* Pagination
* Object creation/update endpoints
* Authentication/session endpoints
* Payment/order/subscription APIs
* User/profile/account APIs

Prioritize endpoints that:

* modify persistent state
* reference object IDs
* accept role/account/user identifiers
* accept prices or quantities
* control permissions
* control workflow states
* accept hidden/internal fields
* perform administrative actions
* process financial/business operations

---

# 3. Threat Model

Parameter manipulation becomes a vulnerability when the backend incorrectly trusts client-controlled input.

Common failure patterns:

```text
Client-controlled parameter
        ↓
Missing / weak validation
        ↓
Unexpected value / field / type
        ↓
Backend accepts manipulated input
        ↓
Security-sensitive behavior
```

The important question is:

> "What security property does this parameter control, and what happens if I change it?"

---

# 4. Parameter Taxonomy

Build a parameter inventory for every API endpoint.

### 4.1 Identity Parameters

Examples:

```text
id
user_id
account_id
customer_id
owner_id
organization_id
tenant_id
profile_id
member_id
```

Test for:

* cross-user access
* cross-tenant access
* ownership bypass
* authorization confusion
* object substitution

---

### 4.2 Authorization Parameters

Examples:

```text
role
is_admin
admin
permission
permissions
access_level
privilege
scope
approved
verified
status
```

Test whether changing these values causes unauthorized privileges.

---

### 4.3 State Parameters

Examples:

```text
status
state
stage
workflow_state
is_active
is_verified
is_approved
completed
published
enabled
```

Test:

```text
normal → privileged state
normal → completed state
pending → approved
unverified → verified
disabled → enabled
```

Do not report a state change by itself. Establish the security/business impact.

---

### 4.4 Financial Parameters

Examples:

```text
price
amount
quantity
discount
discount_code
currency
tax
fee
total
balance
credit
limit
```

Test whether manipulation causes:

* unauthorized discount
* negative/zero-value transaction
* price modification
* balance manipulation
* quantity manipulation
* payment bypass
* incorrect billing

Use test accounts and non-destructive transactions where possible.

---

### 4.5 Feature-Control Parameters

Examples:

```text
feature
feature_enabled
debug
preview
internal
beta
experimental
include_private
include_deleted
include_sensitive
```

Determine whether client-controlled feature flags expose privileged functionality or sensitive information.

---

### 4.6 Pagination Parameters

Examples:

```text
page
offset
limit
size
cursor
start
end
```

Test for:

* excessive data exposure
* authorization inconsistencies
* hidden records
* private objects appearing across pages
* inconsistent filtering

---

### 4.7 Filtering Parameters

Examples:

```text
filter
search
query
where
status
type
category
owner
created_by
```

Test whether filters can bypass server-side authorization.

---

### 4.8 Sorting Parameters

Examples:

```text
sort
order
sort_by
order_by
direction
```

Look for:

* unauthorized fields
* hidden fields
* internal metadata
* backend query manipulation
* information disclosure

---

# 5. Core Manipulation Classes

## 5.1 Value Substitution

Change a legitimate value to another valid value.

Example:

```json
{
  "user_id": "1001"
}
```

→

```json
{
  "user_id": "1002"
}
```

Compare:

* status code
* response body
* returned object
* authorization behavior
* side effects

This overlaps with IDOR/BOLA. If the primary security issue is unauthorized object access, classify it accordingly rather than forcing it into Parameter Manipulation.

---

# 6. Type Confusion

Change parameter types.

Example:

```json
{
  "enabled": true
}
```

Test alternatives:

```json
{
  "enabled": false
}
```

```json
{
  "enabled": "true"
}
```

```json
{
  "enabled": 1
}
```

```json
{
  "enabled": 0
}
```

```json
{
  "enabled": null
}
```

```json
{
  "enabled": []
}
```

```json
{
  "enabled": {}
}
```

Relevant types:

```text
string
integer
float
boolean
null
array
object
```

Look for inconsistent authorization or business logic.

---

# 7. Missing Parameters

Remove parameters entirely.

Original:

```json
{
  "user_id": "123",
  "role": "user",
  "status": "active"
}
```

Test:

```json
{
  "user_id": "123"
}
```

Also test removing one parameter at a time.

Look for:

* insecure defaults
* authorization bypass
* unintended state transitions
* server-side assumptions
* privileged defaults

---

# 8. Null Manipulation

Test:

```json
{
  "role": null
}
```

```json
{
  "user_id": null
}
```

```json
{
  "status": null
}
```

Compare with:

* parameter omitted
* empty string
* zero
* false

Important distinction:

```text
missing ≠ null ≠ empty ≠ false ≠ 0
```

Different backend frameworks may process them differently.

---

# 9. Empty-Value Manipulation

Test:

```text
""
" "
[]
{}
```

For query parameters:

```text
?role=
?user_id=
?status=
```

Observe whether validation, authorization, or filtering changes.

---

# 10. Duplicate Parameters

Test duplicate query parameters:

```text
?id=100&id=200
```

```text
?role=user&role=admin
```

Duplicate JSON keys where accepted:

```json
{
  "role": "user",
  "role": "admin"
}
```

Test duplicates in:

* query strings
* form data
* JSON
* headers when relevant

Determine which value wins:

```text
first wins
last wins
merged
array
backend-dependent
proxy/backend disagreement
```

Pay special attention when different components parse duplicates differently.

---

# 11. Parameter Pollution

Test:

```text
?id=1&id=2&id=3
```

and:

```text
?role=user&role=admin
```

Look for inconsistencies between:

```text
WAF
CDN
proxy
API gateway
application
database layer
```

Parameter pollution can become significantly more important when one component validates one value while another component uses a different value.

---

# 12. Array/Object Substitution

If a parameter normally expects:

```json
{
  "role": "user"
}
```

test:

```json
{
  "role": ["user", "admin"]
}
```

and:

```json
{
  "role": {
    "value": "admin"
  }
}
```

Likewise test object parameters as arrays and arrays as scalars.

Goal:

* parser confusion
* validation bypass
* authorization inconsistencies
* business-logic bypass

---

# 13. Numeric Manipulation

For numeric parameters test boundary conditions:

```text
0
1
-1
-100
MAX_INT
MAX_INT + 1
very large value
decimal
scientific notation
leading zeros
```

Examples:

```text
quantity=0
quantity=-1
amount=0
amount=-1
limit=0
limit=-1
page=0
page=-1
```

For financial operations, avoid real destructive transactions unless explicitly authorized.

---

# 14. Boolean Manipulation

Test:

```text
true
false
"true"
"false"
1
0
"1"
"0"
null
""
```

Especially target:

```text
is_admin
is_verified
approved
enabled
public
private
active
```

A boolean manipulation becomes security-relevant when it changes a protected state without proper authorization.

---

# 15. Enum Manipulation

If an API expects:

```text
role=user
```

identify the complete accepted enum set.

Examples:

```text
user
admin
moderator
owner
staff
internal
```

Do not assume the frontend-visible options are the complete backend-supported options.

Look for:

* hidden roles
* internal states
* undocumented workflow states
* deprecated privileged states

---

# 16. Case Manipulation

Test case sensitivity:

```text
admin
Admin
ADMIN
aDmIn
```

For values such as:

```text
role
status
type
username
email
scope
```

A dangerous case-normalization discrepancy can cause authorization failures.

---

# 17. Encoding Manipulation

Test equivalent representations where appropriate:

```text
URL encoding
double URL encoding
Unicode representations
JSON escaping
HTML encoding
percent encoding
```

Compare:

```text
server parsing
authorization
validation
routing
logging
```

Do not treat encoding differences as vulnerabilities unless they produce meaningful security impact.

---

# 18. Parameter Name Manipulation

Change parameter names or casing:

```text
user_id
userId
UserId
USER_ID
userid
```

Test alternative naming conventions only when supported by the API/framework context.

Also look for:

```text
legacy parameters
deprecated parameters
undocumented parameters
mobile-only parameters
internal parameters
```

---

# 19. Mass Assignment / Hidden Parameter Testing

This is one of the highest-value areas.

Suppose the normal request is:

```json
{
  "name": "Alice",
  "email": "alice@example.com"
}
```

Look for sensitive server-side fields such as:

```text
role
is_admin
permissions
owner_id
organization_id
verified
approved
status
balance
credits
subscription
plan
```

Test adding **one candidate field at a time**.

Example:

```json
{
  "name": "Alice",
  "role": "admin"
}
```

or:

```json
{
  "name": "Alice",
  "is_verified": true
}
```

A valid finding requires evidence that the server actually honors the unauthorized field and that this creates security impact.

---

# 20. HTTP Method + Parameter Interaction

Compare equivalent operations where authorized:

```text
GET
POST
PUT
PATCH
DELETE
```

Look for differences in parameter validation.

Example:

```text
PUT /api/user/123
PATCH /api/user/123
```

The same field may be:

```text
blocked by PUT
accepted by PATCH
```

This can reveal inconsistent server-side authorization or validation.

---

# 21. Query vs Body Parameter Conflicts

Test whether the same logical parameter can appear in multiple locations.

Example:

```http
POST /api/account/update?role=user
```

Body:

```json
{
  "role": "admin"
}
```

Reverse the values:

```http
POST /api/account/update?role=admin
```

Body:

```json
{
  "role": "user"
}
```

Determine which source takes precedence.

This is particularly valuable when:

```text
gateway validates query
application trusts body
```

or vice versa.

---

# 22. Header + Parameter Conflicts

Look for parameters represented in multiple locations:

```text
Authorization
X-User-ID
X-Account-ID
query parameter
JSON body
path parameter
cookie
```

The objective is to identify trust-boundary inconsistencies.

Do not assume a custom header is authoritative merely because the frontend sends it.

---

# 23. Path Parameter Manipulation

Examples:

```http
/api/users/123
/api/accounts/123
/api/orders/123
```

Test legitimate alternate object identifiers where authorization testing is in scope.

Also compare:

```text
path ID
query ID
body ID
```

for inconsistent authorization.

---

# 24. Parameter Order

For APIs where ordering can matter, test:

```text
parameter A → parameter B
```

versus:

```text
parameter B → parameter A
```

For JSON/object parameters, determine whether duplicate keys or parser behavior causes different interpretation.

This is mainly useful when application components use different parsers.

---

# 25. Unknown Parameter Injection

Add harmless unknown fields:

```json
{
  "name": "Alice",
  "test": "123"
}
```

Then test candidate security-sensitive fields discovered through:

* frontend JavaScript
* mobile applications
* API documentation
* OpenAPI specifications
* Burp history
* previous responses
* GraphQL schemas
* error messages

Do not blindly fuzz thousands of fields without understanding the target.

---

# 26. Parameter Discovery

Build a parameter source map.

### Frontend

Inspect:

```text
JavaScript
source maps
API clients
forms
React/Vue/Angular state
mobile bundles
```

### API Documentation

Look for:

```text
OpenAPI
Swagger
GraphQL schema
Postman collections
SDKs
developer documentation
```

### Traffic

Use:

```text
Burp HTTP history
Repeater
Proxy
Logger
```

### Responses

Search for:

```text
field names
IDs
state names
internal metadata
feature flags
object properties
```

### Errors

Validation errors can reveal:

```text
accepted parameter names
expected types
enum values
internal fields
backend framework behavior
```

---

# 27. Differential Testing

Every meaningful parameter mutation should be compared against a baseline.

Record:

```text
Original request
Modified request
Status code
Response length
Response body
Headers
Timing
State change
Authorization result
Side effects
```

Use a simple model:

```text
Baseline
   ↓
One mutation
   ↓
Compare
   ↓
Interesting difference?
   ↓
Reproduce
   ↓
Determine security impact
```

Avoid changing multiple variables simultaneously unless specifically testing parser interaction.

---

# 28. Two-Account Testing

For access-control-related parameter manipulation, use:

```text
Account A = attacker/test account
Account B = victim/test account
```

Capture the legitimate request from A.

Then manipulate only the relevant parameter toward B's object/data.

Validate:

```text
Can A read B's data?
Can A modify B's data?
Can A delete B's data?
Can A trigger B's workflow?
Can A change B's privileges?
```

If successful, classify the underlying issue appropriately, usually as:

```text
IDOR/BOLA
Broken Access Control
Privilege Escalation
```

rather than merely Parameter Manipulation.

---

# 29. Multi-Tenant Testing

For applications with organizations/workspaces/projects:

```text
Tenant A
Tenant B
```

Identify parameters:

```text
tenant_id
organization_id
workspace_id
project_id
account_id
```

Test whether manipulating them allows:

* cross-tenant reads
* cross-tenant writes
* cross-tenant actions
* tenant configuration changes

The primary vulnerability should be classified according to the resulting security violation.

---

# 30. High-Value Parameter Targets

Prioritize:

### Critical

```text
role
permissions
owner_id
tenant_id
account_id
user_id
balance
payment_amount
subscription
status
approval
```

### High

```text
is_admin
is_verified
organization_id
workflow_state
discount
price
quantity
visibility
```

### Medium

```text
filter
sort
page
limit
type
category
search
```

Priority depends on actual application context and impact.

---

# 31. Business Logic Testing

Parameter manipulation is especially valuable when parameters control workflows.

Examples:

```text
cart → checkout → payment → fulfillment
```

Test whether changing:

```text
quantity
price
discount
status
payment_state
shipping_cost
currency
```

causes an invalid transition.

Another workflow:

```text
registration
    ↓
email verification
    ↓
approval
    ↓
account activation
```

Test whether manipulating:

```text
verified
approved
status
state
```

can skip required steps.

---

# 32. API Response Analysis

Do not focus only on HTTP status codes.

Inspect:

```text
response body
embedded objects
metadata
permissions
roles
IDs
pagination
error messages
debug fields
internal flags
```

An endpoint returning `200` does not automatically mean a vulnerability exists.

Likewise, a `403` does not prove the backend is secure if another representation of the same parameter bypasses the check.

---

# 33. Automation Strategy

Automation should come **after manual understanding**.

Recommended workflow:

```text
1. Discover endpoint
2. Capture baseline
3. Identify parameters
4. Classify parameters
5. Select high-value parameters
6. Mutate one variable
7. Compare response
8. Verify state change
9. Reproduce
10. Determine impact
```

Useful tooling:

```text
Burp Suite
Repeater
Intruder
Logger
Proxy
Param Miner
HTTP history
OpenAPI/Swagger
custom scripts
```

For automation, keep mutation sets targeted rather than blindly generating huge request volumes.

---

# 34. Safe Mutation Matrix

For a candidate parameter, test:

```text
Original
Missing
Null
Empty
Zero
Negative
Large
Wrong type
Array
Object
Boolean
Duplicate
Alternative case
Alternative enum
Unexpected field
```

Example:

```text
role=user

→ role=admin
→ role=
→ role=null
→ role=["admin"]
→ role={"value":"admin"}
→ role=true
→ role=ADMIN
→ duplicate role values
→ parameter omitted
```

Only retain mutations that reveal meaningful behavior.

---

# 35. False Positive Controls

Do not report:

* generic `400` responses
* generic validation errors
* harmless ignored parameters
* UI-only changes
* different response formatting
* parameters that are intentionally client-controlled
* undocumented parameters with no impact
* duplicate parameters that produce no security consequence

Always establish:

```text
Manipulation
      +
Server acceptance
      +
Security-sensitive effect
      +
Unauthorized or unintended impact
```

---

# 36. Vulnerability Validation

Before reporting, answer:

### 1. Is the parameter attacker-controlled?

If no → stop.

### 2. Does the server accept the manipulated value?

If no → usually stop.

### 3. Does behavior change?

If no → stop.

### 4. Is the changed behavior security-sensitive?

If no → likely informational.

### 5. Does the attacker gain unauthorized capability or impact?

If yes → validate further.

### 6. Can the result be reproduced?

If no → gather stronger evidence.

---

# 37. Severity Reasoning

Severity should be based on actual impact.

Potential impact categories:

```text
Confidentiality
Integrity
Availability
Authentication
Authorization
Privilege
Financial
Cross-tenant isolation
Business workflow
```

Examples:

```text
Changing a cosmetic preference
→ Low / Informational

Changing another user's data
→ High potential impact

Changing role to admin
→ Critical/High depending on scope

Bypassing payment validation
→ High/Critical depending on financial impact

Cross-tenant access
→ High/Critical depending on exposed data and platform policy
```

Do not assign severity merely because a parameter is named `admin`.

---

# 38. Evidence Requirements

A strong finding should contain:

```text
Endpoint
HTTP method
Original parameter
Manipulated parameter
Original request
Modified request
Original response
Modified response
Observed security impact
Reproduction steps
Affected accounts/objects
Business/security consequence
```

Prefer a minimal proof-of-concept.

Avoid unnecessary destructive actions.

---

# 39. Reporting Structure

Use:

```text
Title

Summary

Affected Endpoint

Parameter

Preconditions

Steps to Reproduce

Original Request

Modified Request

Observed Result

Security Impact

Why Server-Side Validation/Authorization Failed

Suggested Remediation
```

The title should describe the actual impact.

Bad:

```text
API parameter can be manipulated
```

Better:

```text
Unauthorized role modification through mass-assignment of the `role` parameter
```

Or:

```text
Cross-tenant object modification through manipulated `organization_id`
```

---

# 40. Remediation Guidance

Recommend:

### Server-side validation

Never rely exclusively on frontend validation.

### Allowlisting

Accept only expected parameters.

### Strong schemas

Validate:

```text
type
format
range
enum
required fields
nested structure
```

### Authorization

Authorization decisions must be derived from trusted server-side identity/context.

Do not trust:

```text
role
user_id
tenant_id
owner_id
permissions
```

provided by the client.

### Mass-assignment protection

Use explicit writable-field allowlists.

### Consistent parsing

Ensure:

```text
proxy
gateway
WAF
application
framework
database
```

interpret parameters consistently.

### Business-rule validation

Validate state transitions server-side.

---

# 41. Agent Decision Tree

```text
START
  |
  v
Discover API endpoint
  |
  v
Inventory parameters
  |
  v
Classify each parameter
  |
  +--> Identity?
  |      |
  |      +--> Test authorization / BOLA
  |
  +--> Privilege?
  |      |
  |      +--> Test role/permission manipulation
  |
  +--> State?
  |      |
  |      +--> Test workflow bypass
  |
  +--> Financial?
  |      |
  |      +--> Test business-rule integrity
  |
  +--> Object update?
  |      |
  |      +--> Test mass assignment
  |
  +--> Generic input?
         |
         +--> Test type/null/empty/duplicate/enum
                    |
                    v
             Compare baseline
                    |
                    v
             Security impact?
                /       \
              NO         YES
              |           |
           discard     reproduce
                          |
                          v
                     classify bug
                          |
                          v
                     assess impact
                          |
                          v
                       report
```

---

# 42. Priority Workflow

For each new API:

```text
P0
├── role
├── permissions
├── owner_id
├── tenant_id
├── user_id
├── account_id
└── privileged state parameters

P1
├── mass-assignment candidates
├── workflow states
├── financial values
├── price/quantity
├── approval/verification
└── hidden/internal fields

P2
├── type confusion
├── null/empty
├── duplicate parameters
├── enum manipulation
├── filtering
└── pagination

P3
├── cosmetic parameters
├── low-impact preferences
└── parameters with no security-sensitive effect
```

---

# 43. Operational Rules for the Hunting Agent

The agent MUST:

1. Establish a baseline before mutation.
2. Change one parameter at a time whenever possible.
3. Preserve authentication context.
4. Track which account/session performed every request.
5. Distinguish client-side behavior from server-side behavior.
6. Prioritize authorization, identity, privilege, state, and financial parameters.
7. Test hidden writable fields on create/update endpoints.
8. Test parameter type inconsistencies.
9. Test duplicate parameters where parser discrepancies are plausible.
10. Compare query/body/path representations when the same logical value exists in multiple locations.
11. Verify persistent state changes.
12. Use a second account for cross-user authorization testing.
13. Use separate tenants for cross-tenant testing.
14. Avoid destructive actions unless explicitly authorized.
15. Stop escalating once sufficient proof of impact exists.
16. Classify the final issue according to its actual root vulnerability.

---

# 44. Root-Cause Classification

Parameter manipulation is often a **technique**, not the final vulnerability category.

Map findings as follows:

```text
Parameter manipulation
        |
        +--> Unauthorized object access
        |       → IDOR / BOLA
        |
        +--> Unauthorized action
        |       → Broken Access Control
        |
        +--> Privilege modification
        |       → Privilege Escalation
        |
        +--> Protected field injection
        |       → Mass Assignment
        |
        +--> Workflow skipping
        |       → Business Logic Vulnerability
        |
        +--> Parser discrepancy
        |       → Parameter Pollution / Parsing issue
        |
        +--> Data/query manipulation
        |       → Injection class if applicable
        |
        +--> No security impact
                → Not a vulnerability
```

---

# 45. Compact Testing Checklist

* [ ] Capture baseline request.
* [ ] Inventory all parameters.
* [ ] Identify identity parameters.
* [ ] Identify authorization parameters.
* [ ] Identify state/workflow parameters.
* [ ] Identify financial parameters.
* [ ] Identify hidden/internal fields.
* [ ] Test parameter omission.
* [ ] Test `null`.
* [ ] Test empty values.
* [ ] Test type confusion.
* [ ] Test boolean variants.
* [ ] Test numeric boundaries.
* [ ] Test enum alternatives.
* [ ] Test duplicate parameters.
* [ ] Test parameter pollution.
* [ ] Test array/object substitutions.
* [ ] Test case variations.
* [ ] Test query/body conflicts.
* [ ] Test path/body conflicts.
* [ ] Test mass-assignment candidates.
* [ ] Test with a second authorized account where applicable.
* [ ] Test cross-tenant boundaries where applicable.
* [ ] Verify persistent state.
* [ ] Determine actual security impact.
* [ ] Reproduce with minimal requests.
* [ ] Map to the correct vulnerability class.
* [ ] Document evidence.
* [ ] Assign severity based on impact.

---

# 46. Skill Success Condition

The skill succeeds when it can move from:

```text
API endpoint
    ↓
parameter inventory
    ↓
high-value parameter identification
    ↓
targeted mutation
    ↓
differential behavior
    ↓
security-impact validation
    ↓
root-cause classification
    ↓
reproducible finding
```

The skill must **not** equate "parameter accepted" with "vulnerability".

The final decision must always be based on:

```text
Attacker Control
        +
Server-Side Acceptance
        +
Security Boundary Violation
        +
Meaningful Impact
        +
Reproducibility
```

Only when these conditions are sufficiently demonstrated should the agent escalate the result as a vulnerability.
