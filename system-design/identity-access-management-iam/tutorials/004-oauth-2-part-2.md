# OAuth2 Flow Explained: SPA to Backend API (Direct PKCE & BFF Patterns, With and Without API-GW)

---

## Overview

### Executive Summary

This tutorial explains how a Single Page Application (SPA) securely talks to a Backend API using OAuth2, following the two industry-standard patterns end to end:

- **Part 1: Direct SPA + PKCE.** The SPA is a public client and runs the Authorization Code Flow with PKCE itself: it gets tokens from the Authorization Server and calls the API with a Bearer token.
- **Part 2: BFF (Backend for Frontend).** A confidential server-side client owns the OAuth2 flow. Tokens never reach the browser; the SPA only holds an opaque `HttpOnly` session cookie while the BFF attaches tokens to outgoing calls.

Each pattern is covered in two deployment variants: **Without API-GW** (the SPA or BFF calls the Backend API directly) and **With API-GW** (an API-GW becomes the single public entry point, handling TLS termination, JWT validation, rate limiting, WAF, and routing, while the Backend API stays private and re-validates tokens under zero trust).

Every variant ships with a sequence diagram, step-by-step walkthrough, parameter tables, and a hop-by-hop best-practices analysis showing how each layer (SPA, BFF, API-GW, Backend API) secures the layer behind it. The tutorial closes with a BFF vs. direct-PKCE comparison, the drawbacks of the BFF pattern, and guidance on when BFF is actually worth it.

### Key Outcomes

After reading this tutorial, you will be able to:

1. **Run the Authorization Code Flow + PKCE** from a SPA: generate `code_verifier`/`code_challenge`, build the `/authorize` request, verify `state` against Login CSRF, and exchange the code at the token endpoint.
2. **Explain every parameter** of the authorize, callback, and token requests: what is mandatory, what each one protects against, and why.
3. **Implement the BFF pattern**: confidential client with `client_secret`, server-side session store, `HttpOnly`/`Secure`/`SameSite` cookies, and invisible token refresh and logout.
4. **Place an API-GW correctly** in both patterns, including the two BFF topologies (gateway between BFF and backends, or gateway in front of everything for same-origin deployment).
5. **Apply hop-by-hop security best practices**: PKCE and `state` at login, strict CORS and TLS at the public edge, JWT validation and rate limiting at the API-GW, mTLS and network isolation between API-GW and Backend API, and zero-trust token re-validation in the Backend API itself.
6. **Choose the right pattern** for your context, using the BFF vs. direct-PKCE comparison, the BFF drawbacks list, and the closing decision guidance.

---

## Part 1: OAuth2 Flow for a SPA (Authorization Code Flow + PKCE)

### Without API-GW

#### The 4 Actors

1. **User** – the person using your app
2. **SPA** – your frontend (React/Angular/Vue) running in the browser
3. **Authorization Server** – the identity provider (Auth0, Keycloak, Cognito, Azure AD, Google…)
4. **Backend API** – your server that holds the protected data (Spring boot Backend)

> **Key fact:** A SPA is a *public client*: anyone can view its source code, so it **cannot store a secret**. That's why we use **PKCE** (pronounced "pixy"), a dynamic one-time secret generated per login attempt.

---

#### Step-by-Step Flow

```
User ──► SPA ──► Authorization Server ──► SPA ──► Backend API
```

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant SPA as SPA (Browser)
    participant AS as Authorization Server
    participant API as Backend API

    User->>SPA: Click "Login"
    SPA->>SPA: Generate code_verifier + code_challenge (S256)
    SPA->>AS: GET /authorize (client_id, redirect_uri, scope, state, code_challenge)
    AS->>User: Login page (credentials, MFA, consent)
    User->>AS: Authenticate
    AS-->>SPA: 302 redirect to redirect_uri (code, state)
    SPA->>SPA: Verify state matches (CSRF check)
    SPA->>AS: POST /token (code, code_verifier, client_id, redirect_uri)
    AS->>AS: Verify PKCE: S256(code_verifier) == code_challenge
    AS-->>SPA: access_token + id_token (+ refresh_token)
    SPA->>API: GET /orders (Authorization: Bearer access_token)
    API->>API: Validate JWT (signature via JWKS, exp, iss, aud, scope)
    API-->>SPA: 200 OK (protected data)
```

**Step 1: User clicks "Login"**

The SPA generates two random values:
- `code_verifier` = a random secret string (kept in the SPA, never sent yet)
- `code_challenge` = SHA256 hash of the verifier (safe to send)

**Step 2: SPA redirects the browser to the Authorization Server**

```
GET https://auth-server.com/authorize?
    response_type=code
    &client_id=my-spa
    &redirect_uri=https://myapp.com/callback
    &scope=openid profile read:data
    &state=xyz123          ← CSRF protection
    &code_challenge=ABC... ← the hashed verifier
    &code_challenge_method=S256
```

| Parameter               | Sample Values                                                | Requirement     | Usage                 | Description                                                  |
| :---------------------- | :----------------------------------------------------------- | :-------------- | :-------------------- | :----------------------------------------------------------- |
| `client_id`             | `my-spa`<br>`ios-mobile-app`<br>`0oa1b2c3d4e5`               | **Mandatory**   | Client Identification | *Who is asking?* The public identifier for your SPA or mobile app. *The server cannot process any request without knowing which registered application is calling it.* |
| `response_type`         | `code` (Auth Code)<br>`token` (Implicit)<br>`id_token token` (Hybrid) | **Mandatory**   | Flow Selection        | *What kind of flow is this?* **`code` initiates the secure Authorization Code flow.** `token` initiates the older, less secure Implicit flow. |
| `redirect_uri`          | `https://myapp.com/callback`<br>`myapp://auth` (Mobile)<br>`http://localhost:3000` | **Mandatory**   | Callback Routing      | *Where do we send the answer?* The exact URL **where the server redirects the user after authentication.** Prevents attackers from sending the code to a malicious domain. |
| `code_challenge`        | `E9Melhoa2O...`<br>`vFrEMTJguC...`                           | **Mandatory**   | PKCE Security         | *How do we secure the handoff?* A hashed version of a secret random string (the `code_verifier`). It proves the app exchanging the code later is the exact app that requested it. |
| `state`                 | `xyz123`<br>`8a3f9b2e-4c1d...`<br>`eyJ0YXJnZXQi...` (Base64) | **Recommended** | CSRF Protection       | *Is this a forged request?* A random string used to prevent Cross-Site Request Forgery. Often used to store UUIDs or Base64-encoded app state to restore the user's UI after login. |
| `code_challenge_method` | `S256` (Standard)<br>`plain` (Insecure)                      | **Optional**    | Hashing Algorithm     | *How was the challenge secured?* Tells the server how the `code_challenge` was hashed. If omitted, it defaults to `plain` (insecure), so setting it to `S256` is highly critical. |
| `scope`                 | `openid profile email`<br>`read:data write:data`<br>`offline_access` | **Optional**    | Permission Request    | *What access is needed?* Defines the permissions requested. `offline_access` is commonly used to request a Refresh Token alongside the Access Token. |

**Step 3: User logs in at the Authorization Server**

Username/password, MFA, consent screen: all happen on the provider's page. **Your SPA never sees the password.** This is the whole point of OAuth2.

**Step 4: Redirect back with an Authorization Code**

```
https://myapp.com/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz123
```

The SPA verifies `state` matches what it sent (blocks CSRF attacks).

| Parameter | Sample Values                             | Requirement                       | Usage                | Description                                                  |
| :-------- | :---------------------------------------- | :-------------------------------- | :------------------- | :----------------------------------------------------------- |
| `code`    | `SplxlOBeZQQYb...`<br>`4/P7q7W91a-oMs...` | **Mandatory** (on success)        | Temporary Credential | The short-lived, one-time-use authorization code issued by the Authorization Server. The SPA will **immediately** send this to the token endpoint to *exchange for an Access Token.* |
| `state`   | `xyz123`<br>`8a3f9b2e-4c1d...`            | **Mandatory** (if sent in Step 2) | CSRF Protection      | The exact same random string the SPA sent in Step 2. The SPA *must* verify this matches its locally stored value before proceeding. If it doesn't match, the login is instantly aborted. |
| `error`   | `access_denied`<br>`invalid_request`      | **Conditional** (on failure)      | Error Handling       | If the user denies access (clicks "Cancel") or the request is invalid, the server returns this parameter instead of a `code`. |

#### What is CSRF (Cross-Site Request Forgery)?

Imagine you are attending a major tech conference. At check-in, the registration kiosk assigns you a unique, random pairing code that is securely saved in your conference mobile app. When you walk over to the VIP booth to claim your reserved swag bag (the callback), the scanner checks your app to verify your hidden pairing code before handing over the gear.

In OAuth, **CSRF** (specifically "Login CSRF") happens when an attacker tries to trick your browser into logging into *their* account instead of yours.

**How the attack works:**

1. The attacker starts a login process on their own machine and gets a valid `code` tied to *their* account.
2. They embed a hidden link on a malicious website you visit: `[https://myapp.com/callback?code=ATTACKER_CODE](https://myapp.com/callback?code=ATTACKER_CODE)`
3. Your browser visits that link without you knowing.
4. If your SPA doesn't check for that "pairing code," it exchanges the attacker's code, logs you into the attacker's account, and you might unknowingly save sensitive payment info or personal data into a profile the attacker completely controls.

**How the `state` parameter prevents it:** The `state` parameter is that unique pairing code. When your SPA starts the login, it generates a random `state` (e.g., `xyz123`), saves it in your browser's local memory, and sends it to the Auth Server. When the browser comes back, the SPA checks if the URL's `state` matches the one in memory. If an attacker tries to force a login using their own link, they won't know your browser's secret `state`, the match will fail, and the SPA will block the attack.

**Step 5: SPA exchanges the code for tokens**

```
POST https://auth-server.com/token
    grant_type=authorization_code
    &code=SplxlOBeZQQYbYS6WxSbIA
    &redirect_uri=https://myapp.com/callback
    &client_id=my-spa
    &code_verifier=ORIGINAL_SECRET ← proves "I'm the one who started this login"
```

| Parameter       | Sample Values                                                | Requirement   | Usage                 | Description                                                  |
| :-------------- | :----------------------------------------------------------- | :------------ | :-------------------- | :----------------------------------------------------------- |
| `grant_type`    | `authorization_code`<br>`refresh_token`<br>`client_credentials` | **Mandatory** | Flow Identification   | Tells the token endpoint exactly what kind of credential is being exchanged. For this specific step, it must strictly be `authorization_code`. |
| `code`          | `SplxlOBeZQQYb...`<br>`4/P7q7W91a-oMs...`                    | **Mandatory** | Authorization Proof   | The short-lived, one-time-use authorization code extracted from the URL in Step 4. |
| `redirect_uri`  | `https://myapp.com/callback`<br>`myapp://auth`               | **Mandatory** | Routing Verification  | Must exactly match the `redirect_uri` sent in the initial authorization request (Step 2). The server does *not* redirect the user here; it strictly uses it to verify the request's origin. |
| `client_id`     | `my-spa`<br>`ios-mobile-app`                                 | **Mandatory** | Client Identification | The public identifier for your SPA. Required so the server knows which application is attempting to exchange the code. |
| `code_verifier` | `a8b9c7d6e5f4...`<br>`secret_string_123...`                  | **Mandatory** | PKCE Verification     | The original, unhashed secret string the SPA generated before Step 1. The Auth Server hashes this value using `S256` and checks if it matches the `code_challenge` sent in Step 2. This proves the app exchanging the code is the exact same app that requested it. |

The server hashes the verifier, compares it to the challenge from Step 2: if they match, tokens are issued:

- **Access Token** (usually a JWT) → used to call the API
- **ID Token** → tells the SPA *who* the user is
- **Refresh Token** (optional) → gets new access tokens silently

**Step 6: SPA calls your Backend API**

```
GET https://api.myapp.com/orders
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

**Step 7: Backend API validates the token** (on every request)

- ✅ Signature valid? (checked against the Auth Server's public keys: JWKS)
- ✅ Not expired? (`exp` claim)
- ✅ Correct issuer? (`iss`)
- ✅ Meant for this API? (`aud`: audience)
- ✅ Has required permission? (`scope` / roles)

**Step 8: Response**

- Valid → API returns the data 🎉
- Expired → `401 Unauthorized`, and the SPA silently uses its **refresh token** to get a fresh access token, then retries.

---

#### Simple Analogy

- **Authorization Code** = a check-in voucher (useless alone, expires fast)
- **code_verifier / PKCE** = the ID you show at the desk to prove the voucher is yours
- **Access Token** = the room key card: you tap it everywhere (API calls), and it expires
- **Refresh Token** = your reservation, letting the desk issue a new key card without re-doing the whole check-in

---

#### Things to Remember

| Rule | Why |
|---|---|
| Never use the old **Implicit Flow** | Deprecated: tokens in the URL are unsafe |
| Never store tokens in `localStorage` if avoidable | XSS can steal them; prefer memory or `httpOnly` BFF cookies |
| Access token = for the **API**; ID token = for the **SPA** | Don't confuse authentication with authorization |
| Backend must validate the token itself | Never trust the frontend's word |

---

### With API-GW

The login journey (Steps 1-5 above) is **identical**: the gateway plays no role in authentication. What changes is everything *after* the SPA holds an access token. The SPA no longer calls the backend directly; all traffic goes through the **API-GW**, which becomes the single public entry point.

```
User ──► SPA ──► Authorization Server ──► SPA ──► API-GW ──► Backend API
```

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant SPA as SPA (Browser)
    participant AS as Authorization Server
    participant GW as API-GW
    participant API as Backend API (private network)

    User->>SPA: Click "Login"
    SPA->>SPA: Generate code_verifier + code_challenge (S256)
    SPA->>AS: GET /authorize (client_id, redirect_uri, scope, state, code_challenge)
    AS->>User: Login page (credentials, MFA, consent)
    User->>AS: Authenticate
    AS-->>SPA: 302 redirect (code, state)
    SPA->>AS: POST /token (code, code_verifier)
    AS-->>SPA: access_token + id_token (+ refresh_token)
    Note over SPA,AS: Login identical to "Without API-GW"
    SPA->>GW: GET /orders (Authorization: Bearer access_token, HTTPS)
    GW->>GW: Edge checks: TLS, JWT (JWKS, exp, iss, aud, scope), rate limit, WAF
    alt Token invalid or expired
        GW-->>SPA: 401 Unauthorized (rejected at the edge)
    else Token valid
        GW->>API: Forward GET /orders over mTLS (Bearer token propagated)
        API->>API: Re-validate JWT + per-resource authorization (zero trust)
        API-->>GW: 200 OK (protected data)
        GW-->>SPA: 200 OK (+ security headers)
    end
```

**Step 6: SPA calls the API-GW (not the backend)**

```
GET https://gw.myapp.com/orders
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

The backend API has **no public endpoint anymore**. It lives in a private network segment and only accepts traffic arriving from the gateway.

**Step 7: The gateway validates the request (first line of defense)**

Before any backend code runs, the gateway checks:

- ✅ TLS: the connection is encrypted; the gateway is the TLS termination point
- ✅ JWT signature valid? (verified against cached JWKS public keys from the Auth Server)
- ✅ Not expired? (`exp`), correct issuer? (`iss`), correct audience? (`aud`)
- ✅ Required scopes present? (coarse checks, e.g. `read:data` for `GET /orders`)
- ✅ Rate limits / quotas for this client or user? (throttling)
- ✅ IP allowlists, WAF rules, known attack patterns (SQLi, XSS payloads, oversized payloads)

Invalid or expired tokens are rejected with `401` **at the edge**: they never consume backend resources.

**Step 8: The gateway forwards to the backend over a secured internal channel**

```
GET https://orders-backend.internal/orders        (private network, over mTLS)
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...     (original token propagated)
```

- The gateway routes to the right service (`/orders` → orders service, `/payments` → payments service).
- The gateway-to-backend connection uses **mTLS**: both sides prove their identity with certificates.
- **Best practice: the backend re-validates the JWT anyway** (zero trust). Never rely on "the gateway already checked it": if the gateway is bypassed or misconfigured, the backend still protects itself. Fine-grained authorization (can *this* user see *this* order?) always happens in the backend.

**Step 9: Response flows back through the gateway**

The backend responds to the gateway; the gateway can add security headers (`Strict-Transport-Security`, `Content-Security-Policy`), strip internal headers, and returns the response to the SPA. On `401`, the SPA refreshes its token and retries, exactly as before.

---

#### Backend Re-validation: Zero Trust vs. Gateway Trust

Once the API-GW validates the JWT at the edge, you face a design decision: should the Backend API **re-validate the same token anyway** (zero trust), or **trust the API-GW's verdict** and skip validation (gateway trust, where the gateway asserts identity via headers like `X-User-Id` or a gateway-signed token)?

| Aspect | Option A: Backend re-validates JWT (zero trust) | Option B: Backend trusts the API-GW (gateway trust) |
|---|---|---|
| **Trust model** | Trust nothing: every layer independently verifies identity | Trust the edge: validation happens once at the API-GW |
| **If the API-GW is bypassed or misconfigured** | Backend still rejects invalid/missing tokens: attack fails | Backend accepts anything that reaches it: full unauthenticated access |
| **Backend implementation** | Every service needs a JWT validation library + JWKS caching | Backend just reads identity headers: no auth code at all |
| **Latency & CPU** | Small cost per request (signature check, usually cached keys) | Minimal: header parsing only |
| **Header spoofing risk** | None: headers alone never grant access | Real: if the backend is reachable directly, forged `X-User-Id` headers are accepted |
| **Hard infrastructure requirements** | None beyond standard JWT libraries | Strict: private subnet, mTLS, network policies allowing *only* the API-GW, and you must be certain they can never loosen |
| **Lateral movement** | An attacker inside the network still needs valid tokens | An attacker inside the network can call backends freely |
| **Compliance / audit** | Fits zero-trust mandates (NIST SP 800-207, banking/finance standards) | Often rejected by auditors: one checkpoint is a single point of failure |
| **Token flexibility** | Backend sees the real token: full claims, scopes, fine-grained authz | Backend sees only what the gateway chooses to forward |

**How to choose:**

- ✅ **Default to Option A (re-validate).** It is the industry-recommended posture: edge validation stays as a fast, coarse filter, and backend validation is your guarantee when anything upstream fails.
- ⚠️ **Option B is acceptable only when** all of these hold: the Backend API has *no* other network path (proven, enforced, monitored), mTLS is mandatory on the internal hop, the data is low-sensitivity, and you accept that a single network misconfiguration becomes a security incident.
- 🔀 **Common middle ground:** re-validate the JWT *signature and expiry* cheaply in the backend, but rely on the API-GW for expensive checks (revocation lists, introspection calls).

> **Rule of thumb:** the API-GW's validation is an optimization; the backend's validation is the control. Never let the optimization replace the control.

---

#### Best Practices: SPA → API-GW → Backend API

How each hop secures the layer behind it:

| Hop | Exposure | Security controls | How it protects the next layer |
|---|---|---|---|
| **SPA → Auth Server** | Public | PKCE, `state`, HTTPS | The SPA never sees the password; a stolen authorization code is useless without the `code_verifier` |
| **SPA → API-GW** | Public (internet-facing) | HTTPS/TLS 1.2+, short-lived access tokens, strict CORS (exact SPA origin only, never `*`) | Only authenticated, well-formed calls from your own SPA origin reach the edge |
| **At the gateway** | Edge | JWT validation (signature, `exp`, `iss`, `aud`, scopes), rate limiting, WAF, DDoS protection, request size limits | Attacks and invalid tokens are absorbed at the edge; the backend only sees legitimate, validated traffic |
| **API-GW → Backend API** | Private network | mTLS, private subnet (no public ingress), network policy allowing *only* the gateway, JWT re-validation in the backend | Even if someone reaches the internal network, they cannot call the backend without the gateway's certificate and a valid token |
| **Backend itself** | Last line | Re-validate JWT, enforce fine-grained per-resource authorization, never trust gateway-added headers blindly | Zero trust: a misconfigured or bypassed gateway does not expose data |

Key rules:

| Rule | Why |
|---|---|
| Backend must never be publicly reachable | Forces all traffic through the gateway so edge policies cannot be bypassed |
| Validate the JWT twice (edge + backend) | Edge validation is fast and cheap; backend validation is your zero-trust guarantee |
| Keep edge validation coarse, backend validation fine | The gateway checks the token is valid and broadly authorized; the backend decides per-resource access |
| Use short-lived access tokens | A leaked token expires quickly, limiting damage if any hop is compromised |
| Lock CORS to the SPA's exact origin | Prevents other websites from riding the user's token |

---

## Part 2: OAuth2 with BFF (Backend for Frontend)

In this pattern, the **SPA never touches tokens at all**. Your BFF (a small backend dedicated to your frontend, e.g., Node/Express, .NET, Java) becomes the OAuth2 client. The browser only holds an **opaque session cookie**.

---

### Without API-GW

#### The Flow (Step by Step)

```
Browser (SPA)              BFF                    Auth Server            Backend API
     │                      │                         │                      │
     │── /auth/login ──────►│── 302 redirect ────────►│                      │
     │                      │    (+client_id, PKCE,   │                      │
     │                      │     state, redirect_uri)│                      │
     │◄────────── user logs in on Auth Server page ───│                      │
     │                      │◄── callback?code=... ───│                      │
     │                      │── code + client_secret ─►│  ← token exchange   │
     │                      │◄── access + refresh ────│     (server-to-server)
     │                      │   stores tokens in       │                      │
     │                      │   SESSION (Redis/mem)    │                      │
     │◄─ Set-Cookie: sid ───│                          │                      │
     │                      │                          │                      │
     │── GET /api/orders ──►│── looks up session,      │                      │
     │   (cookie only)      │   attaches Bearer token ─┼─────────────────────►│
     │◄──────── data ───────│◄─────────────── data ────┼──────────────────────│
```

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant SPA as Browser (SPA)
    participant BFF as BFF
    participant AS as Auth Server
    participant API as Backend API

    User->>SPA: Click "Login"
    SPA->>BFF: GET /auth/login
    BFF->>BFF: Generate state, nonce, PKCE verifier/challenge, store in session
    BFF-->>SPA: 302 redirect to Auth Server /authorize
    SPA->>AS: GET /authorize (client_id, PKCE, state, redirect_uri)
    AS->>User: Login page (credentials, MFA, consent)
    User->>AS: Authenticate
    AS-->>BFF: GET /auth/callback (code, state)
    BFF->>BFF: Validate state against session
    BFF->>AS: POST /token (code, code_verifier, client_id, client_secret)
    AS-->>BFF: access_token + refresh_token
    BFF->>BFF: Store tokens in session store (Redis)
    BFF-->>SPA: Set-Cookie: session=abc123 (HttpOnly, Secure, SameSite)
    SPA->>BFF: GET /api/orders (Cookie: session=abc123)
    BFF->>BFF: Look up session, attach access token
    BFF->>API: GET /orders (Authorization: Bearer access_token)
    API->>API: Validate JWT (signature, exp, iss, aud, scopes)
    API-->>BFF: 200 OK (data)
    BFF-->>SPA: 200 OK (data)
```

**Step 1: Login starts at the BFF, not the SPA**

User clicks "Login" → SPA calls `GET https://bff.myapp.com/auth/login`.

**Step 2: BFF redirects to the Authorization Server**

The BFF generates `state`, `nonce`, and the PKCE `code_verifier`/`code_challenge`, **stores them in a server-side session**, then sends back a `302` redirect to `/authorize`.

**Step 3: User authenticates**

Login, MFA, consent: all on the Auth Server's page. Neither the SPA nor the BFF sees the password.

**Step 4: Callback hits the BFF (not the SPA!)**

```
https://bff.myapp.com/auth/callback?code=SplxlOBe...&state=xyz123
```

The BFF validates `state` against its session.

**Step 5: BFF exchanges the code for tokens**

```
POST /token
    grant_type=authorization_code
    &code=SplxlOBe...
    &code_verifier=...
    &client_id=my-bff
    &client_secret=██████   ← the BFF is a CONFIDENTIAL client; the secret lives safely on the server
```

**Step 6: Tokens stay server-side; browser gets only a cookie**

The BFF stores `access_token` + `refresh_token` in its session store (Redis, DB, memory) and replies with:

```
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax
```

`HttpOnly` = JavaScript **cannot read this cookie**. The SPA literally cannot see any token.

**Step 7: SPA calls the BFF like a normal API**

```
GET https://bff.myapp.com/api/orders
Cookie: session=abc123
```

No `Authorization` header, no token handling in the frontend at all.

**Step 8: BFF proxies to the real API**

The BFF looks up the session, grabs the access token, and calls the downstream Backend API:

```
GET https://api.myapp.com/orders
Authorization: Bearer eyJhbGciOi...
```

The Backend API validates the JWT exactly as before (signature, `exp`, `iss`, `aud`, scopes).

**Step 9: Refresh & logout are invisible to the SPA**

- Token expired? The BFF silently uses the refresh token server-side and retries.
- Logout? The BFF deletes the session, clears the cookie, and optionally calls the Auth Server's end-session endpoint. Tokens die instantly: nothing lingers in the browser.

---

### With API-GW

With a BFF, the gateway sits **between the BFF and the backend services**. The browser journey (login, code exchange, session cookie) is unchanged; what changes is where the BFF's outgoing calls land. Two common topologies:

1. **Gateway between BFF and backends:** `Browser → BFF → API-GW → Backend services`. The BFF stays next to the SPA; the gateway fronts all microservices.
2. **Gateway in front of everything:** `Browser → API-GW → BFF → API-GW → Backend services`. One public entry point; the gateway routes `/` to the SPA hosting and `/auth`, `/api` to the BFF. This keeps the SPA and BFF **same-origin**, which removes CORS entirely and keeps the session cookie first-party.

**Topology 1: gateway between BFF and backends**

```
Browser (SPA)        BFF                 Auth Server        API-GW         Backend API
     │                │                       │                  │                  │
     │ (Steps 1-6: login, code exchange, session cookie: unchanged)                  │
     │                │                       │                  │                  │
     │── GET /api/orders ►│                   │                  │                  │
     │   (cookie only) │── looks up session,  │                  │                  │
     │                │   attaches Bearer ────┼─────────────────►│── JWT validated, │
     │                │                       │                  │   rate-limited,  │
     │                │                       │                  │   routed ───────►│
     │◄────── data ───│◄──────────────────────┼──────────────────│◄──── data ──────│
```

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant SPA as Browser (SPA)
    participant BFF as BFF
    participant AS as Auth Server
    participant GW as API-GW
    participant API as Backend API (private network)

    User->>SPA: Click "Login"
    SPA->>BFF: GET /auth/login
    BFF-->>SPA: 302 redirect to /authorize (PKCE, state)
    SPA->>AS: Login + MFA + consent
    AS-->>BFF: GET /auth/callback (code, state)
    BFF->>AS: POST /token (code, code_verifier, client_secret)
    AS-->>BFF: access_token + refresh_token
    BFF->>BFF: Store tokens in session store
    BFF-->>SPA: Set-Cookie: session (HttpOnly, Secure, SameSite)
    Note over SPA,BFF: Login identical to "Without API-GW"
    SPA->>BFF: GET /api/orders (Cookie only)
    BFF->>GW: GET /orders (Authorization: Bearer access_token, mTLS / API key)
    GW->>GW: Edge checks: JWT (JWKS, exp, iss, aud, scope), rate limit, WAF, routing
    alt Token invalid or expired
        GW-->>BFF: 401 Unauthorized
        BFF->>AS: Refresh token grant (server-side, silent)
        AS-->>BFF: New access_token
        BFF->>GW: Retry GET /orders (new Bearer token)
    end
    GW->>API: Forward over mTLS (Bearer token propagated)
    API->>API: Re-validate JWT + per-resource authorization (zero trust)
    API-->>GW: 200 OK (data)
    GW-->>BFF: 200 OK (data)
    BFF-->>SPA: 200 OK (data)
```

**Topology 2: gateway in front of everything**

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant SPA as Browser (SPA)
    participant GW as API-GW (single public entry)
    participant BFF as BFF
    participant AS as Auth Server
    participant API as Backend API (private network)

    User->>SPA: Click "Login"
    SPA->>GW: GET /auth/login (same-origin)
    GW->>BFF: Route /auth/* to BFF
    BFF-->>SPA: 302 redirect to /authorize (PKCE, state)
    SPA->>AS: Login + MFA + consent
    AS-->>BFF: GET /auth/callback (code, state)
    BFF->>AS: POST /token (code, code_verifier, client_secret)
    AS-->>BFF: access_token + refresh_token
    BFF->>BFF: Store tokens in session store
    BFF-->>SPA: Set-Cookie: session (HttpOnly, Secure, SameSite, first-party)
    SPA->>GW: GET /api/orders (Cookie only, same-origin)
    GW->>BFF: Route /api/* to BFF
    BFF->>GW: GET /orders (Authorization: Bearer access_token, mTLS / API key)
    GW->>GW: Edge checks: JWT (JWKS, exp, iss, aud, scope), rate limit, WAF, routing
    GW->>API: Forward over mTLS (Bearer token propagated)
    API->>API: Re-validate JWT + per-resource authorization (zero trust)
    API-->>GW: 200 OK (data)
    GW-->>BFF: 200 OK (data)
    BFF-->>GW: 200 OK (data)
    GW-->>SPA: 200 OK (data)
```

**Step 7: SPA calls the BFF (cookie only)**: unchanged.

```
GET https://bff.myapp.com/api/orders
Cookie: session=abc123
```

**Step 8: BFF calls the API-GW with the Bearer token**

The BFF looks up the session, grabs the access token, and calls the gateway instead of calling a backend service directly:

```
GET https://gw.myapp.com/orders
Authorization: Bearer eyJhbGciOi...
```

The gateway applies the edge policies: JWT validation, rate limiting, WAF, and routes to the right backend service. The forward to the backend happens over mTLS, and the backend re-validates the token and enforces per-resource authorization.

> Whether the Backend API should re-validate the token or simply trust the API-GW's checks is a design decision with real trade-offs. See the full comparison in Part 1: **With API-GW → Backend Re-validation: Zero Trust vs. Gateway Trust**. The conclusion applies identically here.

**Step 9: Refresh & logout**: unchanged. The BFF handles refresh server-side; neither the SPA nor the gateway is involved.

**The separation of concerns gets cleaner with a gateway:** the BFF keeps *token custody* (sessions, refresh, logout), while the gateway owns *edge policy* (validation, throttling, routing, WAF). Neither one needs to do the other's job.

---

#### Best Practices: SPA → BFF → API-GW → Backend API

How each hop secures the layer behind it:

| Hop | Exposure | Security controls | How it protects the next layer |
|---|---|---|---|
| **SPA → BFF** (cookie hop) | Public | `HttpOnly` + `Secure` + `SameSite` cookie, anti-CSRF token, origin/referer checks, HTTPS | Tokens are invisible to JavaScript: XSS cannot exfiltrate what it cannot read; CSRF defenses stop forged cookie calls before they reach the BFF |
| **BFF → Auth Server** (login hop) | Public (Auth Server) | Confidential client (`client_secret`), PKCE, `state`, HTTPS, tokens stored server-side | Credentials and tokens never touch the browser; the token exchange happens server-to-server where the secret is safe |
| **BFF → API-GW** (token hop) | Private (or gateway as the single public entry) | Bearer token from the server-side session, mTLS or API key so the gateway knows the caller is a trusted BFF, token audience (`aud`) restricted to the target APIs | The gateway only accepts calls carrying valid user tokens from known clients; a token leaked from another context is rejected by `aud` checks |
| **API-GW → Backend API** | Private network | mTLS, network policy allowing only the gateway, JWT re-validation, per-resource authorization | The backend trusts nothing by default: every request re-proves identity and permission |
| **BFF session store** | Server-side | Redis with encryption at rest, short session lifetimes, refresh handled server-side | A browser compromise reveals nothing; a session can be killed instantly on the server |

Key rules specific to BFF + Gateway:

| Rule | Why |
|---|---|
| Same-origin SPA + BFF (via gateway routing or a shared domain) | Removes CORS entirely and keeps the session cookie first-party |
| Restrict the token audience (`aud`) to the APIs the BFF actually needs | A stolen token cannot be replayed against unrelated services |
| Never forward the session cookie downstream | The cookie is a browser-BFF contract; backend services only understand Bearer tokens |
| Put the BFF behind the gateway's TLS/WAF where possible | One hardened public edge instead of two exposed surfaces |
| BFF authenticates to the gateway (mTLS client cert or API key) | The gateway can reject token-bearing calls that did not come from a trusted BFF |

---

### BFF vs. Direct SPA + PKCE

| Aspect | SPA + PKCE (direct) | BFF pattern |
|---|---|---|
| **Where tokens live** | Browser (JS memory / storage) | Server-side session store |
| **Client type** | Public client (no secret) | Confidential client (has `client_secret`) |
| **What JS can see** | Access + refresh tokens | Nothing: only an opaque `HttpOnly` cookie |
| **XSS damage** | Attacker can exfiltrate tokens | Attacker can't read the cookie; tokens unreachable |
| **Refresh tokens** | Risky in browser (needs rotation) | Safe server-side |
| **OAuth logic** | Lives in the SPA (oidc-client, MSAL.js…) | Lives in the BFF; SPA is "dumb" |
| **CSRF** | Not an issue (Bearer headers aren't auto-sent) | **Back on the table**: cookies are auto-attached → need `SameSite`, CSRF tokens, origin checks |
| **CORS** | Often needed (SPA → API cross-origin) | Can deploy same-origin → CORS disappears |
| **Statelessness** | Fully stateless API | BFF is **stateful** (session store required) |
| **Logout / revocation** | Hard: token copies float in the browser | Instant: delete the server session |
| **Architecture** | Simple: SPA → API | Extra hop: SPA → BFF → API |
| **One API, many clients** | API is generic, reusable by anyone | BFF is tailored *for one specific frontend* |

**The one-line summary:** you trade *frontend simplicity and stronger token protection* for *backend complexity and statefulness*. That's why security-sensitive industries (banking, finance) often mandate it.

---

### Drawbacks of the BFF Approach

**🏗️ Operational complexity**
- A whole extra service to build, deploy, monitor, and scale.
- It becomes a **single point of failure**: BFF down = frontend is dead, even if all APIs are healthy.

**💾 Statefulness & scaling pain**
- Sessions need a shared store (Redis) or sticky sessions; pure horizontal scaling gets harder.
- Alternative: encrypt tokens into the cookie itself: but cookies cap at ~4 KB, and JWTs + refresh tokens often exceed that.

**🍪 CSRF is your problem again**
- Bearer tokens in headers are immune to CSRF; cookies are not. You must implement `SameSite`, anti-CSRF tokens, and strict origin/referer checks.

**🐢 Latency & proxy friction**
- Every API call takes an extra hop (browser → BFF → API).
- File uploads, streaming, WebSockets, and SSE become awkward to proxy through the BFF.

**📱 Doesn't generalize to other clients**
- The cookie/session model fits browsers. A mobile app or third-party integrator can't use it: you still need the direct token flow (PKCE) for them, so you may end up maintaining **both** patterns.

**🎯 The BFF becomes a high-value target**
- It holds *every* logged-in user's tokens. A server-side compromise is far worse than a single browser's XSS.

**🔁 Release coupling**
- Since the BFF is shaped around one frontend's needs, frontend and BFF changes often ship together: more coordination, less independent deployability.

---

## When is BFF actually worth it?

- ✅ **Use it when:** compliance forbids tokens in the browser, you need instant revocation, or the app is browser-only with high security demands.
- ❌ **Skip it when:** you have mobile clients, a public API, or a small team that can't afford the operational overhead: SPA + PKCE with short-lived tokens is still a perfectly secure, standards-approved pattern.
