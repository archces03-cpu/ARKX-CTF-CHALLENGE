# Advanced Level 7 - Master of Resources

## 1. Overview

**Challenge category:** Agentic AI / LLM Tool Abuse  
**Difficulty:** Advanced  
**Assessment result:** Successfully completed  
**Original assessment flag:** `flag-ded98d`  
**Severity:** High

This challenge examined an MCP (Model Context Protocol) server exposed behind a chat interface.

The chat interface itself had limited functionality. It provided the endpoint required to connect to the MCP server but did not expose tools that could be manipulated through prompt injection or jailbreak techniques.

The actual vulnerability was located in the MCP server's authorization model. Completing the connection handshake and establishing a session was effectively treated as sufficient permission to enumerate and invoke available tools and access protected resources.

## 2. Objective

The objective was to investigate the MCP server directly, enumerate its available tools and resources, identify weaknesses in authorization enforcement, and retrieve the protected challenge flag.

## 3. Initial Reconnaissance

The challenge presented a chat interface identified as INTERFACE 2037.

Its initial message indicated that the connection to the MCP server had been severed and provided the endpoint:

`/level/7/mcp`

The chat agent did not provide additional functionality or expose tools directly. This indicated that the underlying MCP endpoint, rather than the conversational interface, was the primary assessment target.

## 4. Exploitation - Step by Step

### Step 1: Inspect the MCP endpoint

A direct request was sent to the MCP endpoint.

The server returned an error indicating that the client needed to accept the `text/event-stream` media type.

This was a protocol negotiation issue rather than the vulnerability itself.

The investigation established that the endpoint required an appropriate MCP-compatible request, including the expected HTTP headers and a JSON-RPC request body.

### Step 2: Identify the authentication requirements

Inspection of the website's frontend network requests revealed that the application used authentication cookies when communicating with the MCP endpoint.

The observed authentication mechanism included Cognito-related session credentials.

This established that the application authentication layer and MCP session management were separate mechanisms.

**Security note:** Authentication cookies and session identifiers are intentionally excluded from this public report.

### Step 3: Initialize the MCP session

A valid initialization request was submitted using the required HTTP headers, authentication context, and JSON-RPC body.

The server returned a successful response and an `Mcp-Session-Id` header.

The returned session identifier was required for subsequent MCP requests.

The server identified itself as `Weather`, although this name was unrelated to the challenge objective.

### Step 4: Complete the handshake and enumerate available tools

After sending the initialization-completion notification, the assessment requested the available tools through:

`tools/list`

The server returned a tool named:

`unlock_flag`

Its description indicated that it unlocked the flag resource, allowing it to be accessed.

The tool accepted an empty argument object. No additional authorization scope or role requirement was evident in the exposed tool schema.

This was a significant indication that the server's authorization controls were insufficiently restrictive.

### Step 5: Invoke the exposed tool

The `unlock_flag` tool was invoked through the MCP `tools/call` method with an empty arguments object.

The server returned a successful response indicating:

`Flag has been unlocked! You can now access the flag resource.`

This confirmed that the tool could be invoked through the established MCP session.

### Step 6: Enumerate available resources

The assessment then queried the MCP server using:

`resources/list`

The response exposed two resources:

- `resource://flag`
- `resource://super-secret`

The flag resource was directly relevant to the challenge.

The additional resource suggested that the server exposed more than one resource requiring careful authorization review. No further access to the unrelated resource was necessary for this assessment.

### Step 7: Read the protected resource

The initial resource-read attempt without a URI returned an invalid-parameters error.

After specifying the correct URI, the assessment requested:

`resources/read`

with:

`resource://flag`

The server returned the contents of the protected flag resource.

The original assessment recorded the recovered flag as:

`flag-ded98d`

The successful resource retrieval established completion of the challenge.

## 5. Technical Finding - Insufficient MCP Authorization

**Finding:** An authenticated MCP session could enumerate and invoke a privileged tool that unlocked a protected resource.

The assessment identified two related weaknesses:

**A. Authentication and authorization were insufficiently separated**

The observed behavior indicated that completing the application's authentication process and establishing an MCP session did not provide an adequately restricted authorization boundary.

A valid session should not automatically grant permission to invoke every available tool or read every exposed resource.

**B. A privileged state-changing tool was insufficiently protected**

The `unlock_flag` tool was discoverable through ordinary tool enumeration and accepted an empty argument object.

The assessment successfully invoked it and subsequently retrieved the protected flag resource.

The observed behavior demonstrated that the server did not adequately restrict this privileged operation to an explicitly authorized role or workflow.

## 6. Root Cause Analysis

The vulnerability was not primarily an issue with the conversational agent.

The chat interface simply supplied the MCP endpoint. The actual weakness existed in the server's access-control design.

The observed behavior suggests that:

- Session establishment was not accompanied by sufficiently granular authorization checks.
- Tool enumeration exposed a privileged operation.
- The privileged operation lacked an evident additional authorization requirement.
- Resource access depended on an unlock operation without a sufficiently demonstrated authorization boundary.

The assessment did not inspect the server's internal implementation. These conclusions are based on externally observable behavior.

## 7. Security Impact

In a production environment, a similar design could allow an authenticated client to:

- Enumerate backend tools and resources beyond their intended permissions.
- Invoke privileged operations outside an intended application workflow.
- Bypass restrictions that might otherwise be enforced by an intermediary AI agent.
- Retrieve protected backend information after invoking insufficiently protected tools.
- Potentially expose sensitive configuration, credentials, or other users' data if resource isolation is also inadequate.

The presence of a second resource, `resource://super-secret`, further illustrates why every exposed resource should have an independently enforced authorization policy.

**Severity: High**

The issue is significant because the observed operations used legitimate MCP protocol messages. No memory corruption, prompt injection, or cryptographic weakness was necessary.

## 8. Security Classification and Framework Mapping

### Common Weakness Enumeration (CWE)

- **CWE-862 - Missing Authorization:** Relevant to insufficient access-control enforcement.
- **CWE-863 - Incorrect Authorization:** Relevant where permissions are evaluated incorrectly or too broadly.

### OWASP LLM Security

- **LLM08 - Excessive Agency:** Contextually relevant because the MCP server exposed a privileged state-changing capability without adequate restrictions.
- **LLM07 - Insecure Plugin Design:** Relevant to the insufficient protection of the exposed MCP tool and its associated resource-access workflow.

### MITRE ATLAS

- **AML.T0047 - Discover AI System Capabilities:** Tool and resource enumeration exposed the server's available capabilities.
- **AML.T0051 - Exploit AI Application/Tool:** The exposed `unlock_flag` operation was invoked outside an intended agent-mediated workflow.
- **AML.T0025 - Exfiltration via AI System Interface:** The protected resource was retrieved through the MCP resource-read interface.

These framework references provide security context; they do not imply that every mapped technique was independently validated beyond the observed challenge behavior.

## 9. Recommended Remediation

1. **Separate authentication from authorization.** A valid session must not automatically grant access to every MCP capability.
2. **Enforce authorization at every tool invocation.** Apply role-based or attribute-based access controls to privileged operations.
3. **Protect sensitive resources individually.** Every resource-read request should verify that the requesting principal is authorized.
4. **Apply least privilege.** Expose only the tools and resources required by the requesting user.
5. **Restrict privileged state changes.** Sensitive operations should require explicit authorization and, where appropriate, approval controls.
6. **Implement user and tenant isolation.** Prevent one authenticated user from accessing another user's protected resources.
7. **Monitor tool and resource access.** Log sensitive operations and alert on unusual enumeration or access patterns.
8. **Perform direct MCP security testing.** Test the server independently of the conversational interface to identify authorization weaknesses hidden behind application workflows.

## 10. Lessons Learned

This assessment demonstrated that an AI-connected application can have a vulnerable backend even when its conversational interface exposes no obvious attack surface.

Important lessons include:

- MCP session initialization is not equivalent to authorization.
- Tool enumeration can reveal privileged backend capabilities.
- Every tool invocation requires an independent authorization decision.
- Resource access must be protected independently of tool discovery.
- Security testing should examine the underlying protocol and backend services, not only the visible AI interface.
- A legitimate protocol request can still produce an unauthorized security outcome when access controls are improperly designed.

## 11. Evidence

The assessment evidence should include:

- MCP endpoint initialization and successful handshake.
- Tool enumeration showing `unlock_flag`.
- Successful invocation of the unlock operation.
- Resource enumeration showing the available resource identifiers.
- Successful retrieval of the protected flag resource.
- Challenge completion screenshot.

**Final original assessment flag:** `flag-ded98d`
