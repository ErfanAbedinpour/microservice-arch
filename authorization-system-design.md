# Authorization System Design
## Enterprise Open Platform — Built In-House with Node.js + PostgreSQL

**Stack:** Node.js / TypeScript (NestJS recommended)
**Database:** PostgreSQL
**Approach:** No external authorization platforms. Everything is built and owned by your team.

---

## Table of Contents

1. [What This System Does](#1-what-this-system-does)
2. [The Three Layers Explained Simply](#2-the-three-layers-explained-simply)
3. [Database Tables](#3-database-tables)
4. [How a Permission Check Works](#4-how-a-permission-check-works)
5. [User Request — Step by Step](#5-user-request--step-by-step)
6. [Partner Request — Step by Step](#6-partner-request--step-by-step)
7. [Service-to-Service — Step by Step](#7-service-to-service--step-by-step)
8. [How to Add or Remove Permissions Without Redeploying](#8-how-to-add-or-remove-permissions-without-redeploying)
9. [How to Handle Exceptions and Blocks](#9-how-to-handle-exceptions-and-blocks)
10. [Where Authorization Lives in Each Service](#10-where-authorization-lives-in-each-service)
11. [Audit Logging](#11-audit-logging)
12. [Building This in Phases](#12-building-this-in-phases)

---

## 1. What This System Does

The authentication layer (from `arch2.md`) already answers **who you are** — it issues JWT tokens and validates them at the gateway. This system answers a different question: **what are you allowed to do?**

Think of it as a separate service your other services can ask at runtime:

```
"Can user alice read account ACC-001?"     →   YES or NO
"Can partner acme read payment rates?"     →   YES or NO
"Can the payment service read ledger?"     →   YES or NO
```

That question is always answered by one place — the **Authorization Service** — and the answer is always based on data in the database, not logic hardcoded inside your business services.

### The Golden Rule

> No service should contain `if (user.role === 'admin')` or any other permission check in its business logic.
> Every access decision comes from the Authorization Service by querying the database.

This means you can change who can do what by updating database rows, not by shipping new code.

---

## 2. The Three Layers Explained Simply

This system uses three concepts stacked on top of each other. You do not need external tools for any of them — they all live in PostgreSQL tables and TypeScript service logic.

---

### Layer 1 — Roles and Permissions (RBAC)

**What it is:** A role is a named group of permissions. Users and partners are assigned roles. When you check if someone can do something, you first check their role.

**What it handles:**
- "Admins can approve payments"
- "Partners with the readonly role can read public rates"
- "Auditors can read the audit log"

**What it does NOT handle:**
- "Alice can read account ACC-001 but not ACC-002" — this is too specific for a role
- "All users except Bob can read this" — roles cannot express exceptions

**Think of it as:** The default permission shape of your system. Roles cover the common cases.

```
Role: payment-viewer
  └── can: read payment
  └── can: read account

Role: payment-writer
  └── can: read payment
  └── can: create payment
  └── can: read account

Role: partner-readonly
  └── can: read rate
```

---

### Layer 2 — Entity Relationships

**What it is:** A simple table that records which user (or role or service) has which access to which specific resource. It is a list of relationships stored as rows.

**What it handles:**
- "Alice is the owner of account ACC-001"
- "Bob is a viewer of account ACC-001"
- "Charlie is blocked from account ACC-001"
- "The payment service can read ledger entries"

**What it does NOT handle:** Complex conditional logic like "only on weekdays" or "only if risk score is low". That is Layer 3.

**Think of it as:** A flexible access list. A row in this table means a direct grant or block between a person/service and a specific resource. You can add and remove rows at runtime with no deployment.

```
Table: resource_access
──────────────────────────────────────────────────────────
alice    │ owner   │ account │ ACC-001
bob      │ viewer  │ account │ ACC-001
charlie  │ blocked │ account │ ACC-001
payment  │ reader  │ ledger  │ (all)
──────────────────────────────────────────────────────────
```

---

### Layer 3 — Rules Table (Conditional Logic as Data)

**What it is:** A table of rules stored in the database. Each rule is a JSON object that the Authorization Service reads and evaluates at runtime. Instead of hardcoding `if` statements, you write rules as data.

**What it handles:**
- "Partners can only call the API during business hours"
- "Payments above 10,000 require the approver role"
- "Users from suspended accounts are denied everything"
- Any context-aware condition that is too complex for a role or a relationship row

**Think of it as:** The escape hatch. Layer 1 and Layer 2 handle 90% of cases. Layer 3 handles the remaining 10% that needs conditions, context, or overrides. Because rules are rows in a table, you can add new rules, disable them, or edit them without redeploying.

```
Table: authz_rules
───────────────────────────────────────────────────────────────────
name            │ effect  │ condition (JSON)
────────────────┼─────────┼─────────────────────────────────────────
high-value-gate │ DENY    │ { "action": "approve", "amount_gte": 10000,
                │         │   "missing_role": "senior-approver" }
partner-hours   │ DENY    │ { "principal_type": "partner",
                │         │   "outside_hours": [8, 20] }
suspended-block │ DENY    │ { "account_status": "suspended" }
───────────────────────────────────────────────────────────────────
```

---

## 3. Database Tables

These are all the tables you need in PostgreSQL. No external services. No graph databases.

---

### 3.1 Principals — Everyone Who Can Make a Request

A principal is any identity that can make a request: a user, a service, or a partner company.

```
TABLE: principals
┌──────────────────┬─────────────────┬──────────────────────────────────────────┐
│ id               │ type            │ external_id                              │
├──────────────────┼─────────────────┼──────────────────────────────────────────┤
│ usr_alice        │ user            │ auth0|user_alice (from your Auth Service) │
│ usr_bob          │ user            │ auth0|user_bob                           │
│ svc_payment      │ service         │ payment-service                          │
│ prt_acmecorp     │ partner         │ acme-corp                                │
└──────────────────┴─────────────────┴──────────────────────────────────────────┘

Why: Every permission check starts with "who is asking?"
     This table is the single source of truth for all identities.
```

---

### 3.2 Roles — Named Groups of Permissions

```
TABLE: roles
┌──────────────────┬────────────────────┬──────────────────────────────────────┐
│ id               │ name               │ description                          │
├──────────────────┼────────────────────┼──────────────────────────────────────┤
│ role_pay_view    │ payment-viewer     │ Can read payments and accounts        │
│ role_pay_write   │ payment-writer     │ Can create payments                  │
│ role_admin       │ admin              │ Full platform access                  │
│ role_auditor     │ auditor            │ Read-only access to audit logs        │
│ role_partner_ro  │ partner-readonly   │ External partner, public data only    │
└──────────────────┴────────────────────┴──────────────────────────────────────┘

Why: Roles give you a stable, named way to group permissions.
     Instead of assigning 10 permissions to every user, you assign one role.
```

---

### 3.3 Permissions — The Allowed Actions

```
TABLE: permissions
┌──────────────────┬───────────────┬──────────────────┬────────────────────────────┐
│ id               │ action        │ resource_type    │ description                │
├──────────────────┼───────────────┼──────────────────┼────────────────────────────┤
│ perm_pay_read    │ read          │ payment          │ Read payment records        │
│ perm_pay_create  │ create        │ payment          │ Create a new payment        │
│ perm_pay_approve │ approve       │ payment          │ Approve a pending payment   │
│ perm_acc_read    │ read          │ account          │ Read account details        │
│ perm_led_read    │ read          │ ledger_entry     │ Read ledger entries         │
│ perm_rate_read   │ read          │ rate             │ Read public exchange rates  │
└──────────────────┴───────────────┴──────────────────┴────────────────────────────┘

Why: A permission is the smallest unit of access.
     It says: "this action is allowed on this type of resource."
     It does NOT say who can do it — that comes from the role assignment.
```

---

### 3.4 Role Permissions — Which Roles Get Which Permissions

```
TABLE: role_permissions
┌──────────────────┬──────────────────┐
│ role_id          │ permission_id    │
├──────────────────┼──────────────────┤
│ role_pay_view    │ perm_pay_read    │
│ role_pay_view    │ perm_acc_read    │
│ role_pay_write   │ perm_pay_read    │
│ role_pay_write   │ perm_pay_create  │
│ role_pay_write   │ perm_acc_read    │
│ role_admin       │ perm_pay_approve │
│ role_partner_ro  │ perm_rate_read   │
└──────────────────┴──────────────────┘

Why: This is the join table between roles and permissions.
     To give a role a new permission, you insert one row here.
     No deployment. No code change.
```

---

### 3.5 Principal Roles — Who Has Which Role

```
TABLE: principal_roles
┌──────────────────┬──────────────────┬────────────────┬────────────────────────┐
│ principal_id     │ role_id          │ granted_by     │ expires_at             │
├──────────────────┼──────────────────┼────────────────┼────────────────────────┤
│ usr_alice        │ role_pay_view    │ usr_admin01    │ NULL                   │
│ usr_bob          │ role_pay_write   │ usr_admin01    │ NULL                   │
│ prt_acmecorp     │ role_partner_ro  │ system         │ 2025-12-31 23:59:59    │
└──────────────────┴──────────────────┴────────────────┴────────────────────────┘

Why: This is where you assign roles to actual people, services, and partners.
     expires_at lets you create time-limited access without any scheduled job —
     the Authorization Service simply checks if expires_at is in the past.
```

---

### 3.6 Resource Access — Entity-Level Grants and Blocks

This is the heart of the relationship layer. Each row says: this principal has this relationship to this specific resource.

```
TABLE: resource_access
┌──────────────────┬────────────────────┬───────────────┬────────────────┬────────────┐
│ principal_id     │ relation           │ resource_type │ resource_id    │ created_by │
├──────────────────┼────────────────────┼───────────────┼────────────────┼────────────┤
│ usr_alice        │ owner              │ account       │ ACC-001        │ system     │
│ usr_bob          │ viewer             │ account       │ ACC-001        │ usr_alice  │
│ usr_charlie      │ blocked            │ account       │ ACC-001        │ usr_admin  │
│ svc_payment      │ reader             │ ledger_entry  │ *              │ system     │
│ prt_acmecorp     │ viewer             │ rate          │ *              │ system     │
└──────────────────┴────────────────────┴───────────────┴────────────────┴────────────┘

resource_id = '*' means access to ALL resources of that type.

Why: This lets you say "alice owns account ACC-001" without giving alice a role
     that grants her access to ALL accounts.
     Adding or removing a row here takes effect immediately.

Relations you will use:
  owner   →  full read + write access to this specific resource
  viewer  →  read-only access to this specific resource
  blocked →  explicit deny, overrides any role-based permission
  reader  →  read-only, used for service-to-service access
```

---

### 3.7 Authorization Rules — Conditional Logic Stored as Data

```
TABLE: authz_rules
┌──────────────────┬─────────────┬──────────────────────────────────────────────────┬──────────┐
│ id               │ effect      │ condition                                        │ enabled  │
├──────────────────┼─────────────┼──────────────────────────────────────────────────┼──────────┤
│ rule_suspended   │ DENY        │ { "principal_status": "suspended" }              │ true     │
│ rule_high_value  │ DENY        │ { "action": "approve",                           │ true     │
│                  │             │   "amount_gte": 10000,                           │          │
│                  │             │   "missing_role": "senior-approver" }            │          │
│ rule_partner_hrs │ DENY        │ { "principal_type": "partner",                   │ true     │
│                  │             │   "outside_business_hours": true }               │          │
└──────────────────┴─────────────┴──────────────────────────────────────────────────┴──────────┘

Why: These rules let you add conditional logic without touching service code.
     enabled = false disables a rule instantly without deleting it.
     The Authorization Service loads all enabled rules at startup and re-reads
     them on a short interval (e.g. every 60 seconds).
```

---

### 3.8 Audit Log — Every Decision Recorded

```
TABLE: authz_audit_log
┌────────────┬──────────────────┬───────────────┬────────────────┬────────────┬────────────┬─────────────────────┐
│ id         │ principal_id     │ resource_type │ resource_id    │ action     │ decision   │ timestamp           │
├────────────┼──────────────────┼───────────────┼────────────────┼────────────┼────────────┼─────────────────────┤
│ log_0001   │ usr_alice        │ account       │ ACC-001        │ read       │ PERMIT     │ 2024-06-01 14:01:33 │
│ log_0002   │ usr_charlie      │ account       │ ACC-001        │ read       │ DENY       │ 2024-06-01 14:01:55 │
│ log_0003   │ prt_acmecorp     │ payment       │ PAY-999        │ read       │ DENY       │ 2024-06-01 14:02:10 │
│ log_0004   │ svc_payment      │ ledger_entry  │ LED-123        │ read       │ PERMIT     │ 2024-06-01 14:03:05 │
└────────────┴──────────────────┴───────────────┴────────────────┴────────────┴────────────┴─────────────────────┘

Why: Required for financial compliance. Every PERMIT and DENY is written here.
     Never delete from this table. Archive old rows to cold storage instead.
```

---

## 4. How a Permission Check Works

When any service needs to check if a request is allowed, it calls the Authorization Service with three things: **who, what action, on what resource.**

The Authorization Service runs through these steps in order and stops as soon as it has a definitive answer.

```
                  Incoming check:
         { principal, action, resource_id }
                         │
                         ▼
         ┌───────────────────────────────┐
         │  Step 1: Check resource_access│
         │  for a 'blocked' row          │
         │                               │
         │  Is there a row where         │
         │  principal = this user AND    │
         │  resource_id = this resource  │
         │  AND relation = 'blocked'?    │
         └───────────────┬───────────────┘
                         │
               ┌─────────┴─────────┐
           row found           no row found
               │                   │
               ▼                   ▼
            DENY             continue to Step 2
         (log it)
                                   │
                                   ▼
         ┌───────────────────────────────┐
         │  Step 2: Check resource_access│
         │  for an explicit grant row    │
         │  (owner, viewer, reader)      │
         │                               │
         │  Does a relationship row      │
         │  exist for this principal     │
         │  and this specific resource?  │
         └───────────────┬───────────────┘
                         │
               ┌─────────┴─────────┐
          row found           no row found
               │                   │
               ▼                   ▼
       continue to Step 4    continue to Step 3
                                   │
                                   ▼
         ┌───────────────────────────────┐
         │  Step 3: Check roles          │
         │                               │
         │  Does the principal have a    │
         │  role that includes a         │
         │  permission for this          │
         │  action + resource_type?      │
         └───────────────┬───────────────┘
                         │
               ┌─────────┴─────────┐
           role found          no role found
               │                   │
               ▼                   ▼
       continue to Step 4        DENY
                               (log it)
                         │
                         ▼
         ┌───────────────────────────────┐
         │  Step 4: Evaluate rules table │
         │                               │
         │  Load all enabled rules.      │
         │  Run each condition against   │
         │  the request context.         │
         │  If any rule effect = DENY    │
         │  and condition matches → DENY │
         └───────────────┬───────────────┘
                         │
               ┌─────────┴─────────┐
           rule fired          no rule fired
               │                   │
               ▼                   ▼
            DENY                PERMIT
         (log it)              (log it)
```

**Key point:** A `blocked` row in `resource_access` is checked first and always wins. This is how you block a specific user from a specific resource even if they have a role that would otherwise allow it.

---

## 5. User Request — Step by Step

```
User (browser or mobile app)
         │
         │  1. User logs in
         │     POST /auth/login
         ▼
┌─────────────────────────────────┐
│           Auth Service           │
│                                 │
│  Validates credentials.         │
│  Looks up the user's roles      │
│  from principal_roles table.    │
│  Issues a signed JWT.           │
│                                 │
│  JWT contains:                  │
│    sub: "usr_alice"             │
│    roles: ["payment-viewer"]    │
│    scope: "payments:read"       │
│    exp: <15 minutes from now>   │
└───────────────┬─────────────────┘
                │
                │  2. JWT returned to user
                ▼
User stores token, makes API call
                │
                │  3. GET /payments/ACC-001
                │     Authorization: Bearer <jwt>
                ▼
┌─────────────────────────────────┐
│           API Gateway            │
│                                 │
│  Validates JWT signature.       │
│  Checks token is not expired.   │
│  Checks scope matches route.    │
│  If any check fails → 401.      │
│                                 │
│  Gateway does NOT check         │
│  resource-level permissions.    │
│  That is the service's job.     │
└───────────────┬─────────────────┘
                │
                │  4. Forwards request over mTLS
                │     with the JWT attached
                ▼
┌─────────────────────────────────┐
│         Payment Service          │
│                                 │
│  Receives request.              │
│  Extracts user id from JWT.     │
│  Before doing anything, calls   │
│  the Authorization Service.     │
└───────────────┬─────────────────┘
                │
                │  5. Authorization check
                │  { principal: "usr_alice",
                │    action: "read",
                │    resource_type: "account",
                │    resource_id: "ACC-001" }
                ▼
┌─────────────────────────────────┐
│      Authorization Service       │
│                                 │
│  Step 1 — blocked check:        │
│    No blocked row for alice     │
│    on ACC-001.          → pass  │
│                                 │
│  Step 2 — relationship check:   │
│    alice is owner of ACC-001.   │
│                         → pass  │
│                                 │
│  Step 3 — role check:           │
│    role:payment-viewer          │
│    includes perm_acc_read.      │
│                         → pass  │
│                                 │
│  Step 4 — rules check:          │
│    No rules fire.       → pass  │
│                                 │
│  Decision: PERMIT               │
│  Written to audit_log.          │
└───────────────┬─────────────────┘
                │
                │  6. PERMIT
                ▼
Payment Service runs business logic
                │
                │  7. Needs ledger data.
                │     Calls Ledger Service over mTLS
                │     with a service token.
                ▼
┌─────────────────────────────────┐
│          Ledger Service          │
│                                 │
│  Receives request.              │
│  Calls Authorization Service:   │
│  { principal: "svc_payment",    │
│    action: "read",              │
│    resource_type: "ledger_entry"│
│    resource_id: "LED-123" }     │
│                                 │
│  Authorization Service:         │
│    svc_payment has reader row   │
│    for ledger_entry:*  → PERMIT │
└───────────────┬─────────────────┘
                │
                │  8. Ledger data returned
                ▼
Payment Service builds response
                │
                │  9. HTTP 200 + data
                ▼
             User
```

---

## 6. Partner Request — Step by Step

```
External Partner (Acme Corp)
         │
         │  1. Partner authenticates
         │     POST /oauth/token
         │     grant_type: client_credentials
         │     client_id + client_secret
         ▼
┌─────────────────────────────────┐
│           Auth Service           │
│                                 │
│  Validates client credentials.  │
│  Looks up partner in principals │
│  and partner_configs tables.    │
│  Issues JWT scoped to only      │
│  what partner is configured for.│
│                                 │
│  JWT contains:                  │
│    sub: "prt_acmecorp"          │
│    principal_type: "partner"    │
│    scope: "rates:read"          │
│    aud: "public-api"            │
└───────────────┬─────────────────┘
                │
                │  2. JWT returned
                ▼
Partner calls the API
                │
                │  3. GET /rates/USD-EUR
                │     Authorization: Bearer <jwt>
                ▼
┌─────────────────────────────────┐
│           API Gateway            │
│                                 │
│  Validates token.               │
│  Checks aud = "public-api".     │
│  Checks scope = "rates:read".   │
│  Enforces rate limit from       │
│  partner_configs table.         │
│                                 │
│  If partner tries /payments     │
│  → 403 immediately (wrong scope)│
└───────────────┬─────────────────┘
                │
                │  4. mTLS + JWT forwarded
                ▼
┌─────────────────────────────────┐
│           Rates Service          │
│                                 │
│  Calls Authorization Service:   │
│  { principal: "prt_acmecorp",   │
│    action: "read",              │
│    resource_type: "rate",       │
│    resource_id: "USD-EUR" }     │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐
│      Authorization Service       │
│                                 │
│  Step 1 — blocked check:        │
│    No block row.        → pass  │
│                                 │
│  Step 2 — relationship check:   │
│    prt_acmecorp has viewer      │
│    row for rate:*       → pass  │
│                                 │
│  Step 3 — role check:           │
│    role:partner-readonly        │
│    includes perm_rate_read.     │
│                         → pass  │
│                                 │
│  Step 4 — rules check:          │
│    rule_partner_hrs fires?      │
│    Current time: 14:00 → NO.    │
│                         → pass  │
│                                 │
│  Decision: PERMIT               │
└───────────────┬─────────────────┘
                │
                │  5. Rate data returned
                ▼
            Partner

Note: The partner never sees the Ledger Service, Payment Service,
      or any internal service URL.
      The gateway scope check blocks that before it ever reaches a service.
```

---

## 7. Service-to-Service — Step by Step

Internal services must also be authorized when calling each other. A compromised service should not be able to call anything it wants.

```
Payment Service needs ledger data
         │
         │  1. Payment Service requests
         │     a short-lived service token
         │     POST /internal/token
         │     (over mTLS, no user credentials)
         ▼
┌─────────────────────────────────┐
│           Auth Service           │
│                                 │
│  Validates the mTLS certificate │
│  of the calling service.        │
│  Issues a service token that    │
│  expires in 5 minutes.          │
│                                 │
│  JWT contains:                  │
│    sub: "svc_payment"           │
│    type: "service"              │
│    scope: "ledger:read"         │
│    exp: <5 minutes from now>    │
└───────────────┬─────────────────┘
                │
                │  2. Service token returned
                ▼
Payment Service makes internal call
                │
                │  3. GET /internal/ledger/LED-123
                │     over mTLS
                │     Bearer <service_token>
                ▼
┌─────────────────────────────────┐
│          Ledger Service          │
│                                 │
│  Two things are validated:      │
│                                 │
│  a) The mTLS certificate must   │
│     belong to payment-service.  │
│     (transport-level identity)  │
│                                 │
│  b) The JWT sub must be         │
│     svc_payment and type must   │
│     be "service".               │
│     (token-level identity)      │
│                                 │
│  Both must pass. Then it calls  │
│  Authorization Service.         │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐
│      Authorization Service       │
│                                 │
│  resource_access row exists:    │
│  svc_payment | reader |         │
│  ledger_entry | *               │
│                                 │
│  Decision: PERMIT               │
└───────────────┬─────────────────┘
                │
                │  4. Ledger data returned
                ▼
           Payment Service
```

---

## 8. How to Add or Remove Permissions Without Redeploying

Every type of access change maps to a database operation. No service needs to restart.

```
WHAT YOU WANT TO DO                HOW YOU DO IT
─────────────────────────────────────────────────────────────────────────────
Give a user a new role             INSERT into principal_roles

Remove a role from a user          DELETE from principal_roles

Give a role a new permission       INSERT into role_permissions

Remove a permission from a role    DELETE from role_permissions

Grant a user access to one         INSERT into resource_access
specific entity                    (relation = 'viewer' or 'owner')

Revoke a user's access to one      DELETE from resource_access
specific entity

Block a user from one entity       INSERT into resource_access
                                   (relation = 'blocked')

Unblock a user                     DELETE the 'blocked' row

Onboard a new partner              INSERT into principals
                                   INSERT into principal_roles
                                   INSERT into resource_access (for their scope)

Offboard a partner immediately     UPDATE principals SET status = 'suspended'
                                   Auth Service checks status on token issuance
                                   and refuses to issue new tokens

Add a new business rule            INSERT into authz_rules (enabled = true)

Disable a rule temporarily         UPDATE authz_rules SET enabled = false

Grant a service access to a        INSERT into resource_access
resource type                      (principal_id = service id, resource_id = '*')
─────────────────────────────────────────────────────────────────────────────
```

The only cases that require a deployment are:

- Adding a brand new action type (e.g. a new verb like `export` that has never existed before)
- Changing how the rules evaluator interprets a new condition key
- Adding a new resource type to the system

Everything else is a database row.

---

## 9. How to Handle Exceptions and Blocks

### "All Users Except This User Can Read This Resource"

This is the most common exception pattern. Here is how it works with your tables:

```
1. The role grants default access to everyone with that role:
   role_permissions: role_pay_view → perm_acc_read

2. Users are assigned the role:
   principal_roles: usr_alice   → role_pay_view
   principal_roles: usr_bob     → role_pay_view
   principal_roles: usr_charlie → role_pay_view

3. Add a blocked row only for the exception user:
   resource_access: usr_charlie | blocked | account | ACC-001

Result:
  Alice calls read account ACC-001   →  PERMIT  (role grants it, no block)
  Bob calls read account ACC-001     →  PERMIT  (role grants it, no block)
  Charlie calls read account ACC-001 →  DENY    (blocked row found in Step 1)

To unblock Charlie later:
  DELETE from resource_access
  WHERE principal_id = 'usr_charlie'
  AND resource_id = 'ACC-001'
  AND relation = 'blocked'
```

### "Only This User Can Access This Resource"

```
1. Do NOT assign a role that includes access to this resource type.

2. Add a specific viewer or owner row:
   resource_access: usr_alice | owner | account | ACC-001

Result:
  Alice calls read account ACC-001   →  PERMIT  (relationship row found)
  Bob calls read account ACC-001     →  DENY    (no role, no relationship row)
```

### "This Partner Can Only Call During Business Hours"

```
1. Partner has their normal role assigned (role:partner-readonly).

2. Add a rule to authz_rules:
   {
     "effect": "DENY",
     "condition": {
       "principal_type": "partner",
       "outside_business_hours": [8, 20]
     }
   }

3. The Authorization Service evaluates this rule as part of Step 4.
   During business hours  →  rule does not fire  →  PERMIT
   Outside business hours →  rule fires          →  DENY
```

---

## 10. Where Authorization Lives in Each Service

Authorization is not scattered across services. It lives in one place and services call it.

```
┌───────────────────────────────────────────────────────────────────────────┐
│                          Your Microservices                                │
│                                                                           │
│  ┌─────────────────────┐          ┌─────────────────────┐                 │
│  │   Payment Service    │          │   Ledger Service    │                 │
│  │                     │          │                     │                 │
│  │  1. Receive request │          │  1. Receive request │                 │
│  │  2. Extract JWT     │          │  2. Extract JWT     │                 │
│  │  3. Call AuthZ ─────┼──────┐   │  3. Call AuthZ ─────┼──────┐         │
│  │  4. If PERMIT:      │      │   │  4. If PERMIT:      │      │         │
│  │     run business    │      │   │     run business    │      │         │
│  │     logic           │      │   │     logic           │      │         │
│  │  5. If DENY: 403    │      │   │  5. If DENY: 403    │      │         │
│  └─────────────────────┘      │   └─────────────────────┘      │         │
│                                │                                │         │
└────────────────────────────────┼────────────────────────────────┼─────────┘
                                 │                                │
                                 └──────────────┬─────────────────┘
                                                │
                                                ▼
                             ┌──────────────────────────────────┐
                             │      Authorization Service        │
                             │                                  │
                             │  The only service that reads:    │
                             │    - principals                  │
                             │    - roles                       │
                             │    - permissions                 │
                             │    - role_permissions            │
                             │    - principal_roles             │
                             │    - resource_access             │
                             │    - authz_rules                 │
                             │    - writes to authz_audit_log   │
                             │                                  │
                             │  Returns: PERMIT or DENY         │
                             └──────────────────────────────────┘
```

### What Services Must Do

```
Every service MUST:
  ✅ Call the Authorization Service before touching any data
  ✅ Pass the exact resource_id being accessed (not just the type)
  ✅ Return HTTP 403 immediately on DENY without running business logic
  ✅ Never cache authorization decisions longer than one request

Every service MUST NOT:
  ❌ Check roles or permissions directly from the database themselves
  ❌ Hardcode any role name or permission name in if/else logic
  ❌ Trust that the API Gateway already checked everything
  ❌ Skip the authorization check because the call came from another service
```

### A Note on Caching

The Authorization Service itself can maintain a short in-memory cache (e.g. 5 seconds) for repeated identical checks within a burst. Individual services should never cache decisions because a `blocked` row could be inserted at any moment and must take effect immediately.

---

## 11. Audit Logging

Every decision — PERMIT and DENY — is written to `authz_audit_log`. This is non-negotiable for a financial platform.

```
What gets logged for every check:
───────────────────────────────────────────────────────────
  principal_id     who made the request
  resource_type    what type of resource was accessed
  resource_id      which specific resource
  action           what they tried to do
  decision         PERMIT or DENY
  deny_reason      which step caused the deny (if DENY)
  rule_id          which rule fired (if applicable)
  trace_id         linked to the original HTTP request trace
  timestamp        when it happened
───────────────────────────────────────────────────────────

Example rows:

usr_alice    │ account │ ACC-001 │ read    │ PERMIT │ —                  │ 14:01:33
usr_charlie  │ account │ ACC-001 │ read    │ DENY   │ blocked_row        │ 14:01:55
prt_acmecorp │ payment │ PAY-001 │ read    │ DENY   │ no_role            │ 14:02:10
prt_acmecorp │ rate    │ USD-EUR │ read    │ DENY   │ rule_partner_hrs   │ 21:15:04
```

The audit log should be:

- Written by the Authorization Service only, never by individual services
- Append-only — no updates, no deletes
- Archived to cold storage after 90 days but never permanently deleted
- Queryable by your compliance and security teams

---

## 12. Building This in Phases

You do not need to build everything at once. Here is the order that gives you value fastest.

```
PHASE 1 — Role-Based Access (Week 1–2)
──────────────────────────────────────
Build:
  ✅ principals table
  ✅ roles table
  ✅ permissions table
  ✅ role_permissions table
  ✅ principal_roles table
  ✅ Authorization Service with Steps 3 and 4 of the check flow
  ✅ authz_audit_log table

What this gives you:
  Roles work. Users get permissions via roles.
  No hardcoded permission logic in any service.
  Audit trail from day one.

Not yet working:
  Per-entity access (alice owns account ACC-001 but not ACC-002)
  Exceptions and blocks

──────────────────────────────────────────────────────────────
PHASE 2 — Entity-Level Access (Week 3–4)
──────────────────────────────────────────────────────────────
Build:
  ✅ resource_access table
  ✅ Add Steps 1 and 2 to the Authorization Service check flow
  ✅ Admin API to manage resource_access rows

What this gives you:
  Per-entity grants and blocks work.
  "Alice owns account ACC-001" works.
  "Charlie is blocked from ACC-001" works.
  All access changes via API, no deployments.

──────────────────────────────────────────────────────────────
PHASE 3 — Conditional Rules (Week 5–6)
──────────────────────────────────────────────────────────────
Build:
  ✅ authz_rules table
  ✅ Rule evaluator in the Authorization Service (Step 4)
  ✅ The condition types your platform needs first:
     partner hours, account status, high-value payment threshold

What this gives you:
  Context-aware rules with no code deployment.
  New business rules added by inserting a database row.

──────────────────────────────────────────────────────────────
PHASE 4 — Operations and Observability (Ongoing)
──────────────────────────────────────────────────────────────
Build:
  ✅ Admin UI for managing roles, relationships, and rules
  ✅ Authorization decision trace endpoint (for debugging denies)
  ✅ Metrics: decisions per second, deny rate, rule hit rate
  ✅ Alerting on unusual deny spikes (potential attack signal)
  ✅ Audit log archive pipeline to cold storage
```

---

## Summary

The entire authorization system is built from **eight PostgreSQL tables** and **one Authorization Service** written in TypeScript. No external platforms, no graph databases, no policy language runtimes.

```
principals          →  who exists in the system
roles               →  named groups of permissions
permissions         →  individual allowed actions
role_permissions    →  which roles grant which permissions
principal_roles     →  which principal has which role
resource_access     →  direct grants and blocks per entity
authz_rules         →  conditional rules stored as data
authz_audit_log     →  every decision, forever
```

Every permission change is a database write. Every access decision is a query. No service contains authorization logic. The Authorization Service is the only component that reads these tables and returns a single answer: **PERMIT** or **DENY**.
