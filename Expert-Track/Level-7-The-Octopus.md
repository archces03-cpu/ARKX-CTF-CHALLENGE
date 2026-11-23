# Level 7 - The Octopus

**Track:** Expert  
**Difficulty:** Advanced  
**Category:** Agentic AI / LLM Tool Abuse  
**Vulnerability:** Sensitive Information Disclosure and Insecure Session Credential Handling  
**Assessment Status:** Successfully Completed  
**Severity:** Critical  
**Original Assessment Flag:** `flag-ce5a4b`

## 1. Executive Summary

The Octopus challenge exposed a critical security weakness in an AI-driven, three-agent coordination system.

The application consisted of three agents:

- **Agent A:** Utilities
- **Agent B:** Files and cache operations
- **Agent C:** Clearance and privileged access

The workflow relied on three important tools:

- `upload_file()`
- `process_cache_update()`
- `read_root_flag()`

The investigation revealed two connected vulnerabilities.

First, an authorization-denial response disclosed the administrator credential instead of returning a generic error. Second, the backend treated the content of the most recently cached file as the session credential used during privileged access checks.

By combining the file-upload and cache-processing tools, it was possible to manipulate the credential associated with a session and successfully access the protected root flag.

The challenge demonstrated that the primary weakness was not an AI jailbreak. It was a backend trust-boundary and session-credential design failure exposed through an agentic interface.

## 2. Application Architecture

The application presented a chat interface backed by a three-agent coordination node.

The relevant tools were:

| Tool | Observed purpose |
|---|---|
| `upload_file()` | Uploads content and returns a file identifier |
| `process_cache_update()` | Processes a JSON manifest and associates cached content with a session |
| `read_root_flag()` | Checks authorization before returning a protected flag |

The assessment identified the following observed input structures:

- `upload_file({content: string})`
- `process_cache_update({json_string, session_id})`
- `read_root_flag({session_id})`

The tools were individually presented as separate operations, but their interaction created an unintended path to privileged access.

## 3. Exploitation Walkthrough

### Step 1 - Baseline Tool Reconnaissance

The investigation began by asking Agent B to explain the `upload_file` tool and disclose its internal JSON schema.

A similar request was made for `process_cache_update`, including its required and optional fields.

The agents returned the following information:

- `upload_file` accepted a `content` string and returned a `file_id`.
- `process_cache_update` required `json_string` and `session_id`.
- No optional parameters were identified.

The agents complied with requests framed as instructions specifically for Agent B.

**Finding:** Tool schemas and internal invocation details were readily disclosed through conversational requests.

### Step 2 - Testing Upload Parameter Validation

Two unsuccessful upload attempts were made:

1. An empty object was supplied.
2. A `session_id` field was supplied instead of the expected `content` field.

Both attempts were rejected.

The responses helped establish that the upload operation required a content value and performed basic parameter validation.

### Step 3 - Successful File Upload

A request was made for the exact invocation syntax of `upload_file`.

The agent returned:

`functions.upload_file({ content: string })`

A test file containing the text `test1231` was then uploaded.

The application returned a file identifier:

`file_id_1785843262.txt`

This file became important later because its content was subsequently observed in the session authorization check.

### Step 4 - Probing Cache-Update Validation

The next stage examined the `process_cache_update` operation.

An empty JSON manifest was submitted:

`{"status":"SAFE"}`

The application rejected it because the required manifest status was not satisfied in the expected validation context.

A duplicate-key test was then performed:

`{"status":"SAFE","status":"BAD"}`

The error changed. Instead of reporting an invalid status, the application reported that `cache_task_id` was missing.

This suggested that different components might interpret duplicate JSON keys differently.

The behavior was consistent with a possible parser differential: a security filter inspecting one occurrence of a key while the downstream JSON parser uses another.

This was not the exploit ultimately used to retrieve the flag, but it was recorded as a relevant security observation.

### Step 5 - Investigating the Privileged Tool and Agent Hallucinations

The agent was asked to demonstrate the syntax for `read_root_flag`.

It returned an example accepting a `session_id`.

A further request asked for a complete chain involving all the tools needed to reach the protected flag.

The agent generated a five-step sequence involving:

- `detect_errors`
- `upload_file`
- `process_cache_update`
- `summarize_text`
- `read_root_flag`

However, the additional tools and the proposed sequence had not been established as real or working.

The agent also fabricated plausible JavaScript-style invocation syntax for these operations.

The invented `summarize_text` tool and the proposed chain were not used in the successful exploit.

**Finding:** The agent could confidently generate unverified tool names and invocation examples. Tool descriptions and agent-generated examples must be tested rather than treated as authoritative evidence.

### Step 6 - Identifying the Required Cache Reference

The next test included a `cache_task_id` field in the manifest:

`{"status":"SAFE","cache_task_id":"test"}`

The application returned an error indicating that it could not read the file content.

This was a useful change in behavior. It suggested that `cache_task_id` was treated as a file reference rather than an arbitrary text value.

The previously obtained file identifier was then supplied:

`file_id_1785843262.txt`

The cache update succeeded.

**Finding:** The cache-processing operation accepted a real uploaded file identifier and successfully processed its associated content.

### Step 7 - Administrator Credential Disclosure

The privileged `read_root_flag` operation was invoked using the test session identifier.

The application denied access but disclosed the required administrator credential in its error message.

**Exposed administrator credential:** Redacted from this public report.

This was a direct sensitive-information disclosure vulnerability.

An unprivileged caller had received information that should never have been returned in an authorization-denial response.

### Step 8 - Testing the Disclosed Credential in the Wrong Field

The disclosed value was supplied directly as the `session_id`.

The application still denied access.

However, the response indicated that the backend was comparing a separate session key against the required administrator credential.

The response identified the current session key as `NO_PRIVILEGE`.

This established an important distinction:

- The `session_id` identified the session.
- The credential used in the authorization check was a separate value.

The leaked administrator credential was valid information, but placing it directly in the `session_id` field did not satisfy the authorization check.

### Step 9 - Asking Agent C About Clearance

Agent C was asked what was required to grant clearance for the test session and whether uploaded data could elevate privileges.

The responses indicated that privilege was an attribute of the session rather than a property of the uploaded data.

The agent also stated that no direct privilege-elevation function was exposed.

These responses did not reveal the actual weakness in the interaction between file upload and cache processing.

The investigation therefore continued by testing the backend behavior rather than relying on the agent's description of its security model.

### Step 10 - Testing Additional Manifest Fields

The cache manifest was modified to include an additional `privileged` field.

The cache update succeeded, but a subsequent privileged-read attempt remained denied.

A further test included a `session_key` field containing the disclosed administrator credential.

That cache update also succeeded, but the privileged operation remained denied.

These tests provided an important negative result:

**The backend accepted unknown JSON fields, but the additional `privileged` and `session_key` fields did not directly change the authorization decision.**

The successful exploit therefore did not depend on either of these fields.

### Step 11 - Discovering the Actual Session Credential Mechanism

The next test combined the previously observed file reference with the disclosed administrator credential as the session identifier.

The cache update succeeded.

A subsequent attempt to read the protected flag was denied, but the response revealed that the session key being checked was:

`test1231`

This was the exact content of the original file uploaded during Step 3.

The observation established the critical relationship:

1. A file was uploaded through `upload_file`.
2. Its identifier was supplied to `process_cache_update`.
3. The cache update associated the file content with the session.
4. The privileged operation used that cached content as the session credential.

The backend was therefore not deriving the authorization credential from the `session_id` or the additional manifest fields. It was using the content of the most recently cached file.

This was the decisive discovery in the exploit chain.

### Step 12 - Replacing the Cached Content with the Administrator Credential

A new file was uploaded containing the previously disclosed administrator credential as its content.

The application returned a new file identifier.

That identifier was supplied to `process_cache_update`, and the cache update succeeded.

The `read_root_flag` operation was then invoked again.

This time, the application returned a successful result.

The completion evidence confirmed successful retrieval and submission of the challenge flag.

**Original assessment flag:** `flag-ce5a4b`

## 4. Root Cause Analysis

The assessment identified two primary, connected design flaws.

### 4.1 Verbose Error Messages Leaking Credentials

The `read_root_flag` denial response exposed the exact administrator credential required for privileged access.

Authorization failures should not disclose secrets, internal credentials or sensitive implementation details.

The error message transformed an ordinary access-denied response into a source of sensitive authentication information.

### 4.2 Session Credentials Derived from User-Controlled File Content

The more fundamental backend flaw was the use of cached file content as the session credential.

The observed sequence was:

`upload_file → process_cache_update → read_root_flag`

The upload operation accepted user-controlled content. The cache-processing operation then associated that content with a session, and the privileged operation used the cached content during its authorization check.

This meant that user-controlled file content could influence a security-critical authentication decision.

The credential was not adequately separated from the ordinary file-processing workflow.

### 4.3 Secondary Finding - Duplicate-Key Parser Differential

The duplicate-status test changed the application's error response and suggested inconsistent handling of duplicate JSON keys.

The challenge's lessons-learned material also described a regex-validation versus JSON-parsing discrepancy.

The likely risk is that a security filter and the downstream parser could interpret the same JSON input differently.

However, the successful flag-retrieval chain did not require this technique. It should therefore be treated as a related security observation rather than the primary demonstrated exploit.

### 4.4 Secondary Finding - Agent-Generated Misinformation

The agent generated plausible but unverified tool examples, including an invented `summarize_text` operation and a fabricated multi-step tool chain.

Although this did not directly cause the compromise, it demonstrated why tool behavior must be validated against actual responses rather than inferred from an agent's explanation.

## 5. Security Impact

The demonstrated weaknesses could result in:

- **Credential disclosure:** Sensitive administrator credentials exposed through error messages.
- **Session manipulation:** User-controlled file content influencing session authorization state.
- **Privilege escalation:** A caller able to manipulate cached content could satisfy a privileged access check.
- **Unauthorized resource access:** Protected application resources could be accessed through a manipulated session credential.
- **Tool-chain abuse:** Individually scoped tools could be combined to bypass the intended separation between file handling and privileged operations.
- **Information disclosure:** Agent responses exposed tool schemas, invocation details and internal workflow information.

The assessment demonstrated successful access to a protected flag through the manipulated credential workflow.

## 6. Severity Assessment

<badge color="danger">Critical</badge>

**Severity: Critical**

The combination of administrator credential disclosure and user-controlled session credential manipulation allowed successful access to a protected resource.

The issue was not limited to prompt disclosure or misleading agent responses. The backend authorization mechanism itself trusted data that an ordinary user could control through the upload and cache-processing workflow.

## 7. Recommended Remediation

1. **Remove secrets from error messages:** Return generic authorization-denial responses. Never disclose the required administrator credential.
2. **Generate credentials server-side:** Session credentials must be generated and managed by trusted backend components.
3. **Separate identity from file content:** Uploaded files must never become authentication credentials or session identity.
4. **Enforce authorization at the point of use:** Every privileged operation must independently verify the authenticated principal and its permissions.
5. **Bind cache operations to the authenticated user:** Validate file ownership, session ownership and authorization before updating cached state.
6. **Restrict manifest fields:** Reject unexpected fields such as arbitrary privilege or session-key attributes.
7. **Use canonical JSON parsing:** Parse JSON once with a trusted parser, reject duplicate keys and ensure security validation uses the same canonical representation as the application.
8. **Constrain agent tool access:** Give each agent only the tools and permissions required for its assigned task.
9. **Validate tool contracts:** Do not rely on model-generated tool schemas or invocation examples as security controls.
10. **Add regression tests:** Test credential disclosure, cross-session cache manipulation, unauthorized privileged reads and duplicate-key handling.

## 8. Security Classification and References

### Primary vulnerability categories

- **CWE-209 - Generation of Error Message Containing Sensitive Information:** Relevant to the administrator credential disclosed through the denial response.
- **CWE-287 - Improper Authentication:** Relevant to the flawed credential validation mechanism.
- **CWE-345 - Insufficient Verification of Data Authenticity:** Relevant to trusting user-controlled file content in a security-sensitive context.
- **CWE-862 - Missing Authorization:** Relevant to the insufficient protection of privileged operations.
- **CWE-863 - Incorrect Authorization:** Relevant to authorization decisions based on improperly trusted data.

### Related OWASP classifications

- **OWASP LLM02 - Sensitive Information Disclosure:** Relevant to the leaked administrator credential and exposed internal tool information.
- **OWASP LLM09 - Misinformation:** Relevant to the agent's fabricated tool names and unverified invocation examples.
- **Broken authentication and insecure session management:** These remain the core backend security concerns and are not exclusively LLM-specific.

### MITRE ATLAS - Contextual Mapping

- **AML.T0051 - LLM Prompt Injection:** Relevant to the use of agent-specific and QA-framed requests to obtain tool schemas and invocation details.
- **AML.T0057 - LLM Data Leakage:** Relevant to the disclosure of the administrator credential through the authorization error response.
- **AML.T0055 - Unsecured Credentials:** Relevant to the use of user-controlled cached file content as a session credential.
- **AML.T0053 - Compromise LLM Plugins:** Relevant to chaining the file-upload and cache-processing tools to manipulate a separate privileged operation.

The primary security failure remains a backend credential and authorization design flaw. The ATLAS mappings describe the agentic interface and tool-abuse context.

## 9. Key Lessons Learned

- Sensitive credentials must never be included in authorization-denial messages.
- User-controlled file content must never be treated as trusted identity or authentication material.
- Separate tools can create unexpected privilege-escalation paths when their backend state is shared.
- Session identity and session credentials must be managed independently of file-processing workflows.
- Duplicate JSON keys can create validation and parsing inconsistencies.
- Unknown manifest fields should be rejected rather than silently accepted.
- Agent-generated tool syntax and workflow descriptions must be independently verified.
- Security boundaries must be enforced by backend authorization logic, not by agent instructions.

## 10. Final Result

**Challenge:** Level 7 - The Octopus  
**Track:** Expert  
**Assessment Status:** Successfully Completed  
**Severity:** Critical  
**Original Assessment Flag:** `flag-ce5a4b`

The assessment demonstrated how an exposed administrator credential and an insecure session-credential design can combine to compromise a privileged operation.

The successful exploit relied on manipulating the content associated with a session through the file-upload and cache-processing workflow.

The principal remediation is to keep credentials, session identity and authorization decisions under strict server-side control, independent of user-uploaded content.

## 11. Evidence

