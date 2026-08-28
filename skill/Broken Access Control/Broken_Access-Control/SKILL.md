---

name: broken-access-control
description: Detects and validates broken authorization in web applications and APIs by mapping actors, resources, actions, roles, tenants, states, and trust boundaries, then performing prioritized differential tests for horizontal/vertical escalation, object-level access, forced browsing, function-level bypass, property-level authorization, and authorization-state manipulation. Trigger when testing authenticated or privileged functionality for unauthorized reads, writes, deletes, execution, or privilege changes.
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Broken Access Control Hunting Skill

## 1. Purpose

Find **real authorization failures**, not merely unusual responses.

Primary objective:

```text
Actor
+
Resource
+
Action
+
Expected authorization boundary
↓
Controlled differential test
↓
Unauthorized capability?
↓
Validate impact
```

Treat:

```text
"Can I reach it?"
```

as different from:

```text
"Can I perform an unauthorized security-relevant action?"
```

Do not classify a finding from status codes, UI visibility, parameter reflection, or endpoint discovery alone.

---

# 2. Operating Model

Model every interesting operation as:

```text
ACTOR → ACTION → RESOURCE → CONTEXT
```

Record:

```text
Actor:
- anonymous
- authenticated user
- owner
- same-role different user
- lower role
- higher role
- tenant A
- tenant B

Action:
- read
- create
- update
- delete
- execute
- approve
- invite
- export
- change security setting
- change ownership
- administer

Resource:
- account
- user
- organization
- project
- invoice
- order
- file
- message
- API object
- configuration
- permission
- integration
- administrative function

Context:
- tenant
- workflow state
- ownership
- object relationship
- authentication state
- session
- feature flag
- approval state
- subscription tier
```

The important question is:

> Which authorization relationship is the application supposed to enforce, and can the current actor violate it?

---

# 3. Core Hunt State

Maintain only this compact state:

```text
Target:
Relevant feature:
Actors / sessions:
Roles:
Tenants:
Resources:
Actions:
Authorization assumptions:
Interesting endpoints:
Interesting parameters:
Tests completed:
Observed boundary behavior:
Current hypothesis:
Next test:
Confidence:
```

Do not carry raw request/response history unless required for reproduction.

Compress:

```text
50 requests
↓
3 authorization facts
↓
1 hypothesis
↓
1 next test
```

---

# 4. Trigger Conditions

Activate when any of these appear:

* Multiple user accounts
* Roles or permissions
* Organizations / teams
* Ownership
* Object IDs
* Admin functionality
* API resources
* User-specific resources
* Sensitive actions
* Hidden functionality
* Export/download operations
* Invitations
* Sharing
* Collaboration
* Approval workflows
* Security settings
* Subscription/plan boundaries
* Tenant boundaries
* Backend requests controlled by client state

Prioritize authenticated functionality over generic public pages.

---

# 5. First Move: Build the Authorization Map

Do not immediately fuzz IDs.

First identify:

```text
Who?
↓
Can do what?
↓
To which object?
↓
Under which state?
```

Create a compact matrix:

```text
Actor       Resource       Action      Expected
User A      own invoice    READ        ALLOW
User A      User B invoice READ        DENY
User A      own invoice    DELETE      ALLOW/DENY
User A      admin config   UPDATE      DENY
Admin       User B invoice READ        ALLOW
Tenant A    Tenant B data  READ        DENY
```

Only create rows supported by observed application behavior.

Do not invent permissions.

---

# 6. Attack-Surface Mapping

Prioritize surfaces where authorization decisions are likely to depend on attacker-controlled input.

## High-value surfaces

```text
/api/*
/admin/*
/users/*
/accounts/*
/organizations/*
/teams/*
/projects/*
/files/*
/orders/*
/invoices/*
/messages/*
/exports/*
/settings/*
/permissions/*
/roles/*
/integrations/*
```

Also inspect:

```text
GraphQL
REST
WebSockets
background requests
mobile API calls
download endpoints
export endpoints
share/invite flows
internal-looking APIs
admin UI APIs
```

Do not test every endpoint equally.

Prioritize endpoints containing:

```text
object identifiers
user identifiers
tenant identifiers
role identifiers
permission identifiers
ownership identifiers
resource paths
workflow state
security-sensitive actions
```

---

# 7. High-Signal Priority Order

Use:

### P0 — Very high signal

```text
Cross-user sensitive object access
Cross-tenant access
Unauthorized admin action
Unauthorized security-setting modification
Unauthorized account ownership/control
Unauthorized financial action
Unauthorized delete/write
```

### P1 — High signal

```text
Horizontal object access
Vertical function access
Forced browsing
Hidden API functionality
Role/permission manipulation
Unauthorized export/download
Property-level authorization
Workflow-state bypass
```

### P2 — Medium signal

```text
Metadata exposure
Non-sensitive object access
UI-only restrictions
Minor object enumeration
Low-impact read access
```

### P3 — Low signal

```text
UI hiding
Predictable IDs without unauthorized access
403 vs 404 differences alone
Endpoint existence
Public resources
Client-side-only restrictions with no privileged effect
```

Do not spend expensive reasoning on P3 unless it creates a stronger hypothesis.

---

# 8. Hypothesis Format

Before every meaningful test:

```text
Hypothesis:
Why plausible:
Actor:
Resource:
Action:
Variable:
Control:
Expected denied behavior:
Expected vulnerable signal:
Next pivot:
```

Example:

```text
Hypothesis:
POST /api/invoices/{id}/download may lack object authorization.

Why:
User A owns invoice 100.
The request directly supplies invoice ID.

Variable:
invoice_id

Control:
User B session + User A invoice

Expected safe:
403/404/no sensitive file.

Vulnerable:
User B receives User A's invoice.
```

---

# 9. Differential Testing

Differential testing is the primary technique.

Prefer:

```text
A vs B
```

over:

```text
100 payloads against A
```

## Horizontal

Use:

```text
User A → own object
User A → User B object
```

Keep constant:

```text
method
endpoint
headers
body
state
```

Change only:

```text
object reference
```

Then repeat with User B where necessary.

OWASP's authorization testing methodology explicitly recommends maintaining separate sessions and comparing equivalent requests across identities.

---

# 10. Vertical Authorization Testing

Create:

```text
Low-privileged session
High-privileged session
```

Capture privileged operations.

Then test:

```text
Low privilege
+
High-privilege request
```

Prioritize:

```text
create user
delete user
change role
change permissions
view sensitive records
export data
approve transactions
change billing
change security settings
manage integrations
impersonate users
```

Do not assume the UI is the authorization boundary.

A hidden admin button is not protection.

---

# 11. Unauthenticated / Forced Browsing

For sensitive functionality:

```text
Authenticated request
↓
remove authentication
↓
request same resource
```

Then test direct access to known protected functionality.

Sources for discovery:

```text
browser traffic
JavaScript
API documentation
mobile traffic
sitemap
robots.txt
links
historical endpoints
known application routes
```

Do not brute-force huge path lists unless the application provides a strong reason.

A successful request is only interesting if the returned capability/resource should actually be protected.

OWASP WSTG treats unauthenticated, horizontal, and vertical authorization bypass as separate authorization objectives.

---

# 12. IDOR / BOLA

Treat IDOR as one **mechanism** inside BAC.

Look for object references in:

```text
path
query
JSON
form data
GraphQL variables
headers
cookies
WebSocket messages
file identifiers
download tokens
export IDs
message IDs
order IDs
user IDs
tenant IDs
UUIDs
opaque strings
```

Do not require sequential IDs.

UUIDs and opaque identifiers can still be vulnerable if obtained from another legitimate application response.

Test:

```text
own object
↓
different user's object
↓
different tenant's object
```

Test both:

```text
READ
WRITE
DELETE
EXECUTE
```

A write/delete primitive is generally more significant than a harmless metadata difference.

---

# 13. Object Reference Discovery

When an object ID is not obvious, look for it in:

```text
list responses
detail pages
search results
notifications
messages
comments
activity logs
shared links
exports
emails
WebSocket events
GraphQL responses
client-side state
mobile API responses
```

Important:

```text
identifier secrecy ≠ authorization
```

Do not stop because the identifier is random.

The actual question is whether the server enforces access to the referenced object.

OWASP explicitly states that guessed or manipulated identifiers must not grant access merely because they are valid references.

---

# 14. Object-Level Read Testing

For each object-bearing endpoint:

```text
GET own object
↓
identify object reference
↓
replace with another authorized test account's object
↓
compare
```

Strong signals:

```text
private data returned
object metadata returned
download succeeds
sensitive fields returned
state-specific information returned
```

Weak signals:

```text
same generic wrapper
same empty response
same public metadata
404/403 without leakage
```

Do not report object enumeration without unauthorized access unless the program considers enumeration independently impactful.

---

# 15. Object-Level Write Testing

Write operations deserve high priority.

Test:

```text
PUT
PATCH
POST action
DELETE
mutation
```

Example:

```text
User A modifies own object
↓
replace object ID with User B object
↓
observe whether User B's state changes
```

Validate with User B's session or another safe observable.

Do not rely solely on:

```text
200 OK
```

Confirm actual state change.

---

# 16. Function-Level Authorization

Find sensitive functionality first.

Examples:

```text
/admin/users
/admin/settings
/api/users/delete
/api/roles/update
/api/export
/api/billing/refund
/api/integrations
```

Then ask:

```text
Can low-privileged actor invoke function?
Can anonymous actor invoke function?
Can another role invoke function?
Can function be reached through another endpoint?
```

The API/UI distinction is critical:

```text
UI denies
+
API allows
=
potential BAC
```

But:

```text
UI hides
+
API also denies
=
not a finding
```

---

# 17. Method-Level Authorization

When a restricted endpoint exists, compare relevant methods only when supported by the application behavior:

```text
GET
POST
PUT
PATCH
DELETE
```

Look for:

```text
GET allowed but DELETE protected
POST protected but alternate mutation method allowed
frontend restricts one method while backend accepts another
```

Do not blindly send every HTTP method.

Only test methods that correspond to the operation or where routing behavior suggests a plausible alternate path.

PortSwigger documents method-based access-control inconsistencies as a practical BAC pattern.

---

# 18. Path / Route Authorization Discrepancies

When authorization appears to be enforced by a gateway/router, test only plausible path-normalization variants:

```text
/case-sensitive
/CAsE-sensitive
/path/
/path
/path.ext
/path;parameter
```

Also investigate application-specific routing normalization.

Goal:

```text
Frontend sees:
DENIED ROUTE

Backend sees:
PROTECTED FUNCTION
```

A discrepancy matters only if it results in unauthorized functionality.

Do not report a routing difference without an authorization boundary violation.

---

# 19. Client-Controlled Authorization State

Inspect:

```text
role
is_admin
admin
permission
permissions
scope
access
plan
tier
user_type
account_type
organization_role
```

Locations:

```text
query
body
form
cookie
local state
headers
JWT claims
GraphQL variables
```

Do not mutate everything.

First identify whether the value actually participates in an authorization decision.

Test:

```text
original value
↓
single modified value
↓
privileged behavior?
```

A client-side value changing UI state is not sufficient.

The server must grant an unauthorized capability.

PortSwigger specifically identifies parameter-based access control as a BAC pattern when server-side authorization decisions depend on user-controllable state.

---

# 20. Property-Level Authorization

Do not stop at object-level authorization.

Ask:

```text
Can I access the object?
+
Which properties should I be allowed to read/write?
```

High-value properties:

```text
role
permissions
email
verified
billing_status
owner_id
tenant_id
security settings
password-related fields
API keys
internal flags
approval state
```

Test:

```text
allowed property
+
sensitive property
```

For APIs, inspect both:

```text
response exposure
request-side modification
```

OWASP API3 specifically treats unauthorized sensitive object properties and unauthorized modification of sensitive properties as broken object property-level authorization.

---

# 21. Mass Assignment as Authorization Testing

If an update endpoint accepts structured objects:

```text
PATCH /api/user
{
  "display_name": "...",
  ...
}
```

Do not immediately fuzz hundreds of fields.

First identify security-sensitive fields already visible elsewhere:

```text
role
owner
organization
permissions
verified
status
plan
is_admin
```

Then test one field at a time.

Signal:

```text
field accepted
+
security-sensitive state changes
```

Ignore:

```text
field silently ignored
```

unless another response proves it affected authorization.

---

# 22. Multi-Tenant Authorization

Treat tenant boundaries as first-class authorization boundaries.

Model:

```text
Tenant A User
↓
Tenant A Object = expected ALLOW

Tenant A User
↓
Tenant B Object = expected DENY
```

Look for:

```text
tenant_id
organization_id
workspace_id
company_id
account_id
project_id
```

Do not assume:

```text
same role
=
same authorization
```

Tenant isolation often matters more than ordinary horizontal access.

Prioritize:

```text
cross-tenant READ
cross-tenant WRITE
cross-tenant DELETE
cross-tenant ADMIN
```

---

# 23. Context-Dependent Authorization

Some operations are authorized only in a particular state.

Map:

```text
state A → action
state B → action
```

Examples:

```text
before payment
after payment

before approval
after approval

invited
accepted

draft
published

active
suspended

owner
non-owner
```

Test whether the server validates the current state or trusts client-controlled workflow transitions.

Use:

```text
valid workflow
↓
capture request
↓
repeat at invalid state
```

A workflow bypass is BAC when a protected action becomes available outside its intended authorization/state boundary.

---

# 24. Ownership Transfer

Ownership changes deserve special attention.

Look for:

```text
owner_id
user_id
account_id
organization_id
assignee_id
recipient_id
```

Test:

```text
User A owns object
↓
attempt controlled ownership change
↓
does User B gain control?
```

Then verify:

```text
Can A still control it?
Can B control it?
Did permissions actually change?
```

Do not stop at a successful API response.

---

# 25. Sharing / Invitation Boundaries

Test:

```text
share
invite
accept
remove
revoke
transfer
collaborate
```

Important questions:

```text
Can User A modify another user's share?
Can removed users retain access?
Can an invite be accepted by the wrong actor?
Can a non-owner create privileged invitations?
Can a revoked user continue using an existing resource?
```

These flows often expose authorization state inconsistencies.

---

# 26. Delete / Destructive Operations

For destructive functionality:

```text
DELETE
archive
disable
revoke
remove
cancel
reset
```

Use the safest available test object.

Prefer:

```text
test object
+
controlled second account
```

Never perform destructive actions against unrelated real users or production data.

A successful unauthorized state change is stronger evidence than a status code.

---

# 27. Download / Export Authorization

Prioritize:

```text
/download
/export
/report
/archive
/file
/attachment
```

Test:

```text
own resource
↓
other test user's resource
```

Also check whether:

```text
page access denied
but download endpoint allowed
```

or:

```text
UI denies
but direct file endpoint allows
```

Exports often expose more data than normal object views.

---

# 28. Alternate-Endpoint Consistency

For the same resource/action, identify multiple representations:

```text
web endpoint
API endpoint
mobile endpoint
GraphQL
legacy endpoint
download endpoint
bulk endpoint
internal-looking endpoint
```

If one path enforces authorization and another performs the same sensitive action without it:

```text
same capability
+
different authorization result
=
high-value hypothesis
```

Do not duplicate identical tests across endpoints unless the implementation boundary differs.

---

# 29. GraphQL

Prioritize:

```text
query
mutation
object IDs
nested objects
field selection
aliases
bulk operations
```

Test:

```text
Can User A query User B's object?
Can a mutation modify another user's object?
Can a hidden sensitive field be selected?
Can nested authorization be bypassed?
```

Separate:

```text
object authorization
property authorization
function authorization
```

Do not treat GraphQL introspection alone as BAC.

---

# 30. WebSockets / Async Operations

When the application uses WebSockets:

```text
capture message
identify resource reference
identify actor/session
repeat with second account
```

Test:

```text
subscribe
read
update
delete
join
leave
admin action
```

Async authorization must be tested at the server operation, not just the initial UI action.

---

# 31. Token / Session Boundary Tests

Only when relevant, compare:

```text
authenticated
expired
logged-out
different user
different role
```

Check whether sensitive operations remain authorized after:

```text
logout
role downgrade
membership removal
permission revocation
tenant change
```

Preserve session state carefully.

Do not confuse authentication failure with authorization failure.

---

# 32. Differential Result Classification

Use:

```text
RESULT A = expected authorization
RESULT B = modified authorization condition
```

Classify:

### VULNERABLE

```text
Unauthorized resource/capability obtained
+
reproducible
+
security-relevant impact
```

### INTERESTING

```text
Behavior differs
+
authorization explanation unclear
```

Run a control.

### SAFE

```text
Authorization boundary consistently enforced
```

### NOISE

```text
UI-only difference
generic response
public resource
non-security metadata
```

### INCONCLUSIVE

```text
state changed
+
result cannot be attributed confidently
```

---

# 33. False-Positive Gates

Do not report:

```text
200 instead of 403
```

by itself.

Do not report:

```text
predictable ID
```

without unauthorized access.

Do not report:

```text
hidden admin endpoint
```

without successful unauthorized functionality.

Do not report:

```text
UI restriction bypass
```

if the server still rejects the action.

Do not report:

```text
same response length
```

without evidence of unauthorized data/capability.

Do not report:

```text
UUID discovered
```

unless access control is actually bypassed.

---

# 34. Validation Gate

Before finding classification:

```text
1. Reproduce.
2. Identify actor.
3. Identify protected resource/function.
4. Identify expected boundary.
5. Change one authorization variable.
6. Confirm unauthorized result.
7. Verify impact.
8. Run a control.
9. Remove benign explanations.
```

Minimum proof:

```text
Actor A
→
protected resource belonging to B
→
server accepts operation
→
B's resource/state is actually exposed or changed
```

---

# 35. Impact Escalation

Once a BAC primitive is confirmed, ask:

```text
Can READ become WRITE?
Can WRITE become DELETE?
Can horizontal become vertical?
Can object access expose credentials?
Can object access expose recovery mechanisms?
Can tenant access become tenant administration?
Can a low-privilege action affect privileged users?
```

Do not escalate blindly.

Only follow chains supported by observed relationships.

---

# 36. High-Value Chains

### Chain A

```text
Horizontal object access
+
admin-owned object
↓
vertical escalation
```

### Chain B

```text
Object read
+
sensitive security property
↓
account compromise path
```

### Chain C

```text
Object write
+
role/permission property
↓
privilege escalation
```

### Chain D

```text
Tenant isolation failure
+
tenant administration
↓
cross-tenant compromise
```

### Chain E

```text
Unauthorized export
+
sensitive dataset
↓
mass data exposure
```

Never claim the final chain impact until the required primitive is demonstrated.

---

# 37. Test Ledger

Maintain:

```text
TEST:
Actor:
Endpoint:
Resource:
Action:
Hypothesis:
Variable changed:
Control:
Result:
Confidence:
Next pivot:
```

Before testing:

```text
Have I already tested this authorization relationship?
```

Repeat only when:

```text
new actor
new resource
new action
new state
new endpoint
new evidence
```

---

# 38. One Variable at a Time

Default rule:

```text
CHANGE:
object_id

KEEP:
session
method
endpoint
body
headers
state
```

For vertical testing:

```text
CHANGE:
session/role

KEEP:
resource
action
endpoint
body
```

For tenant testing:

```text
CHANGE:
tenant/resource reference

KEEP:
actor
action
endpoint
```

If multiple variables must change, record why.

---

# 39. Failed Test → Pivot

When authorization is correctly enforced:

```text
FAILED
↓
classify
```

### If object access denied

Try:

```text
different object type
different action
related endpoint
download/export endpoint
nested object
bulk endpoint
```

### If role enforcement works

Try:

```text
client-controlled role state
alternate endpoint
workflow transition
property-level authorization
tenant boundary
```

### If API denies but UI appears vulnerable

Verify:

```text
server-side result
```

If server denies:

```text
discard UI-only branch
```

### If one endpoint is protected

Search for:

```text
legacy endpoint
mobile endpoint
GraphQL
download
bulk
async
```

### If all related paths are protected

```text
stop branch
```

Do not generate random payloads.

---

# 40. Anti-Loop Rule

If:

```text
same actor
+
same resource
+
same action
+
same hypothesis
+
same result
```

appears repeatedly:

```text
STOP
```

unless:

```text
state changed
new evidence appeared
previous result was inconclusive
```

Then pivot to a different authorization relationship.

---

# 41. Review Gates

## Gate A — Before Expensive Testing

Ask:

```text
What authorization boundary am I testing?
Why is it plausible?
What is the cheapest discriminating test?
What result would justify deeper testing?
```

If no answer:

```text
return to mapping
```

## Gate B — Interesting Behavior

```text
observe
↓
reproduce
↓
control
↓
remove benign explanation
↓
validate
```

## Gate C — Before Reporting

```text
unauthorized?
reproducible?
security boundary crossed?
impact demonstrated?
false positive excluded?
evidence sufficient?
```

## Gate D — Low Signal

If:

```text
3+ related tests
+
no new signal
+
no stronger hypothesis
```

stop that branch.

---

# 42. Context Compression

When context grows:

```text
Raw requests
↓
deduplicate
↓
extract authorization facts
↓
retain strongest evidence
↓
discard equivalent requests
```

Compress to:

```text
User A owns object 101.
User B owns object 202.
GET /objects/{id} enforces ownership.
PATCH /objects/{id} returned 200 for B → A object but state did not change.
Export endpoint not tested.
```

This is preferable to retaining dozens of requests.

Never discard:

```text
exact vulnerable request
exact response evidence
identity/session relationship
resource identifier
state required for reproduction
```

---

# 43. Tool Strategy

Use tools to answer specific questions.

Preferred flow:

```text
Burp/browser traffic
↓
extract authorization-bearing requests
↓
filter object/function candidates
↓
controlled replay
↓
compare
↓
record ledger
```

Useful automation:

```text
request diffing
session comparison
endpoint clustering
parameter extraction
response similarity
authorization matrices
```

Avoid:

```text
millions of requests
+
no hypothesis
+
full output in context
```

Automation should reduce repetitive comparison, not replace authorization reasoning.

---

# 44. Stop / Continue

## Continue when

```text
cross-user signal exists
cross-tenant signal exists
privileged function is reachable
sensitive property is exposed
state changes unexpectedly
new authorization boundary appears
strong chain exists
```

## Stop branch when

```text
authorization consistently enforced
same hypothesis repeatedly fails
impact is negligible
behavior is benign
tests become equivalent
evidence is already sufficient
```

## Pivot when

```text
same feature has another authorization surface
another actor exists
another resource type exists
another action exists
another state exists
```

---

# 45. Finding Record

For confirmed findings:

```text
Title:
Target:
Actor:
Role:
Tenant:
Endpoint:
Method:
Resource:
Action:
Expected authorization:
Actual authorization:
Original request:
Modified request:
Control:
Observed result:
Security impact:
Reproduction:
Evidence:
Confidence:
Potential chain:
```

Keep evidence minimal and reproducible.

---

# 46. Final Decision Tree

```text
START
 |
 v
Identify actor/resource/action
 |
 v
Is there an authorization boundary?
 |                     |
 NO                    YES
 |                     |
 map more              v
                  Find control request
                        |
                        v
                  Identify variable
                        |
                        v
                 Change ONE variable
                        |
                        v
                  Compare result
                  /      |       \
               DENIED  DIFFERENT  ALLOWED
                 |        |          |
                stop   investigate  reproduce
                           |          |
                           v          v
                         control   impact?
                                    |
                              +-----+-----+
                              |           |
                             NO          YES
                              |           |
                            discard    validate
                                          |
                                          v
                                      assess chain
                                          |
                                          v
                                       report/stop
```

---

# 47. Agent Behavior Rules

The agent MUST:

* Think in authorization relationships.
* Prefer two-account differential testing.
* Change one meaningful variable at a time.
* Test server-side behavior, not UI appearance.
* Treat IDOR as a mechanism, not the whole BAC category.
* Prioritize cross-user, cross-tenant, and privilege-boundary failures.
* Validate actual state/data changes.
* Keep a compact test ledger.
* Stop duplicate hypotheses.
* Escalate only after evidence improves.
* Preserve exact reproduction evidence.
* Never manufacture impact.
* Use safe test objects and authorized accounts.
* Stop when the branch is exhausted.

The agent MUST NOT:

* blindly fuzz every identifier;
* equate `200 OK` with authorization bypass;
* equate hidden UI elements with security;
* report predictable IDs without unauthorized access;
* repeat identical requests;
* dump raw tool output into context;
* claim privilege escalation without demonstrating privileged capability;
* perform unnecessary destructive actions.

---

# 48. Progressive Disclosure

Load:

```text
references/techniques.md
```

when:

```text
object/function/property/state authorization
```

requires deeper testing.

Load:

```text
references/advanced.md
```

only when:

```text
normal authorization tests are exhausted
+
a strong implementation-specific bypass hypothesis exists
```

Do not load advanced material for weak signals.

---

# 49. Completion Criteria

A BAC branch is complete when:

```text
authorization model understood
+
high-value actor/resource/action combinations tested
+
strong differentials validated
+
false positives eliminated
+
plausible chains evaluated
+
evidence sufficient
+
remaining tests are low-value or equivalent
```

The goal is not to test every endpoint.

The goal is to discover the highest-value authorization failures with the fewest meaningful tests.
