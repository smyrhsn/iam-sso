## 1 · Tokens — id_token vs access_token
 
| Token | Answers | Who uses it | Analogy |
|---|---|---|---|
| `id_token` | *Who are you?* — contains name, email, sub, auth time | The client app | ID Card |
| `access_token` | *What can you do?* — sent in `Authorization: Bearer` header to APIs | The backend API | Access Badge |
| `refresh_token` | Gets a new access_token without re-authenticating | The client app (sent to `/token`) | Long-term pass |
 
> **Key rule:** `id_token` stays at the app — never send it to APIs. `access_token` is what you send to APIs.
 
**Troubleshoot by symptom:**
 
| Problem | Check | Where |
|---|---|---|
| App shows wrong name / email | `id_token` | Decode at jwt.io — check `name`, `email`, `sub` claims |
| Logged in but can't access data | `access_token` | Check `scope`, `exp` (expiry), `aud` (audience) |
| User keeps getting logged out | `refresh_token` | Check if issued — requires `offline_access` scope |
| SSO works but wrong role shown | `id_token` or `access_token` | Check where roles/groups are mapped in IdP |
 
---
