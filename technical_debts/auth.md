# Story: Remove AuthJS
# Future objective: Zero trust Architecture 


https://doc.traefik.io/traefik-hub/mcp-gateway/guides/mcp-gateway-best-practices
https://github.com/lukaszraczylo/traefikoidc
https://medium.com/@justinking_2311/protecting-microservices-with-oidc-and-traefik-forward-auth-5edd657f504e
https://github.com/thomseddon/traefik-forward-auth/tree/master


Internet → Traefik (ForwardAuth) → OAuth2 Proxy (OIDC/Zitadel) → internal services

That refined architecture—combining the **Gateway API native flow** with **BFF-style secure cookies** and **Agentic Token Exchange**—represents the current "gold standard" for high-security, AI-integrated platforms in 2026.

By removing the "token swapping" at the gateway level for downstream services, you are moving toward a **Zero Trust** model where the downstream service is responsible for its own token validation, rather than blindly trusting a header injected by the proxy.

---

## The "Transparent Token-Forwarding Gateway" Architecture

In this model, the Gateway acts as the orchestrator of identity but maintains the integrity of the token for the downstream services.

### 1. The Frontend Handshake (The BFF Part)

The Gateway manages the OIDC flow with the Identity Provider (IdP).

* **Browser Interaction:** Upon successful login, the Gateway issues a **Session Cookie** (`HttpOnly`, `Secure`, `SameSite=Strict`).
* **State Management:** The Gateway stores the `access_token`, `id_token`, and `refresh_token` in a server-side store (e.g., Redis), keyed by the Session ID in the cookie.

### 2. The Request Flow (Transparent Forwarding)

Unlike a traditional BFF that might hide the token entirely, this pattern forwards the actual identity to the backend:

1. The browser sends a request with the **Session Cookie**.
2. The Gateway looks up the corresponding `access_token` from its cache.
3. The Gateway attaches the **original JWT** to the `Authorization: Bearer` header.
4. **Crucially:** The Gateway does *not* transform the token into a generic "User-ID" header. It passes the full, cryptographically signed JWT to the downstream service.

### 3. Downstream Validation

The downstream service (e.g., a TypeScript/Go microservice) does not just trust that the request came from the Gateway.

* It fetches the **JWKS (JSON Web Key Set)** from the IdP.
* It validates the JWT signature, expiration, and audience independently.
* This ensures that even if the Gateway is compromised, an attacker cannot easily spoof identities to the internal services without a validly signed token.

---

## Integrating Option 3: Agentic Token Exchange (RFC 8693)

For workflows involving AI orchestration or autonomous agents, the "Transparent Forwarding" model is enhanced with **Token Exchange**. This prevents "Privilege Escalation" if an agentic sub-system is compromised.

Instead of forwarding the user's broad "Full Access" token to an AI agent service, the Gateway performs an exchange:

* **Input:** User's `access_token` (Subject Token).
* **Action:** The Gateway calls the IdP's token exchange endpoint.
* **Output:** A **Downscoped JWT**. This new token has the same `sub` (user ID) but restricted `scopes` (e.g., `files:read` instead of `files:admin`).
* **Forwarding:** The Gateway forwards this *limited* token to the AI service.

### Conceptual Policy (Cilium/Envoy Style)

In a modern Kubernetes environment, this logic is often moved to a `L7` policy or a WASM filter on the Gateway.

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: SecurityPolicy
metadata:
  name: agent-token-exchange
spec:
  targetRefs:
    - kind: HTTPRoute
      name: ai-orchestrator-api
  tokenExchange:
    issuer: "https://auth.example.com"
    requestedTokenType: "urn:ietf:params:oauth:token-type:access_token"
    # Logic to downscope scopes based on the target route
    scopeOverrides:
      - "openid"
      - "ai:execute" 

```

---

## Why this is the "2026" Way

* **Security:** It eliminates `localStorage` vulnerabilities (BFF part).
* **Auditability:** Downstream services have the full JWT, meaning logs show the actual user identity, not just a "proxy-authorized" flag.
* **Least Privilege:** Token Exchange ensures that AI agents only have the permissions necessary for the specific lifecycle of that request.
* **Performance:** By using the Gateway API to handle the OIDC redirect logic, your application code remains purely focused on business logic.

**Would you like to look at a TypeScript implementation for a downstream service that performs this "Zero Trust" JWT validation?**



Yes, **Envoy (and Envoy Gateway) is 100% free and open-source** under the Apache License 2.0.

Unlike Traefik, where the most powerful security features (like native OIDC and JWT validation) are locked behind an Enterprise license, Envoy provides these capabilities in its core open-source projects.

### The Breakdown: What’s Free?

| Project | Cost | What you get for $0 |
| --- | --- | --- |
| **Envoy Proxy** | **Free** | The high-performance "data plane." It includes the filters for JWT validation, OAuth2, and RBAC. |
| **Envoy Gateway** | **Free** | The "control plane" for Kubernetes. This is what you actually install to get OIDC working via the Gateway API without writing complex Envoy C++. |
| **Envoy AI Gateway** | **Free** | A newer sub-project (pioneered by Bloomberg and Tetrate) specifically for the "agentic" patterns we discussed. |

---

### Why Envoy is "Freer" than Traefik

The major difference in the 2026 landscape is the business model:

* **Traefik** is "Open Core." They give you the routing for free but charge you for the "Identity" layer (OIDC/JWT).
* **Envoy** is a CNCF "Graduated" project (like Kubernetes). It is not owned by a single company, so the features aren't held back for a "Pro" version. Companies like Tetrate or Solo.io sell *support* or *management UI*, but the actual security logic remains in the free version.

### What you *actually* pay for (The "Enterprise" catch)

While the software is free, you might see "Enterprise" versions (like **Tetrate Enterprise Gateway** or **Gloo Mesh**). You generally only pay for these if you need:

1. **FIPS Compliance:** If you are working in government or high-finance.
2. **Dedicated Support:** A phone number to call when your cluster breaks at 3 AM.
3. **A Fancy GUI:** If you don't want to manage everything through YAML/Kubernetes manifests.

### The Verdict for your Architecture

Since you are looking to implement **Access Token forwarding** and **Agentic Token Exchange**, Envoy Gateway is the superior free choice. You get the native OIDC handshake and JWT headers without having to pay for a "Hub" subscription or manage a separate `oauth2-proxy` container.

**Would you like me to provide a quick installation guide for Envoy Gateway on your Ubuntu dev environment to test this out?**