# Vlag self-hosted installation candidate

Use is limited to non-production alpha evaluation under the accompanying
LICENSE. Third-party licenses and the SDK's MIT license remain separate.
Runtime binaries/browser assets are inspectable; private source access is not
granted. Availability, scans and acceptance must be verified for the exact
release version; these instructions alone do not establish publication or
external tester acceptance.

Use Docker Engine and Compose v2, with at least 2 GiB free RAM and sufficient
disk for both old/new images and a database backup. Release `0.1.0-alpha.1`
provides backend and web images for `linux/amd64` only. Use an x86-64 Linux
Docker host or VM for this release. Check the release notes for other versions.
The bundle needs no source checkout, Node, Clojure CLI or hosted Vlag account.

## Fresh installation

1. Verify the archive against the release checksum before extracting it. After
   extraction, run `sha256sum -c SHA256SUMS` (or `shasum -a 256 -c SHA256SUMS`)
   from the extracted directory and compare its three image digests with the
   release notes.
2. In a private directory, copy `.env.example` to `.env`; make it mode 0600.
   Fill in the three release digests. Generate independent random database and
   privacy secrets, each at least 32 bytes, using your password/secret manager.
   Keep the privacy secret stable across restarts and upgrades.
3. For private GHCR images, get Read access to both packages. Run
   `docker login ghcr.io --username YOUR_GITHUB_USERNAME` and enter a GitHub
   classic personal access token with `read:packages` at the password prompt.
   Run `docker compose pull`, then `docker compose up -d --wait db`.
4. Explicitly run `docker compose run --rm backend flag-platform.migrate`.
5. Create an owner-only `bootstrap-password` file (12+ characters, at most 72
   UTF-8 bytes). Bootstrap without printing it or passing its value as an argument:

```sh
docker compose run --rm --user "$(id -u):$(id -g)" --volume "$PWD/bootstrap-password:/run/bootstrap-password:ro" \
  backend flag-platform.bootstrap you@example.test Operator --password-file /run/bootstrap-password
```

The one-off bootstrap uses your UID to read the owner-only mounted file without
widening its permissions. Remove the password file after confirmed bootstrap. Bootstrap refuses a second user.

6. Run `docker compose up -d --wait` and open `http://localhost:8080`. Sign in as
   the operator and create a project and flag. Create an evaluate-only key as
   described below, then follow the [public SDK guide](https://www.npmjs.com/package/@olegkoval/vlag-sdk/v/0.1.0)
   from a server process using the local URL. Keep runtime keys and operator
   credentials out of browser-delivered application code.

Default HTTP binds only to loopback and disables Secure cookies for local use.
For remote use, put an operator-managed TLS proxy on the same host in front of
loopback port 8080, preserve Host and forwarded headers, set COOKIE_SECURE=true
and PUBLIC_DASHBOARD_URL to the HTTPS origin. Never expose local HTTP directly.
No hosted log exporter, Sentry DSN, email or identity-provider credential is
configured by this bundle. Those optional integrations need separate setup.

## Create an evaluate-only key

In `0.1.0-alpha.1`, the dashboard's key dialog grants both evaluation and event
ingestion. Use the management API to limit a key to evaluation in one environment.
Send these requests to your local dashboard URL. Set `Accept: application/json`
and `Origin` to the dashboard's origin (for example, `http://localhost:8080`).
Set `Content-Type: application/json` for both POST requests.

1. Send `POST /v1/auth/login` with the operator's `email` and `password` in a
   JSON body. Use a cookie jar to retain the session cookie. Read `data.csrfToken`
   from the response. Keep credentials, cookies and responses out of logs.
2. With the session cookie, use `GET /v1/projects` to find your project's `id`,
   then `GET /v1/projects/{projectId}/environments` to find the desired
   environment's `id`. The Development environment has key `dev`.
3. Send `POST /v1/projects/{projectId}/api-keys` with the session cookie and set
   `X-CSRF-Token` to the token from step 1. Set the JSON body to:

   ```json
   {"name":"SDK evaluate only","environmentId":"YOUR_ENVIRONMENT_ID","permissions":["evaluate"]}
   ```

4. On HTTP 201, verify `data.environmentId` and `data.permissions` match the
   request. Save `data.secret` to your server-side secret store when it is
   returned. Use the environment's **key** (such as `dev`) in SDK configuration.

## Operations and upgrades

`docker compose stop` / `start` preserves data; `up -d --force-recreate` replaces
containers while preserving the named PostgreSQL volume. `down` removes only
containers/network. `down --volumes` permanently deletes this installation's data;
use it only for an explicitly intended uninstall after saving a verified backup.

Before upgrading, save `.env`, record current image digests, stop writes and take
an owner-readable `pg_dump --format=custom --no-owner --no-acl` backup through
`docker compose exec -T db`. Verify recovery on a separate disposable stack.
Change both application image digests together, pull, run migrations explicitly,
then `up -d --wait` and verify `/readyz`, sign-in, stored flags and audit history.
Keep the old images/backup until acceptance. Do not assume older binaries work
with a migrated schema; no automatic image/database rollback is provided.

A failed migration must stop the upgrade. A failed readiness check is a failed
release, not permission to delete/recreate the volume. Inspect `docker compose
logs backend` locally and redact credentials/personal data before sharing logs.
Record image pulls with the intended registry access, offline operation after
pull, persistence, version upgrades and recovery, and independent installation
as separate checks for each release.
