| Communication Type                  | Recommended Auth               |
| ----------------------------------- | ------------------------------ |
| User → API Gateway                  | JWT / OAuth2                   |
| External Company → API Gateway      | OAuth2 Client Credentials      |
| Internal Service → Internal Service | mTLS + Service Identity Tokens |


# Secure Internal Service-to-Service Communication

## Using mTLS + JWT in a Financial Microservices Architecture

---

# 1. Introduction

As our platform evolves into a large-scale financial ecosystem with:

* multiple internal microservices
* third-party integrations
* open platform APIs
* external partner access
* sensitive financial operations

we must establish a secure and scalable communication model between services.

This document proposes a **Zero Trust internal communication architecture** using:

* **mTLS (Mutual TLS)** for transport-level authentication
* **JWT (JSON Web Tokens)** for service identity and authorization

This architecture is aligned with modern cloud-native and financial-industry security practices.

---

# 2. Problem Statement

Our current internal architecture assumes:

> “Internal services trust each other.”

This approach introduces significant security risks:

| Risk                      | Description                                         |
| ------------------------- | --------------------------------------------------- |
| Lateral Movement          | A compromised service can access other services     |
| Token Theft               | Stolen tokens can be reused internally              |
| Service Spoofing          | Any internal client may impersonate another service |
| Lack of Identity          | No cryptographic proof of caller identity           |
| Weak Auditability         | Difficult to trace trusted internal requests        |
| Future Scalability Issues | Unsafe for open platform ecosystems                 |

For a financial platform, internal trust alone is not sufficient.

---

# 3. Security Objectives

The proposed architecture aims to achieve:

* Strong service identity
* Encrypted internal communication
* Fine-grained authorization
* Zero Trust networking
* Secure partner integrations
* Scalable future architecture
* Regulatory/security compliance readiness
* Protection against replay and impersonation attacks

---

# 4. Core Security Model

The proposed architecture uses two complementary mechanisms:

| Technology | Responsibility                         |
| ---------- | -------------------------------------- |
| mTLS       | Verifies *who is connecting*           |
| JWT        | Verifies *what they are allowed to do* |

These technologies solve different security problems and should be used together.

---

# 5. High-Level Architecture

```text
┌────────────────────────────┐
│        Auth Service        │
│ OAuth2 / Identity Provider │
└─────────────┬──────────────┘
              │
      Issues Service JWTs
              │
              ▼

┌────────────────────────────┐
│       User Service         │
│  Service Identity + Cert   │
└─────────────┬──────────────┘
              │
      JWT + mTLS Connection
              │
              ▼

┌────────────────────────────┐
│      Payment Service       │
│ Verifies JWT + Certificate │
└────────────────────────────┘
```

---

# 6. Why JWT Alone Is Not Enough

JWT provides:

* identity
* scopes
* permissions
* audience validation

Example:

```json
{
  "sub": "user-service",
  "aud": "payment-service",
  "scope": ["payment:create"]
}
```

However, JWT does NOT prove:

* the caller is the real service
* the request came from trusted infrastructure
* the token was not stolen
* the transport channel is trusted

If a JWT token leaks, it can be reused from any machine until expiration.

---

# 7. Why mTLS Is Required

mTLS (Mutual TLS) ensures:

* both services authenticate each other
* communication is encrypted
* only trusted services may connect
* service impersonation becomes difficult

Each service owns a certificate:

```text
user-service.crt
payment-service.crt
notification-service.crt
```

During communication:

1. Caller presents certificate
2. Receiver validates certificate
3. Both establish encrypted trusted channel

This creates cryptographic proof of service identity.

---

# 8. JWT + mTLS Together

## Combined Security Model

| Layer               | Protection        |
| ------------------- | ----------------- |
| Transport Layer     | mTLS              |
| Application Layer   | JWT               |
| Authorization Layer | JWT scopes/claims |

This creates defense-in-depth security.

---

# 9. Internal Communication Flow

## Step 1 — Service Authentication

The calling service authenticates with the Auth Service using:

* OAuth2 Client Credentials Flow
* private keys or client secrets

---

## Step 2 — JWT Issuance

Auth Service issues a short-lived access token.

Example:

```json
{
  "iss": "auth-service",
  "sub": "user-service",
  "aud": "payment-service",
  "scope": ["payment:create"],
  "exp": 1710000300
}
```

---

## Step 3 — Secure Service Request

The caller sends request:

```http
POST /payments
Authorization: Bearer eyJ...
```

over:

```text
HTTPS + mTLS
```

---

## Step 4 — Receiver Validation

Receiving service validates:

### mTLS Validation

* client certificate
* trusted CA
* certificate expiration

### JWT Validation

* signature
* issuer
* audience
* expiration
* scopes

Only if ALL validations pass:

```text
Request is accepted
```

---

# 10. Authorization Model

Internal services should NEVER have unrestricted access.

Instead, every service receives scoped permissions.

---

## Example

| Service              | Allowed Permissions |
| -------------------- | ------------------- |
| user-service         | payment:create      |
| notification-service | payment:read        |
| analytics-service    | market:read         |
| cms-service          | content:write       |

---

## Principle of Least Privilege

Each service should only receive the minimum required permissions.

This reduces blast radius if a service becomes compromised.

---

# 11. Recommended Token Design

## Required Claims

```json
{
  "iss": "auth-service",
  "sub": "user-service",
  "aud": "payment-service",
  "scope": ["payment:create"],
  "iat": 1710000000,
  "exp": 1710000300
}
```

---

## Claim Description

| Claim | Description     |
| ----- | --------------- |
| iss   | Token issuer    |
| sub   | Calling service |
| aud   | Target service  |
| scope | Allowed actions |
| iat   | Issued at       |
| exp   | Expiration time |

---

# 12. Token Best Practices

| Recommendation                       | Reason                       |
| ------------------------------------ | ---------------------------- |
| Short-lived tokens (5–15 min)        | Limits token theft impact    |
| Rotate signing keys                  | Reduces long-term compromise |
| Validate audience                    | Prevents token reuse         |
| Use asymmetric signing (RS256/ES256) | Better key security          |
| Avoid long-lived internal secrets    | Reduces risk                 |

---

# 13. Certificate Management

Certificates should:

* be automatically rotated
* have expiration policies
* be signed by internal CA
* never be manually distributed

---

## Recommended Solutions

| Tool         | Purpose                           |
| ------------ | --------------------------------- |
| Istio        | Automatic mTLS                    |
| Linkerd      | Lightweight service mesh          |
| SPIRE        | Service identity                  |
| cert-manager | Kubernetes certificate automation |

---

# 14. Recommended Long-Term Architecture

As the platform grows, implementing a Service Mesh is strongly recommended.

---

## Benefits of Service Mesh

| Capability             | Description                    |
| ---------------------- | ------------------------------ |
| Automatic mTLS         | No manual certificate handling |
| Traffic Policies       | Fine-grained routing           |
| Retry/Circuit Breaking | Improved resilience            |
| Observability          | Metrics and tracing            |
| Identity Management    | Service identity automation    |

---

# 15. External Partner Integration

External companies should NEVER communicate directly with internal services.

All external traffic must pass through:

```text
API Gateway
```

---

## Gateway Responsibilities

* OAuth2 validation
* rate limiting
* API key management
* logging
* DDoS protection
* request transformation
* tenant isolation

---

# 16. Zero Trust Security Model

This architecture follows:

# Zero Trust Principles

Core assumption:

> No service or network should be trusted by default.

Every request must independently prove:

1. identity
2. authorization
3. transport authenticity

---

# 17. Recommended Technology Stack

## Identity Provider

Recommended solutions:

* [Keycloak](https://www.keycloak.org?utm_source=chatgpt.com)
* [ORY Hydra](https://www.ory.sh/hydra/?utm_source=chatgpt.com)
* [Zitadel](https://zitadel.com?utm_source=chatgpt.com)

---

## Service Mesh

Recommended solutions:

* [Istio](https://istio.io?utm_source=chatgpt.com)
* [Linkerd](https://linkerd.io?utm_source=chatgpt.com)

---

## Secret Management

Recommended solutions:

* [HashiCorp Vault](https://www.vaultproject.io?utm_source=chatgpt.com)
* [Infisical](https://infisical.com?utm_source=chatgpt.com)

---

# 18. Proposed Final Architecture

```text
                    ┌────────────────────┐
                    │ External Clients   │
                    └─────────┬──────────┘
                              │
                              ▼
                    ┌────────────────────┐
                    │ API Gateway        │
                    │ OAuth2 Validation  │
                    │ Rate Limiting      │
                    └─────────┬──────────┘
                              │
                              ▼
                    ┌────────────────────┐
                    │ Auth Service       │
                    │ OAuth2 / OIDC      │
                    └─────────┬──────────┘
                              │
                    Issues JWT Tokens
                              │
                              ▼

┌────────────────────────────────────────────────────┐
│                 Internal Services                  │
│                                                    │
│  mTLS + JWT + Scope Validation + Zero Trust        │
│                                                    │
│  User Service ↔ Payment Service ↔ Notification     │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

# 19. Security Benefits

| Benefit              | Description                             |
| -------------------- | --------------------------------------- |
| Strong Identity      | Services cryptographically authenticate |
| Secure Transport     | All traffic encrypted                   |
| Reduced Blast Radius | Scoped permissions                      |
| Safer Open Platform  | External integrations become manageable |
| Better Auditability  | Clear service identity tracking         |
| Compliance Readiness | Better alignment with fintech standards |
| Zero Trust Security  | No implicit internal trust              |

---

# 20. Recommended Adoption Strategy

## Phase 1 — Immediate

* JWT validation between services
* short-lived tokens
* audience validation
* scope-based authorization

---

## Phase 2 — Infrastructure Hardening

* introduce mTLS
* automate certificates
* centralize identity provider

---

## Phase 3 — Platform Scale

* service mesh adoption
* advanced observability
* tenant isolation
* external developer ecosystem
* policy-based authorization

---

# 21. Final Recommendation

For a growing financial platform, relying solely on internal network trust is not sufficient.

The recommended architecture is:

# JWT for authorization

# mTLS for transport authentication

# Zero Trust for internal networking

This provides a scalable, secure, and industry-aligned foundation for future platform growth and external ecosystem expansion.

