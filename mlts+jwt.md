## Secure Service-to-Service Communication with mTLS and JWT
This document explains how to secure internal communication between microservices using **mutual TLS (mTLS)** for transport security and **JWT** for authentication and authorization. It also includes ASCII sequence diagrams you can paste directly into Markdown viewers or repositories. [yanniznik](https://yanniznik.com/implementing-mtls-across-your-microservices/)

***
## 1. Concepts Overview
- **mTLS (Mutual TLS)**  
  - Extends normal TLS so that **both client and server present certificates** and verify each other. [yanniznik](https://yanniznik.com/implementing-mtls-across-your-microservices/)
  - Provides encryption in transit, integrity, and **strong service identity** (which service is calling which). [c-sharpcorner](https://www.c-sharpcorner.com/article/securing-microservices-communication-with-mtls-explained/)

- **JWT (JSON Web Token)**  
  - A signed token that encodes **claims** (like user id, roles, permissions, or service identity).  
  - Used by microservices to **authorize** a request after the connection is already secured by mTLS. [stackoverflow](https://stackoverflow.com/questions/45616764/authentication-between-microservice-approach)

- **Why use both?**  
  - mTLS: “Is this really Service A talking to Service B, on our internal network?”  
  - JWT: “Is this caller allowed to perform this specific action on this resource?” [reddit](https://www.reddit.com/r/microservices/comments/1pa2vwh/how_should_authentication_work_in/)

What part of this high-level picture feels less clear to you: the certificate side (mTLS) or the token side (JWT)?

***
## 2. Architecture: Where mTLS and JWT Live
In a typical microservice architecture:

- **Edge / API Gateway**  
  - Terminates external TLS from clients.  
  - Exchanges user credentials for a **user JWT** (for example via OAuth2 / OpenID Connect). [stackoverflow](https://stackoverflow.com/questions/45616764/authentication-between-microservice-approach)

- **Internal Services (Service A, Service B, …)**  
  - Communicate over **mTLS-only** internal network.  
  - Use **service JWTs** (sometimes also including user context) for authorization between services. [reddit](https://www.reddit.com/r/microservices/comments/1pa2vwh/how_should_authentication_work_in/)

High-level goals:

- Every internal HTTP call uses **HTTPS with mutual TLS**.  
- Every business operation checks a **JWT** that expresses either the user, the calling service, or both. [c-sharpcorner](https://www.c-sharpcorner.com/article/securing-microservices-communication-with-mtls-explained/)

***
## 3. Basic mTLS Flow Between Two Services
### 3.1. Textual Flow
1. Service A opens a TCP connection to Service B on port 443.  
2. mTLS handshake happens: both sides exchange and verify certificates issued by a shared CA. [yanniznik](https://yanniznik.com/implementing-mtls-across-your-microservices/)
3. If verification passes, an encrypted channel is established.  
4. Only after that, HTTP request/response data flows over the secure channel.
### 3.2. Sequence Diagram: Service A → Service B (mTLS only)
```mermaid
sequenceDiagram
    participant A as Service A
    participant B as Service B

    A->>B: TCP connect (port 443)
    A->>B: TLS ClientHello (mTLS requested)
    B-->>A: TLS ServerHello + Server Certificate
    A->>B: Verify B's certificate (CA, expiry, CN/SAN)
    A-->>B: Client Certificate
    B->>B: Verify A's certificate (CA, expiry, CN/SAN)
    A-->>B: TLS Finished (keys established)
    B-->>A: TLS Finished (secure channel ready)

    A->>B: HTTPS Request (e.g. GET /orders)
    B-->>A: HTTPS Response (200 OK, JSON data)
```

In this diagram, **transport is secure and identities are known**, but **no fine‑grained permissions** are expressed yet. mTLS only tells “A is really A, B is really B.” [yanniznik](https://yanniznik.com/implementing-mtls-across-your-microservices/)

***
## 4. Adding JWT for Authorization
### 4.1. Why JWT on top of mTLS?
- mTLS answers **“who is this service?”**, not **“what is this service allowed to do?”** [yanniznik](https://yanniznik.com/implementing-mtls-across-your-microservices/)
- JWT lets you encode **claims** like `sub` (subject), `roles`, `permissions`, `aud` (audience), and `iss` (issuer). [stackoverflow](https://stackoverflow.com/questions/45616764/authentication-between-microservice-approach)
- The receiving service validates the signature and claims to decide **allow / deny**. [reddit](https://www.reddit.com/r/microservices/comments/1pa2vwh/how_should_authentication_work_in/)

Common patterns:

- **User JWT**: created at the edge after login; propagated downstream so each service can enforce user-level permissions. [stackoverflow](https://stackoverflow.com/questions/45616764/authentication-between-microservice-approach)
- **Service JWT (client credentials flow)**: each service authenticates to an auth server and gets its own JWT used when calling other services. [reddit](https://www.reddit.com/r/microservices/comments/1pa2vwh/how_should_authentication_work_in/)
### 4.2. Flow: Service A Calls Service B with mTLS + JWT
```mermaid
sequenceDiagram
    participant Auth as Auth Server
    participant A as Service A
    participant B as Service B

    Note over A,Auth: 1) Service A obtains a service JWT
    A->>Auth: POST /oauth/token (client_credentials)
    Auth-->>A: 200 OK (service JWT)

    Note over A,B: 2) A calls B over mTLS with JWT
    A->>B: TCP connect (443) + mTLS handshake
    B-->>A: mTLS established (mutual auth OK)

    Note over A,B: 3) HTTP request with JWT
    A->>B: HTTPS POST /orders<br/>Authorization: Bearer {JWT}
    B->>B: Validate JWT (signature, iss, aud, exp, scopes)
    alt JWT valid and authorized
        B-->>A: 200 OK (response body)
    else JWT invalid or unauthorized
        B-->>A: 401/403 (error)
    end
```

Here:

- mTLS ensures **only trusted workloads** can reach Service B at all. [yanniznik](https://yanniznik.com/implementing-mtls-across-your-microservices/)
- JWT ensures the caller (Service A) has **the right to perform the requested operation**. [stackoverflow](https://stackoverflow.com/questions/45616764/authentication-between-microservice-approach)

In your own words, how would you explain the difference between “identifying the caller” and “authorizing the caller” to someone new to security?

***
## 5. Combining User JWT and Service JWT
A common real-world scenario:

1. User logs in at the edge (API Gateway or BFF), gets a **User JWT**.  
2. Edge validates the user and may create or forward a token to internal services.  
3. When Service A calls Service B, it can:  
   - Forward the **User JWT** so B knows who the end-user is.  
   - Attach its own **Service JWT** so B can verify that “Service A” is itself authorized to call B. [stackoverflow](https://stackoverflow.com/questions/45616764/authentication-between-microservice-approach)

High-level flows used in practice:

- **User context propagation**: The original user JWT is included so all services can enforce user-level rules. [reddit](https://www.reddit.com/r/microservices/comments/1pa2vwh/how_should_authentication_work_in/)
- **Service authorization**: Each service has a client-credentials–based JWT for calling others. [stackoverflow](https://stackoverflow.com/questions/45616764/authentication-between-microservice-approach)

***
## 6. Example Internal Network Topology (Diagram)
This diagram shows multiple services in an internal network using mTLS for transport and JWT for authorization.

```mermaid
flowchart LR
    subgraph External
        C[Client (Browser / Mobile)]
    end

    subgraph Internal Network (Zero Trust)
        G[API Gateway / Edge]
        S1[Service A]
        S2[Service B]
        S3[Service C]
        Auth[Auth / Token Service]
    end

    C -- HTTPS (TLS) --> G

    G -- mTLS + User JWT --> S1
    S1 -- mTLS + User JWT + Service JWT --> S2
    S2 -- mTLS + Service JWT --> S3

    S1 --- Auth
    S2 --- Auth
    S3 --- Auth

    note right of Auth: Issues JWTs (user + service)
```

Key ideas:

- All internal communication (**G ↔ S1 ↔ S2 ↔ S3**) uses **mTLS**.  
- JWTs carry **user and/or service claims** for authorization decisions in each service. [stackoverflow](https://stackoverflow.com/questions/45616764/authentication-between-microservice-approach)

***
## 7. Best Practices Checklist
- **Always-on mTLS**  
  - Issue certificates for each service from a central CA or via a service mesh (Istio, Linkerd, etc.). [yanniznik](https://yanniznik.com/implementing-mtls-across-your-microservices/)
  - Automate certificate rotation and use short lifetimes. [c-sharpcorner](https://www.c-sharpcorner.com/article/securing-microservices-communication-with-mtls-explained/)

- **JWT design**  
  - Use strong signing keys and algorithms (for example, RS256 or ES256). [stackoverflow](https://stackoverflow.com/questions/45616764/authentication-between-microservice-approach)
  - Validate `iss`, `aud`, `exp`, and `nbf` in every service that consumes tokens. [reddit](https://www.reddit.com/r/microservices/comments/1pa2vwh/how_should_authentication_work_in/)
  - Consider separate lifetimes for **user tokens** vs **service tokens**. [reddit](https://www.reddit.com/r/microservices/comments/1pa2vwh/how_should_authentication_work_in/)

- **Zero trust mindset**  
  - Do not fully trust the internal network; require both mTLS and token validation at each hop. [reddit](https://www.reddit.com/r/kubernetes/comments/g0yxqa/why_in_the_world_do_i_need_mtls_between_my/)
  - Log failed verifications for auditing and incident response. [openlogic](https://www.openlogic.com/blog/patterns-microservices-authentication-client-certificates)

***
## 8. Minimal Example HTTP Exchange (Conceptual)
Below is a conceptual, language-agnostic example of how an HTTP request might look once mTLS is already established at the transport layer:

```http
POST /orders HTTP/1.1
Host: service-b.internal
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
X-User-Id: 12345
Content-Type: application/json

{
  "productId": "abc",
  "quantity": 2
}
```

On Service B’s side:

- The connection is accepted only if the **client certificate is valid** (mTLS).  
- Then B validates the **JWT** and uses claims plus headers to enforce business rules. [yanniznik](https://yanniznik.com/implementing-mtls-across-your-microservices/)

***

If you share what stack you use (for example, Kubernetes + Istio, Spring Boot, .NET, Go), a next step could be to add **concrete code snippets** and **config examples** to this document. What environment or language are you primarily targeting for these microservices?
