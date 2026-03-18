# Authentication — SHELL AUTHENTICATION SERVICE

## Purpose

The `shell-auth-svc` is a Go HTTP service that acts as a **Backend for Frontend (BFF)** authentication proxy between the SHELL React SPA and Keycloak (the NCMT Identity Provider). It implements the **OIDC Authorization Code Flow with PKCE** entirely on the server side, ensuring that tokens never reach the browser's JavaScript context.

The service is responsible for:
- Redirecting unauthenticated users to Keycloak
- Exchanging the authorization code for tokens after login
- Verifying ID and access token signatures via OIDC Discovery (JWKS)
- Storing tokens in a server-side session (not in cookies or browser storage)
- Exposing safe session metadata to the frontend via `/auth/session`
- Terminating sessions and triggering Keycloak global logout via `/auth/logout`
- Securing all inbound and outbound traffic with TLS (mTLS optional)

---

## Authentication Architecture

```
                          Kubernetes cluster
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  ┌──────────────┐  HTTPS (mTLS optional)  ┌──────────────────────┐  │
│  │  SHELL React  │ ──────────────────────▶ │  shell-auth-svc   │  │
│  │  Frontend    │ ◀── session cookie ───── │  :8081               │  │
│  └──────────────┘                         └─────────┬────────────┘  │
│                                                     │ HTTPS          │
│                                                     ▼ (CA verified)  │
│                                          ┌──────────────────────┐   │
│                                          │  Keycloak / OIDC     │   │
│                                          │  (NCMT IAM Provider) │   │
│                                          └──────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

The frontend **never** receives or stores tokens. The backend holds all tokens in a server-side session store, issuing only a `sessionId` cookie to the browser.

---

## Why Backend Session-Based OIDC vs SPA-Only OIDC

| Feature                                | SPA-only OIDC (insecure) | Backend Session-based (current) |
|----------------------------------------|--------------------------|----------------------------------|
| Keycloak login redirect                | ✅                        | ✅                               |
| Tokens stored in browser JS            | ✅                        | ❌                               |
| Tokens stored server-side              | ❌                        | ✅                               |
| Uses secure HttpOnly cookies           | ❌                        | ✅                               |
| APIs called via backend                | ❌                        | ✅                               |
| Resistant to XSS                       | ❌                        | ✅                               |

Implementing the OIDC flow directly in the frontend (even in its server-side runtime) is discouraged because:

- The Keycloak `client_secret` would live inside the frontend deployment, increasing its attack surface.
- Frontend servers are not hardened for sensitive credential handling.
- Mixing authentication concerns into the UI layer violates separation of concerns and complicates scaling independently.

A dedicated Go backend provides a clear security boundary, full control over TLS and session lifecycle, and keeps the frontend focused solely on rendering.

---

## Authentication Flow

```
User → React Frontend → GET /auth/login → [Go Backend]
                                               │
                          Redirect to Keycloak ▼
                              [Keycloak Login UI]
                                               │
                   Keycloak → Redirect to [Go Backend /auth/callback?code=...]
                                               │
              Backend exchanges code for tokens (server-side POST to Keycloak)
              Verifies ID token + access token signatures via JWKS
              Stores tokens in server-side session
              Sets HttpOnly sessionId cookie
                                               │
       Frontend reads /auth/session ──────────▶│──▶ Returns username, role, email
       Frontend reads /auth/status  ──────────▶│──▶ Returns 200 OK or 401
```

### Step-by-step

1. **User navigates to SHELL** — the frontend checks `/auth/status`. A `401 Unauthorized` triggers a redirect to `/auth/login`.

2. **`GET /auth/login`** — the backend checks for an existing server-side session. If none exists, it constructs an OAuth2 authorization URL (including `state`, `code_challenge`, `code_challenge_method=S256`) and redirects the browser to Keycloak.

3. **Keycloak authentication** — the user enters credentials on the Keycloak login page. After successful authentication, Keycloak redirects to the backend's registered `redirect_uri` with an authorization `code` and `state` in the query parameters.

4. **`GET /auth/callback`** — the backend:
   - Validates the `code` parameter is present
   - Makes a server-side POST to the Keycloak token endpoint (`grant_type=authorization_code`) exchanging the code for `access_token`, `id_token`, and `refresh_token`
   - Verifies the ID token signature and standard claims (expiry, issuer, audience) using OIDC Discovery JWKS
   - Verifies the access token
   - Extracts claims: `preferred_username`, `email`, `sub`, and realm roles
   - Creates a server-side session and stores all tokens there
   - Sets an `HttpOnly` `sessionId` cookie (1-hour lifetime, `Secure` flag in production)
   - Redirects the browser back to the frontend origin

5. **`GET /auth/session`** — returns a JSON payload with safe user information (`username`, `role`, `email`). No tokens are exposed.

6. **`GET /auth/status`** — returns `200 OK` if a valid, non-expired session exists; `401 Unauthorized` otherwise.

7. **`GET /auth/logout`** — deletes the server-side session, clears the `sessionId` cookie, and redirects to the Keycloak `end_session` endpoint (passing `id_token_hint` and `post_logout_redirect_uri`) to terminate the Keycloak session globally.

---

## API Endpoints

| Endpoint            | Method | Description                                                              |
|---------------------|--------|--------------------------------------------------------------------------|
| `/auth/login`       | GET    | Checks for active session; if none, redirects browser to Keycloak login  |
| `/auth/callback`    | GET    | Receives Keycloak redirect; exchanges code for tokens; sets session cookie |
| `/auth/session`     | GET    | Returns safe user info (`username`, `role`, `email`) from session        |
| `/auth/status`      | GET    | Returns `200` if session is active and not expired, else `401`           |
| `/auth/logout`      | GET    | Clears session and redirects to Keycloak global logout                   |

> Tokens are **never** returned to the frontend. The `sessionId` cookie is `HttpOnly`, `Secure` (in production), and `SameSite=Lax`.

---

## Running Locally (Without TLS)

By default `TLS_ENABLED=true` (production-safe). For local development, explicitly disable TLS:

```bash
TLS_ENABLED=false \
SESSION_COOKIE_SECURE=false \
TLS_INSECURE_SKIP_VERIFY=true \
go run ./src/cmd/main.go
```

| Variable                  | Local value | Why                                                                 |
|---------------------------|-------------|---------------------------------------------------------------------|
| `TLS_ENABLED`             | `false`     | Skips cert loading, starts on plain HTTP                           |
| `SESSION_COOKIE_SECURE`   | `false`     | Allows cookies over non-HTTPS origin                               |
| `TLS_INSECURE_SKIP_VERIFY`| `true`      | Skips Keycloak cert verification when Keycloak is also HTTP         |

> **Never** set `TLS_INSECURE_SKIP_VERIFY=true` in production.

---

## Environment Variables Reference

### OIDC Configuration

| Variable              | Default                                                                | Description                                          |
|-----------------------|------------------------------------------------------------------------|------------------------------------------------------|
| `OIDC_CLIENT_ID`      | `shell-auth-svc`                                                         | Client ID registered in Keycloak                     |
| `OIDC_CLIENT_SECRET`  | *(dev default)*                                                        | Client secret — inject via Kubernetes Secret         |
| `OIDC_ISSUER`         | `http://localhost:8080/access/realms/ncmt`                             | Keycloak realm issuer URL                            |
| `OIDC_REDIRECT_URI`   | `http://localhost:8081/auth/callback`                                  | Backend callback URI registered in Keycloak          |
| `OIDC_TOKEN_ENDPOINT` | `http://localhost:8080/access/realms/ncmt/protocol/openid-connect/token` | Token exchange endpoint                            |
| `OIDC_AUTH_URL`       | `http://localhost:8080/access/realms/ncmt/protocol/openid-connect/auth`  | Authorization endpoint                             |
| `OIDC_END_SESSION`    | `http://localhost:8080/access/realms/ncmt/protocol/openid-connect/logout`| Keycloak end-session endpoint for global logout    |

### CORS Configuration

| Variable            | Default                          | Description                            |
|---------------------|----------------------------------|----------------------------------------|
| `ALLOW_ORIGIN`      | `http://localhost:5175`          | Allowed frontend origin                |
| `ALLOW_HEADERS`     | `Content-Type, Authorization`    | Allowed request headers                |
| `ALLOW_METHODS`     | `GET, POST, OPTIONS`             | Allowed HTTP methods                   |
| `ALLOW_CREDENTIALS` | `true`                           | Allow cookies with cross-origin requests |

### TLS Configuration

| Variable                   | Default          | Description                                                                 |
|----------------------------|------------------|-----------------------------------------------------------------------------|
| `TLS_ENABLED`              | `true`           | Master TLS switch. Set `false` for local dev only                           |
| `ENABLE_MTLS`              | `true`           | Require client certificates. Set `false` for browser-facing deployments     |
| `SERVER_CERTS_PATH`        | `/service_certs` | Directory containing `ca.crt`, `tls.crt`, `tls.key` for inbound HTTPS      |
| `KEYCLOAK_CERTS_PATH`      | `/keycloak_certs`| Directory containing `ca.crt` for verifying the Keycloak TLS certificate   |
| `CA_CRT_FILE_NAME`         | `ca.crt`         | CA certificate file name                                                    |
| `TLS_CRT_FILE_NAME`        | `tls.crt`        | Server certificate file name                                                |
| `TLS_KEY_FILE_NAME`        | `tls.key`        | Server private key file name                                                |
| `TLS_INSECURE_SKIP_VERIFY` | `false`          | Skip Keycloak TLS verification — **development only**                       |
| `SESSION_COOKIE_SECURE`    | `true`           | Mark session cookie as `Secure` — set `false` for plain HTTP dev            |

---

## TLS Architecture

```
TLS_ENABLED (env, default: true)
├── ServiceTLSEnabled = true    → server listens on HTTPS (:8081)
├── ServiceMTLSEnabled          → read from ENABLE_MTLS (default: true)
│     true  → ClientAuth = RequireAndVerifyClientCert  (service-to-service)
│     false → server-side TLS only                     (browser-compatible)
└── KeycloakTLSEnabled = true   → Keycloak HTTP client verifies CA from /keycloak_certs/ca.crt
```

Certificate volumes are mounted from Kubernetes Secrets:

| Path              | Secret               | Contains                           | Used by                 |
|-------------------|----------------------|------------------------------------|-------------------------|
| `/service_certs`  | `shell-auth-svc-tls`   | `ca.crt`, `tls.crt`, `tls.key`     | Inbound HTTPS server    |
| `/keycloak_certs` | `keycloak-tls`       | `ca.crt`                           | Outbound Keycloak client|

Setting `TLS_ENABLED=false` disables all three derived flags — plain HTTP, no cert files required.

---

## Keycloak Client Setup

Register a **confidential** backend client in the `ncmt` realm:

| Setting                     | Value                                       |
|-----------------------------|---------------------------------------------|
| **Client ID**               | `shell-auth-svc`                              |
| **Access Type**             | Confidential                                |
| **Valid Redirect URIs**     | `https://<domain>/auth/callback`            |
| **Post Logout Redirect URIs** | `https://<frontend-domain>/*`             |
| **Standard Flow**           | ✅ Enabled                                  |
| **Direct Access Grants**    | ✅ Enabled                                  |
| **Implicit Flow**           | ❌ Disabled                                 |
| **Service Accounts**        | ❌ Disabled                                 |
| **Client Authentication**   | ✅ Enabled (provides `client_secret`)       |

---

## Session Behavior

| Behavior                                              | Status                    |
|-------------------------------------------------------|---------------------------|
| Session stored via secure HttpOnly cookie             | ✅ Yes                    |
| Frontend never sees tokens                            | ✅ Yes                    |
| `/auth/session` exposes only safe user info           | ✅ Yes                    |
| `/auth/status` returns 401 when session is expired    | ✅ Yes                    |
| Keycloak Admin UI shows active sessions               | ❌ No (expected — the app does not use `keycloak-js` or Keycloak's browser session) |

---

## Project Structure (relevant packages)

```
src/
├── cmd/
│   └── main.go                        # Entry point; wires routes, starts TLS-aware server
└── internal/
    ├── auth/
    │   └── endpoints.go               # Registers all /auth/* HTTP handlers
    ├── config/
    │   └── definitions.go             # All environment variable definitions and TLS flag cascade
    ├── controller/
    │   ├── login/handleLogin.go       # GET /auth/login
    │   ├── callback/handleCallBack.go # GET /auth/callback — token exchange & session creation
    │   ├── session/handleSession.go   # GET /auth/session, GET /auth/status
    │   └── logout/handleLogout.go     # GET /auth/logout — session teardown + Keycloak redirect
    ├── services/
    │   └── oidcProvider.go            # OAuth2 config, shared HTTP client, token verification
    ├── session/                       # Server-side session store and cookie utilities
    └── common/httpx/
        ├── tls.go                     # TLS config helpers (cert loading, CA pools)
        └── server/server.go           # TLS-aware HTTP server with graceful shutdown
```

---

## Tech Stack

| Component              | Technology                                |
|------------------------|-------------------------------------------|
| Web framework          | `net/http` + `chi`                        |
| OIDC client            | `coreos/go-oidc` + `golang.org/x/oauth2`  |
| Session storage        | Server-side in-memory store               |
| Session cookie         | `HttpOnly`, `Secure`, `SameSite=Lax`      |
| Identity Provider      | Keycloak (`ncmt` realm)                   |
| TLS                    | `crypto/tls` with optional mTLS           |
| Module management      | `go mod`                                  |
