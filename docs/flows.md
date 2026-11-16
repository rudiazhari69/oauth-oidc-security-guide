# OAuth 2.0 / OIDC Flows

The flow is the sequence of steps between "user clicks Sign in" and "the app
knows who the user is". The flow you use decides which attacks are possible.

## The parties

- **Resource Owner** — the user.
- **Client** — the application asking for access (e.g. `app.example.com`).
- **Authorization Server** — the service that authenticates the user and
  issues tokens (e.g. `login.example.com`).
- **Resource Server** — the API that accepts access tokens.

## 1. Authorization Code flow (the standard)

The only flow new applications should use.

```
User            Client (app.example.com)     Authorization Server (login.example.com)
 |                       |                                   |
 |  1. Click "Sign in"  |                                   |
 |---------------------->|                                   |
 |                       |  2. Redirect to authorize         |
 |                       |    response_type=code               |
 |                       |    client_id=YOUR_CLIENT_ID         |
 |                       |    redirect_uri=https://app.example.com/callback
 |                       |    scope=openid profile             |
 |                       |    state=RANDOM_STATE               |
 |                       |    code_challenge=SHA256(verifier)  |
 |                       |---------------------------------->|
 |                       |                                   | 3. User logs in,
 |                       |                                   |    approves scopes
 |                       |  4. Redirect back with code       |
 |                       |<----------------------------------|
 |                       |  5. Exchange code + code_verifier |
 |                       |    for tokens (server-to-server)    |
 |                       |---------------------------------->|
 |                       |  6. access_token + id_token       |
 |                       |<----------------------------------|
 |  7. Logged in         |                                   |
 |<----------------------|                                   |
```

Why this flow is the safe default:

- The **authorization code is short-lived and single-use**.
- Tokens are exchanged **server-to-server** — they never pass through the
  browser.
- Combined with **PKCE** (see [pkce.md](pkce.md)), a stolen code is useless
  without the verifier.
- The **state** parameter binds the response to the request the user started
  (CSRF protection).

## 2. PKCE (Proof Key for Code Exchange)

PKCE adds a per-request secret to the authorization code flow. The client
creates a random `code_verifier`, sends its hash (`code_challenge`) in step 2,
and must present the original verifier in step 5. The authorization server
checks that they match.

PKCE was designed for public clients (mobile apps, single-page apps) that
cannot keep a client secret, but it is now **recommended for all clients** —
including confidential ones. See [pkce.md](pkce.md) for what breaks when it is
missing or not enforced.

## 3. Implicit flow (legacy — do not use)

In the implicit flow the authorization server returns the **access token
directly in the URL fragment** (`#access_token=...`) after login. No code
exchange happens.

Why it is legacy:

- Tokens travel through the browser and end up in **browser history, logs,
  and referrer headers**.
- There is no client authentication at any step, so a token in the URL can be
  stolen by any script or extension that can read the fragment.
- The OAuth 2.0 Security Best Current Practice document recommends against it
  for new implementations.

If you are reviewing an application and still see
`response_type=token`, that alone is a finding worth reporting (severity
depends on what the token can do — see [token-issues.md](token-issues.md)).

## 4. Other flows, briefly

- **Client Credentials** — server-to-server, no user involved. Fine for
  machine-to-machine, but the token must be tightly scoped.
- **Resource Owner Password Credentials (ROPC)** — the app collects the
  user's password directly. Legacy; avoid it. It defeats the whole point of
  OAuth (the app should never see the password).
- **Device Authorization Grant** — for input-constrained devices (TVs,
  CLIs). Watch for slow polling abuse and for codes that stay valid too long.

## What to check first

1. Which `response_type` does the login request use? `code` is expected.
2. Is `state` present, random per request, and validated on return?
3. Is PKCE in use (`code_challenge` in the authorize request)?
4. Where do tokens go — server-to-server exchange, or through the browser?
