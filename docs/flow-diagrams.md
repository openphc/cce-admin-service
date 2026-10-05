# Flow Diagrams

Sequence-level detail for the scenarios referenced elsewhere. Entity/column names per
[data-dictionary.md](data-dictionary.md); endpoint names per
[api-reference.md](api-reference.md); the reasoning behind the split of
responsibilities shown here lives in [security-model.md](security-model.md).

## 1. Login and role-aware menu resolution

The scenario that motivates this whole service: a user logs in and sees only their
role's menus.

```mermaid
sequenceDiagram
    actor User
    participant UI as cce-insights-ui
    participant KC as Keycloak
    participant GW as gateway-service
    participant ADM as cce-admin-service
    participant DB as admin_service DB

    User->>UI: opens app
    UI->>KC: PKCE login redirect
    KC-->>UI: access token (realm_access.roles: [FACILITY_ADMIN])
    UI->>GW: GET /admin/v1/me/menus (Bearer token)
    GW->>GW: validate JWT, check api_permissions for this route
    GW->>ADM: proxy request (Authorization header intact)
    ADM->>ADM: validate JWT again, resolve sub -> users.keycloak_id
    ADM->>DB: SELECT role names for user (user_roles JOIN roles)
    ADM->>DB: SELECT menus JOIN role_menus WHERE role_id IN (...)
    DB-->>ADM: menu rows
    ADM-->>GW: menu tree
    GW-->>UI: menu tree
    UI->>UI: render sidebar from response, not the old static list
```

## 2. Admin onboards a new user

```mermaid
sequenceDiagram
    actor Admin
    participant AdminUI as cce-admin-ui
    participant GW as gateway-service
    participant ADM as cce-admin-service
    participant KC as Keycloak Admin API
    participant DB as admin_service DB

    Admin->>AdminUI: fill in user + role(s)
    AdminUI->>GW: POST /admin/v1/users
    GW->>ADM: proxy
    ADM->>KC: create user (username, email, temp password / required actions)
    KC-->>ADM: keycloak_id
    ADM->>KC: assign realm role(s) matching selected role name(s)
    ADM->>DB: INSERT users (keycloak_id, ...)
    ADM->>DB: INSERT user_roles
    ADM->>DB: INSERT audit_log (USER_CREATED)
    ADM-->>AdminUI: 201 Created, user representation
```

If the Keycloak create succeeds but the local insert fails, the endpoint responds
`502 KEYCLOAK_SYNC_FAILED` with the Keycloak `keycloak_id` recorded in the audit entry so
the orphaned Keycloak account can be reconciled by hand rather than retried blindly (a
blind retry risks a second Keycloak account for the same person).

## 3. Admin configures a role

Shows why menus and permissions are set together even though they're independent axes
(see [security-model.md §5](security-model.md#5-menus-vs-permissions-two-axes-one-role)).

```mermaid
sequenceDiagram
    actor Admin
    participant AdminUI as cce-admin-ui
    participant ADM as cce-admin-service
    participant DB as admin_service DB
    participant GW as gateway-service

    Admin->>AdminUI: create role FACILITY_ADMIN
    AdminUI->>ADM: POST /admin/v1/roles
    ADM->>DB: INSERT roles
    Admin->>AdminUI: select menus (Compliance, Patients)
    AdminUI->>ADM: PUT /admin/v1/roles/:id/menus
    ADM->>DB: replace role_menus rows
    Admin->>AdminUI: select API permissions (matching the same screens)
    AdminUI->>ADM: PUT /admin/v1/roles/:id/permissions
    ADM->>DB: replace api_permissions rows for permission_name = FACILITY_ADMIN
    ADM->>GW: POST /api/permissions/sync-to-keycloak
    GW->>GW: reconcile realm roles from api_permissions
    GW-->>ADM: 200 OK
    ADM->>DB: INSERT audit_log (ROLE_MENUS_UPDATED, ROLE_PERMISSIONS_UPDATED)
```

## 4. Admin updates a role's API permissions

Detail on the sync-ordering referenced in
[security-model.md §6](security-model.md#6-keycloak-provisioning-who-does-what): the
realm role definition must exist in Keycloak *before* any user is assigned it, so the
sync call happens as part of the permission update, not deferred.

```mermaid
sequenceDiagram
    participant ADM as cce-admin-service
    participant DB as admin_service DB
    participant GW as gateway-service
    participant KC as Keycloak

    ADM->>DB: DELETE existing api_permissions WHERE permission_name = :role
    ADM->>DB: INSERT new api_permissions rows
    ADM->>GW: POST /api/permissions/sync-to-keycloak
    GW->>DB: (its own read) SELECT DISTINCT permission_name FROM api_permissions
    GW->>KC: create/update/delete realm roles to match
    KC-->>GW: ok
    GW-->>ADM: 200 OK
    Note over ADM,GW: If this call fails, the role's menus/DB permissions are already\nsaved but the realm role isn't updated yet - role assignment to a\nuser (Flow 2) would then assign a stale/missing realm role.\nADM surfaces 502 KEYCLOAK_SYNC_FAILED so the admin retries the sync\nbefore assigning the role to anyone.
```

## 5. Gateway request authorization (existing behavior, shown for context)

Not implemented by this service — included because every write in flows 2-4 is only
reachable through this check.

```mermaid
sequenceDiagram
    actor Caller
    participant GW as gateway-service
    participant DB as admin_service DB (api_permissions)

    Caller->>GW: request + Bearer token
    GW->>GW: verify signature (JWKS), issuer, audience
    GW->>GW: extract roles from JWT_ROLES_CLAIM_PATH
    GW->>DB: match (method, uri) against api_permissions for caller's roles
    alt match found
        GW->>GW: proxy to target service
    else no match
        GW-->>Caller: 403 Forbidden
    end
```

## 6. User deactivation

```mermaid
sequenceDiagram
    actor Admin
    participant ADM as cce-admin-service
    participant KC as Keycloak Admin API
    participant DB as admin_service DB

    Admin->>ADM: PATCH /admin/v1/users/:id/status {status: INACTIVE}
    ADM->>KC: disable user (enabled=false)
    ADM->>DB: UPDATE users SET status='INACTIVE'
    ADM->>DB: INSERT audit_log (USER_DEACTIVATED)
    Note over ADM: user_roles rows are left intact (not deleted) so\nre-activation restores prior access without re-configuring roles
```
