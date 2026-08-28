---

name: authentication
description: Hunts for web authentication vulnerabilities across login, identity verification, MFA, password recovery, session establishment, OAuth/OIDC, SSO, and authentication state transitions. Use when testing whether an application can be made to authenticate the wrong identity, skip a required authentication step, weaken authentication controls, or incorrectly bind credentials, factors, tokens, or sessions to users.
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Authentication Hunting Skill

## 1. Purpose

Find flaws that allow an attacker to:

* authenticate as another user;
* bypass or weaken a required authentication factor;
* obtain or reuse another user's authentication state;
* confuse the application about which identity is being authenticated;
* manipulate authentication state transitions;
* bypass password/recovery protections;
* abuse OAuth/OIDC/SSO authentication;
* exploit weak login defenses;
* chain a low-impact authentication weakness into account takeover or privilege escalation.

Do not confuse Authentication with Authorization.

* **Authentication:** who is this user?
* **Authorization:** what may this authenticated user do?
* **Session management:** how does the application preserve the authenticated identity?

When the finding crosses into authorization, continue testing the authentication root cause and document the chain.

---

# 2. Trigger Conditions

Activate when the target exposes one or more of:

* login/logout;
* registration;
* password authentication;
* MFA/2FA;
* email/phone verification;
* password reset;
* password change;
* account recovery;
* "remember me";
* magic links;
* OTP;
* passkeys/WebAuthn;
* SSO;
* OAuth/OIDC;
* social login;
* API authentication;
* JWT/session-token exchange;
* device/session management;
* re-authentication;
* step-up authentication;
* identity verification;
* invitation-based onboarding.

Prefer targets where multiple authentication mechanisms coexist.

---

# 3. Core Hunting Model

Use:

```text
Map identity flow
↓
Map authentication states
↓
Map trust boundaries
↓
Identify identity-bearing inputs
↓
Identify verification gates
↓
Test state transitions
↓
Test identity binding
↓
Test factor binding
↓
Test recovery paths
↓
Test alternate authentication paths
↓
Compare responses/states
↓
Pivot from anomalies
↓
Validate account impact
↓
Check chaining
↓
Collect minimal proof
```

The primary question is:

> Can I make the application believe that authentication succeeded for an identity or authentication state that I should not possess?

---

# 4. Build the Authentication Map First

Before payload testing, identify:

### Entry points

* `/login`
* `/signin`
* `/auth`
* `/authenticate`
* `/session`
* `/token`
* `/oauth/*`
* `/authorize`
* `/callback`
* `/sso/*`
* `/verify`
* `/otp`
* `/mfa`
* `/2fa`
* `/reset`
* `/forgot-password`
* `/recover`
* `/change-password`

Also inspect:

* GraphQL mutations;
* mobile/API endpoints;
* WebSocket authentication;
* background XHR/fetch requests;
* server-rendered authentication routes;
* undocumented endpoints;
* alternate hostnames;
* legacy authentication endpoints.

### Identity-bearing values

Record every value that appears to identify a user:

* email;
* username;
* user ID;
* account ID;
* phone;
* tenant ID;
* client ID;
* identity-provider subject;
* OAuth `sub`;
* JWT claims;
* session IDs;
* reset-token references;
* OTP transaction IDs;
* MFA challenge IDs;
* device IDs;
* invitation IDs.

For each value ask:

```text
Who creates it?
Who validates it?
Who can modify it?
What security decision depends on it?
Is it cryptographically or server-side bound to the authenticated identity?
```

---

# 5. Establish Baseline Accounts

When permitted, create controlled accounts.

Prefer:

```text
Account A = attacker-controlled
Account B = second controlled identity
Account C = privileged/test identity if provided
```

Record independently:

* credentials;
* session cookies;
* access tokens;
* refresh tokens;
* device/session identifiers;
* MFA state;
* recovery state;
* OAuth identities.

Never assume two accounts are equivalent merely because they use the same application.

---

# 6. Authentication State Machine

Model the flow explicitly.

Example:

```text
UNAUTHENTICATED
      ↓
PRIMARY CREDENTIAL VERIFIED
      ↓
MFA REQUIRED
      ↓
MFA VERIFIED
      ↓
FULL SESSION
```

Also model:

```text
UNAUTH
→ PASSWORD_OK
→ EMAIL_VERIFY_REQUIRED
→ MFA_REQUIRED
→ STEP_UP_REQUIRED
→ AUTHENTICATED
→ REAUTH_REQUIRED
→ AUTHENTICATED
```

For every transition ask:

1. What server-side state changed?
2. What token/cookie was issued?
3. Can the next state be accessed directly?
4. Can the previous state access authenticated functionality?
5. Is the identity fixed to the same user across every step?
6. Can another browser/session continue the transaction?
7. Can a step be replayed?
8. Can steps be reordered?
9. Can a state transition be triggered without completing the previous state?

Authentication bugs frequently exist in the transitions rather than the login endpoint itself.

---

# 7. High-Signal Test Priority

Use this order unless the target indicates otherwise:

## P0 — Authentication State / Identity Confusion

Test first:

* authentication step skipping;
* partial-authentication session abuse;
* identity mismatch between steps;
* user-controlled identity parameters;
* verification-state manipulation;
* session issued before required factor;
* alternate endpoint accepting incomplete authentication.

Reason:

```text
Low request cost
+
High impact
+
Strong signal
=
Priority
```

## P1 — MFA / Recovery Logic

Test:

* MFA bypass;
* MFA identity binding;
* OTP transaction binding;
* recovery-factor substitution;
* password reset authorization;
* password-change reauthentication;
* recovery token reuse.

## P1 — OAuth/OIDC/SSO

Test:

* `state`;
* `nonce`;
* `redirect_uri`;
* authorization-code handling;
* token/session exchange;
* identity claim binding;
* account linking;
* unverified identity claims;
* scope/client binding.

## P2 — Login Enumeration / Brute-Force Controls

Test:

* username enumeration;
* inconsistent responses;
* timing differences;
* account-lock behavior;
* per-IP vs per-account controls;
* alternate login endpoints;
* inconsistent rate limits.

Keep testing bounded and within program limits.

## P2 — Session Establishment

Test:

* session fixation;
* token rotation;
* login/logout invalidation;
* cross-device session behavior;
* pre-authentication cookies becoming authenticated;
* token reuse after security events.

## P3 — Edge Authentication Mechanisms

Only when present:

* magic links;
* passwordless login;
* passkeys;
* device trust;
* email verification;
* phone verification;
* invitation authentication;
* legacy endpoints;
* API/mobile-specific login flows.

---

# 8. Identity-Binding Test

This is one of the highest-value tests.

For every multi-step flow:

```text
Step 1 → Identity A
Step 2 → Identity A
```

Then determine whether any identity-bearing value can be changed to B.

Conceptually:

```text
POST /auth/step1
identity=A
credentials=A

↓

challenge=XYZ

↓

POST /auth/step2
challenge=XYZ
identity=B
factor=valid-factor
```

Ask:

```text
Does the server derive identity from trusted server-side state?
OR
Does it trust a client-controlled identity value?
```

If changing the identity changes the authenticated account, investigate immediately.

---

# 9. One-Variable Testing

Change one security-relevant variable at a time.

Example:

```text
Baseline:
identity=A
challenge=X
factor=valid-A

Test:
identity=B
challenge=X
factor=valid-A
```

Keep constant:

* endpoint;
* method;
* session;
* headers;
* body;
* timing;
* unrelated parameters.

This makes authentication logic failures easier to isolate.

---

# 10. Differential Testing Matrix

Where applicable compare:

| Dimension      | Compare                                    |
| -------------- | ------------------------------------------ |
| Identity       | A vs B                                     |
| Authentication | authenticated vs unauthenticated           |
| MFA            | completed vs incomplete                    |
| Recovery       | own token vs unrelated token               |
| Session        | pre-login vs post-login                    |
| Device         | trusted vs untrusted                       |
| State          | expected vs skipped                        |
| Credential     | valid vs invalid                           |
| Factor         | A's factor vs B's factor                   |
| OAuth          | own identity vs controlled second identity |
| Endpoint       | modern vs legacy                           |
| Client         | browser vs API/mobile                      |

Look for differences in:

* status code;
* response body;
* redirects;
* cookies;
* tokens;
* authenticated resources;
* server-side state;
* timing;
* error messages.

---

# 11. Login Testing

Inspect:

```text
username
email
password
remember-me
captcha
device ID
client ID
return URL
next
continue
state
nonce
```

Test whether authentication depends on any value that should not be trusted.

High-signal questions:

```text
Can login succeed without the expected credential?
Can a valid username be paired with an invalid password?
Can the password check be bypassed through alternate parameters?
Does another endpoint authenticate the same account differently?
Does an API/mobile endpoint implement weaker verification?
Does a partial login create useful authenticated state?
```

Do not assume the visible login form is the only authentication boundary.

---

# 12. Username Enumeration

Compare invalid and valid candidate identities.

Observe:

* response text;
* status;
* response length;
* redirect;
* JSON fields;
* error code;
* timing;
* lockout behavior;
* password-reset behavior;
* registration behavior.

High-signal enumeration locations:

```text
/login
/password-reset
/register
/invite
/email verification
/account recovery
/username availability
```

Do not report enumeration alone as account takeover unless the program considers it impactful.

Instead ask:

```text
Can enumeration enable credential attacks?
Does it expose sensitive account existence?
Does it unlock a stronger authentication weakness?
```

PortSwigger specifically documents enumeration through different responses, subtle differences, timing, and account-lock behavior.

---

# 13. Brute-Force / Rate-Limit Logic

Do not blindly spray credentials.

First determine the protection model:

```text
Per IP?
Per account?
Per session?
Per device?
Per endpoint?
Per authentication method?
```

Then test whether protection is consistently enforced.

Look for:

* login endpoint vs API endpoint;
* web vs mobile;
* alternate hostname;
* password reset vs login;
* username-based vs email-based login;
* IPv4/IPv6 inconsistencies;
* authenticated vs unauthenticated paths;
* different HTTP methods;
* different content types.

Important distinction:

```text
Rate-limit weakness
≠
Account takeover
```

Validate actual security impact before escalating severity.

---

# 14. MFA / 2FA

When MFA exists, map:

```text
password verification
↓
MFA challenge creation
↓
MFA delivery
↓
MFA verification
↓
session elevation
```

Test:

### Skip

Can the post-password user access authenticated functionality before MFA?

### Identity mismatch

Can a challenge created for A be completed using B's factor?

### Challenge binding

Can a challenge ID be replaced?

### Replay

Can a successful OTP/challenge be reused?

### Expiration

Does an expired challenge remain valid?

### Attempt control

Are verification attempts bounded?

### Session binding

Can another browser/session complete the challenge?

### Recovery

Does "lost MFA" provide a weaker authentication path?

### Alternate channel

Does email/SMS/recovery bypass stronger MFA?

The critical validation question is:

```text
Does the weakness produce a fully authenticated session?
```

PortSwigger specifically identifies MFA bypasses caused by directly accessing authenticated pages after the first authentication step and flaws where the second factor is not properly bound to the identity.

---

# 15. Password Reset / Recovery

Treat recovery as an authentication mechanism.

Map:

```text
request reset
↓
identity selection
↓
token/challenge
↓
verification
↓
new password
↓
session creation
```

Test:

* token predictability;
* token reuse;
* token expiration;
* token invalidation after use;
* token/account binding;
* host/header influence;
* email/user identity confusion;
* alternate recovery endpoints;
* changing identity between steps;
* password reset without sufficient verification;
* session issuance after reset;
* whether old sessions remain valid;
* recovery fallback weaker than normal authentication.

High-signal rule:

> If the recovery mechanism can authenticate the wrong identity, treat it as an authentication bypass/account takeover candidate.

Password reset is not secondary functionality; it is an authentication boundary. PortSwigger explicitly recommends auditing reset/change mechanisms with the same rigor as login.

---

# 16. Password Change

Test:

```text
current password
new password
confirm password
MFA/re-authentication
session
```

Ask:

```text
Is the current password actually verified?
Can the identity be changed?
Can another user's reset state authorize the change?
Does changing the password invalidate existing sessions?
Does a weaker endpoint perform the same operation?
```

Also inspect API/mobile implementations.

---

# 17. Session Establishment

Authentication is incomplete until the resulting session is correct.

Check:

```text
Before login:
session=A

After login:
session=B
```

Investigate whether:

* session ID rotates;
* pre-authentication session becomes privileged;
* authentication state can be fixed;
* logout invalidates the session;
* password change invalidates old sessions;
* MFA completion changes authentication state;
* tokens remain valid after account-security changes.

OWASP treats authentication and session management as closely connected because the session is what allows subsequent requests to retain the authenticated identity.

---

# 18. "Remember Me"

Inspect persistent authentication tokens.

Ask:

```text
Is token entropy sufficient?
Is token tied to the user?
Can token data be modified?
Can a token be reused after logout/password change?
Does remember-me authentication bypass MFA?
Does it create a stronger session than intended?
```

If token structure is predictable, validate whether it actually permits account access.

Do not report token structure alone.

---

# 19. OAuth / OIDC / SSO

When social login or SSO exists, map:

```text
Client
↓
Authorization endpoint
↓
Identity provider
↓
Callback
↓
Code/token exchange
↓
Identity claims
↓
Local session
```

Identify:

```text
client_id
redirect_uri
response_type
response_mode
scope
state
nonce
code
access_token
id_token
PKCE
```

PortSwigger identifies OAuth authentication attack surfaces including flawed CSRF protection, code/token leakage, redirect URI validation, scope validation, and unverified user registration.

### Priority tests

#### `state`

Determine whether it exists and whether it is bound to the initiating browser/session.

#### `redirect_uri`

Determine whether validation is exact or weak.

Test parser/normalization discrepancies only when in scope.

#### Identity claim binding

Ask:

```text
Does the application trust email alone?
Does it validate issuer?
Does it validate audience/client?
Does it validate the identity-provider subject?
Does it bind the OAuth identity to the correct local account?
```

#### Account linking

Test:

```text
Account A local
+
OAuth identity B
```

Determine whether an attacker can bind their OAuth identity to another local account.

#### Callback/session exchange

Identify whether client-controlled identity data is accepted by the endpoint responsible for creating the local session.

#### Scope/client binding

Where applicable, verify that tokens are accepted only for the intended client and scope.

#### OIDC

Inspect:

* issuer;
* audience;
* subject;
* nonce;
* signature;
* algorithm;
* expiration;
* token source;
* local-account mapping.

Do not treat decoding a JWT as a vulnerability.

---

# 20. Authentication State Confusion

Look for endpoints that consume state indirectly.

Examples:

```text
POST /authenticate
POST /verify
POST /complete-login
GET /callback
POST /exchange-token
POST /session
```

Questions:

```text
Which fields determine identity?
Which fields determine authentication status?
Which fields determine MFA completion?
Which fields determine session creation?
```

If these are client-controlled, test whether they can become inconsistent.

Example model:

```text
identity = A
credential = valid-A
verification_state = verified
session_target = B
```

A secure implementation should reject inconsistent states.

---

# 21. Multi-Step Flow Manipulation

For every authentication sequence test:

```text
Skip step
Repeat step
Reorder step
Replay step
Change identity
Change transaction ID
Change session
Change device
Change factor
Use stale state
Use state from another account
```

Do not change everything simultaneously.

Start with the smallest mutation that can reveal the trust boundary.

---

# 22. Race Conditions in Authentication

Prioritize races when authentication has hidden state transitions.

Interesting sequences include:

```text
login + authenticated endpoint
password reset + login
MFA completion + privileged endpoint
session revocation + authenticated request
email verification + sensitive action
```

The key question:

> Is there a temporary security state in which the application believes authentication succeeded before all required checks have completed?

PortSwigger documents authentication-related race conditions where a session can temporarily become authenticated before MFA enforcement is applied.

Only pursue race testing after identifying a plausible state transition; avoid blind high-volume concurrency.

---

# 23. Alternate Authentication Paths

Always compare:

```text
Web
API
Mobile
GraphQL
Legacy
SSO
OAuth
Passwordless
Recovery
Invitation
Magic link
```

A common high-value pattern is:

```text
Strong authentication path
        ↓
Weak alternate path
        ↓
Same account/session
```

If two mechanisms authenticate the same identity, they must enforce equivalent security guarantees.

---

# 24. Headers / Routing / Host Influence

When authentication or password-reset URLs depend on host/origin information, inspect:

* `Host`;
* forwarded host headers;
* origin;
* referer;
* callback URLs;
* absolute URLs generated by the server.

Do not report header reflection alone.

Look for security impact such as:

```text
host manipulation
↓
authentication/recovery link manipulation
↓
credential/token exposure
↓
account compromise
```

Host-header attacks can intersect directly with password-reset poisoning and authentication bypass scenarios.

---

# 25. API Authentication

Do not assume browser behavior represents API security.

Compare:

```text
Browser login
vs
REST API
vs
GraphQL
vs
Mobile API
```

Inspect:

* token issuance;
* token exchange;
* refresh;
* login endpoints;
* alternate versions;
* authentication headers;
* cookies;
* CSRF requirements;
* authentication middleware;
* error behavior.

High-signal finding:

```text
Same identity
+
Different endpoint
+
Different authentication requirement
=
Investigate
```

---

# 26. False Positive Control

Do not report:

* decoded JWTs without exploitability;
* predictable-looking IDs without authentication impact;
* username enumeration without meaningful impact;
* missing CAPTCHA alone;
* weak password policy alone;
* verbose authentication errors without practical consequence;
* session cookies merely existing;
* OAuth parameters merely being visible;
* HTTP Basic Auth merely being used over HTTPS;
* a token being replayable when replay is intended and adequately protected.

Require:

```text
Security boundary crossed
+
Attacker-controlled condition
+
Reproducible behavior
+
Meaningful impact
```

---

# 27. Validation Standard

A valid authentication finding should ideally demonstrate:

```text
Attacker-controlled action
        ↓
Authentication control weakened/bypassed
        ↓
Unexpected identity or authentication state
        ↓
Protected resource/session obtained
```

Prefer proving the smallest safe impact.

For account takeover:

```text
Authenticated as victim
```

is stronger evidence than:

```text
Login endpoint returned HTTP 200
```

Do not access unnecessary sensitive data.

Use controlled accounts whenever possible.

---

# 28. Chaining

Authentication findings often become much stronger when chained.

Look for:

```text
Enumeration
+
Credential weakness
→ Account compromise

MFA weakness
+
Session persistence
→ Persistent account takeover

OAuth identity confusion
+
Account linking
→ Account takeover

Password reset flaw
+
Host/header manipulation
→ Token theft / takeover

Partial authentication
+
Unprotected endpoint
→ Authentication bypass

Weak recovery
+
Existing session
→ Account takeover

Authentication bypass
+
Admin functionality
→ Administrative compromise
```

Document each causal step.

Do not inflate severity by chaining unrelated issues.

---

# 29. Pivot Logic

Use targeted pivots.

```text
Login bypass failed
→ test alternate login/API/mobile endpoint

MFA bypass failed
→ test recovery and step-up paths

Reset token appears strong
→ test token/account binding and flow state

OAuth redirect_uri blocked
→ inspect callback handling and identity binding

Enumeration found
→ test whether it enables a meaningful downstream attack

Session appears fixed
→ test whether it becomes authenticated or privileged

Token manipulation fails
→ inspect token-to-user binding rather than guessing signatures

One endpoint is protected
→ search for equivalent legacy/API endpoint
```

Never pivot randomly.

Every pivot should answer:

> What assumption did the failed test prove, and what adjacent assumption remains untested?

---

# 30. Stop / Continue Rules

### Continue when

* identity binding is unclear;
* a partial-authentication state exists;
* multiple authentication mechanisms coexist;
* recovery differs from login;
* OAuth/OIDC is present;
* alternate API/mobile endpoints exist;
* a response difference is unexplained;
* a token/challenge is not clearly bound to an identity.

### Stop when

* the behavior is demonstrably intended;
* the security boundary remains intact;
* the anomaly has no security impact;
* further testing would require unnecessary destructive actions;
* the same root cause has already been established.

Avoid duplicating tests across equivalent endpoints unless implementation differences exist.

---

# 31. Evidence Requirements

Capture:

1. Baseline request.
2. Modified request.
3. Relevant response.
4. Authentication-state change.
5. Identity affected.
6. Required prerequisites.
7. Exact security boundary crossed.
8. Minimal impact demonstration.

For multi-step vulnerabilities show the complete chain:

```text
Request 1
→ State/token

Request 2
→ State/token

Modified request
→ unexpected state

Final request
→ protected account/resource
```

Remove unrelated secrets from evidence.

---

# 32. Final Hunter Checklist

```text
[ ] Authentication architecture mapped
[ ] All authentication entry points identified
[ ] Identity-bearing inputs identified
[ ] Authentication states mapped
[ ] Partial-authentication states tested
[ ] Identity binding tested
[ ] Login logic tested
[ ] Enumeration tested
[ ] Rate-limit model understood
[ ] MFA flow tested
[ ] MFA identity binding tested
[ ] Recovery flow tested
[ ] Password reset tested
[ ] Password change tested
[ ] Session establishment tested
[ ] Logout/session invalidation tested
[ ] Remember-me tested
[ ] OAuth/OIDC identified
[ ] OAuth state tested
[ ] OAuth redirect handling tested
[ ] OAuth identity mapping tested
[ ] Alternate/API/mobile authentication tested
[ ] Legacy authentication paths tested
[ ] Relevant race conditions considered
[ ] False positives eliminated
[ ] Impact validated
[ ] Chaining evaluated
[ ] Minimal evidence collected
```

---

# 33. References

Load only when the active test requires deeper technique-specific detail:

* `references/password-login.md`
* `references/mfa-and-recovery.md`
* `references/oauth-oidc.md`
* `references/advanced-auth-flows.md`

# Password Login & Recovery Reference

## Username Enumeration

Test equivalent authentication operations with controlled valid/invalid identities.

Compare:

* status;
* body;
* length;
* redirect;
* error code;
* timing;
* lock state.

Prioritize enumeration when it can support a meaningful downstream attack.

## Brute-Force Controls

Determine whether controls are:

* per IP;
* per account;
* per session;
* per device;
* endpoint-specific.

Compare equivalent authentication interfaces.

Do not perform uncontrolled credential spraying.

## Login Logic

Test whether the server actually validates:

```text
identity
+
credential
+
authentication state
```

Treat any client-controlled value involved in session creation as a trust-boundary candidate.

## Password Reset

Map:

```text
request
→ token
→ verification
→ password change
→ session
```

Test:

* identity binding;
* token reuse;
* expiration;
* invalidation;
* step skipping;
* session binding;
* alternate recovery methods.

## Password Change

Verify:

* current-password enforcement;
* identity binding;
* reauthentication;
* MFA/step-up requirements;
* old-session invalidation.

## Remember Me

Determine:

* token entropy;
* user binding;
* expiration;
* revocation;
* MFA interaction.

Do not report token format alone.

## High-Signal Patterns

```text
Client-controlled identity
+
trusted authentication endpoint
→ investigate immediately

Weak recovery
+
same account
→ compare against normal login

Enumeration
+
weak rate limiting
→ test downstream account compromise safely
```
# Advanced Authentication Flows

## State Transition Testing

For every multi-step flow test:

```text
skip
repeat
reorder
replay
replace identity
replace transaction
replace session
replace device
replace factor
reuse stale state
```

Change one variable at a time.

## API / Mobile Parity

Compare:

```text
Web authentication
API authentication
Mobile authentication
Legacy authentication
```

Look for the same identity being authenticated through different security requirements.

## Authentication Headers

Inspect:

```text
Authorization
Cookie
Origin
Referer
Host
Forwarded
X-Forwarded-Host
```

Only escalate when the value affects an authentication or recovery security decision.

## Magic Links

Test:

* token binding;
* expiration;
* replay;
* session binding;
* redirect behavior;
* whether opening the link authenticates the intended identity.

## Email / Phone Verification

Test whether verification is bound to:

```text
account
+
session
+
transaction
```

A verification artifact from account A should not satisfy account B.

## Invitation Flows

Model:

```text
invite creation
→ invite delivery
→ acceptance
→ identity binding
→ account/session creation
```

Test whether invitation acceptance can authenticate or attach the wrong identity.

## Passkeys / WebAuthn

Focus on implementation logic:

* credential-to-account binding;
* challenge freshness;
* challenge/session binding;
* origin/RP validation;
* registration vs authentication state;
* recovery fallback.

Do not confuse a normal WebAuthn credential identifier with a vulnerability.

## Authentication + Authorization Chain

If authentication bypass yields access to another identity:

```text
Authentication bypass
→ victim session
→ victim authorization context
→ sensitive functionality
```

Stop the proof at the minimum point necessary to establish impact.
