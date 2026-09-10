# Chewie RBAC via FundFlow Capabilities

Chewie access is controlled by FundFlow RBAC capabilities, not by a separate
OpenWebUI user list.

## Claim Flow

```text
FundFlow user/personas/capability overrides
  -> FundFlow OIDC claim: chewie_role
  -> Keycloak broker/user attribute
  -> Keycloak token claim: chewie_role
  -> OpenWebUI OAuth role management
```

FundFlow emits one of these values:

- `chewie-admin`: user has the FundFlow admin wildcard capability `*`.
- `chewie-user`: user has at least one configured Chewie capability.
- `chewie-denied`: user does not have a configured Chewie capability.

Production currently grants Chewie access through:

```text
ai_research:view
```

## OpenWebUI Enforcement

The production deployment enables:

```env
ENABLE_OAUTH_ROLE_MANAGEMENT=true
OAUTH_ROLES_CLAIM=chewie_role
OAUTH_ALLOWED_ROLES=chewie-user
OAUTH_ADMIN_ROLES=chewie-admin
```

OpenWebUI is patched to fail closed when role management is enabled but the
configured role claim is missing.

## Keycloak Mapping

In realm `artha`, use the existing identity provider `Artha Group` with alias
`arthatest`.

1. Go to `Identity providers` -> `Artha Group` -> `Mappers`.
2. Add an OIDC broker mapper that imports claim `chewie_role` into user
   attribute `chewie_role`.
3. Set sync mode to `Force`, so changed FundFlow capabilities are refreshed on
   subsequent login.
4. Go to client `open-webui-production`.
5. Add a client protocol mapper for user attribute `chewie_role`.
6. Emit token claim name `chewie_role` as a string.
7. Include the claim in ID token, access token, and userinfo.

After mapping, a user without `ai_research:view` should receive a token with:

```json
{"chewie_role": "chewie-denied"}
```

OpenWebUI should reject that user with `403`.

