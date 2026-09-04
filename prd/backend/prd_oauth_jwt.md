# Frappe JWT Auth

JWT-based authentication for Frappe/ERPNext. Works identically on **mobile**,
**web**, and **API** clients — install once on the server, use everywhere.

## Why this exists

- Vanilla Frappe login (`/api/method/login`) uses session cookies — fine for
  browser-based web apps, terrible for native mobile and API clients.
- The Frappe Mobile SDK ships with `mobile_auth` which requires per-doctype
  mobile configuration, doctype sync, translation sync — overkill when all
  you need is authentication.
- This app does **one thing**: issue JWT access + refresh tokens from
  username/password credentials. No UI configuration. No mobile form setup.

## Quick start

### Server (one-time)

```bash
bench get-app git@github.com:autopilotstore/frappe_jwt_auth.git
bench --site your-site install-app frappe_jwt_auth
```

That's it. No UI configuration needed — the `JWT Refresh Token` DocType is
created automatically during install.

### Client (any platform)

**Login:**

```http
POST /api/method/frappe_jwt_auth.api.login
Content-Type: application/json

{"usr": "user@example.com", "pwd": "password"}
```

Response:
```json
{
  "access_token": "eyJhbGciOi...",
  "refresh_token": "eyJhbGciOi...",
  "token_type": "Bearer",
  "expires_in": 900,
  "user": "user@example.com",
  "full_name": "John Doe"
}
```

**All subsequent requests** — pass the access token in the Authorization header:

```http
GET /api/method/frappe.client.get_list?doctype=Customer&fields=["name","customer_name"]
Authorization: Bearer eyJhbGciOi...
```

The Bearer token middleware (`auth_hooks`) validates the JWT and sets
`frappe.session.user` — so **all standard Frappe API endpoints work**
without needing a separate API key or session cookie.

**Refresh token:**

When the access token expires (default: 15 minutes), call:

```http
POST /api/method/frappe_jwt_auth.api.refresh
Content-Type: application/json

{"refresh_token": "eyJhbGciOi..."}
```

Response includes a **new** access token AND a **new** refresh token. The
old refresh token is revoked (rotation policy — a stolen token can only
be used once).

**Logout:**

```http
POST /api/method/frappe_jwt_auth.api.logout
Authorization: Bearer eyJhbGciOi...
```

Revokes all refresh tokens for the current user.

## API reference

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `frappe_jwt_auth.api.login` | Guest | Username/password → JWT tokens |
| POST | `frappe_jwt_auth.api.refresh` | Guest | Refresh token → new token pair |
| POST | `frappe_jwt_auth.api.logout` | Bearer | Revoke refresh tokens |
| GET | `frappe_jwt_auth.api.me` | Bearer | Current user info from JWT |

## Token lifecycle

- **Access token**: 15 minutes (configurable via `ACCESS_TOKEN_EXPIRY_MINUTES`)
- **Refresh token**: 7 days (configurable via `REFRESH_TOKEN_EXPIRY_DAYS`)
- **Rotation**: Each refresh revokes the old refresh token and issues a new
  pair. If a revoked refresh token is used, all tokens for that user are
  revoked (theft detection).

## Configuration

All config lives in `frappe_jwt_auth/auth.py` at the top of the file:

```python
ACCESS_TOKEN_EXPIRY_MINUTES = 15
REFRESH_TOKEN_EXPIRY_DAYS = 7
```

No UI, no site config, no doctype settings — just constants in the source.

## Compared to mobile_auth

| | frappe_jwt_auth | mobile_auth (Frappe Mobile SDK) |
|---|---|---|
| Auth method | JWT (access + refresh) | JWT (access + refresh) |
| Server config | Zero — `install-app` only | Mobile configuration doctype required |
| Mobile form setup | None | Per-doctype activation in UI |
| Doctype sync | No | Yes (meta, translation, data sync) |
| Works on web | Yes | Yes (but mobile-branded) |
| Works on API clients | Yes | Yes |
| Refresh token rotation | Yes | Yes |
| Bearer middleware (all endpoints) | Yes | Yes |

## License

MIT
