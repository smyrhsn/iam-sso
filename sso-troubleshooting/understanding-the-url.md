# SSO Troubleshooting — URL Dissection One Pager
> Read any SSO URL in seconds · OIDC · SAML

---

## 1 · Identify Protocol From URL

| Signal in URL | Protocol | Means |
|---|---|---|
| `client_id=` & `response_type=` | OIDC | Authorization request → IdP `/authorize` endpoint |
| `SAMLRequest=` in URL | SAML | SP-initiated — decode to inspect XML |
| `SAMLResponse=` in POST body | SAML | IdP assertion POSTed to SP's ACS URL |
| `scope=openid` present | OIDC | id_token will be returned |
| `code_challenge=` present | OIDC PKCE | Public client — SPA or mobile app |
| `RelayState=` present | SAML / some OIDC IdPs | Return-URL tracking. In OIDC, `state=` is the spec equivalent |

---

## 2 · OIDC — Grant Type (read `response_type=`)

| `response_type=` value | Grant Type | Notes |
|---|---|---|
| `code` | Authorization Code | Server-side app — most common |
| `code` + `code_challenge=` | Auth Code + PKCE | Public client — prevents code interception |
| `token` | Implicit | Legacy / deprecated — token in URL fragment |
| `code id_token` | Hybrid | id_token from `/authorize`, access_token from `/token` |
| *(POST only — no URL redirect)* | Client Credentials | Server-to-server — check POST body |
| *(POST `grant_type=refresh_token`)* | Refresh Token | Renewing expired access token |

---

## 3 · OIDC — Key URL Parameters

| Parameter | Identifies | Failure if wrong |
|---|---|---|
| `client_id=` | App registered at IdP | `invalid_client` error |
| `redirect_uri=` | Where token/code is returned | Must match exactly — even trailing slash matters |
| `scope=` | Claims requested (must include `openid`) | Missing `openid` → not an OIDC request |
| `state=` | CSRF protection token | Mismatch on callback → session/CSRF error |
| `nonce=` | Replay protection for id_token | Required for Implicit & Hybrid flows |
| `code_challenge=` | PKCE verifier hash (S256) | Missing when IdP requires PKCE → failure |
| `prompt=none` | Silent auth — no UI shown | Fails with `login_required` if no active session |
| `acr_values=` | Auth strength / MFA level required | IdP can't satisfy → auth failure |

---

## 4 · OIDC — Callback Error Codes

| `error=` value | Root cause | First action |
|---|---|---|
| `access_denied` | User denied or IdP policy block | Check user is assigned to the app |
| `invalid_client` | `client_id` or `client_secret` wrong | Verify app registration in IdP |
| `invalid_request` | Missing / malformed parameter | Check `client_id`, `redirect_uri`, `scope` |
| `invalid_scope` | Scope not registered for this client | Check allowed scopes in IdP app config |
| `invalid_grant` | Code expired or `redirect_uri` mismatch | Confirm `redirect_uri` identical on both legs |
| `login_required` | `prompt=none` but no active session | User must re-authenticate interactively |
| `unauthorized_client` | Grant type not allowed for this client | Check allowed flows in IdP registration |

---

## 5 · SAML — Decoded SAMLRequest Fields

| XML Field | Must equal | Failure if wrong |
|---|---|---|
| `Destination` | IdP SSO URL exactly | IdP rejects: invalid destination |
| `AssertionConsumerServiceURL` | ACS URL registered at IdP | IdP posts to wrong URL or rejects |
| `<saml:Issuer>` (EntityID) | SP EntityID in IdP registration | IdP cannot identify the SP |
| `IssueInstant` | Within 5 min of current time | Request rejected as expired |
| `NameIDPolicy Format` | email / persistent / transient | Wrong NameID format returned to SP |
| `ProtocolBinding` | HTTP-POST or HTTP-Redirect | Response delivered via wrong binding |
| `RequestedAuthnContext` | MFA level IdP can satisfy | `NoAuthnContext` status in response |

---

## 6 · SAML — Decoded SAMLResponse Fields

| XML Field | Must equal / contain | Failure if wrong |
|---|---|---|
| `StatusCode` | `urn:…:Success` | Any other value = auth failed at IdP |
| `<saml:Issuer>` | IdP EntityID registered at SP | SP rejects: unknown or untrusted issuer |
| `Destination` | This SP's ACS URL | SP rejects: invalid destination |
| `AudienceRestriction` | SP's EntityID | SP returns: invalid audience error |
| `NotBefore / NotOnOrAfter` | Current time within window | Clock skew > 5 min → assertion expired |
| `<saml:NameID>` | Format SP expects (email, etc.) | SP can't map user to local account |
| `<saml:Attribute>` list | All required claims present | SP missing email / groups / roles |
| `<ds:Signature>` | Signed with current IdP cert | Expired or rotated cert → invalid signature |

---

## 7 · SAML — StatusCode Values

| StatusCode (suffix) | Meaning | First action |
|---|---|---|
| `Success` | Auth succeeded at IdP | If SP still fails → check attribute mapping |
| `Requester` | SP request is invalid | Decode SAMLRequest; check EntityID & ACS URL |
| `Responder` | IdP-side processing error | Check IdP server logs |
| `AuthnFailed` | Credential / MFA failure | Check IdP auth logs for the user |
| `NoAuthnContext` | Can't meet MFA requirement | Enable MFA for user or relax AuthnContext |
| `RequestDenied` | IdP policy blocks the request | Confirm user is assigned to the SP app |
| `InvalidAttrNameOrValue` | Attribute value format mismatch | Fix attribute mapping rules in IdP |
| `VersionMismatch` | SAML version conflict (1.1 vs 2.0) | Check SP SAML library configuration |

---

## 8 · Quick Find — Where Is That Parameter?

| Looking for | Protocol | Where to find it |
|---|---|---|
| Client / App ID | OIDC | Auth URL → `client_id=` |
| Client / App ID | SAML | SAMLRequest XML → `<saml:Issuer>` |
| Redirect / ACS URL | OIDC | Auth URL → `redirect_uri=` |
| Redirect / ACS URL | SAML | SAMLRequest XML → `AssertionConsumerServiceURL` |
| Flow / grant type | OIDC | Auth URL → `response_type=` |
| User identifier | OIDC | id_token JWT payload → `sub` / `email` claim |
| User identifier | SAML | SAMLResponse XML → `<saml:NameID>` |
| Token expiry | OIDC | JWT payload → `exp` (Unix timestamp) |
| Assertion expiry | SAML | SAMLResponse `<Conditions>` → `NotOnOrAfter` |
| Return URL after auth | SAML | URL / POST body → `RelayState=` |

---

## 9 · Decode Commands & Tools

| Task | Command / URL |
|---|---|
| Decode SAMLRequest (inflate) | `echo "VALUE" \| base64 -d \| python3 -c "import zlib,sys; sys.stdout.buffer.write(zlib.decompress(sys.stdin.buffer.read(),-15))"` |
| Decode SAMLResponse | `echo "VALUE" \| base64 -d`   or paste at **samltool.io** |
| Decode OIDC id_token / JWT | Paste at **jwt.io** — view header · payload · verify signature |
| URL-decode redirect_uri / scope | `python3 -c "import urllib.parse; print(urllib.parse.unquote('VALUE'))"` |
| OIDC discovery document | `https://<idp>/.well-known/openid-configuration` |
| SAML metadata (SP & IdP) | `https://<sp>/saml/metadata`   ·   `https://<idp>/saml/metadata` |
| Capture SAMLResponse in browser | DevTools → Network → POST to `/acs` → Payload tab → `SAMLResponse` field |

---

*OIDC: RFC 6749 / OpenID Connect Core 1.0   ·   SAML: OASIS SAML 2.0   ·   Consult IdP docs for vendor-specific variations*
