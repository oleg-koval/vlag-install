# Vlag self-hosted installation candidate

Use is limited to non-production alpha evaluation under the accompanying
LICENSE. Third-party licenses and the SDK's MIT license remain separate.
Runtime binaries/browser assets are inspectable; private source access is not
granted. Availability, scans and acceptance must be verified for the exact
release version; these instructions alone do not establish publication or
external tester acceptance.

Use Docker Engine and Compose v2, with at least 2 GiB free RAM and sufficient
disk for both old/new images and a database backup. Exact CPU and capacity
support remains release-specific; current local acceptance is arm64 only.
The bundle needs no source checkout, Node, Clojure CLI or hosted Vlag account.

## Fresh installation

1. Verify the bundle checksum and its published image digests before extraction.
2. In a private directory, copy `.env.example` to `.env`; make it mode 0600.
   Fill in the three release digests. Generate independent random database and
   privacy secrets, each at least 32 bytes, using your password/secret manager.
   Keep the privacy secret stable across restarts and upgrades.
3. Run `docker compose pull`, then `docker compose up -d --wait db`.
4. Explicitly run `docker compose run --rm backend flag-platform.migrate`.
5. Create an owner-only `bootstrap-password` file (12+ characters, at most 72
   UTF-8 bytes). Bootstrap without printing it or passing its value as an argument:

```sh
docker compose run --rm --user "$(id -u):$(id -g)" --volume "$PWD/bootstrap-password:/run/bootstrap-password:ro" \
  backend flag-platform.bootstrap you@example.test Operator --password-file /run/bootstrap-password
```

The one-off bootstrap uses your UID to read the owner-only mounted file without
widening its permissions. Remove the password file after confirmed bootstrap. Bootstrap refuses a second user.

6. Run `docker compose up -d --wait`; open `http://localhost:8080`. Create a
   project/flag and an environment-scoped evaluate-only key. Follow the public
   `@olegkoval/vlag-sdk@0.1.0` guide with the local URL from a server process.
   Browser code must never receive the runtime key or operator credentials.

Default HTTP binds only to loopback and disables Secure cookies for local use.
For remote use, put an operator-managed TLS proxy on the same host in front of
loopback port 8080, preserve Host and forwarded headers, set COOKIE_SECURE=true
and PUBLIC_DASHBOARD_URL to the HTTPS origin. Never expose local HTTP directly.
No hosted log exporter, Sentry DSN, email or identity-provider credential is
configured by this bundle. Those optional integrations need separate setup.

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
Anonymous image pulls, offline operation after pull, persistence, exact-version
upgrade/recovery and independent installation still require release evidence.
