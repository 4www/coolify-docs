---
title: "Matrix"
description: "Run Matrix Synapse server on Coolify for decentralized chat, end-to-end encryption, federation, OIDC SSO, and secure real-time communication."
---

# Matrix (Synapse)

<ZoomableImage src="/docs/images/services/matrix-logo.svg" alt="Matrix dashboard" />

## What is Matrix?

Matrix is an open-source, decentralized communication protocol that enables secure, real-time communication. It provides end-to-end encrypted messaging, voice and video calls, file sharing, and room-based conversations. Matrix serves as an excellent alternative to proprietary platforms like Slack or Discord, offering federation capabilities that allow different Matrix servers to communicate with each other.

## What is Synapse?

Synapse is a [Matrix homeserver](https://matrix.org/ecosystem/servers/) written in Python/Twisted, [developed and maintained](https://github.com/element-hq/synapse) by the team [Element](https://element.io/), creators of Matrix.

## Deployment variants

Synapse Matrix server is available in two deployment configurations in Coolify:

### Synapse with SQLite

- **Database:** SQLite (embedded)
- **Use case:** Simple deployments, testing, or personal Matrix hosting
- **Components:** Single Synapse container with built-in SQLite database

### Synapse with PostgreSQL (recommended)

- **Database:** PostgreSQL
- **Use case:** Production deployments requiring better performance and scalability
- **Components:**
  - Synapse container
  - PostgreSQL container
  - Automatic database configuration and health checks

## Installation steps

For all deployment variants the installation steps are the same.

### Matrix domain setup (important)

Matrix uses a value called the **server name** to generate user IDs and room aliases.

- Server name `example.org` results in:
  - `@user:example.org`
  - `#room:example.org`

The Matrix server itself can run on a different domain, for example `matrix.example.org`.

### Recommended setup

- Matrix server name: `example.org`
- Matrix Synapse server service domain: `matrix.example.org`

This allows users and rooms to use `:example.org` while hosting Synapse on a subdomain.

### Coolify configuration

#### Domains

In the service configuration, set the domain to `matrix.example.org:8008`.

#### Environment variables

Set the Matrix server name before the first deployment:

```env
SYNAPSE_SERVER_NAME=example.org
```

You can optionally set an explicit public base URL. This is useful for SSO/OIDC and avoids relying on an auto-generated service URL:

```env
SYNAPSE_PUBLIC_BASEURL=https://matrix.example.org
```

> [!WARNING]
> Choose `SYNAPSE_SERVER_NAME` carefully. It becomes part of every Matrix user ID and room alias and is not easy to change later.

## User registration

Coolify's Matrix recipe keeps open registration disabled by default:

```env
ENABLE_REGISTRATION=false
```

Enable it only if you want anyone who can reach your homeserver to create an account:

```env
ENABLE_REGISTRATION=true
```

For public registration, also configure reCAPTCHA:

```env
RECAPTCHA_PUBLIC_KEY=your-public-key
RECAPTCHA_PRIVATE_KEY=your-private-key
```

To allow registration without email or other verification checks, set:

```env
ENABLE_REGISTRATION_WITHOUT_VERIFICATION=true
```

> [!WARNING]
> `ENABLE_REGISTRATION_WITHOUT_VERIFICATION=true` is only intended for trusted/private deployments. For public servers, use verified registration and abuse controls.

## Admin user

The recipe can automatically register one administrator account after Synapse starts:

```env
SERVICE_USER_ADMIN=admin
SERVICE_PASSWORD_ADMIN=change-me
```

If either value is empty, the recipe skips admin user creation. If the account already exists, Synapse will not create a duplicate account.

## Guest access

Guest access is disabled by default:

```env
ENABLE_GUEST_ACCESS=false
```

Enable it only if you intentionally want unauthenticated guest users:

```env
ENABLE_GUEST_ACCESS=true
```

## Rooms and federation

The recipe supports a few common room/federation toggles:

```env
AUTO_JOIN_ROOMS=#general:example.org
AUTO_CREATE_AUTO_JOIN_ROOMS=true
AUTO_CREATE_AUTO_JOIN_ROOMS_FEDERATED=false
ALLOW_PUBLIC_ROOMS_OVER_FEDERATION=false
ALLOW_PUBLIC_ROOMS_WITHOUT_AUTH=false
```

Use a comma-separated list for multiple auto-join rooms:

```env
AUTO_JOIN_ROOMS=#general:example.org,#announcements:example.org
```

## OIDC / SSO

Synapse can authenticate users with an OpenID Connect provider such as Authentik, Keycloak, Zitadel, Auth0, or another OIDC-compatible identity provider.

Enable OIDC in Coolify:

```env
OIDC_ENABLED=true
OIDC_PROVIDER_ID=internal
OIDC_PROVIDER_NAME=Internal SSO
OIDC_ISSUER=https://identity.example.org/application/o/matrix-internal/
OIDC_CLIENT_ID=your-client-id
OIDC_CLIENT_SECRET=your-client-secret
OIDC_SCOPES=openid,profile,email
```

With OIDC enabled, Synapse can automatically create Matrix users on first successful SSO login:

```env
OIDC_ENABLE_REGISTRATION=true
```

This is separate from normal Matrix registration. You can keep open registration disabled while allowing trusted SSO users to be created on first login:

```env
ENABLE_REGISTRATION=false
OIDC_ENABLE_REGISTRATION=true
```

Allow existing local Matrix accounts to be linked to matching OIDC users:

```env
OIDC_ALLOW_EXISTING_USERS=true
```

### Redirect URI

Configure this redirect URI in your OIDC provider:

```text
https://matrix.example.org/_synapse/client/oidc/callback
```

It must match Synapse's public base URL. If the public service is `https://matrix.example.org`, set:

```env
SYNAPSE_PUBLIC_BASEURL=https://matrix.example.org
```

### Authentik example

For an Authentik application with slug `matrix-internal` hosted at `identity.example.org`:

```env
OIDC_ENABLED=true
OIDC_PROVIDER_ID=authentik
OIDC_PROVIDER_NAME=Authentik
OIDC_ISSUER=https://identity.example.org/application/o/matrix-internal/
OIDC_CLIENT_ID=your-client-id
OIDC_CLIENT_SECRET=your-client-secret
OIDC_SCOPES=openid,profile,email
OIDC_CLIENT_AUTH_METHOD=client_secret_basic
OIDC_ENABLE_REGISTRATION=true
OIDC_ALLOW_EXISTING_USERS=true
```

In Authentik, configure the provider with:

```text
Redirect URI: https://matrix.example.org/_synapse/client/oidc/callback
Redirect URI mode: Strict
Signing key: RSA key
Client type: Confidential
```

> [!NOTE]
> Some providers require `client_secret_basic`; others may use `client_secret_post`. Keep the default unless your provider requires a different method.

### OIDC claim mapping

The recipe maps OIDC claims to Matrix users with these defaults:

```env
OIDC_LOCALPART_TEMPLATE={{ user.preferred_username|lower }}
OIDC_DISPLAY_NAME_TEMPLATE={{ user.name|default(user.preferred_username) }}
OIDC_EMAIL_TEMPLATE={{ user.email }}
```

For example, a user with `preferred_username=alice` becomes:

```text
@alice:example.org
```

Do not change the localpart mapping after users have started logging in unless you understand the account-linking consequences.

## Testing SSO

After deploying, test the login endpoint:

```bash
curl https://matrix.example.org/_matrix/client/v3/login
```

You should see an SSO login flow similar to:

```json
{
  "type": "m.login.sso"
}
```

To test the OIDC redirect flow, use the provider ID from `OIDC_PROVIDER_ID`:

```bash
curl -I "https://matrix.example.org/_matrix/client/v3/login/sso/redirect/oidc-authentik?redirectUrl=https%3A%2F%2Fmatrix.example.org"
```

A working setup returns a `302` redirect to the configured OIDC issuer.

## Delegation (required when server name differs from service domain)

Because Synapse runs on `matrix.example.org` but identifies as `example.org`, [delegation](https://element-hq.github.io/synapse/latest/delegate.html) is required.

On `https://example.org`, serve the following files:

- `/.well-known/matrix/client` for client discovery

```json
{
  "m.homeserver": {
    "base_url": "https://matrix.example.org"
  }
}
```

- `/.well-known/matrix/server` for federation discovery

```json
{
  "m.server": "matrix.example.org:443"
}
```

## Useful environment variables

| Variable | Default | Description |
| --- | --- | --- |
| `SYNAPSE_SERVER_NAME` | `SERVICE_FQDN_MATRIX` | Matrix server name used in user IDs and room aliases. |
| `SYNAPSE_PUBLIC_BASEURL` | `SERVICE_URL_MATRIX_8008` | Public URL used by Synapse for links and OIDC callbacks. |
| `SYNAPSE_REPORT_STATS` | `no` | Whether Synapse reports anonymous usage statistics. |
| `ENABLE_REGISTRATION` | `false` | Enables normal Matrix account registration. |
| `ENABLE_REGISTRATION_WITHOUT_VERIFICATION` | `false` | Allows unverified registration. Use with care. |
| `ENABLE_GUEST_ACCESS` | `false` | Enables guest users. |
| `RECAPTCHA_PUBLIC_KEY` | empty | Public key for registration CAPTCHA. |
| `RECAPTCHA_PRIVATE_KEY` | empty | Private key for registration CAPTCHA. |
| `AUTO_JOIN_ROOMS` | empty | Comma-separated rooms for new users to join. |
| `AUTO_CREATE_AUTO_JOIN_ROOMS` | `true` | Automatically create configured auto-join rooms. |
| `AUTO_CREATE_AUTO_JOIN_ROOMS_FEDERATED` | `false` | Allow auto-created rooms to be federated. |
| `ALLOW_PUBLIC_ROOMS_OVER_FEDERATION` | `false` | Allow publishing public rooms over federation. |
| `ALLOW_PUBLIC_ROOMS_WITHOUT_AUTH` | `false` | Allow unauthenticated users to view the public room directory. |
| `OIDC_ENABLED` | `false` | Enables OIDC SSO. |
| `OIDC_PROVIDER_ID` | `oidc` | Internal provider ID used in Synapse SSO URLs. |
| `OIDC_PROVIDER_NAME` | `OpenID Connect` | Display name for the SSO provider. |
| `OIDC_ISSUER` | empty | OIDC issuer URL. |
| `OIDC_CLIENT_ID` | empty | OIDC client ID. |
| `OIDC_CLIENT_SECRET` | empty | OIDC client secret. |
| `OIDC_SCOPES` | `openid,profile,email` | Comma-separated OIDC scopes. |
| `OIDC_CLIENT_AUTH_METHOD` | `client_secret_basic` | OIDC client authentication method. |
| `OIDC_ENABLE_REGISTRATION` | `true` | Automatically creates Matrix users after successful OIDC login. |
| `OIDC_ALLOW_EXISTING_USERS` | `true` | Allows matching OIDC users to existing Matrix accounts. |
| `OIDC_LOCALPART_TEMPLATE` | `{{ user.preferred_username|lower }}` | Matrix localpart mapping template. |
| `OIDC_DISPLAY_NAME_TEMPLATE` | `{{ user.name\|default(user.preferred_username) }}` | Matrix display name mapping template. |
| `OIDC_EMAIL_TEMPLATE` | `{{ user.email }}` | Matrix email mapping template. |

## Links

- [The official website](https://matrix.org?utm_source=coolify.io)
- [Synapse documentation](https://element-hq.github.io/synapse/latest/?utm_source=coolify.io)
- [Synapse OIDC documentation](https://element-hq.github.io/synapse/latest/openid.html?utm_source=coolify.io)
- [GitHub](https://github.com/element-hq/synapse?utm_source=coolify.io)
- [Docker image](https://hub.docker.com/r/matrixdotorg/synapse?utm_source=coolify.io)
- [Matrix Federation Tester](https://federationtester.matrix.org?utm_source=coolify.io)
