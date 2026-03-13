While reviewing the source code, I focused on how OpenClaw handles HTTP requests — specifically the `fetchWithSsrFGuard()` function, which is responsible for making server-side fetch calls.

I noticed something off.

When a request followed a cross-origin redirect (meaning the server sent a `3xx` response pointing to a different domain), OpenClaw was supposed to strip sensitive headers before forwarding the request to the new destination. It did — but only for a narrow hardcoded denylist:

```
Authorization, Proxy-Authorization, Cookie, Cookie2
```

The problem? That list is incomplete.

Custom authorization headers like `X-Api-Key`, `Private-Token`, or any other bearer-style header that developers commonly use — none of those were stripped. They were forwarded as-is to the redirect destination.

This means: if an attacker could control or influence where a redirect pointed, they could receive sensitive credentials that were never intended for them.

---

## Why This Is Dangerous

Imagine your application uses OpenClaw to call an internal API with a custom `X-Api-Key` header. A malicious server responds with a redirect to an attacker-controlled URL. OpenClaw follows the redirect — and forwards your API key right along with it.

Game over. Your credentials are now in someone else's hands.

**CVSS Score: 7.5 (High)**
* Attack Vector: Network
* Attack Complexity: High
* Confidentiality Impact: High
* No privileges required, no user interaction needed

---

## The Fix

The maintainers replaced the denylist approach with a safe-header allowlist. Instead of trying to block known bad headers, the new logic only allows known safe headers through on cross-origin redirects — things like content negotiation and cache validators. Everything else gets stripped by default.

This is the correct approach. Denylist-based security is fragile; allowlist-based security is robust.

**References**
* GitHub Advisory: [GHSA-6mgf-v5j7-45cr](https://github.com/advisories/GHSA-6mgf-v5j7-45cr)
* Fix Commit: `46715371b0612a6f9114dffd1466941ac476cef5`
* Affected versions: `<= 2026.3.2`
* Patched version: `>= 2026.3.7`

---

## If You're Using OpenClaw

Update immediately to `>= 2026.3.7`.

If you use custom authorization headers (like `X-Api-Key` or `Private-Token`) and you were on an older version, treat those credentials as potentially compromised and rotate them.

---
