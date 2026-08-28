---
name: authentication-bypass
description: Hunts for authentication-bypass vulnerabilities caused by skipped verification, flawed authentication state machines, identity-binding failures, incomplete authentication, weak MFA or recovery logic, alternate authentication paths, and inconsistent enforcement. Use when testing whether protected functionality or an authenticated session can be obtained without satisfying the application's intended authentication requirements.
---

# Authentication Bypass Hunting Skill

## 1. Purpose

Find cases where an attacker can obtain authenticated access without legitimately satisfying the required authentication controls.

Primary objective:

> Determine whether the application can be made to treat an unauthenticated or incompletely authenticated actor as authenticated.

Prioritize:

- authentication-state confusion;
- skipped authentication steps;
- direct access to protected functionality;
- MFA bypass;
- identity-binding flaws;
- password-reset/recovery bypass;
- alternate authentication endpoints;
- API/mobile/legacy authentication inconsistencies;
- token/session authentication flaws;
- OAuth/OIDC authentication logic;
- race conditions around authentication transitions.

Do not confuse:

- Authentication bypass → becoming authenticated without proper proof.
- Authorization bypass → authenticated user accessing something they should not.
- Session management flaw → incorrect creation, persistence, rotation, or invalidation of authentication state.

If one issue chains into another, preserve the root cause.

---

# 2. Trigger Conditions

Activate when the target contains:

- login;
- MFA/2FA;
- password reset;
- account recovery;
- email/phone verification;
- magic links;
- OAuth/OIDC;
- SSO;
- API authentication;
- JWT/session exchange;
- step-up authentication;
- reauthentication;
- trusted-device functionality;
- invitation-based login;
- passwordless authentication;
- mobile authentication;
- legacy authentication endpoints.

---

# 3. Core Hunting Workflow

Use:

```text
Understand Authentication
        ↓
Map Authentication States
        ↓
Map Protected Functionality
        ↓
Identify Verification Gates
        ↓
Identify Identity-Bearing Inputs
        ↓
Test Direct Access
        ↓
Test State Transitions
        ↓
Test Identity Binding
        ↓
Test Alternate Paths
        ↓
Test MFA / Recovery
        ↓
Compare Results
        ↓
Pivot From Anomalies
        ↓
Validate Authentication Impact
        ↓
Check Chaining
        ↓
Document Evidence

Do not begin with blind payload spraying.

First understand what the application believes proves identity.

4. Authentication Model

Build a minimal state model.

Example:

UNAUTHENTICATED
      ↓
CREDENTIALS_ACCEPTED
      ↓
MFA_REQUIRED
      ↓
MFA_VERIFIED
      ↓
AUTHENTICATED

For each state identify:

session cookie;
token;
server-side transaction;
challenge ID;
identity;
verification flags;
redirect;
accessible endpoints.

Ask:

What creates the state?
What advances the state?
What proves the previous state occurred?
What identity is attached to the state?
Can the client modify any of this?
Can a later state be reached directly?
5. Build the Attack-Surface Map

Find:

/login
/signin
/auth
/session
/token
/verify
/mfa
/2fa
/otp
/reset
/recover
/forgot-password
/change-password
/callback
/oauth/*
/sso/*

Also inspect:

REST APIs;
GraphQL;
mobile endpoints;
AJAX/fetch requests;
legacy endpoints;
alternate subdomains;
internal-looking routes exposed externally;
passwordless flows;
invitation flows;
trusted-device flows.

Do not assume the visible login page is the complete authentication mechanism.

OWASP specifically notes that authentication can sometimes be bypassed by skipping the login page and directly requesting internal functionality, or by manipulating requests so the application incorrectly believes authentication occurred.

6. Establish Controlled Accounts

Prefer:

Account A = attacker-controlled
Account B = second controlled identity

Record:

cookies;
access tokens;
refresh tokens;
session IDs;
MFA state;
recovery state;
OAuth identities;
device identifiers.

Use controlled accounts whenever possible.

7. P0 — Direct Authentication Bypass

Before complex testing, determine whether protected functionality is actually protected.

Map authenticated-only endpoints from:

normal browsing;
Burp history;
JavaScript;
API documentation;
mobile traffic;
predictable application routes.

Then test them without an authenticated session.

Compare:

Authenticated request
vs
Unauthenticated request

Check:

HTTP status;
redirect;
response body;
cookies;
token issuance;
actual functionality;
server-side state.

A 200 OK alone is not proof.

The question is:

Did the unauthenticated request obtain functionality that requires authentication?

8. P0 — Authentication State Skipping

For every multi-step login flow:

Step 1
↓
Step 2
↓
Step 3
↓
Authenticated

Test:

Step 1
↓
Skip Step 2
↓
Request Step 3/protected endpoint

Also test:

Step 1
→ direct protected endpoint

Step 1
→ refresh

Step 1
→ alternate endpoint

Step 1
→ API endpoint

Step 1
→ authenticated UI route

High-signal result:

Incomplete authentication
+
Protected functionality accessible
=
Potential authentication bypass

PortSwigger demonstrates this class through authentication flows where reaching an authenticated page before completing required verification is sufficient to bypass the intended control.

9. Authentication State Machine Testing

For each transition:

A → B

test whether:

A → C
A → protected endpoint
A → B without required proof
A → B with modified identity

Specifically test:

skip;
reorder;
replay;
duplicate;
refresh;
direct navigation;
stale state;
alternate session;
alternate endpoint.

PortSwigger's flawed-state-machine example demonstrates how skipping an intermediate role-selection state caused unintended privileged authentication.

10. Identity-Binding Testing

This is a critical branch.

Suppose:

Login(A)
↓
Challenge(X)
↓
Verify(X)
↓
Session(A)

Determine what actually binds:

Challenge X
→ Account A

Then test only one identity-bearing variable at a time.

Conceptually:

Challenge(X)
Identity=A
Factor=A

versus:

Challenge(X)
Identity=B
Factor=A

Ask:

Does the server derive identity from trusted state, or accept a client-controlled identity value?

Potentially dangerous inputs:

username;
email;
user ID;
account ID;
tenant ID;
challenge ID;
transaction ID;
account cookie;
hidden form field;
JSON field;
JWT claim;
OAuth subject;
session parameter.

Do not report the presence of such fields.

Report only when changing them crosses an authentication boundary.

11. One-Variable Rule

Whenever practical:

Change:
identity

Keep constant:
session
endpoint
method
headers
challenge
factor
body

Then:

Observe
↓
Classify
↓
Change next variable

This prevents false attribution.

12. P0 — Partial Authentication

Identify sessions that exist between:

Unauthenticated

and:

Fully authenticated

Examples:

credentials accepted
MFA pending
email verification pending
password reset pending
OAuth callback pending
device verification pending
step-up pending

For every partial state ask:

Can it access protected endpoints?
Can it create an authenticated session?
Can it perform sensitive actions?
Can it call APIs?
Can it access another identity?
Can it become fully authenticated without completing the missing step?
13. P0 — MFA Bypass

When MFA exists, model:

Password accepted
↓
MFA required
↓
MFA verified
↓
Authenticated

Test:

Direct access

Can protected functionality be accessed after password validation but before MFA?

Alternate route

Can another endpoint perform the same action without MFA?

Identity substitution

Can the MFA transaction be associated with another identity?

Challenge substitution

Can challenge/transaction identifiers be replaced?

Session substitution

Can a different browser/session complete the flow?

Replay

Can a completed challenge be reused?

Recovery downgrade

Does "lost MFA" provide a weaker authentication path?

Verification logic

Can the application accept:

MFA_REQUIRED

as equivalent to:

MFA_VERIFIED

PortSwigger documents both simple MFA bypasses and flaws where the second factor is not correctly bound to the authenticated identity.

14. MFA Brute-Force as a Secondary Path

Only consider code guessing when:

a valid first authentication factor is already available;
the program permits the activity;
rate limiting can be evaluated safely;
there is evidence that verification protection is weak.

Determine:

attempt limit
session invalidation
account lock
code expiration
challenge rotation
per-account controls
per-IP controls

Do not blindly spray production accounts.

PortSwigger identifies weak protection around short MFA codes as a distinct authentication weakness.

15. Password Reset / Recovery Bypass

Treat recovery as an authentication mechanism.

Model:

Request reset
↓
Identity selection
↓
Token/challenge
↓
Verification
↓
Password change
↓
Authenticated session

Test:

identity binding;
token binding;
transaction binding;
step skipping;
token reuse;
stale tokens;
session substitution;
alternate reset endpoints;
verification-state manipulation;
weaker recovery mechanisms;
whether reset automatically authenticates the wrong identity.

Critical question:

Can an attacker complete recovery for an account without possessing the required proof of control?

PortSwigger explicitly treats password reset/change functionality as part of the authentication attack surface.

16. Password Change as an Authentication Boundary

Check whether sensitive password changes require:

current password;
valid authenticated session;
reauthentication;
MFA/step-up when appropriate.

Investigate:

authenticated session
+
missing reauthentication
+
identity manipulation

Do not automatically classify missing reauthentication as bypass.

Determine whether an attacker can actually cross an authentication boundary.

17. P1 — Alternate Authentication Paths

Compare:

Web
API
Mobile
GraphQL
Legacy
OAuth
SSO
Passwordless
Recovery
Magic Link
Invitation

Look for:

Strong authentication path
        ↓
Weak alternate path
        ↓
Same account/session

High-value pattern:

One authentication mechanism correctly enforces a requirement that another mechanism omits.

18. API Authentication Bypass

Compare authenticated and unauthenticated API requests.

Inspect:

Authorization
Cookie
Bearer token
session ID
API key
client ID
identity fields

Test whether:

authentication is missing;
authentication is checked only in the UI;
one API version lacks middleware;
GraphQL differs from REST;
mobile endpoints have weaker verification;
legacy endpoints accept incomplete authentication.

Do not treat an API returning metadata as bypass unless the data/functionality requires authentication.

19. JWT / Token-Based Authentication

Do not confuse token inspection with vulnerability.

Map:

Token issuance
↓
Token validation
↓
Identity extraction
↓
Session creation

Ask:

Who signs the token?
Who validates it?
Which claims determine identity?
Which claims determine authentication state?
Is issuer validated?
Is audience validated?
Is expiration enforced?
Is the token bound to the intended authentication flow?

Prioritize logic flaws over speculative cryptographic attacks.

A decoded JWT is not a finding.

20. OAuth / OIDC / SSO

When present, map:

Authorization
↓
Callback
↓
Code/token
↓
Identity claims
↓
Local session

Inspect:

state;
nonce;
redirect_uri;
client_id;
authorization code;
identity-provider subject;
issuer;
audience;
account-linking logic.

High-signal questions:

Can callback processing create a session without completing the expected flow?

Can an external identity be mapped to the wrong local account?

Can authentication state from one session be consumed by another?

Can account linking bypass authentication requirements?

Can a weaker SSO path authenticate an account protected by stronger local authentication?

OAuth is especially relevant because implementation flaws can result in complete authentication bypass.

21. Login CSRF / Authentication Confusion

Look for flows where an attacker can cause the victim's browser to become authenticated as an attacker-controlled account.

Model:

Attacker controls authentication transaction
↓
Victim browser completes transaction
↓
Victim becomes authenticated as attacker

Determine whether this leads to meaningful impact such as:

sensitive information being entered into attacker-controlled account;
account linking;
identity confusion;
credential/data disclosure.

Do not report login CSRF solely because a login endpoint lacks a CSRF token.

Demonstrate actual security impact.

22. Session Establishment

Check:

Before authentication
↓
Authentication
↓
Authenticated session

Ask:

Does the session change?
Is authentication state server-side?
Can an unauthenticated session become authenticated unexpectedly?
Can a pre-authentication session be fixed?
Can one session complete another session's authentication transaction?
Does logout invalidate the authenticated state?

Authentication bypass can occur when the server incorrectly promotes a pre-authentication session.

23. Authentication Header / Proxy Trust

When authentication depends on infrastructure-provided identity headers, investigate the trust boundary.

Examples:

X-Authenticated-User
X-User
X-Remote-User
X-Forwarded-User
X-SSL-CLIENT-CN

Do not assume these headers are vulnerable.

Determine:

Who sets the header?
Can the external client influence it?
Does the backend trust it?
Does changing it change authenticated identity?

PortSwigger documents cases where upstream authentication information is passed through headers and consumed by backend authorization logic.

24. Direct-Route Testing

Once authenticated functionality is discovered, test the route independently of the UI.

Example:

Login required
↓
Dashboard
↓
Sensitive endpoint

Try:

Unauthenticated → Sensitive endpoint
Partial-auth → Sensitive endpoint
Expired-session → Sensitive endpoint
Logged-out session → Sensitive endpoint

Compare server behavior.

Do not rely on client-side redirects as proof of access control.

25. State-Parameter Testing

For every authentication transaction identifier:

state
challenge
transaction_id
flow_id
session_id
request_id

ask:

Is it unpredictable?
Is it bound to the correct user?
Is it bound to the correct browser/session?
Is it single-use?
Does changing it alter identity?
Can it be replayed?
Can it be exchanged across accounts?

Focus on binding rather than guessing.

26. Race Conditions

Consider races only when there is evidence of a state transition.

Good candidates:

MFA completion
+
protected request

password reset
+
login

session revocation
+
authenticated request

verification
+
sensitive action

Hypothesis:

Security state changes
        ↓
Two requests observe different states
        ↓
Protected action executes before enforcement catches up

Avoid blind concurrency testing.

27. False Positive Control

Do NOT report:

login endpoint returning 200;
protected page returning a generic shell;
client-side redirects alone;
decoded JWTs;
visible authentication parameters;
missing CAPTCHA alone;
username enumeration without meaningful impact;
weak password policy alone;
an endpoint being reachable but not actually usable;
a token being replayable when replay is intended;
an unusual state transition with no security consequence.

Require:

Attacker-controlled condition
+
Authentication requirement
+
Requirement bypassed
+
Protected identity/functionality obtained
28. Validation Standard

Strong evidence:

Unauthenticated/partial-authenticated actor
                ↓
Authentication control bypassed
                ↓
Authenticated state obtained
                ↓
Protected functionality accessed

For account compromise:

Attacker
↓
Bypass
↓
Victim identity
↓
Victim protected resource

Use controlled accounts whenever possible.

Stop at the minimum evidence needed to establish impact.

29. Chaining

Prioritize chains such as:

Authentication bypass
+
admin functionality
→ administrative compromise

MFA bypass
+
persistent session
→ account takeover

Recovery bypass
+
automatic login
→ account takeover

OAuth identity confusion
+
account linking
→ account takeover

Partial authentication
+
unprotected API
→ authenticated data access

Authentication bypass
+
IDOR/BOLA
→ cross-user compromise

Document causal relationships.

Do not inflate severity with unrelated bugs.

30. Pivot Logic

When a test fails:

FAILED
↓
Identify the enforced control
↓
Determine which assumption was disproved
↓
Test the adjacent trust boundary

Examples:

Direct protected endpoint blocked
→ inspect partial-auth state

MFA direct bypass blocked
→ test MFA identity binding

Identity substitution blocked
→ inspect challenge/session binding

Reset token cannot be modified
→ test reset-state transitions

Web endpoint protected
→ compare API/mobile/legacy endpoint

OAuth callback validates state
→ inspect identity/account-linking logic

Token manipulation fails
→ inspect token issuance/session exchange

Never switch to random payloads without a new hypothesis.

31. Stop / Continue Rules
Continue when
authentication state is unclear;
a partial-auth state exists;
different authentication mechanisms coexist;
identity is client-controlled;
verification occurs across multiple requests;
an alternate endpoint implements the same function;
recovery is weaker than normal authentication;
authentication behavior differs between clients;
an unexplained state transition occurs.
Stop when
the server demonstrably enforces authentication;
the anomaly has no security impact;
behavior is clearly intended;
the same root cause is already established;
further testing would require unnecessary destructive actions.
32. Evidence Requirements

Capture:

1. Baseline unauthenticated request
2. Normal authenticated request
3. Authentication flow/state
4. Modified request
5. Server response
6. Resulting authentication state
7. Identity obtained
8. Protected functionality reached

For state-machine flaws:

Request A
→ State A

Skipped/modified request
→ State B

Protected request
→ authenticated result

Keep evidence minimal and redact secrets.

33. Final Decision Tree
Is the target supposed to require authentication?
            |
           YES
            ↓
Can unauthenticated access reach protected functionality?
            |
      +-----+-----+
     YES          NO
      ↓            ↓
Validate impact   Map auth states
                   ↓
          Is there partial authentication?
                   |
              +----+----+
             YES        NO
              ↓          ↓
       Test state       Map login
       transitions      verification
              ↓          ↓
        Can a required   Can a required
        step be skipped? step be bypassed?
              |          |
             YES        YES
              ↓          ↓
         Validate      Validate
              \          /
               \        /
                ↓      ↓
              AUTH BYPASS
                   ↓
             Check identity
             + privilege
                   ↓
                Chain