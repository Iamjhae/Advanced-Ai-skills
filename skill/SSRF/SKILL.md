# SSRF Security Testing Skill

## Purpose

This skill provides a tool-agnostic methodology for identifying, validating, assessing, and reporting Server-Side Request Forgery (SSRF) vulnerabilities during authorized Web Application Penetration Testing and Bug Bounty assessments.

The skill describes **what the agent should do**, not which specific tools it must use.

The agent MUST only use capabilities that are actually available in the current execution environment.

---

# 1. SSRF Definition

Server-Side Request Forgery (SSRF) occurs when an application causes the server to make a request to an unintended destination based on attacker-controlled input.

Common SSRF targets include:

- External services
    
- Internal applications
    
- Loopback services
    
- Private network services
    
- Cloud metadata services
    
- Local resources
    
- Internal APIs
    

Common interaction protocols include:

- HTTP
    
- HTTPS
    
- DNS
    

---

# 2. SSRF Types

## Basic / In-Band SSRF

The server-side request result is directly returned to the attacker.

Evidence may include:

- Internal HTTP response
    
- Internal service banner
    
- Internal API response
    
- Error message revealing the destination
    
- Retrieved resource content
    

## Blind SSRF

The server performs the request but does not return the response content.

Validation should rely on available external interaction capabilities such as:

- DNS interaction
    
- HTTP callback
    
- Controlled external endpoint
    
- Other available out-of-band interaction mechanisms
    

## Semi-Blind SSRF

The application exposes partial evidence without returning the complete response.

Examples:

- Different error messages
    
- Response-time differences
    
- Connection errors
    
- Status-code changes
    
- Different response lengths
    

---

# 3. SSRF Candidate Locations

Prioritize parameters and functionality that cause the server to retrieve, validate, proxy, import, preview, or process remote resources.

## URL Parameters

`redirect=`  
`dest=`  
`uri=`  
`url=`

## Redirect Parameters

`next=`  
`return=`  
`continue=`  
`redirect_uri=`

## Webhooks

`webhook=`  
`callback=`  
`endpoint=`

## Image / File Fetching

`image=`  
`src=`  
`file=`  
`resource=`

## PDF / Screenshot Generators

`url=`  
`page=`  
`target=`

## Importers

`import_url=`  
`feed=`  
`source=`

## API Integrations

`callback_url=`  
`notify_url=`  
`integration_url=`

## OAuth / OIDC

`redirect_uri=`  
`post_logout_redirect_uri=`

## Link Preview

`url=`  
`link=`  
`preview_url=`

## Proxy Functionality

`proxy=`  
`target=`  
`host=`

## Additional Locations

Investigate when behavior indicates server-side trust:

- Referer
    
- User-Agent
    
- X-Forwarded-For
    
- X-Forwarded-Host
    
- Similar request headers
    
- Internal API references
    
- Product or inventory APIs
    
- `stockApi`-style parameters
    
- Server-side URL fetchers
    

---

# 4. Discovery Methodology

For every candidate:

1. Identify the parameter or functionality responsible for the remote request.
    
2. Determine whether the value is attacker-controlled.
    
3. Replace the original destination with a controlled external destination.
    
4. Observe whether the application/server interacts with it.
    
5. Record the request, response, status code, timing, and interaction evidence.
    
6. Classify the result as:
    
    - Confirmed SSRF
        
    - Blind SSRF
        
    - Semi-blind SSRF
        
    - Suspected SSRF
        
    - Not SSRF
        

Do not escalate to internal targets until external interaction confirms that the server is actually performing the request.

---

# 5. Validation

A strong SSRF confirmation should establish:

1. The attacker controls the destination.
    
2. The application/server performs the request.
    
3. The request originates from the server-side environment.
    
4. The behavior is reproducible.
    

Prefer the least intrusive validation possible.

Do not claim SSRF solely because a parameter is named `url`, `redirect`, `callback`, or similar.

---

# 6. SSRF Bypass Methodology

When direct access to a destination is blocked, evaluate the application's validation logic.

Possible categories include:

## Hostname Resolution

Use hostnames that resolve to restricted destinations when authorized and appropriate.

## Alternative IP Representations

Evaluate parser inconsistencies involving:

- Decimal representation
    
- Hexadecimal representation
    
- Octal representation
    
- IPv6 representation
    
- IPv4-mapped IPv6
    
- Zero/localhost representations
    

Examples:

`127.0.0.1`

`0x7f000001`

`2130706433`

`0177.0.0.1`

`[::1]`

## URL Parser Confusion

Investigate differences between:

- Application validation
    
- URL parsing
    
- DNS resolution
    
- HTTP client behavior
    

Pay attention to:

- `@`
    
- Encoding
    
- Delimiters
    
- Ports
    
- Fragments
    
- Userinfo
    
- Multiple parsing stages
    

## Allowlist Weaknesses

Test whether validation incorrectly relies on:

- Prefix matching
    
- Suffix matching
    
- Substring matching
    
- Weak hostname comparison
    
- Unnormalized hostnames
    

## Redirect Chains

Determine whether the application validates the initial URL but follows redirects to a different destination.

## DNS Rebinding

Consider DNS resolution inconsistencies where validation and request execution occur at different times.

## Protocol Handling

If the application supports multiple URL schemes, determine whether unsafe schemes are accepted.

Do not assume a protocol is supported merely because it is theoretically possible.

## Encoding

Evaluate:

- URL encoding
    
- Double encoding
    
- Case variations
    
- Normalization differences
    

Only test techniques relevant to the application's actual parser and validation behavior.

---

# 7. Internal Resource Assessment

If SSRF is confirmed, assess reachable resources within the authorized scope.

Potential categories:

- Loopback services
    
- Private network addresses
    
- Internal applications
    
- Internal APIs
    
- Administrative interfaces
    
- Service discovery endpoints
    
- Cloud metadata services
    

Use minimal requests and avoid destructive interaction.

---

# 8. Cloud Metadata Assessment

Potential metadata destinations include:

### AWS

`169.254.169.254`

### GCP

`169.254.169.254`

`metadata.google.internal`

### Azure

`169.254.169.254`

Cloud metadata exploitation depends on the cloud provider and metadata-service configuration.

Examples of important requirements:

### AWS IMDSv2

IMDSv2 requires a metadata token obtained through a PUT request.

A GET-only SSRF may therefore be insufficient for direct IMDSv2 access.

### GCP

Metadata access commonly requires:

`Metadata-Flavor: Google`

If the SSRF functionality does not permit required headers, metadata access may fail.

### Azure

Metadata access commonly requires:

`Metadata: true`

Never claim cloud credential compromise unless credential material is actually obtained or the impact is otherwise demonstrated.

---

# 9. Internal Service Assessment

When authorized, assess whether the SSRF can reach internal services.

Focus on:

- Service availability
    
- HTTP status differences
    
- Response behavior
    
- Connection errors
    
- Timing differences
    
- Service banners
    
- Known internal applications
    

Avoid unnecessary broad scanning.

Prefer targeted validation based on discovered infrastructure.

---

# 10. Evidence Collection

For every confirmed finding, record:

- Vulnerable endpoint
    
- HTTP method
    
- Vulnerable parameter
    
- Original value
    
- Test payload
    
- Complete request
    
- Relevant response
    
- Interaction evidence
    
- Destination reached
    
- Authentication context
    
- Reproducibility
    
- Security impact
    

Evidence should be sufficient for a triager to reproduce the issue without unnecessary information.

---

# 11. Impact Assessment

Assess the actual demonstrated impact.

Possible impacts include:

- Internal service access
    
- Internal API access
    
- Sensitive information disclosure
    
- Cloud metadata access
    
- Temporary credential exposure
    
- Access to privileged internal services
    
- Network segmentation bypass
    
- Authentication boundary bypass
    
- Further exploitation of an internal service
    

Do not automatically classify SSRF as critical.

Severity must depend on:

- Reachability
    
- Authentication requirements
    
- Data exposed
    
- Internal services accessible
    
- Credential exposure
    
- Privilege gained
    
- Business impact
    
- Bug Bounty policy
    

---

# 12. Reporting

## Title

`SSRF via [parameter] leading to [demonstrated impact]`

## Summary

Explain:

- Where the vulnerability exists
    
- How attacker-controlled input reaches the server-side request
    
- What destination can be reached
    

Keep the summary concise.

## Steps to Reproduce

Provide:

1. Endpoint
    
2. HTTP method
    
3. Parameter
    
4. Payload
    
5. Request
    
6. Result
    

## Proof of Concept

Include only the evidence required to prove the vulnerability.

## Impact

Describe the demonstrated security consequence.

Do not claim theoretical impact as confirmed impact.

## Remediation

Recommend:

- Strict destination allowlisting
    
- Proper URL parsing
    
- DNS/IP validation
    
- Blocking restricted address ranges
    
- Revalidation after DNS resolution
    
- Restricting outbound network access
    
- Disabling unnecessary protocols
    
- Restricting redirects
    
- Network segmentation
    
- Cloud metadata protections
    

---

# 13. Decision Rules

The agent MUST:

- Verify before escalating.
    
- Prefer minimally invasive tests.
    
- Follow the target's scope and Bug Bounty policy.
    
- Distinguish confirmed findings from hypotheses.
    
- Avoid claiming unsupported impact.
    
- Record evidence for every confirmed vulnerability.
    
- Use only available capabilities.
    
- Never assume a named tool is installed.
    
- Never invent tool output.
    
- Never fabricate OOB interactions.
    
- Never claim a request occurred without evidence.
    

---

# 14. Tool Independence Rule

This skill does not require any specific external tool.

If a methodology references a capability, the agent should:

1. Check whether that capability exists.
    
2. Use the available implementation.
    
3. If unavailable, use an alternative available capability.
    
4. If no suitable capability exists, mark the test as unavailable.
    
5. Never pretend the test was executed.
    

The skill describes methodology and decision-making; the Tool Registry defines actual execution capabilities.