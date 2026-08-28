Password_Reset — Bug Hunting Skill
---
name: Password_Reset
description: >
  Professional bug-bounty skill for discovering, validating, and documenting
  password-reset and account-recovery vulnerabilities in authorized targets.
  Focuses on reset-token security, identity binding, workflow integrity,
  authorization, session handling, rate limiting, and account takeover impact.
---

# Password Reset Vulnerability Hunting

## 1. Mission

Identify vulnerabilities in password-reset and account-recovery workflows that
could allow an attacker to:

- Take over another user's account.
- Reset a password without proving control of the account.
- Obtain or predict another user's reset token.
- Reuse, bypass, or manipulate reset tokens.
- Change the recovery destination.
- Bypass verification steps.
- Abuse reset functionality for account enumeration.
- Abuse reset mechanisms to affect sessions or authentication state.
- Escalate from a low-impact reset weakness into full account compromise.

Only test accounts, systems, and targets for which testing is explicitly authorized.

---

# 2. Core Attack Surface

Map the complete password-reset lifecycle before testing.

Typical flow:

```text
Forgot Password
      ↓
Identify Account
      ↓
Verification / Challenge
      ↓
Reset Token / OTP
      ↓
Reset Password
      ↓
Session / Authentication State
      ↓
Post-Reset Account Access

Potential endpoints include:

POST /forgot-password
POST /password-reset
POST /reset-password
GET  /reset-password
POST /verify-reset-token
POST /verify-otp
POST /resend-otp
POST /change-password
POST /set-password
GET  /account/recovery

Do not assume endpoint names. Discover the real workflow from browser
traffic, Burp history, JavaScript, API documentation, and application behavior.

3. Recon Before Testing
3.1 Map the Workflow

Capture:

Forgot-password request.
Account identifier submitted.
Server response.
Email/SMS/OTP delivery.
Reset-token generation.
Token verification.
Password submission.
Session creation.
Existing-session behavior.
Post-reset redirects.
Recovery-email changes.
Recovery-phone changes.
MFA interactions.

Record:

Endpoint
HTTP Method
Parameters
Cookies
Authorization headers
CSRF token
Reset token
OTP
User identifiers
Email
Phone
Session identifiers
Response behavior
Redirect behavior
4. Establish Test Accounts

Prefer multiple controlled accounts.

Minimum recommended setup:

Account A = attacker-controlled
Account B = second controlled user
Account C = optional privileged/test account

Record unique identifiers:

user_id
account_id
email
phone
organization_id
tenant_id
session_id

The objective is to determine whether reset operations remain bound to the
correct identity throughout the entire workflow.

5. Reset Token Security

Investigate the properties of reset tokens.

5.1 Token Predictability

Check whether tokens appear:

Sequential.
Timestamp-derived.
User-ID-derived.
Email-derived.
Short numeric values.
Repeated across requests.
Generated using predictable patterns.

Compare multiple tokens generated:

Token A
Token B
Token C
Token D

Look for structural relationships.

Do not perform unnecessary high-volume token guessing against production systems.

5.2 Token Entropy

Determine:

Length.
Character set.
Randomness.
Number of possible values.
Whether the token is sufficiently unpredictable.

High-value finding:

Attacker can predict or practically brute-force a valid reset token
5.3 Token Reuse

Test whether a token remains valid after:

Password reset
Token verification
Password change
Requesting another reset
Logging in
Logging out

Expected behavior:

Used token → invalid
Expired token → invalid
Superseded token → invalid
5.4 Multiple Active Tokens

Generate:

Token A
Token B

Determine whether:

Token A remains valid after Token B is generated

Multiple simultaneously valid tokens may be intentional, but investigate
whether this creates an exploitable race or account-takeover condition.

6. Identity Binding

This is one of the highest-value areas.

Determine whether the reset token is cryptographically or server-side bound
to the intended account.

Test controlled accounts:

Account A
Account B

Conceptually evaluate:

Request reset for A
Obtain legitimate reset token for A

Attempt to use the token while supplying:
- A's user ID
- B's user ID
- A's email
- B's email
- A's account ID
- B's account ID

The server must not allow:

Token(A) + Identity(B) → Reset B

A successful cross-account reset is potentially critical account takeover.

7. Parameter Manipulation

Inspect every parameter associated with the reset flow.

Examples:

user_id
userId
account_id
accountId
email
username
phone
reset_token
token
code
otp
verification_id
reset_id
challenge_id

Test whether changing one identifier affects the target account.

Example conceptual matrix:

Token A + user A
Token A + user B
Token B + user A
Token B + user B

Expected:

Only valid token/account combinations succeed.

Do not assume that hidden or frontend parameters are trusted.

8. Reset-Link Manipulation

Inspect reset URLs.

Example:

/reset-password?token=TOKEN

Investigate:

Token handling.
User identifiers.
Redirect parameters.
Verification state.
Additional query parameters.
Client-side state.
Fragment parameters.
Host/origin handling.

Potentially dangerous pattern:

/reset-password?user_id=TARGET&token=ATTACKER_TOKEN

The backend must validate the complete reset context server-side.

9. OTP Testing

If OTP is used, test:

Rate Limiting

Determine whether repeated incorrect attempts are restricted.

Expiration

Check whether OTPs expire appropriately.

Reuse

Check whether an OTP can be reused.

Invalidation

Determine whether requesting a new OTP invalidates the previous one.

Identity Binding

Check whether:

OTP(A) → Account B

is rejected.

Response Leakage

Look for responses revealing:

correct/incorrect
remaining attempts
partial OTP
OTP length
verification state
account existence

Do not perform large-scale OTP brute forcing against production.

10. OTP / Token Race Conditions

Investigate state transitions such as:

Generate OTP
Verify OTP
Generate second OTP
Verify first OTP
Reset password

Also inspect concurrent requests where authorized testing permits it.

Look for inconsistent server-side state such as:

Expired token accepted
Invalidated token accepted
Already-used token accepted
Verification bypassed due to request ordering
11. Verification-Step Bypass

Map every security checkpoint.

Example:

Forgot password
      ↓
Email verification
      ↓
OTP verification
      ↓
Reset authorization
      ↓
Password change

Test whether later steps can be reached without completing earlier steps.

Look for:

Direct access to reset endpoint
Missing verification state
Client-side-only verification
Manipulable verification flags
Missing server-side authorization
Alternative endpoint bypass
HTTP method differences
API/mobile endpoint inconsistencies

The key question:

Does the server independently enforce every required recovery condition?

12. Alternate Recovery Paths

Do not test only the primary web flow.

Compare:

Web
Mobile API
Old API version
GraphQL
OAuth/account recovery
Email recovery
SMS recovery
Support recovery
Password change
Session recovery

A weak alternate recovery endpoint can undermine an otherwise secure primary
workflow.

13. Recovery Destination Manipulation

Investigate whether the attacker can influence:

recovery_email
email
phone
phone_number
recovery_phone
contact
notification_email

High-value scenario:

Attacker initiates recovery
        ↓
Manipulates recovery destination
        ↓
Reset instructions reach attacker
        ↓
Victim account compromised

Do not actually redirect real users' recovery messages.

Use controlled accounts and controlled destinations.

14. Account Enumeration

Check whether the reset endpoint reveals whether an account exists.

Compare:

Existing account
Non-existing account

Observe:

HTTP status.
Response body.
Response length.
Error message.
Response timing.
Headers.
Redirect behavior.
OTP/email behavior.

Example:

"Account does not exist"

is stronger enumeration than:

"If the account exists, instructions were sent."

Enumeration severity depends heavily on target policy and practical impact.

15. Host / Link Generation Issues

Inspect how reset links are generated.

Investigate whether user-controlled input can influence:

Host
X-Forwarded-Host
Forwarded
Origin
Referer
redirect parameters

The objective is to determine whether a reset link or sensitive recovery
material can be generated pointing to an attacker-controlled destination.

Do not send real users' reset tokens to external infrastructure.

Use controlled accounts and harmless canary values where permitted.

16. Token Leakage

Search for reset tokens in:

URL
Referer
Browser history
Analytics
Logs
HTML
JavaScript
API responses
Redirects
Third-party requests
Emails
Notifications

Pay special attention to:

/reset?token=...

followed by external resources or redirects.

Determine whether an attacker can realistically obtain a valid token.

17. Session Handling After Reset

After a successful password reset, determine:

Are old sessions invalidated?
Are refresh tokens invalidated?
Are remembered devices invalidated?
Are API tokens revoked?
Are active browser sessions revoked?

Important distinction:

Password reset works

vs.

Password reset allows attacker to maintain or obtain persistent access

Session persistence can materially increase impact.

18. Password Reset vs Change Password

Compare:

/change-password
/reset-password
/set-password

The authenticated password-change endpoint should normally require the current
password or another appropriate strong authentication factor.

The unauthenticated reset endpoint should require its own secure recovery
authorization.

Look for security assumptions being incorrectly shared between these flows.