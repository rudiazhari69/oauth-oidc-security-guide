# Token Issues

Access tokens and ID tokens are the keys to the account. Most OAuth
vulnerabilities are, at the end, a story about a token ending up somewhere it
should not be — or a token that can do more than it should.

All examples below use placeholders (`login.example.com`, `REDACTED`).
Nothing here targets a real service.

## 1. Tokens in URLs

Access tokens (or authorization codes) that appear in a URL can leak through:

- **Browser history** — anyone with access to the device can read them.
- **Server logs** — URLs are commonly logged by proxies, CDNs, and the
  application itself.
- **Referer header** — when the page with the token in its URL loads a
  third-party resource (image, script, analytics), the full URL — token
  included — is sent in the `Referer` header.

Where this happens:

- The **implicit flow** returns the access token in the URL fragment
  (`#access_token=REDACTED`). See [flows.md](flows.md) — this flow is legacy.
- A callback page that **redirects onward** with the code or token still in
  the URL (open redirect after login — see
  [redirect-uri.md](redirect-uri.md)).
- An application that puts the access token in a **query parameter** for API
  calls (`GET /api/profile?access_token=REDACTED`) instead of the
  `Authorization: Bearer` header.

Rule: tokens belong in headers or POST bodies, never in URLs.

## 2. Over-scoped tokens

A token's **scope** is the list of permissions it grants. Problems:

- The client requests broad scopes it does not need (`read write admin`
  when it only reads a profile). A leaked token then grants far more than
  the feature requires.
- The authorization server **ignores the requested scope** and issues a
  token with everything the user has. Check what the issued token can
  actually do, not just what was requested.
- **Offline access / refresh tokens** issued by default: a refresh token
  turns a short-lived leak into a long-lived one. It should only be issued
  when the client genuinely needs it, and it should be rotatable and
  revocable.

Check: decode the issued token (ID tokens are JWTs — decode the payload,
never the signature, on your own token) and compare the scopes/claims with
what the feature needs.

## 3. Missing `state` parameter (login CSRF)

The `state` parameter binds the login response to the login request the user
started. Without it — or if the server does not validate it — an attacker
can perform **login CSRF**:

1. The attacker starts a login flow on their own account and gets a valid
   authorization code (or a login link) for the victim's browser.
2. The attacker tricks the victim's browser into completing that flow
   (e.g. the victim visits a page that triggers the callback).
3. The victim ends up logged in as the **attacker**. Anything the victim
   does next — entering payment details, uploading files, changing settings
   — goes into the attacker's account.

This is often rated low on its own, but its impact is real wherever the
logged-in session accumulates user data. `state` must be random per request,
stored server-side (or in a signed cookie), and validated on return.

## 4. Token leakage via Referer

Even when tokens are not in the URL by design, they can leak through the
`Referer` header:

- A page that received a token in its URL loads third-party content.
- An application **echoes the token** into a page that links out.
- A redirect chain carries the token as a query parameter to the next hop.

Mitigations to look for: `Referrer-Policy: no-referrer` (or
`strict-origin-when-cross-origin`) on token-handling pages, and tokens never
placed in URLs in the first place.

## 5. Account-takeover shapes (described generically)

These are the impact chains these issues combine into. They are described
here so you can judge severity honestly — not as instructions against any
real service:

- **Code interception → token exchange.** A code leaked through a loose
  redirect URI, combined with missing or unenforced PKCE, is exchanged for
  tokens. Impact: full account access for the victim who clicked.
- **Login CSRF → data planting.** Missing `state` lets an attacker log the
  victim in as the attacker; victim data lands in the attacker's account.
- **Token in URL → token theft.** A token in a URL is read from history,
  logs, or a `Referer` header and reused directly.
- **Over-scoped token → privilege escalation.** A token issued for a
  low-privilege feature can call admin APIs because the scope was not
  enforced.
- **ID token confusion.** An ID token issued for one client is accepted by
  another (missing `aud` validation), or a token from one issuer is
  accepted as if from another (missing `iss` validation). The fix is
  strict validation of `aud`, `iss`, `exp`, and signature on every token.

## Severity notes

- Judge by **what the token can actually do**, proven on your own test
  account — not by the worst case you can imagine.
- A token that can only read a public profile is a low-severity issue even
  if it leaks; a token that can change email, password, or payment details
  is not.
- If you cannot prove the token is usable (for example PKCE blocks the
  exchange), say so in the report. An unproven chain is a hardening
  suggestion, not a critical finding.
