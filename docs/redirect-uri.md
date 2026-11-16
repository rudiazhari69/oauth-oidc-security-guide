# Redirect URI Validation

The `redirect_uri` tells the authorization server where to send the user (and
the authorization code) after login. If the server does not validate it
strictly, an attacker can redirect the code to a site they control — and a
code is as good as a login.

## How it is supposed to work

1. The client registers one or more **exact** redirect URIs when it is created
   (e.g. `https://app.example.com/callback`).
2. At login time the client sends a `redirect_uri` in the authorize request.
3. The authorization server checks that the sent URI **exactly matches** a
   registered one before redirecting. Anything else is rejected.

## What goes wrong

### 1. Loose matching (prefix / substring / regex)

Some servers check only that the redirect URI *starts with* or *contains* the
registered value:

- Registered: `https://app.example.com/callback`
- Accepted: `https://app.example.com/callback.evil.example` (prefix match)
- Accepted: `https://app.example.com/callback@evil.example` (userinfo trick)
- Accepted: `https://app.example.com/callback%2f..%2f..` (path confusion)

A code sent to an attacker-controlled host is a full account takeover of the
victim who clicked the link.

### 2. Wildcard redirect URIs

Some providers allow patterns such as `https://*.example.com/callback`. A
wildcard is only as safe as the weakest subdomain it covers:

- an unclaimed or dangling subdomain under `*.example.com` can be taken over;
- any subdomain that hosts user content, open redirects, or file uploads can
  be used to receive the code.

If a wildcard is in use, check whether any matching subdomain is
attacker-controllable.

### 3. Open redirect on the registered URI itself

Even an exactly-matched redirect URI can be unsafe if the page it points to
is itself an open redirect (e.g. `https://app.example.com/redirect?url=...`).
The code flows to the client first, then the open redirect forwards the
browser — including the code in the URL — to the attacker's site.

### 4. Scheme and host confusion

- `http://` allowed alongside `https://` (code sent in cleartext on a
  network an attacker can observe).
- Trailing-slash, case, port, and encoding differences treated as equal by
  the server but not by the browser (`https://app.example.com:443/callback`
  vs `https://app.example.com/callback`).
- Mobile / custom-scheme URIs (`myapp://callback`) claimed by more than one
  app on a device.

## How to test safely

Only on systems you are authorized to test. Use **your own accounts** and a
domain you control (e.g. `https://attacker.example` in your own lab, or a
request-catcher you own) as the target — never a third party's domain.

1. Start a normal login and capture the authorize request.
2. Replay it with the `redirect_uri` changed to your own test host. Does the
   server accept it? (If yes, that is a finding — record the exact request
   and response, redact the code.)
3. Try variations of the registered URI: add a path segment, a subdomain,
   `@evil.example`, an encoded slash, a trailing dot. Note which ones are
   accepted.
4. Check whether the registered callback page itself redirects onward
   (open redirect after the code arrives).
5. Confirm the impact honestly: a code leak is only an account takeover if
   the code can actually be exchanged (check PKCE enforcement and whether
   the code is bound to the client — see [pkce.md](pkce.md)). Report the
   chain as it is, not as it could be in the worst case.

## What good looks like

- Exact-match validation against a registered list; no wildcards.
- `https` only; no `http` fallback.
- One redirect URI per environment, not a shared list across apps.
- The callback page never redirects onward with the code in the URL.
- Authorization codes are short-lived, single-use, and (with PKCE) bound to
  the client that started the flow.
