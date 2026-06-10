---
layout: post
title: "Authentral: Centralized Auth + Externalized Authorization for Airline Ground Ops"
date: 2026-06-10
categories: [security, architecture]
tags: [oauth2, pkce, opa, rego, fastapi, microsoft-entra, authorization]
author: swapnak15
---

This post walks through **Authentral** — a portfolio project that demonstrates centralized authentication via Microsoft Entra ID (OAuth 2.0 + PKCE) and externalized authorization via Open Policy Agent (OPA). The scenario is an airline ground operations portal where two roles — Gate Manager and Station Master — have distinct, policy-enforced capabilities.

Live demo: [authentral.vercel.app](https://authentral.vercel.app)  
Source: [github.com/swapnak15/authentral](https://github.com/swapnak15/authentral)

---

## The Problem

Most applications mix _who you are_ (authentication) with _what you can do_ (authorization) inside the same service. This makes policy changes expensive — you rewrite code, re-test, redeploy. It also makes auditing hard: the rules are scattered across `if user.role == "admin"` checks rather than in a single, inspectable policy.

Authentral separates the two concerns:

| Concern | Responsibility | Technology |
|---|---|---|
| Authentication | Verify identity and issue tokens | Microsoft Entra ID (Azure AD) |
| Authorization | Decide what the token-holder may do | Open Policy Agent (OPA) |
| Enforcement | Enforce the decision at the API layer | FastAPI middleware (the PEP) |

---

## Authentication: OAuth 2.0 + PKCE via Microsoft Entra ID

### Why PKCE?

The classic OAuth 2.0 Authorization Code flow relies on a `client_secret` to redeem an authorization code for tokens. That secret cannot be safely stored in a browser or mobile app — it can be extracted from JavaScript bundles or app storage. **PKCE (Proof Key for Code Exchange, RFC 7636)** replaces the static secret with a per-request cryptographic challenge:

1. The browser generates a random `code_verifier` (a 43–128 character random string).
2. It computes `code_challenge = BASE64URL(SHA256(code_verifier))`.
3. The authorization request sends only the `code_challenge` and `code_challenge_method=S256`.
4. The token request sends the original `code_verifier`. Entra hashes it and compares — no secret ever leaves the client.

This means a stolen authorization code is worthless without the verifier that only the original browser tab holds.

### The Flow Step by Step

```
Browser                      Entra ID                    FastAPI (API)
   |                             |                             |
   |-- 1. Login button clicked   |                             |
   |   Generate code_verifier    |                             |
   |   Compute code_challenge    |                             |
   |                             |                             |
   |-- 2. GET /authorize ------->|                             |
   |   (code_challenge, scopes)  |                             |
   |                             |                             |
   |<- 3. Login page shown ------|                             |
   |   User enters credentials   |                             |
   |                             |                             |
   |<- 4. Redirect: code=... ----|                             |
   |                             |                             |
   |-- 5. POST /token ---------->|                             |
   |   (code, code_verifier)     |                             |
   |                             |                             |
   |<- 6. {access_token, ...} ---|                             |
   |                             |                             |
   |-- 7. GET /me ---------------|---------------->            |
   |   Authorization: Bearer <access_token>                   |
   |                             |   Validate token (JWKS)    |
   |                             |   Extract claims           |
   |<- 8. {upn, roles, ...} -----|<----------------            |
```

### Token Validation on the API

When the access token arrives at the FastAPI backend, `auth.py` validates it against Entra's public keys:

```python
# auth.py (simplified)
jwks_client = PyJWKS(config.JWKS_URL)          # fetches https://login.microsoftonline.com/{tenant}/discovery/v2.0/keys

def verify_token(token: str) -> dict:
    signing_key = jwks_client.get_signing_key_from_jwt(token)
    return jwt.decode(
        token,
        signing_key.key,
        algorithms=["RS256"],
        audience=config.API_AUDIENCE,           # must match the registered app URI
        issuer=config.ISSUER,                   # must match tenant's issuer URL
    )
```

Three claims matter here:
- **`aud` (audience)** — confirms the token was issued for _this_ API, not some other resource.
- **`iss` (issuer)** — confirms it came from your specific Entra tenant, not a rogue identity provider.
- **`roles`** — custom app roles (`gate-manager`, `station-master`) assigned to users in Entra.

A token that passes signature verification but fails audience or issuer validation is rejected — this prevents token replay across different APIs in the same tenant.

### App Registration in Azure Portal

Key settings that enforce the above:

| Setting | Value | Why |
|---|---|---|
| Supported account types | Single tenant | Prevents logins from other organizations |
| Platform | SPA (Single Page Application) | Enables PKCE; disables client_secret requirement |
| Redirect URI | `https://authentral.vercel.app` | Entra only redirects to this URL after auth |
| Access token version | 2 | Returns v2 JWT format with `roles` claim |
| App roles defined | `gate-manager`, `Station.Master` | Embedded in token; used by OPA policy |

---

## Authorization: Open Policy Agent + Rego

Once identity is established, a second question arises: _is this identity allowed to perform this action on this resource?_ That decision belongs to OPA — a general-purpose policy engine that evaluates Rego policies against a JSON input document.

### The PEP → PDP Pattern

```
FastAPI endpoint (PEP — Policy Enforcement Point)
        |
        | build input doc from:
        |   - token claims (roles, UPN)
        |   - user attributes (assigned_station, assigned_zones)
        |   - request context (action, resource station, resource zone)
        |
        v
   OPA HTTP endpoint (PDP — Policy Decision Point)
        |
        | evaluate authz.rego against input
        |
        v
   {"allow": true/false, "reason": "..."}
        |
        v
FastAPI: 200 OK  —or—  403 Forbidden
```

The PEP (FastAPI) never contains policy logic — it only gathers facts and enforces the decision. The PDP (OPA) never executes application code — it only evaluates rules. This strict separation means policy changes are deployable independently of the application.

### The Rego Policy

```rego
# policy/authz.rego
package authentral.authz

import rego.v1

default allow := false

# --- view a zone / view seat summary ---
allow if {
    input.action in {"view", "view_seat_summary"}
    input.resource.station == input.user.assigned_station
    some role in input.user.roles
    role in {"gate-manager", "station-master"}
    _view_zone_ok(role, input.user.assigned_zones, input.resource.zone)
}

# --- gate reassignment ---
allow if {
    input.action == "reassign"
    input.resource.station == input.user.assigned_station
    _reassign_ok(input.user.roles, input.user.assigned_zones, input.resource.zone)
}

# --- seat manifest + upgrade (station-master only) ---
allow if {
    input.action in {"view_seat_manifest", "process_upgrade"}
    input.resource.station == input.user.assigned_station
    "station-master" in input.user.roles
}

# helpers
_view_zone_ok("station-master", _, _) := true
_view_zone_ok("gate-manager", assigned_zones, zone) := zone in assigned_zones

_reassign_ok(roles, assigned_zones, zone) if {
    "station-master" in roles
}
_reassign_ok(roles, assigned_zones, zone) if {
    "gate-manager" in roles
    zone in assigned_zones
}
```

Every `allow` block must pass _all_ its conditions — Rego uses conjunction within a block and disjunction across blocks. If none of the `allow` blocks fires, the default `allow := false` applies.

### The Input Document

A typical authorization call from the flight service looks like this:

```python
# opa_client.py — build and send input doc
input_doc = {
    "user": {
        "upn": "gate.manager@swapnakm15gmail.onmicrosoft.com",
        "roles": ["gate-manager"],
        "assigned_station": "ORD",
        "assigned_zones": ["T1-B"]
    },
    "action": "reassign",
    "resource": {
        "station": "ORD",
        "zone": "T2-A"   # ← gate manager is NOT assigned to T2-A
    }
}
# OPA response: {"allow": false, "reason": "denied: 'reassign' not permitted ..."}
```

The `user.assigned_station` and `user.assigned_zones` fields come from `attributes.json` — a side-channel store that holds ground-ops-specific attributes not present in the JWT. This keeps the token lean while allowing the policy to reason about physical assignments.

---

## Airline Scenario: Roles and Permissions Matrix

The scenario models the ORD (Chicago O'Hare) hub. Zones `T1-A`, `T1-B`, `T2-A`, `T2-B` are terminal gate clusters.

| Action | Gate Manager | Station Master | Notes |
|---|---|---|---|
| View flight list | Own zones only | All zones | Filtered by `assigned_zones` |
| View seat summary | Own zones only | All zones | Zone-level aggregate |
| Reassign gate | Own zones only | Any zone | Write action; audit-logged |
| View seat manifest | ✗ | ✓ | PII-adjacent; station-master only |
| Process upgrade | ✗ | ✓ | Inventory change; station-master only |

**Why this distinction matters in the policy:**

A Gate Manager assigned to `T1-B` can reassign a flight _within_ `T1-B` — this is their day-to-day job. But they cannot touch `T2-A` gates (wrong zone) and cannot view seat manifests (not their purview). A Station Master overrides zone restrictions because they oversee the whole station, but the station restriction still applies — an ORD station master cannot act on LAX resources.

**Example deny decision:**

```json
{
  "allow": false,
  "reason": "denied: 'reassign' not permitted for role(s) ['gate-manager'] on resource station=ORD zone=T2-A (caller assigned_station=ORD, assigned_zones=['T1-B'])"
}
```

The reason string is returned to the frontend as a `403` detail — useful for debugging, and it makes the policy auditable without grepping application logs.

---

## Vercel Deployment: Inline Policy Engine

OPA runs as a sidecar in Docker dev (where the PEP→PDP separation is fully demonstrable). On Vercel, OPA isn't available as a sidecar, so `opa_client.py` switches to an inline Python translation of the same Rego rules:

```python
# opa_client.py
_USE_INLINE = bool(os.environ.get("VERCEL"))

async def evaluate(input_doc: dict) -> dict:
    if _USE_INLINE:
        from policy_engine import evaluate as _inline
        return _inline(input_doc)
    # ... real OPA HTTP call
```

`policy_engine.py` is a faithful Python mirror of `authz.rego` — same input shape, same output shape, same allow/deny logic. The architectural property (policy separated from enforcement code) is preserved even without the OPA server.

---

## Security Checklist

- **PKCE** — no client_secret in browser; code interception useless without verifier
- **Token audience validation** — `aud` claim must match this API's App ID URI
- **Issuer pinning** — `iss` must match the specific Entra tenant
- **Role claims from Entra** — roles embedded in signed JWT, not user-supplied
- **Attribute side-channel** — station/zone assignments in `attributes.json`, not in token (token stays lean)
- **Default-deny policy** — OPA's `default allow := false`; unlisted actions are denied
- **Station boundary** — cross-station actions are always denied, regardless of role
- **Fail-closed OPA client** — if the OPA server is unreachable, `evaluate` returns `{"allow": false}`
- **S2S JWT** — internal service-to-service calls (flight service → seat service) use a shared `SERVICE_JWT_SECRET`; requests without a valid internal token are rejected before they reach business logic
- **Read-only filesystem** — logging is guarded by `os.environ.get("VERCEL")` to avoid crashes on Vercel's read-only runtime

---

## Running Locally

```bash
# 1. Clone and configure
git clone https://github.com/swapnak15/authentral
cp .env.example api/.env   # fill in TENANT_ID, CLIENT_ID, etc.

# 2. Start all services (FastAPI + OPA sidecar)
docker compose up -d --build

# 3. Open the app
open http://localhost:3000

# 4. Run OPA policy tests
docker compose exec opa opa test /policy -v
```

The OPA test suite (`policy/authz_test.rego`) covers every allow/deny combination for each role and action.

---

## What's Next

- Replace `attributes.json` with a database-backed attribute store
- Add OPA bundle signing to verify policy integrity at load time
- Extend the policy to support time-based rules (e.g. gate managers can only reassign during their shift window)
- Add structured audit logging with correlation IDs across PEP and PDP
