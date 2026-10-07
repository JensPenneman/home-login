# home-login

Passkey sign-in for self-hosted admin apps, packaged as a Docker Compose app
for Launchway:

- **[Pocket ID](https://pocket-id.org)** is the identity provider. It knows
  passkeys only; there are no passwords.
- **[oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/)** is the
  gate. The edge asks it about every request to a protected route
  (`forward_auth`): with a valid session cookie it answers 202 and the request
  goes through, without one it sends the browser through Pocket ID.

One passkey sign-in gives a session cookie on `.<COOKIE_DOMAIN>` that covers
every protected host for 24 hours. Only members of the Pocket ID group
`ALLOWED_GROUPS` get in.

## Deploy in Launchway

1. Create the app from this repository (`compose.yaml`) on the **edge node**:
   the gate and oauth2-proxy's calls to Pocket ID go over the edge's proxy
   network.
2. Set the [environment variables](#environment-variables).
3. Create the [routes](#routes) **before the first deployment**: Launchway
   joins a service to its proxy network only through a route, and
   oauth2-proxy reads Pocket ID's discovery document through the edge when it
   starts.
4. Deploy, then set the [gate](#gate).

### Routes

A Launchway route serves one whole host name and joins its service to the
proxy network under the alias `<app-slug>-<service>`. oauth2-proxy needs a
route as well: the edge must reach it for the gate, and the OIDC callback
`/oauth2/callback` must reach it. So each service gets a host name:

| Host name | Service | Port | Protected |
| --- | --- | --- | --- |
| `login.<domain>` (`LOGIN_HOST`) | `pocket-id` | 1411 | no |
| `auth.<domain>` (`OAUTH2_HOST`) | `oauth2-proxy` | 4180 | no |

The OIDC client in Pocket ID then has the callback URL
`https://<OAUTH2_HOST>/oauth2/callback`.

Routing only the login host (to Pocket ID) and pointing the gate at
oauth2-proxy is not enough: without a route oauth2-proxy is not on the proxy
network, and the callback would land on Pocket ID.

**One host name**, as in a standalone setup where `/oauth2/*` on the login
host goes to oauth2-proxy: once a route can carry extra Caddy directives, add
`handle /oauth2/* { reverse_proxy <app-slug>-oauth2-proxy:4180 }` to the login
route (see [closed paths](#closed-paths)), leave `OAUTH2_HOST` empty and use
`https://<LOGIN_HOST>/oauth2/callback` as the callback URL. Keep the
`auth.<domain>` route while Launchway attaches only routed services: it is
what puts oauth2-proxy on the proxy network.

### Gate

In Launchway's settings, set the forward-auth URL to
`http://<app-slug>-oauth2-proxy:4180/` (the path `/` becomes the
`forward_auth` `uri`). Every route marked *protected* then asks oauth2-proxy
first: 202 for a signed-in member of `ALLOWED_GROUPS`, a redirect to the
sign-in for everyone else.

### Closed paths

The login host must not offer Pocket ID's first-admin setup or sign-up, and
must not accept admin API keys from outside. Add these extra directives to the
`login.<domain>` route (the last `handle` only for the one-host variant):

```caddy
# admin API keys only on the node itself (127.0.0.1:20014)
request_header -X-API-KEY
@closed path /setup /setup/* /signup /signup/* /api/signup /api/signup/*
handle @closed {
	respond 404
}
handle /oauth2/* {
	reverse_proxy <app-slug>-oauth2-proxy:4180
}
```

Caddy runs `handle` before the route's own `reverse_proxy`, so every other
path still reaches Pocket ID.

Until Launchway routes accept extra directives: sign-ups stay off in Pocket ID
itself (`ALLOW_USER_SIGNUPS=disabled`) and the setup page refuses once an admin
exists (create it right after the first deployment), but a static API key
would work from the internet, so set `POCKET_ID_STATIC_API_KEY` only while you
need the admin API.

## Environment variables

Set them as the app's (encrypted) environment variables in Launchway. Compose
reads them for the `${…}` in `compose.yaml`; nothing else reaches the
containers.

| Variable | Required | Meaning | Value |
| --- | --- | --- | --- |
| `LOGIN_HOST` | yes | Host name of Pocket ID; the passkeys are bound to it | `login.example.com` |
| `COOKIE_DOMAIN` | yes | Parent domain of the login host and every protected host: the session cookie is set on `.<COOKIE_DOMAIN>`, and oauth2-proxy redirects only to hosts under it | `example.com` |
| `POCKET_ID_ENCRYPTION_KEY` | yes | Encrypts Pocket ID's signing keys at rest; at least 16 bytes | `openssl rand -hex 32` |
| `OAUTH2_PROXY_COOKIE_SECRET` | yes | Encrypts the session cookie; 32 bytes as URL-safe base64 | `openssl rand -base64 32 \| tr -- '+/' '-_'` |
| `OAUTH2_PROXY_CLIENT_ID` | yes | Client ID of the OIDC client in Pocket ID | from Pocket ID |
| `OAUTH2_PROXY_CLIENT_SECRET` | yes | Its client secret | from Pocket ID |
| `OAUTH2_HOST` | no | Host name routed to oauth2-proxy, under `COOKIE_DOMAIN`; the callback is `https://<OAUTH2_HOST>/oauth2/callback` | default `LOGIN_HOST` |
| `TRUSTED_PROXY_IP` | no | The edge's fixed address on the proxy network: the only source of `X-Forwarded-*` both services trust, and where oauth2-proxy reaches `LOGIN_HOST` | default `10.210.0.2` |
| `POCKET_ID_STATIC_API_KEY` | no | Admin API key (header `X-API-KEY`) for automation, at least 16 characters; empty disables it | `openssl rand -hex 32` |
| `ALLOWED_GROUPS` | no | Pocket ID group whose members get in | default `admins` |
| `LOGIN_APP_NAME` | no | Name on the sign-in pages | default `Pocket ID` |
| `LOGIN_ACCENT_COLOR` | no | Accent colour of the sign-in pages, such as `#2563eb` | default `default` |
| `TZ` | no | Time zone of Pocket ID | default `Europe/Brussels` |

The cookie secret must decode to 32 bytes as URL-safe base64. Without the
`tr`, an output with `+` or `/` is read as a 44-byte string and oauth2-proxy
refuses to start.

**OIDC client:** in Pocket ID, create the group `ALLOWED_GROUPS`, then an OIDC
client with the callback URL from [Routes](#routes), PKCE on and only that
group allowed. Its client ID and secret go into `OAUTH2_PROXY_CLIENT_ID` and
`OAUTH2_PROXY_CLIENT_SECRET`. On a first install, deploy once with
placeholders for both, create the client, then set the real values and
redeploy.

## Migrating from a standalone deployment

1. Stop the old `pocket-id` and `oauth2-proxy`: the database must be at rest
   while it is copied, and the old stack publishes the same `127.0.0.1:20014`.
2. Keep the secrets and the login host. `POCKET_ID_ENCRYPTION_KEY` is the exact
   content of the old `ENCRYPTION_KEY_FILE` (Pocket ID uses that file byte for
   byte, without trimming); another key cannot read the old data. The passkeys
   are bound to `LOGIN_HOST`. The cookie secret, client ID and secret and the
   static API key carry over as they are.
3. Before the first deployment, create the new project's volume and copy the
   data into it. Launchway's Compose project is `launchway-<app-slug>`; the
   labels let Compose adopt the volume as its own:

   ```sh
   docker volume create \
     --label com.docker.compose.project=launchway-<app-slug> \
     --label com.docker.compose.volume=pocket-id-data \
     launchway-<app-slug>_pocket-id-data
   docker run --rm -v <old-project>_pocket-id-data:/from:ro \
     -v launchway-<app-slug>_pocket-id-data:/to alpine cp -a /from/. /to/
   ```

4. With the two-host routes, add `https://<OAUTH2_HOST>/oauth2/callback` to the
   OIDC client's callback URLs.

## License

[MIT](LICENSE)
