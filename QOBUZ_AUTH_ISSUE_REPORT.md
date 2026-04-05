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

## Extended Investigation (April 5, 2026)

### Tested and Failed
- GET vs POST for /user/login: Both return 401
- Adding `device_manufacturer_id` (79d511e3-9a3f-4a78-baaa-89e2d78fed83): Still 401
- Different password formats (plain, MD5, SHA256): All return 401
- Fresh credentials from web player bundle.js: Still 401

### Key Discovery: Request Signing
The web player bundle.js contains evidence of new signature mechanism:
- Uses `_raw_sig`, `sha256`, and `_requestIdentifier` for request signing
- The `/user/login` endpoint appears to require this signature
- Simple username/password POST is no longer sufficient

### Possible Solutions
1. **Reverse engineer the new signature algorithm** from web player's JavaScript
2. **Extract auth token from browser** using DevTools/mitmproxy and use `login_with_token`
3. **Wait for upstream projects** to release a fix (qobuz-api-rust, streamrip, etc.)

## WORKAROUND: Token-Based Authentication

QobuzDownloaderX-MOD uses an alternate login method that bypasses the broken email/password auth.

### How to Get Your Token

1. **Open Qobuz Web Player**: Go to https://play.qobuz.com/ and log in with your credentials
2. **Open DevTools**: Press F12 (or Cmd+Option+I on Mac)
3. **Go to Application tab**: Click on "Application" in DevTools
4. **Navigate to Local Storage**: Expand "Storage" → "Local Storage" → "https://play.qobuz.com"
5. **Find `localuser`**: Click on it to see the stored data
6. **Copy these values**:
   - `id` → This is your `user_id`
   - `token` → This is your `user_auth_token`

### How to Configure qobuz-dl

Edit your config file (`~/.config/qobuz-dl/config.ini`) and add:

```ini
user_id = YOUR_USER_ID_HERE
user_auth_token = YOUR_TOKEN_HERE
```

Leave `email` and `password` empty or as-is.

Then run qobuz-dl as normal.

### Related Tools Status
- qobuz-api-rust: Author only tested token-based auth (email/password untested)
- QobuzDownloaderX-MOD: May have working solution (C# project, check their approach)
