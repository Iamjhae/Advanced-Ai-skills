---

name: burp-match-replace-hunting
description: Use this skill when performing authorized web application or bug bounty testing with Burp Suite Match and Replace. Apply it whenever request or response rewriting, parameter manipulation, header injection/removal, authentication-state comparison, access-control testing, method manipulation, WebSocket message rewriting, or automated differential testing can expose security-relevant behavior. Treat Match and Replace as a hypothesis-driven testing primitive rather than a vulnerability by itself, and prioritize high-signal transformations that reveal authorization, trust-boundary, validation, routing, caching, or business-logic weaknesses.
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Burp Match & Replace Hunting

## 1. Purpose

Use Burp Match & Replace to turn observed application behavior into controlled, repeatable differential tests.

Core loop:

```text
Observe
→ Identify security-relevant variable
→ Define hypothesis
→ Rewrite one variable
→ Compare behavior
→ Classify result
→ Pivot
→ Validate
```

Match & Replace is **not itself a vulnerability**. A finding exists only when a transformation produces security-relevant behavior with a credible impact.

Use only against assets and accounts you are authorized to test.

---

## 2. Operating Rules

### Scope first

Before enabling broad rules:

* Confirm target scope.
* Restrict rules to in-scope traffic where possible.
* Prefer temporary/project-local rules.
* Disable rules after the test.
* Avoid destructive transformations on state-changing endpoints unless necessary.
* Preserve the original request for comparison.

### One variable at a time

When investigating a hypothesis, change one security-relevant property while keeping the rest constant.

```text
Baseline:
session=A
endpoint=/api/resource
method=GET
object_id=123
role=user

Test:
session=A
endpoint=/api/resource
method=GET
object_id=124
role=user
```

Do not simultaneously change identity, method, headers, body, and object ID unless the hypothesis specifically requires a combination.

### Treat automation as an experiment

Every rule should have:

```text
Purpose
Match condition
Replacement
Expected signal
Validation step
Stop condition
```

Delete rules that produce no useful signal.

---

## 3. Hunting Workflow

### Phase 1 — Establish baseline

Capture representative requests for:

* Authentication
* Account/profile operations
* Object reads
* Object updates
* Administrative functions
* File operations
* API requests
* Sensitive workflows
* WebSocket traffic when relevant

Record:

* Method
* Path
* Query parameters
* Body parameters
* Headers
* Cookies
* Tokens
* Object identifiers
* Role/tenant indicators
* Response status
* Response length
* Important response fields
* Side effects

Do not start with global replacement.

---

### Phase 2 — Map rewriteable security variables

Prioritize variables that appear to control trust or authorization.

#### High-signal request locations

| Location    | Look for                                               |
| ----------- | ------------------------------------------------------ |
| Headers     | IP, role, tenant, auth state, origin, forwarded values |
| Query       | IDs, flags, role values, feature switches              |
| JSON body   | ownership, permissions, workflow state                 |
| Form body   | account state, privilege fields                        |
| Cookies     | role/state/session indicators                          |
| JWT claims  | role, tenant, subject, permissions                     |
| Path        | object IDs, admin paths, API versions                  |
| HTTP method | GET/POST/PUT/PATCH/DELETE/OPTIONS                      |
| WebSocket   | event names, IDs, client-controlled state              |

---

## 4. Priority Order

Start with transformations that are cheap and highly diagnostic.

### Priority 1 — Authorization and trust boundaries

Test:

* Role-like parameters
* Tenant identifiers
* User/object identifiers
* IP-related headers
* Authentication-state indicators
* Privilege-related fields
* Ownership fields

### Priority 2 — Request semantics

Test:

* HTTP method changes
* Duplicate parameters
* Parameter value substitutions
* Header insertion/removal
* Content-type changes when relevant
* Alternate representations

### Priority 3 — Validation boundaries

Test:

* Boolean/state transitions
* Null/empty values
* Unexpected enum values
* Duplicate values
* Encoded variants
* Case variants

### Priority 4 — Response transformation

Use response rewriting primarily for:

* Detecting client-side security controls
* Revealing hidden UI behavior
* Testing whether a control is enforced server-side
* Isolating client-side assumptions

Do not mistake modified client-visible content for proof of a server-side vulnerability.

### Priority 5 — Advanced scripting

Use Bambda/script-based Match & Replace only when static rules cannot express the hypothesis.

Prefer a simple rule over a script whenever both provide equivalent signal.

---

## 5. Header Manipulation

### Hypothesis

The application or intermediary may trust a client-controlled header.

Search observed traffic for:

* Forwarded/IP-related headers
* Role indicators
* Internal-network indicators
* Authentication-related headers
* Tenant headers
* Custom application headers

Do not assume a header is trusted merely because it exists.

### Test

1. Capture baseline.
2. Identify the header.
3. Add, replace, or remove only that header.
4. Repeat the same request.
5. Compare authorization and response behavior.
6. Validate the behavior with the minimum-impact request.

### Signal

Strong signals include:

* Previously inaccessible functionality becomes reachable.
* Server-side authorization changes.
* A role/identity changes.
* Internal-only functionality becomes available.
* Response content changes in a security-relevant way.

A status-code change alone is insufficient.

---

## 6. Parameter Manipulation

Search application traffic for values representing:

```text
role
admin
permission
owner
user
tenant
account
approved
verified
active
internal
debug
read
write
enabled
disabled
```

These are indicators, not automatic vulnerabilities.

### Test pattern

```text
baseline value
→ controlled replacement
→ compare response
→ determine whether server behavior changed
```

Example reasoning:

```text
is_admin=false
        ↓
replace with true
        ↓
response changes?
        ↓
YES
        ↓
Does server-side privilege actually change?
        ↓
YES
        ↓
Validate impact
```

If only the UI changes, classify it as client-side behavior until server-side impact is demonstrated.

PortSwigger specifically documents using Match & Replace to test parameter-based access-control behavior by replacing observed parameter values and remapping the application.

---

## 7. Cross-User / Cross-Tenant Testing

When multiple authorized accounts are available, Match & Replace can automate controlled identity-variable substitution.

Prioritize:

* User IDs
* Tenant IDs
* Organization IDs
* Resource IDs
* Owner IDs
* Account IDs

Use differential testing:

```text
User A baseline
        ↓
same request + User B-controlled identifier
        ↓
compare authorization
```

Then:

```text
Tenant A baseline
        ↓
same request + Tenant B identifier
        ↓
compare authorization
```

A changed response is only a lead.

Validate that the unauthorized principal can actually:

* Read protected data
* Modify protected data
* Delete protected data
* Perform privileged actions
* Trigger a meaningful cross-boundary side effect

Stop after sufficient evidence; do not enumerate unrelated users or tenants.

---

## 8. Method Manipulation

Test alternate HTTP methods when endpoint behavior suggests method-dependent authorization or routing.

Useful comparisons:

```text
GET
POST
PUT
PATCH
DELETE
OPTIONS
HEAD
```

Also consider an invalid method when mapping server behavior.

Observe:

* Status
* Response body
* Headers
* Authorization behavior
* Method-specific routing
* Side effects

Do not rely on status code alone. PortSwigger explicitly recommends analyzing the complete response when identifying supported methods.

### High-value hypothesis

```text
POST /admin/action
authorization enforced

        ↓

GET /admin/action

        ↓

Does routing reach the same handler?
        ↓
Does authorization still apply?
```

Only report when the alternate method creates a genuine security impact.

---

## 9. Duplicate Parameter Testing

When an application accepts parameters from multiple layers, test duplicate names.

Examples:

```text
role=user&role=admin
id=123&id=124
tenant=A&tenant=B
```

Compare:

* First-value behavior
* Last-value behavior
* Array interpretation
* Framework-specific parsing
* Proxy/backend disagreement

This is useful for detecting HTTP Parameter Pollution and parser inconsistencies. OWASP notes that duplicate parameters can be interpreted differently and may affect validation or internal values.

### Pivot

If duplicate parameters produce inconsistent behavior:

```text
proxy interpretation
→ application interpretation
→ validation layer
→ authorization layer
```

Test only the boundary relevant to the observed behavior.

---

## 10. Encoding / Representation Variants

When a security control appears dependent on exact string representation, test controlled representation changes.

Examples:

```text
admin
ADMIN
Admin
encoded value
double-encoded value
whitespace variant
alternate serialization
```

Do not blindly generate thousands of variants.

Use representation changes only when:

* A parser boundary is suspected.
* A validation check appears string-based.
* Different layers may normalize values differently.

---

## 11. Request vs Response Rewriting

### Request rewriting

Use for:

* Authorization hypotheses
* Header trust
* Parameter manipulation
* Method behavior
* Session-state comparisons
* Parser differences

### Response rewriting

Use for:

* Testing client-side controls
* Revealing hidden client functionality
* Checking whether UI restrictions are enforced by the server
* Investigating client/server trust boundaries

Rule:

```text
Response modification
≠
server-side vulnerability
```

If changing the response enables a privileged UI action, replay the resulting request without response modification and determine whether the server independently authorizes it.

---

## 12. WebSocket Match & Replace

For WebSocket applications, inspect:

* Message direction
* Event/action names
* Object IDs
* User IDs
* Tenant IDs
* Permission-like fields
* State transitions

Use:

```text
client → server baseline
        ↓
modify one security-relevant field
        ↓
server response / state change
```

Prioritize server-side authorization and state validation.

Avoid modifying every WebSocket message globally because this can destroy application state and create misleading results.

---

## 13. Decision Tree

```text
Do you have a reproducible baseline?
        │
        ├─ NO → capture normal traffic first
        │
        └─ YES
             ↓
Is there a security-relevant variable?
             │
             ├─ NO → map more traffic
             │
             └─ YES
                  ↓
Can one variable be rewritten safely?
                  │
                  ├─ NO → use Repeater/manual test
                  │
                  └─ YES
                       ↓
Apply controlled replacement
                       ↓
Did server behavior change?
        │
        ├─ NO → classify failure → pivot
        │
        └─ YES
             ↓
Is the change security-relevant?
        │
        ├─ NO → likely noise
        │
        └─ YES
             ↓
Can the behavior be reproduced manually?
        │
        ├─ NO → investigate rule/parser effects
        │
        └─ YES
             ↓
Can impact be demonstrated safely?
        │
        ├─ NO → informational/weak lead
        │
        └─ YES
             ↓
Validate + collect evidence
```

---

## 14. Failure Classification

Never treat a failed replacement as the end of the investigation.

Classify it first.

### A — Rule failure

The replacement did not match.

Action:

* Inspect exact message.
* Check regex.
* Check encoding.
* Check rule ordering.
* Test the rule against the actual request.

### B — Application normalization

The server normalized the modified value.

Action:

* Determine normalization behavior.
* Test the next relevant representation.
* Avoid blind variant spraying.

### C — Security control rejected it

Action:

* Identify which layer rejected it.
* Determine whether the rejection is expected.
* Look for a different trust boundary only if evidence supports it.

### D — Wrong hypothesis

Action:

* Remove the rule.
* Preserve observations.
* Generate a new hypothesis from the actual response.

### E — Interesting differential

Action:

* Reproduce manually.
* Isolate the variable.
* Validate impact.

---

## 15. Rule Ordering

Match & Replace rules execute sequentially.

Therefore:

```text
Rule A
↓
modified request
↓
Rule B
↓
modified request
↓
server
```

Unexpected behavior can result from rule interaction rather than the target.

When debugging:

1. Disable all unrelated rules.
2. Enable one rule.
3. Test.
4. Add the next rule.
5. Retest.
6. Record ordering dependencies.

PortSwigger documents sequential processing and configurable rule ordering.

---

## 16. Regex Discipline

Use regex when the target value is dynamic.

Prefer:

```text
narrow pattern
+
explicit replacement
```

over:

```text
.* 
```

Avoid broad replacements that modify unrelated parameters, headers, JSON fields, or endpoints.

Before enabling a regex rule:

```text
Original request
→ Test rule
→ Inspect modified request
→ Confirm only intended region changed
→ Enable
```

Burp provides a built-in test function for validating the resulting modified message.

---

## 17. Script / Bambda Usage

Escalate to scripting when:

* Matching requires contextual logic.
* Replacement depends on dynamic request properties.
* Multiple related fields must be handled conditionally.
* Static regex would be error-prone.
* Response/request classification is required.

Keep scripts:

* Deterministic
* Narrow
* Fast
* Scope-aware
* Easy to disable

Avoid expensive logic on every request.

PortSwigger supports Java-based Match & Replace scripts and warns that slow/resource-intensive scripts can affect Burp performance.

---

## 18. Differential Evidence

For every promising result preserve:

```text
Baseline request
Modified request
Baseline response
Modified response
Exact changed variable
Observed behavioral difference
Security consequence
Reproduction condition
```

Prefer evidence that demonstrates:

```text
unauthorized state
→ security control changed
→ protected action/data became available
```

rather than:

```text
HTTP 200
```

---

## 19. False Positive Control

Common false positives:

### UI-only change

A rewritten response exposes an admin button.

Test:

```text
Click/action request without response rewrite
```

If rejected server-side, it is not an access-control bypass.

### Generic status change

A replacement changes `403` to `400`.

This is not evidence of impact.

### Error-message difference

Different parsing errors can look interesting but have no security consequence.

### Cache artifacts

A response change may be caused by caching rather than authorization.

Recheck with controlled cache conditions.

### Session contamination

A global rule may modify unrelated requests and alter authentication state.

Reproduce with a clean session.

### Rule interaction

Another active Match & Replace rule may be responsible.

Disable unrelated rules and retest.

---

## 20. Chaining

Consider chaining only after an individual behavior is confirmed.

Examples:

```text
trusted-header manipulation
        +
weak authorization
        ↓
privileged functionality
```

```text
parameter-controlled role
        +
missing server-side authorization
        ↓
privilege escalation
```

```text
cross-tenant identifier substitution
        +
weak object authorization
        ↓
cross-tenant access
```

```text
client-side control bypass
        +
server-side missing validation
        ↓
real privilege/action bypass
```

Do not inflate severity merely because multiple weak signals exist.

---

## 21. Stop / Continue Rules

### Stop when

* The transformation produces no meaningful differential after reasonable hypothesis-driven variants.
* The behavior is clearly client-side only.
* Server-side authorization remains intact.
* The result cannot be reproduced without unrelated rule interactions.
* Further testing would require unnecessary destructive actions.

### Continue when

* A security-relevant response field changes.
* Authorization behavior changes.
* A previously inaccessible function becomes reachable.
* Cross-user/tenant behavior differs.
* Different processing layers appear inconsistent.
* A parser or normalization boundary is identified.
* A confirmed weakness suggests a closely related attack surface.

---

## 22. Minimal Hunting Checklist

```text
[ ] Establish baseline
[ ] Identify security-relevant variables
[ ] Test headers
[ ] Test role/permission parameters
[ ] Test object/tenant identifiers
[ ] Compare authorized identities
[ ] Test relevant HTTP methods
[ ] Test duplicate parameters when parser ambiguity exists
[ ] Test representation changes when normalization is suspected
[ ] Test WebSocket messages when applicable
[ ] Separate request effects from response-only effects
[ ] Validate promising differentials manually
[ ] Disable unrelated rules
[ ] Preserve baseline/modified evidence
[ ] Stop when signal is exhausted
```

---

## 23. References

Load `references/techniques.md` when deeper Match & Replace patterns are needed.

Load `references/advanced.md` when dealing with scripting, complex rule interactions, parser boundaries, or advanced differential testing.

Do not load reference material when the core workflow is sufficient.



# Match & Replace Hunting Techniques

## 1. Header Trust Testing

### Hypothesis

A backend, proxy, CDN, or application component trusts a client-controlled header.

### Procedure

```text
observe header
→ establish baseline
→ add/replace header
→ compare authorization/routing
→ reproduce manually
```

Prioritize headers that plausibly represent:

* Client IP
* Forwarded identity
* Internal network location
* Role
* Tenant
* Authentication state

Do not assume standard header names are universally trusted.

---

## 2. Parameter-Controlled Authorization

Look for parameters whose values appear to influence:

* Role
* Permission
* Account state
* Ownership
* Tenant
* Workflow stage

Test one value at a time.

```text
false → true
user → admin
owner → other
tenant=A → tenant=B
```

The important observation is **server-side behavior**, not merely changed HTML.

PortSwigger documents this exact Match & Replace workflow for parameter-based access-control testing.

---

## 3. Object Identifier Substitution

Candidate identifiers:

```text
user_id
account_id
organization_id
tenant_id
project_id
document_id
invoice_id
file_id
message_id
```

Use authorized accounts.

```text
A owns object 123
↓
A requests 123
↓
A requests B-owned 124
```

Validate whether the server returns or modifies B's resource.

---

## 4. Role / State Fields

Potential fields:

```text
role
role_id
is_admin
is_staff
approved
verified
active
status
permission
permissions
access
```

These are indicators only.

A changed field is not sufficient unless the server trusts it in a security-sensitive operation.

OWASP identifies permission-related and process-dependent properties as important mass-assignment candidates.

---

## 5. Method Differential

Use when the same endpoint appears to expose multiple methods.

```text
GET
POST
PUT
PATCH
DELETE
OPTIONS
HEAD
```

Compare:

* Authentication
* Authorization
* Routing
* Response body
* Side effects

Do not assume method changes are automatically bypasses.

---

## 6. Duplicate Parameters

Useful when there may be disagreement between:

```text
WAF
proxy
framework
application
authorization middleware
```

Example:

```text
role=user&role=admin
```

Observe which value each layer appears to consume.

OWASP notes that HTTP Parameter Pollution can produce unexpected interpretation and potentially affect validation or internal values.

---

## 7. Request/Response Boundary

Use request rewriting to test server behavior.

Use response rewriting to test client assumptions.

Always separate:

```text
client-visible behavior
```

from:

```text
server-authorized behavior
```

A response modification that unlocks a button is only a lead until the resulting request succeeds without the modification.

---

## 8. Rule Debugging

If expected replacement does not occur:

```text
1. Inspect exact message
2. Confirm match type
3. Confirm regex/literal mode
4. Check encoding
5. Check rule order
6. Disable unrelated rules
7. Test using Burp's rule tester
```

Burp provides a built-in mechanism to inspect the original and automatically modified request/response.

---

## 9. High-Signal Observation Matrix

| Observation                                   | Likely next step                               |
| --------------------------------------------- | ---------------------------------------------- |
| Role change affects only UI                   | Replay action without rewrite                  |
| Object ID returns another user's data         | Confirm cross-user authorization               |
| Tenant ID changes data scope                  | Confirm cross-tenant boundary                  |
| Header changes admin access                   | Reproduce manually and identify trust boundary |
| Method changes authorization                  | Compare all relevant methods                   |
| Duplicate parameter changes security decision | Identify parser disagreement                   |
| Encoding changes validation                   | Test normalization boundary                    |
| Response-only change                          | Determine whether server trusts client state   |
| No response change                            | Remove rule and generate another hypothesis    |
| Rule unexpectedly affects many requests       | Narrow scope/pattern                           |

---

## 10. Evidence Standard

A strong result should contain:

```text
baseline
+
single-variable modification
+
observable differential
+
security-relevant consequence
+
manual reproduction
```

Do not report:

```text
interesting header
```

as a vulnerability.

Report:

```text
client-controlled header
→ authorization decision changes
→ protected resource/action becomes available
```

when that chain is reproducible.

# Advanced Match & Replace

## 1. Script Escalation

Use scripts only when settings-mode rules cannot express the hypothesis cleanly.

Good candidates:

* Context-sensitive matching
* Dynamic replacement
* Conditional header modification
* Request classification
* Response classification
* Complex message transformations

Avoid scripts for simple literal or regex replacements.

PortSwigger's current implementation supports Java-based HTTP Match & Replace scripts through the Montoya API and provides reusable Bambda scripts.

---

## 2. Dynamic Replacement

Conceptually:

```text
if request matches target condition:
    modify only intended security variable
else:
    leave request unchanged
```

Keep conditions narrow.

A global script that rewrites every request creates poor experimental validity and can damage application state.

---

## 3. Rule Interaction Analysis

When multiple rules are active:

```text
Request
 ↓
Rule 1
 ↓
Rule 2
 ↓
Rule 3
 ↓
Target
```

A finding may disappear when one rule is disabled.

Therefore:

```text
Interesting result
→ disable all unrelated rules
→ reproduce with one rule
→ reproduce manually
```

This distinguishes a target behavior from a local Burp transformation artifact.

---

## 4. Parser Boundary Hunting

Use Match & Replace when there is evidence that two layers may interpret the same request differently.

Potential boundaries:

```text
CDN → origin
proxy → application
WAF → application
JSON parser → business logic
query parser → framework
HTTP parser → router
WebSocket gateway → backend
```

Do not test parser tricks blindly.

First establish evidence of inconsistent interpretation, then modify the smallest relevant variable.

---

## 5. Client-Side Security Control Detection

Workflow:

```text
observe UI restriction
        ↓
identify corresponding request
        ↓
remove/alter client-side control
        ↓
send request directly
        ↓
server accepts?
```

If the server rejects the request, classify the UI bypass as non-server-side unless another impact exists.

---

## 6. Authentication-State Rewriting

Potential signals:

* Cookie presence
* Authentication headers
* State parameters
* Token-bearing headers
* Session indicators

Use carefully.

Never treat removal of an authentication artifact as evidence of bypass until the server grants access without valid authentication.

---

## 7. WebSocket Advanced Testing

Map:

```text
connection
→ authentication
→ message types
→ object identifiers
→ state transitions
→ server responses
```

Then alter one field.

High-value candidates:

```text
user_id
tenant_id
resource_id
action
role
state
```

Validate server-side authorization and state handling.

---

## 8. Performance Discipline

Avoid:

* Heavy scripts on every request
* Broad regexes
* Large response rewrites
* Rules applied to irrelevant hosts
* Permanent global transformations

Prefer:

```text
narrow target
+
minimal transformation
+
short test window
+
disable after use
```

PortSwigger warns that slow or resource-intensive Match & Replace scripts can negatively affect Burp performance.

---

## 9. Advanced Pivot Tree

```text
Replacement changes response
        ↓
Is change security relevant?
        │
        ├─ NO → discard
        │
        └─ YES
             ↓
Does server-side behavior change?
        │
        ├─ NO → client/parser artifact
        │
        └─ YES
             ↓
Is identity/authorization involved?
        │
        ├─ YES → cross-user/role/tenant validation
        │
        └─ NO
             ↓
Is routing/method involved?
        │
        ├─ YES → method differential
        │
        └─ NO
             ↓
Is input interpretation involved?
        │
        ├─ YES → parser/normalization testing
        │
        └─ NO → isolate new hypothesis
```

---

## 10. Final Validation

Before treating a result as a finding:

```text
[ ] Scope confirmed
[ ] Baseline preserved
[ ] Single-variable change isolated
[ ] Rule behavior verified
[ ] Unrelated rules disabled
[ ] Manual reproduction completed
[ ] Server-side impact demonstrated
[ ] Authorization/security boundary identified
[ ] Minimal-impact evidence collected
[ ] No client-only artifact mistaken for vulnerability
```
