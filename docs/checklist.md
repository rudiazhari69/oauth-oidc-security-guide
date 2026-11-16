# OAuth / OIDC Testing Checklist

One page. Work through it on your own test accounts, on systems you are
authorized to test. Tick each item, and note the exact request/response for
anything that fails.

## Setup

- [ ] Scope confirmed: the login flow is in scope, and I am using only my
      own accounts.
- [ ] Baseline captured: a normal, successful login (authorize request,
      callback, token exchange) saved with all parameters.

## Flow

- [ ] `response_type` is `code` (not `token` / implicit).
- [ ] `state` is present, random per request, and validated on return.
- [ ] `code_challenge` (PKCE, method `S256`) is present in the authorize
      request.
- [ ] Tokens are exchanged server-to-server, not through the browser.

## Redirect URI

- [ ] `redirect_uri` must exactly match a registered value.
- [ ] Tested: your own test host as `redirect_uri` — accepted or rejected?
- [ ] Tested: path/subdomain/`@` tricks on the registered URI.
- [ ] Tested: wildcard coverage — is any matching subdomain
      attacker-controllable?
- [ ] The callback page does not redirect onward with the code in the URL.
- [ ] `https` only; no `http` fallback.

## PKCE

- [ ] Token exchange with a **wrong** verifier — rejected?
- [ ] Token exchange with an **empty / missing** verifier — rejected?
- [ ] `code_challenge_method=plain` — rejected?
- [ ] Authorize request with **no challenge** — can it complete?

## Tokens

- [ ] No access token or code appears in a URL (fragment or query).
- [ ] API calls use `Authorization: Bearer`, not `?access_token=`.
- [ ] Issued scopes match what was requested and what the feature needs.
- [ ] ID token validated: `aud`, `iss`, `exp`, signature (on your own token).
- [ ] Refresh token only issued when genuinely needed; rotation/revocation
      supported.
- [ ] `Referrer-Policy` set on token-handling pages.

## Login CSRF

- [ ] `state` validated — replaying a callback without the original `state`
      fails.
- [ ] Completing a login started by another session does not log the victim
      in as someone else.

## Before you report

- [ ] Reproduced from a clean state (fresh browser / session).
- [ ] Impact proven on your own test account — what can the token/code
      actually do?
- [ ] Duplicates checked; the asset is in scope.
- [ ] Proof of concept is minimal; all secrets, codes, and tokens are
      `REDACTED`.
- [ ] Severity matches the proven impact, not the worst case.
