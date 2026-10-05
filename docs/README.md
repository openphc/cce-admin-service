# cce-admin-service Documentation

`cce-admin-service` manages users, roles, menus, and API permissions for the CCE
platform, so that a user logging in to `cce-insights-ui` sees only the menus their role
permits, and `gateway-service` only lets them call the APIs their role permits.

Read in this order:

1. [architecture-overview.md](architecture-overview.md) — what this service is, how it
   fits with `cce-insights-ui`, `gateway-service`, and Keycloak, tech stack, and the
   existing constraints it was designed around.
2. [data-dictionary.md](data-dictionary.md) — the `admin_service` database schema.
3. [security-model.md](security-model.md) — authentication, the role/permission/menu
   authorization model, and Keycloak provisioning responsibilities.
4. [api-reference.md](api-reference.md) — REST endpoints, request/response shapes,
   error codes.
5. [flow-diagrams.md](flow-diagrams.md) — sequence diagrams for login/menu resolution,
   user onboarding, role configuration, and deactivation.

Each document links to the others for anything outside its own scope rather than
repeating it — if you're looking for something and don't find it where you expected,
follow the links.
