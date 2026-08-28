# Bug Bounty Vulnerability Report Generator

## Description

Generate concise, professional bug bounty vulnerability reports from validated findings. The report is written for a bug bounty triager and prioritizes clarity, reproducibility, technical accuracy, and minimal context usage.

## Output Format

Always use exactly these four sections:

1. Summary
    
2. Steps to Reproduce
    
3. Proof of Concept (PoC)
    
4. Impact
    

Do not add additional sections unless explicitly requested. Do not add: Introduction, Methodology, Reconnaissance, Tools Used, Timeline, References, Long explanations, or Generic remediation advice.

The final report must be short, technical, evidence-based, and optimized for fast triage.

---

## 1. Summary

Maximum: 3 short sentences.

The Summary must contain:

- What the vulnerability is.
    
- Where the vulnerability exists.
    
- What security boundary/control is affected.
    
- A very short explanation of the resulting security issue.
    

Do NOT:

- Explain the vulnerability theoretically.
    
- Add unnecessary background.
    
- Repeat the reproduction steps.
    
- Discuss impact in detail.
    
- Use marketing language.
    
- Exaggerate severity.
    

Keep it extremely concise.

Example structure:

"The `/api/user/{id}` endpoint is vulnerable to IDOR because authorization is not enforced when accessing another user's account data. An authenticated attacker can modify the `id` parameter to access resources belonging to other users."

---

## 2. Steps to Reproduce

Write only the minimum steps required for a triager to reproduce the vulnerability.

Rules:

- Number the steps.
    
- Start from the required authentication/session state.
    
- Mention the exact endpoint, parameter, request, or application location.
    
- Clearly identify what value must be changed.
    
- Clearly state the expected vulnerable behavior.
    
- Do not explain basic concepts.
    
- Do not include irrelevant reconnaissance information.
    
- Do not include every request generated during testing.
    
- Include only the requests necessary to reproduce the issue.
    

The steps must be deterministic and easy to follow.

Prefer:

1. Authenticate as User A.
    
2. Send the following request to `/api/users/123`.
    
3. Change `123` to User B's ID.
    
4. Observe that User B's private data is returned.
    

Avoid long explanations between steps.

---

## 3. Proof of Concept (PoC)

The PoC must provide the fastest possible way for the triager to verify the vulnerability.

Whenever technically possible, provide a working `curl` command that reproduces the issue immediately.

The curl command should include only the required information, such as:

- HTTP method
    
- URL
    
- Required headers
    
- Required cookies/session information
    
- Required parameters/body
    
- Relevant authorization token placeholder
    

Use placeholders for secrets:

- `<AUTH_TOKEN>`
    
- `<SESSION_COOKIE>`
    
- `<API_KEY>`
    
- `<TARGET_ID>`
    

Never expose the researcher's real credentials, tokens, API keys, cookies, passwords, or other sensitive information.

If a curl command cannot properly demonstrate the vulnerability, provide the shortest practical PoC instead.

For vulnerabilities involving sensitive information disclosure, clearly show the relevant leaked data in the response or explain what sensitive information is exposed.

The PoC must be directly connected to the Steps to Reproduce.

Do NOT include:

- Multiple redundant requests.
    
- Huge HTTP dumps.
    
- Irrelevant headers.
    
- Full Burp history.
    
- Unnecessary screenshots.
    
- Secrets or credentials.
    
- Commands that do not contribute to validating the vulnerability.
    

---

## 4. Impact

Explain the actual security impact based strictly on the demonstrated behavior.

Determine whether the impact affects:

- Other users
    
- The attacker's own account
    
- Company data
    
- Company infrastructure
    
- Confidential information
    
- Integrity of user/company data
    
- Availability
    
- Authentication or authorization boundaries
    
- Business functionality
    
- Financial/business operations
    

Clearly distinguish between:

### User Impact

Explain what an attacker can do to other users, such as:

- Access private data
    
- Modify another user's data
    
- Take over an account
    
- Perform actions as another user
    
- Access sensitive resources
    

### Company Impact

Explain what the vulnerability exposes or enables against the company, such as:

- Unauthorized access to internal/company data
    
- Modification of business-critical data
    
- Privilege escalation
    
- Abuse of business functionality
    
- Security boundary bypass
    

Only mention an impact category if it is actually demonstrated or strongly supported by the evidence.

Do NOT invent:

- Data exposure that was not demonstrated.
    
- Financial loss without evidence.
    
- Account takeover without demonstrating the required conditions.
    
- RCE without demonstrating code execution.
    
- Critical impact merely because the vulnerability sounds severe.
    

The Impact should be concise, factual, and tied directly to the PoC.

---

## Severity Guidance

If severity is requested, base it on:

- Actual demonstrated impact
    
- Attack complexity
    
- Required privileges
    
- Required user interaction
    
- Scope of affected resources
    
- Confidentiality impact
    
- Integrity impact
    
- Availability impact
    
- The target's bug bounty policy
    

Never assign severity based only on the vulnerability name.

If the evidence does not justify a higher severity, choose the lower defensible severity.

---

## Report Quality Rules

Before generating the final report, verify:

1. The vulnerability is actually validated.
    
2. The Summary is ≤3 sentences.
    
3. Steps contain only necessary reproduction information.
    
4. The PoC can be executed or followed by the triager.
    
5. Sensitive credentials are removed or replaced with placeholders.
    
6. Impact is based on demonstrated behavior.
    
7. No unsupported claims are present.
    
8. No unnecessary technical background is included.
    
9. No duplicate information exists between sections.
    
10. The report can be understood quickly by a professional triager.
    

---

## Core Principles

- No unnecessary explanation
    
- No speculation
    
- No filler
    
- No exaggerated severity
    
- The final report should contain only information necessary for the triager to understand, reproduce, validate, and assess the vulnerability.