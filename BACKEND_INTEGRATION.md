# Backend Integration (Optilium Nest API)

This frontend talks to the **Optilium Nest** backend over HTTP/JSON with a single base URL and a standard response envelope.

## 1. Configure the API base URL

Use **one** variable everywhere:

- **Local (recommended):** create `.env.development` (or `.env.local`) with **`/api`** so requests go through the Vite dev proxy (see `vite.config.ts` → `/api` forwarded to `http://localhost:3000`, avoiding browser CORS issues):

```bash
VITE_API_BASE_URL=/api
```

If you insist on talking to the backend directly from the browser (cross-origin), set `VITE_API_BASE_URL=http://localhost:3000` and configure the backend to allow origin `http://localhost:8080`.

- **Production (e.g. Vercel):** set `VITE_API_BASE_URL=https://<your-backend-url>` in the project env settings.

Restart the Vite dev server after changing env vars.

## 2. Response envelope

Successful API responses are expected to look like:

```json
{ "success": true, "data": { ... }, "error": null, "meta": { ... } }
```

Errors use HTTP error status and:

```json
{ "success": false, "data": null, "error": { "message": "..." }, "meta": { ... } }
```

The client in `src/api/client.ts` **unwraps** `data` and throws `OptiliumApiError` on failure.

## 3. Auth

- **Login:** `POST {VITE_API_BASE_URL}/auth/login` with `{ "email", "password" }`.
- **Token:** JWT is stored in `localStorage` under **`accessToken`**.
- **Requests:** `Authorization: Bearer <accessToken>` is added automatically (except when `skipAuth: true`).
- **Tenant:** Do **not** send `companyId` in body/query; tenant is taken from the JWT on the backend.
- **Logout:** `POST /auth/logout` (best-effort; local token is always cleared).

## 4. Where things live

| File | Purpose |
| --- | --- |
| `src/api/client.ts` | Base URL, Bearer token, envelope unwrap, `OptiliumApiError` |
| `src/api/auth.ts` | `login(email, password)` → `POST /auth/login` |
| `src/api/commandCenter.ts` | Command-center endpoints (e.g. ELD overview, drivers) |
| `src/lib/auth.ts` | App-level login/logout + `auth_user` / `isLoggedIn` for UI |
| `src/context/AuthContext.tsx` | `useAuth()` |
| `src/pages/OptiliumSmokeTestPage.tsx` | Smoke test: login + sample GETs |
| `src/types/api.ts` | Shared DTOs (`LoginRequest`, `AuthUser`, …) |

Legacy `src/lib/api-client.ts` (older `auth_token` helper) is **not** used by Optilium auth; prefer `src/api/client.ts`.

## 5. Smoke test

Open **`/optilium-smoke`** (also linked from the sidebar as “API smoke” when logged into the dashboard). It logs in and prints JSON for:

- `GET /command-center/eld-overview`
- `GET /command-center/drivers?page=1&pageSize=20`

## 6. CORS

**Development:** With `VITE_API_BASE_URL=/api`, the browser calls **`http://localhost:8080/api/...`** (same-origin); Vite proxies to the Nest server, so strict CORS for `localhost:8080` on the backend is optional.

**Direct cross-origin requests** (e.g. `VITE_API_BASE_URL=http://localhost:3000`): the Vite app runs at **`http://localhost:8080`**. The backend must allow that origin and the `Authorization` header:

```
Access-Control-Allow-Origin: http://localhost:8080
Access-Control-Allow-Headers: authorization, content-type
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
```

In production, set `Access-Control-Allow-Origin` to your deployed frontend origin (or use dynamic allowlist).
