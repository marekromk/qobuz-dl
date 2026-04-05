# Qobuz Authentication Issue Report

Date: April 5, 2026
Status: OPEN / UNRESOLVED

## Problem

Qobuz changed their API authentication around April 3, 2026. The `/api.json/0.2/user/login`
endpoint now returns `401: "User authentication is required"` for all login attempts, even with
valid credentials.

Affected tools:
- qobuz-dl (https://github.com/vitiko98/qobuz-dl/issues/329)
- streamrip (https://github.com/nathom/streamrip/issues/954)
- Potentially other Qobuz API-based tools

## Error Message

```
qobuz_dl.exceptions.AuthenticationError: Invalid credentials.
Reset your credentials with 'qobuz-dl -r'
```

API Response:
```json
{"code":401,"status":"error","message":"User authentication is required. (Root=1-xxx)"}
```

## Investigation Notes

- Credentials (email/password hash) are correct - they work on Qobuz official apps
- App ID and secrets are fetched fresh from Qobuz web app bundle.js
- Different signature formats were tested but did not resolve the issue
- The authentication mechanism appears to have fundamentally changed on Qobuz's side

## Relevant GitHub Issues

1. qobuz-dl #329: "Credentials reported invalid although they work fine on regular player"
   - Opened: April 3, 2026
   - URL: https://github.com/vitiko98/qobuz-dl/issues/329

2. streamrip #954: "Qobuz login stopped working"
   - Opened: April 3, 2026
   - URL: https://github.com/nathom/streamrip/issues/954

## Workarounds

1. Wait for upstream projects to release a fix
2. Use Qobuz's official apps/website for now
3. Monitor the GitHub issues for updates

## Fix Required

The qobuz-dl project needs to update its authentication mechanism. Possible approaches:
- Reverse-engineer the new authentication from Qobuz web app's JavaScript bundle
- Use a different authentication endpoint if available
- Implement OAuth or token-based authentication

## Investigation Results (April 5, 2026)

### HTTP Method Test
Tested changing `/user/login` from GET to POST (matching qobuz-api-rust library):
- **Result**: Both GET and POST return 401 "User authentication is required"
- **Conclusion**: HTTP method is not the issue

### Signature Test
Tested adding request signature to login (like track/getFileUrl endpoint):
- **Result**: Still returns 401
- **Conclusion**: Simple signing doesn't fix the issue

### Related Projects
- qobuz-api-rust (Rust library): Author reports they couldn't test email/password login - only token-based auth works for them
- streamrip: Same 401 error confirmed

### Key Finding
The Rust library's author explicitly states:
> "I haven't been able to test the login with email/password or username/password, as I only have access to user ID and user authentication token."

This suggests **token-based authentication may be the only working method** now. The library has `login_with_token(user_id, user_auth_token)` which may be the path forward.

### Next Steps
1. Extract a valid `user_auth_token` from browser traffic or Qobuz web app
2. Implement token-based login in qobuz-dl
3. Store and refresh user tokens (they may expire)
