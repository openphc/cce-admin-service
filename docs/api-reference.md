# API Reference

Base path as seen by callers: `/admin/v1` (via `gateway-service`'s existing `/admin/**`
route; see [architecture-overview.md §3](architecture-overview.md#3-system-context)).
Every request requires `Authorization: Bearer <keycloak-access-token>`. Which role may
call which of these endpoints is itself configured through the `api_permissions`
resource below — see [security-model.md §4](security-model.md#4-one-identifier-role--realm-role--permission_name).

For entity/column meaning see [data-dictionary.md](data-dictionary.md). For the
order-of-operations behind the non-trivial endpoints (user creation, permission sync),
see [flow-diagrams.md](flow-diagrams.md).

## Conventions

**Envelope** — success:

```json
{ "data": { }, "pagination": { "page": 1, "pageSize": 20, "total": 57 } }
```

`pagination` is present only on list endpoints. Error:

```json
{ "error": { "code": "ROLE_NOT_FOUND", "message": "Role a1b2... not found" } }
```

### Pagination

List endpoints accept `?page=1&pageSize=20` (`pageSize` capped at 100, default 20) and
return the `pagination` block above.

### Common error codes

| HTTP | code | Meaning |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Request body/query failed validation |
| 401 | `UNAUTHENTICATED` | Missing/invalid/expired token |
| 403 | `FORBIDDEN` | Valid token, but caller's role lacks this permission, or local account is not `ACTIVE` |
| 404 | `USER_NOT_FOUND` / `ROLE_NOT_FOUND` / `MENU_NOT_FOUND` | |
| 409 | `DUPLICATE_ROLE_NAME` / `DUPLICATE_MENU_KEY` / `EMAIL_IN_USE` | Unique constraint conflict |
| 409 | `ROLE_IN_USE` | Attempted to delete a role still referenced by `user_roles` |
| 409 | `SYSTEM_ROLE_IMMUTABLE` | Attempted to modify/delete a `roles.is_system = true` row |
| 502 | `KEYCLOAK_SYNC_FAILED` | Keycloak Admin API call failed; local DB state and Keycloak are now out of sync — see retry note below |

## Current-user context

### `GET /me`

Returns the caller's own profile and role names, resolved from the token's `sub`.

```json
{ "data": { "id": "...", "username": "j.smith", "roles": ["FACILITY_ADMIN"], "facilityScope": "RW-KGL-001" } }
```

### `GET /me/menus`

**The endpoint `cce-insights-ui` calls after login to render a role-aware sidebar.**
Returns the menu tree (only `is_active = true` rows) that at least one of the caller's
roles is mapped to, `NAV` and `DETAIL` entries both included so the frontend can permit
direct navigation to a detail route without listing it.

```json
{
  "data": [
    { "key": "dashboard", "label": "Dashboard", "path": "/", "icon": "ChartBarIcon", "children": [] },
    { "key": "compliance", "label": "Compliance", "path": "/compliance", "icon": "ClipboardDocumentCheckIcon",
      "children": [{ "key": "compliance.protocol_detail", "label": "Protocol Detail", "path": "/compliance/protocols/:id", "menuType": "DETAIL" }] }
  ]
}
```

## Users

| Verb | Path | Purpose |
|---|---|---|
| GET | `/users` | List, filterable by `?status=`, `?roleId=`, `?facilityScope=`, `?q=` (name/email search) |
| POST | `/users` | Create a user — creates the Keycloak account, the local row, and (optionally) initial role assignment in one flow. See [flow-diagrams.md §2](flow-diagrams.md#2-admin-onboards-a-new-user). |
| GET | `/users/:id` | Fetch one, including current role names |
| PUT | `/users/:id` | Update profile fields (`firstName`, `lastName`, `email`, `facilityScope`) |
| PATCH | `/users/:id/status` | Body `{ "status": "ACTIVE" \| "INACTIVE" \| "LOCKED" }` — also enables/disables the Keycloak account |
| PUT | `/users/:id/roles` | Body `{ "roleIds": ["..."] }` — **replaces** the full set of role assignments; diffed against current state to compute which Keycloak realm roles to assign/remove |
| POST | `/users/:id/reset-password` | Triggers a Keycloak "update password" required action / reset email |

`POST /users` request:

```json
{ "username": "j.smith", "email": "j.smith@example.org", "firstName": "Jane", "lastName": "Smith", "facilityScope": "RW-KGL-001", "roleIds": ["<role-uuid>"] }
```

Response mirrors `GET /users/:id`. If the Keycloak account is created but the local
insert fails (or vice versa), the endpoint returns `502 KEYCLOAK_SYNC_FAILED` with the
partial state recorded in `audit_log` for manual reconciliation — this is not silently
retried automatically, since retrying a partially-applied user creation risks duplicate
Keycloak accounts.

## Roles

| Verb | Path | Purpose |
|---|---|---|
| GET | `/roles` | List all roles |
| POST | `/roles` | Create a role. Body: `{ "name": "FACILITY_ADMIN", "displayName": "Facility Admin", "description": "..." }` |
| GET | `/roles/:id` | Fetch one |
| PUT | `/roles/:id` | Update `displayName`/`description`. `name` is immutable after creation (it's already propagated to Keycloak/`api_permissions`) |
| DELETE | `/roles/:id` | Blocked with `409 ROLE_IN_USE` if any `user_roles` row references it, or `409 SYSTEM_ROLE_IMMUTABLE` if `is_system` |
| GET | `/roles/:id/menus` | List menu keys currently mapped to the role |
| PUT | `/roles/:id/menus` | Body `{ "menuIds": ["..."] }` — replaces the full menu set |
| GET | `/roles/:id/permissions` | List `api_permissions` rows for the role |
| PUT | `/roles/:id/permissions` | Body `{ "permissions": [{ "httpMethod": "GET", "uriPattern": "/v1/insights/compliance/**", "description": "...", "resourceCategory": "Compliance API" }] }` — replaces the full set |
| POST | `/roles/:id/sync` | Calls `gateway-service`'s `POST /api/permissions/sync-to-keycloak` so the realm role reflects the latest `api_permissions` rows. Also called automatically after `PUT .../permissions`; exposed standalone for manual re-sync after a gateway-side incident. |

## Menus

Menu records are largely seeded (see [data-dictionary.md §3](data-dictionary.md#3-menus))
and change only when `cce-insights-ui` gains new screens.

| Verb | Path | Purpose |
|---|---|---|
| GET | `/menus` | List the full catalog as a tree |
| POST | `/menus` | Create a menu entry. Body: `{ "key": "...", "label": "...", "path": "...", "icon": "...", "parentId": "...", "menuType": "NAV" }` |
| PUT | `/menus/:id` | Update label/path/icon/parent/sortOrder/isActive |
| DELETE | `/menus/:id` | Also removes matching `role_menus` rows |

## Health

`GET /health` — unauthenticated, used by the container orchestrator. Returns
`{ "status": "UP" }` and, best-effort, DB connectivity.
