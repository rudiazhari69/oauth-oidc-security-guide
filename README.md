# OAuth 2.0 & OpenID Connect Security Guide

A practical guide to the most common OAuth 2.0 and OpenID Connect (OIDC)
misconfigurations — what they are, why they matter, and how to check for them.

## What OAuth and OIDC are

- **OAuth 2.0** is an *authorization* framework: it lets a user grant an
  application limited access to their resources on another service, without
  sharing their password. The application gets an **access token** scoped to
  specific permissions.
- **OpenID Connect (OIDC)** is an *authentication* layer on top of OAuth 2.0:
  it adds an **ID token** (a signed JWT) that tells the application who the
  user is.

Most "Sign in with ..." buttons you see are OAuth 2.0 + OIDC.

## What this guide covers

| Document | Contents |
| -------- | -------- |
| [docs/flows.md](docs/flows.md) | The main flows: authorization code, PKCE, and why implicit is legacy |
| [docs/redirect-uri.md](docs/redirect-uri.md) | Redirect URI validation issues |
| [docs/pkce.md](docs/pkce.md) | PKCE: what it protects and what breaks without it |
| [docs/token-issues.md](docs/token-issues.md) | Token handling issues: URLs, scopes, state, leakage |
| [docs/checklist.md](docs/checklist.md) | A one-page testing checklist |

## ⚠️ Authorized testing only

OAuth flows touch **real user accounts and real login sessions**. Only test
these on systems you are authorized to test — a bug bounty program in scope,
or your own test applications. Never:

- test a login flow against a service you have no permission to test;
- use another person's authorization code, access token, or session;
- attempt an account takeover against a real service or real user;
- automate login attempts (brute force) against any login endpoint.

If you find an issue, report it through the proper channel with a minimal,
redacted proof of concept. Never publish a working exploit.

## Who this is for

- Bug bounty hunters testing web programs with social login.
- Developers implementing "Sign in with ..." who want to avoid the classic
  mistakes.
- Security reviewers checking an OAuth integration.

## License

[MIT](LICENSE)
