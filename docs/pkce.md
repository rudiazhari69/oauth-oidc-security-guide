# PKCE (Proof Key for Code Exchange)

PKCE (pronounced "pixy", RFC 7636) protects the authorization code flow
against **authorization code interception**: if someone steals the code, they
still cannot exchange it for tokens.

## How it works

1. Before redirecting the user, the client generates a random
   `code_verifier` (43–128 characters, high entropy).
2. It sends `code_challenge = BASE64URL(SHA256(code_verifier))` (method
   `S256`) in the authorize request.
3. After login, the authorization server returns the code to the redirect URI.
4. The client exchanges the code **plus the original `code_verifier`**.
5. The server hashes the verifier and compares it to the stored challenge.
   Match = tokens. Mismatch = rejected.

The verifier never travels through the browser's address bar — only the
challenge does, and the challenge cannot be reversed into the verifier.

## What it protects

- **Code interception on mobile / custom schemes:** another app on the
  device that claims the same redirect scheme can receive the code, but
  without the verifier it cannot use it.
- **Code leakage via logs, referrers, or history:** a leaked code alone is
  useless.
- **Confused clients:** the code is bound to the client instance that
  started the flow, not just to the registered client id.

## What breaks when PKCE is missing

- The authorize request has no `code_challenge` at all. Any intercepted code
  can be exchanged by anyone who can present the client id (public clients
  have no secret, so there is nothing else to check).
- This is the classic companion to a redirect-URI issue: a loose redirect
  URI leaks the code, and no PKCE means the code is directly usable.

## What breaks when PKCE is not enforced

These are subtler, and worth checking one by one:

1. **Challenge sent but verifier not checked.** The server accepts the token
   exchange with a wrong, empty, or missing `code_verifier`. Test (on your
   own accounts, authorized scope only): complete a login, then replay the
   token exchange with a wrong verifier. If it succeeds, PKCE is decoration.
2. **`plain` method accepted.** `code_challenge_method=plain` sends the
   verifier itself as the challenge — anyone who sees the authorize request
   learns the verifier. The server should require `S256` (or at least reject
   `plain` for new clients).
3. **Verifier not bound to the code.** The server stores the challenge
   globally or per client rather than per authorization request, so a
   verifier from one login can be reused for another login's code.
4. **Downgrade.** The server accepts a flow that *starts* without a
   challenge and completes without one, even though the client normally sends
   one. An attacker can simply omit the challenge from a crafted authorize
   request.

## How to test safely

Use only your own accounts, on systems you are authorized to test.

1. Capture a normal authorize request. Is `code_challenge` present, and is
   the method `S256`?
2. Complete the flow, capture the token exchange, and replay it with:
   - a wrong `code_verifier`,
   - an empty `code_verifier`,
   - no `code_verifier` at all.
   Any success is a finding — record which variant worked.
3. Try `code_challenge_method=plain` in the authorize request. Is it
   accepted?
4. Try starting the authorize request with no challenge at all. Does the
   server still issue a code, and can it be exchanged without a verifier?

## What good looks like

- PKCE required for all clients, public and confidential.
- Only `S256` accepted; `plain` rejected.
- Verifier checked on every exchange, bound to the specific authorization
  request, single-use.
- A flow started without a challenge cannot complete.
