# OpenID Connect (OIDC) Single Sign-On Lab

A hands-on lab where I configured an **OpenID Connect** login flow between an OpenID Provider and a Relying Party, ran the full authorization request, and then decoded the resulting **ID Token (JWT)** to read the identity claims Auth0 issued. This is the modern counterpart to my [SAML 2.0 lab](../saml-sso-lab) — same goal (federated SSO), very different mechanics.

---

## What this demonstrates

- OpenID Connect / OAuth 2.0 fundamentals
- The difference between authentication (OIDC) and authorization (OAuth 2.0)
- Registering a client, configuring scopes, and the callback-URL security model
- Decoding and reading a JWT ID Token and its claims

---

## Background: how OIDC works (and how it differs from SAML)

**OpenID Connect** is an identity layer built on top of **OAuth 2.0**. OAuth 2.0 by itself handles *authorization* — granting an app delegated access to resources. OIDC adds *authentication* on top: a standard way to prove **who the user is** and pass that identity to the app.

Where SAML uses heavy XML assertions and is common in legacy enterprise apps, OIDC uses lightweight **JSON Web Tokens (JWTs)** and is the standard for modern web and mobile apps. Same two-party trust model, different envelope:

| Concept | SAML | OIDC |
|--------|------|------|
| Identity source | Identity Provider (IdP) | **OpenID Provider (OP)** |
| The app | Service Provider (SP) | **Relying Party (RP)** |
| Token format | XML assertion | **JSON Web Token (JWT)** |
| Built on | Its own spec | **OAuth 2.0** |
| Typical use | Legacy enterprise | Modern web / mobile |

In this lab:
- **OpenID Provider (OP)** — **Auth0**, which authenticates the user and issues the ID Token.
- **Relying Party (RP)** — **[oidcdebugger.com](https://oidcdebugger.com/)**, the app requesting the identity.

---

## Tools used

| Role | Tool |
|------|------|
| OpenID Provider (OP) | Auth0 |
| Relying Party (RP) | oidcdebugger.com |
| Token inspection | [jwt.io](https://www.jwt.io/) |

---

## The login flow

> This lab uses the **implicit flow** (`response_type=id_token`), which returns the ID Token straight from the authorization endpoint. It's ideal for *seeing* the token directly, which is the point of the exercise. (In production the modern choice is Authorization Code + PKCE — see next steps.)

```mermaid
sequenceDiagram
    participant U as User (Browser)
    participant RP as Relying Party (oidcdebugger.com)
    participant OP as OpenID Provider (Auth0)

    U->>RP: 1. Click "Send Request"
    RP->>U: 2. Redirect to OP /authorize<br/>(client_id, redirect_uri, scope, response_type=id_token)
    U->>OP: 3. Follow redirect to Auth0
    OP->>U: 4. Present login form
    U->>OP: 5. Submit credentials
    OP->>U: 6. Consent screen (authorize profile + email)
    U->>OP: 7. Accept
    OP->>U: 8. Redirect to redirect_uri with the ID Token (JWT)
    U->>RP: 9. RP receives & displays the token
    Note over RP: Signature would be validated against the<br/>OP's public key (JWKS); claims iss/aud/exp/nonce checked
```

---

## Walkthrough

### Step 1 — Configured Auth0 as the OpenID Provider

In the Auth0 dashboard I went to **Applications → Applications → Create Application**, named it `OIDC Testing App`, and chose **Single Page Web Application**. The app type isn't cosmetic — it tells Auth0 this is a public, browser-based client (no client secret), which governs which flows are allowed.

On the **Settings** tab I grabbed two values the RP would need:
- **Domain** — the OP's base URL / issuer (e.g. `dev-1234.us.auth0.com`).
- **Client ID** — the RP's public identifier.

Then I scrolled to **Allowed Callback URLs** and entered exactly `https://oidcdebugger.com/debug`, and clicked **Save Changes**.

That callback allowlist is one of the most important security controls in OAuth/OIDC: the OP will **only** return tokens to a pre-registered URL. Without it, an attacker could swap in their own `redirect_uri` and have the token delivered to them. Registering the exact URL shuts that down.

### Step 2 — Configured oidcdebugger as the Relying Party

Over on [oidcdebugger.com](https://oidcdebugger.com/), I filled in the request that the RP would send to the OP:

- **Authorize URI** → `https://YOUR_DOMAIN/authorize` (my Auth0 domain). This is the OP's **authorization endpoint**, where every OIDC flow begins.
- **Redirect URI** → `https://oidcdebugger.com/debug` — matched *exactly* to the callback I registered in Auth0. Any mismatch and the OP refuses.
- **Client ID** → the value I copied from Auth0, identifying the RP.
- **Scope** → `openid profile email`. `openid` is **required** — it's the scope that turns a plain OAuth request into an OIDC one and asks for an ID Token. `profile` and `email` request those specific claims. Asking only for what the app needs is least-privilege in action.
- **Response type** → `id_token`, which selects the implicit flow and returns the ID Token directly.

Then I clicked **Send Request**.

### Step 3 — Authenticated, then decoded the token

Sending the request redirected me to Auth0, where I logged in as my test user. Auth0 then showed a **consent screen** asking whether to authorize the app to access my profile and email — this is the OAuth consent step, where the user explicitly grants the app access to their identity data. I clicked **Accept**.

I was redirected back to the OIDC Debugger success screen, which displayed a long string highlighted in green: the **ID Token**, a JWT. I copied it and pasted it into the **Encoded** box at [jwt.io](https://www.jwt.io/) to decode it.

A JWT is three Base64URL-encoded parts separated by dots — `header.payload.signature`:

- **Header** — the signing algorithm (e.g. `RS256`) and the key ID (`kid`) used to sign the token.
- **Payload** — the **claims**, the actual identity data. Key OIDC claims to look for:
  - `iss` — the **issuer** (my Auth0 domain), proving where the token came from.
  - `sub` — the **subject**, a stable unique ID for the user.
  - `aud` — the **audience** (my Client ID); the token is only valid for this app.
  - `exp` / `iat` — expiry and issued-at timestamps that bound the token's lifetime.
  - `nonce` — replay protection tying the token to the original request.
  - Plus the requested claims: `name`, `email`, `email_verified`, `picture`, etc.
- **Signature** — Auth0's cryptographic signature. A real RP verifies it against the OP's **public key** (published at the OP's JWKS endpoint) before trusting anything inside.

---

## Key takeaways

- **OIDC = authentication on top of OAuth 2.0.** The `openid` scope is literally what makes the request an OIDC request and produces an ID Token.
- **The ID Token is a JWT** — self-contained JSON, far lighter than a SAML XML assertion, and readable/verifiable by anyone with the OP's public key.
- **Claims carry the trust**: `iss` says who issued it, `aud` says who it's for, `exp` limits its lifetime, and the **signature** proves it wasn't tampered with.
- **The callback allowlist and scopes are the security backbone** — restricting where tokens go and what data an app can request.
- Same federated-SSO goal as SAML, but a modern, JSON-based, developer-friendly implementation.

---

## Possible next steps

- Rebuild this with the **Authorization Code flow + PKCE** — the current best practice for SPAs and mobile — and compare it to the implicit flow used here.
- Inspect the OP's **JWKS endpoint** (`/.well-known/jwks.json`) and validate the token signature manually instead of trusting the decoder.
- Request an **access token** alongside the ID token and call the `/userinfo` endpoint.
- Add **custom claims** in Auth0 and confirm they appear in the decoded payload.
- Put this side by side with the [SAML lab](../saml-sso-lab) to contrast token size, format, and flow.
