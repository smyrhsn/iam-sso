## OIDC Flows — Auth Code, PKCE, Hybrid, Implicit
 
| Flow | `response_type=` | id_token from | access_token from | Refresh token | Recommendation |
|---|---|---|---|---|---|
| Auth Code | `code` | `/token` | `/token` | ✅ Yes | ✅ Recommended |
| Auth Code + PKCE | `code` + `code_challenge=` | `/token` | `/token` | ✅ Yes | ✅ Best practice for public clients |
| Hybrid | `code id_token` | `/authorize` (fast identity) | `/token` | ✅ Yes | ⚠️ Acceptable for legacy web apps |
| Implicit | `token` or `id_token` | `/authorize` (unsafe) | `/authorize` (unsafe) | ❌ No | ❌ Deprecated — avoid |
| Client Credentials | *(POST only)* | N/A — no user | `/token` | ❌ No | ✅ Server-to-server only |
 
**Implicit vs Hybrid — the key difference:**
 
Both return tokens from `/authorize`, but Hybrid also returns a `code` to exchange server-side:
 
| | Tokens from `/authorize` | Code for `/token` | Refresh token | `access_token` safe? |
|---|---|---|---|---|
| Implicit | ✅ Both tokens | ❌ No | ❌ No | ❌ Exposed in URL fragment |
| Hybrid | ✅ `id_token` only | ✅ Yes | ✅ Yes | ✅ Comes from `/token` |
| Auth Code | ❌ No | ✅ Yes | ✅ Yes | ✅ Never in browser |
 
> **Why Implicit is deprecated:** Tokens in the URL fragment are visible in browser history, referrer headers, and logs. PKCE solves the same problem securely. OAuth 2.1 removes Implicit entirely.
 
**Hybrid flow step by step:**
 
| Step | Endpoint | What happens | What you get |
|---|---|---|---|
| 1 | `/authorize` | User authenticates at IdP | `code` + `id_token` in URL fragment |
| 2 | Client app | App reads `id_token` immediately | User identity — name, email, sub |
| 3 | `/token` | App exchanges code server-side | `access_token` + `refresh_token` |
| 4 | Resource API | App calls API with `access_token` | User's data |
 
---
