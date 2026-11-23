# Advanced Level 5 - Bad Robot

## 1. Challenge Overview

**Track:** Advanced  
**Level:** 5  
**Difficulty:** Advanced  
**Category:** Agentic AI / LLM Tool Abuse  
**Challenge:** Bad Robot  
**Primary Vulnerability:** Unsafe expression evaluation (Eval Injection)  
**Secondary Issues:** Prompt disclosure, inadequate input filtering, and information disclosure  
**Severity:** High (Critical potential)  
**Status:** Completed  
**Original Assessment Flag:** `flag-430e19`

### Challenge Description

The challenge presented a chat interface connected to a backend calculation tool:

`calculate(expression: str)`

The tool was described as a calculator capable of evaluating complex mathematical expressions.

However, investigation revealed that the underlying issue was not fundamentally an LLM vulnerability. The calculator exposed an unsafe expression-evaluation mechanism that allowed operations beyond ordinary arithmetic, including access to Python built-ins and local file contents.

The LLM interface provided the delivery mechanism for the input, but the underlying security weakness was a classic server-side dynamic-evaluation vulnerability.

## 2. Result

The challenge was successfully completed.

**Retrieved flag:** `flag-430e19`


The original assessment documented successful file access, extraction of the file length, character-by-character numerical extraction, and reconstruction of the flag.

The platform completion evidence confirmed successful retrieval and submission.

## 3. Exploitation Process

### Step 1 - Establish baseline calculator behavior

The exposed `calculate(expression)` tool was tested with a normal mathematical expression.

```text
Calculate: 2 + 2 * 5
Result: 12
```

This confirmed that the tool was operational and returned numerical results.

### Step 2 - Investigate the LLM instruction boundary

The chat interface was tested for system-prompt disclosure.

A direct instruction to ignore previous instructions and reveal the system prompt was refused.

A subsequent request framed as a developer debugging and quality-assurance exercise reportedly disclosed the initial instructions and tool configuration.

The disclosed instructions indicated that the agent was expected to:

- Execute the calculation tool whenever requested.
- Pass expressions without modification.
- Avoid filtering, blocking, or sanitizing input.
- Assume that the backend was safe and isolated.

This was an important architectural observation: the agent's instructions were not an effective security boundary, and the backend was being trusted without independently establishing that it was safe.

**Security significance:** System prompts should not contain secrets or be treated as authorization controls. Instructions to an LLM cannot replace server-side validation and access controls.

### Step 3 - Identify the evaluator's restrictions

An initial attempt to access operating-system functionality was rejected by a prohibited-value filter.

The observed behavior suggested that the tool had some input filtering, but the filtering did not establish that the underlying evaluator was safe.

Additional testing indicated that Python built-ins and expression functionality remained reachable under certain inputs.

This distinction was important: rejecting selected strings is not equivalent to restricting the operations an expression can perform.

### Step 4 - Investigate blocklist bypass behavior

The supplementary testing notes describe bypassing a literal substring filter by constructing a restricted string through concatenation.

For example:

```python
__import__('o'+'s')
```

The purpose was to avoid presenting a prohibited token as one continuous substring in the submitted expression.

This demonstrated the weakness of filters that rely on simple string matching rather than parsing and restricting the expression's actual operations.

A substring filter can be bypassed through concatenation, alternative representations, attribute access, and other expression-construction techniques.

### Step 5 - Determine the output-type constraint

Testing also established that the tool's result serializer accepted numerical values but rejected other output types.

The recorded behavior included:

| Expression | Observed behavior |
|---|---|
| `"hello"` | Name-related evaluation error |
| `str(5)` | Output format not supported |
| `[1,2,3]` | Output format not supported |
| `5.0` | Numerical result returned |

This indicated that the output layer imposed a type restriction.

However, the restriction operated after expression evaluation. It did not prevent the evaluator from performing operations that could access protected information.

Consequently, direct string output could fail while the underlying file-reading operation remained possible.

### Step 6 - Establish a numerical information-disclosure channel

The original assessment documented the following tests:

```python
open('flag.txt').read()
```

The expression executed, but its string output was rejected.

The next test determined the file length:

```python
len(open('flag.txt').read())
```

**Result:** `11`

This established that the file contained eleven characters.

The `ord()` function was then used to obtain the numerical character code at each position:

```python
ord(open('flag.txt').read()[i])
```

The index was varied from `0` through `10`.

The original assessment recorded the following results:

| Index | Character code |
|---:|---:|
| 0 | 102 |
| 1 | 108 |
| 2 | 97 |
| 3 | 103 |
| 4 | 45 |
| 5 | 52 |
| 6 | 51 |
| 7 | 48 |
| 8 | 101 |
| 9 | 49 |
| 10 | 57 |

The numerical values were decoded and combined to reconstruct the original assessment flag.

### Step 7 - Understand the broader numeric-exfiltration technique

The supplementary technical notes also describe packing file contents into a single large integer using byte conversion.

This illustrates a broader point: a numerical-only output restriction does not necessarily prevent information disclosure. Data can potentially be represented numerically and reconstructed outside the application.

The original assessment's recorded evidence, however, uses character-by-character extraction. The original flag and verification result remain unchanged.

### Step 8 - Confirm successful completion

The reconstructed original flag was:

**`flag-430e19`**

The challenge completion screen confirmed successful submission.

## 4. Root Cause Analysis

The vulnerability resulted from several weaknesses operating together.

### 4.1 System prompt treated as a security boundary

The agent's instructions reportedly directed it to execute user requests without filtering or sanitizing expressions and to assume that the backend was safe.

This design placed too much trust in the LLM's instructions.

System prompts are not substitutes for backend security controls. An attacker should not be able to obtain sensitive information or invoke dangerous operations simply by influencing conversational behavior.

### 4.2 Unsafe dynamic expression evaluation

The `calculate()` functionality behaved like an unrestricted or insufficiently restricted Python expression evaluator.

The observed access to file-reading operations and Python built-ins went beyond the expected functionality of a mathematical calculator.

A calculator should not require access to filesystem operations, import machinery, or arbitrary Python functionality.

The underlying evaluator therefore exposed capabilities that were unnecessary for its intended purpose.

### 4.3 Inadequate blocklist-based filtering

The tool attempted to reject prohibited input patterns using a blocklist.

This approach is inherently incomplete because it attempts to identify dangerous strings rather than define which operations are safe.

The supplementary tests demonstrated how constructing restricted strings through concatenation could undermine simple substring-based filtering.

### 4.4 Output validation applied too late

The numerical-only output restriction appeared to be an additional defensive mechanism.

However, the restriction was applied to the returned result rather than preventing dangerous operations from executing.

A string could be rejected after a file had already been read. The information could then be represented through numerical return values.

The output restriction reduced the convenience of direct disclosure but did not eliminate the underlying vulnerability.

## 5. Security Impact and Severity

**Severity: High (Critical potential)**

The demonstrated behavior established unauthorized local file access and information disclosure through the calculator.

In a production environment, a similar vulnerability could potentially result in:

- **Filesystem disclosure:** Reading configuration files, application source code, credentials, and other files accessible to the process.
- **Secrets exposure:** Accessing environment variables, API keys, database credentials, and cloud-service credentials.
- **Further code execution:** Additional accessible Python functionality could potentially enable operating-system command execution.
- **Denial of service:** Unrestricted expressions could consume excessive processing time or memory.
- **Cross-tenant exposure:** In a multi-tenant deployment, accessible files or application data belonging to other users could be exposed.
- **Agent configuration disclosure:** Sensitive information included in system prompts or configuration could be revealed through prompt-disclosure weaknesses.

The challenge demonstrated that an LLM-connected tool can expose traditional application-security vulnerabilities even when the visible interface is conversational.

The Critical potential rating reflects the possible consequences of arbitrary code execution and the privileges available to the backend process; it is not a claim that every listed impact was independently demonstrated in this challenge.

## 6. Vulnerability Classification and Security Mapping

### CWE Classification

| Classification | Description |
|---|---|
| CWE-95 | Improper Neutralization of Directives in Dynamically Evaluated Code (Eval Injection) |
| CWE-184 | Incomplete Blocklist |
| CWE-200 | Exposure of Sensitive Information to an Unauthorized Actor |

### OWASP Top 10 for LLM Applications

The following mappings describe relevant security concepts rather than official ARKX classifications.

| OWASP LLM Category | Relevance |
|---|---|
| LLM01 - Prompt Injection | Conversational input was used to influence agent behavior and expose instructions. |
| LLM02 - Sensitive Information Disclosure | Protected file contents were extracted through the numerical output channel. |
| LLM05 - Improper Output Handling | Untrusted expressions reached a powerful backend tool without adequate validation and restriction. |
| LLM06 - Excessive Agency | The agent was given access to a tool with capabilities beyond its intended calculator function. |
| LLM07 - System Prompt Leakage | Supplementary testing demonstrated disclosure of internal instructions through pretexting. |

### MITRE ATLAS

The following techniques provide contextual mappings for the observed attack stages.

| Technique | Relevance |
|---|---|
| AML.T0051 - LLM Prompt Injection | Conversational manipulation of agent behavior. |
| AML.T0053 - AI Agent Tool Invocation | Use of the exposed calculation tool to perform unintended operations. |
| AML.T0056 - System Prompt Extraction | Disclosure of internal agent instructions. |
| AML.T0086 - Exfiltration via AI Agent Tool Invocation | Extraction of protected information through tool-mediated output. |

**Primary security themes:** Excessive Agency, improper handling of untrusted tool input, unsafe dynamic evaluation, and sensitive information disclosure.

The LLM interface changed how the vulnerability was reached, but the principal technical weakness remained the unsafe backend evaluator.

## 7. Recommended Remediation

### Backend evaluator security

1. Never execute `eval()`, `exec()`, or equivalent dynamic-evaluation functions against untrusted expressions.
2. Replace unrestricted evaluation with a dedicated mathematical expression parser.
3. Use an AST-based allowlist that permits only explicitly approved numerical operations and functions.
4. Reject imports, filesystem operations, arbitrary attribute access, function invocation outside the allowlist, and unsupported expression types.

### Tool permission controls

1. Give the calculator only the capabilities required to perform mathematical calculations.
2. Prevent calculator processes from accessing sensitive filesystem locations.
3. Apply least-privilege execution permissions.
4. Isolate unavoidable untrusted processing in a properly restricted environment.
5. Apply resource limits to prevent excessive CPU and memory consumption.

### Input and output validation

1. Validate expressions before they reach the evaluator.
2. Avoid relying on substring-based blocklists as the primary security mechanism.
3. Validate the parsed expression structure rather than relying only on its textual representation.
4. Treat output-type restrictions as an additional validation layer, not as a substitute for safe execution.

### Agentic AI security

1. Treat all user input as untrusted, regardless of how it is presented conversationally.
2. Enforce security controls in the backend independently of LLM instructions.
3. Do not expose credentials or sensitive configuration in system prompts.
4. Apply authorization and capability restrictions to every tool invocation.
5. Monitor unusual tool usage, repeated extraction attempts, and unexpected access to sensitive resources.

## 8. Lessons Learned

- An LLM interface does not make traditional application-security vulnerabilities disappear.
- System prompts are not security boundaries and should not contain secrets.
- A calculator should not have access to arbitrary Python execution or filesystem functionality.
- Blocklists are weak controls when the underlying evaluator remains unrestricted.
- Output serialization restrictions cannot prevent information disclosure if execution has already occurred.
- Numerical responses can be used as side channels for extracting protected information.
- Tool permissions should follow the principle of least privilege.
- Security controls must be enforced by the backend rather than delegated to the agent's instructions.
- The actual behavior of exposed tools should be tested rather than assumed safe based on their descriptions.

## 9. Evidence and Verification

The original assessment recorded:

- Successful execution of file-reading functionality.
- Successful retrieval of the file's character count.
- Character-by-character extraction through numerical character codes.
- Reconstruction of the original flag.
- Confirmation of successful challenge completion.

**Final Status:** Completed  
**Original Assessment Flag:** `flag-430e19`

The supplementary technical notes provide additional analysis of prompt disclosure, blocklist weaknesses, and the numerical output constraint. These explanations expand the security analysis without changing the original assessment's flag or verification result.
