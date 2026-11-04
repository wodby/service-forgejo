# Forgejo on Wodby

What Wodby sets up for Forgejo on this service. Forgejo runs from the official rootless image `data.forgejo.org/forgejo/forgejo:<version>-rootless` as user and group 1000. Configuration is passed as `FORGEJO__<section>__<KEY>` environment variables, which the image writes to `app.ini` on every start; a variable with an empty section name (`FORGEJO____APP_NAME`) sets a key at the top of the file.

## Linked services

| Link | Variables |
| --- | --- |
| PostgreSQL (required) | `FORGEJO__database__DB_TYPE`, `FORGEJO__database__HOST`, `FORGEJO__database__NAME`, `FORGEJO__database__USER`, `FORGEJO__database__PASSWD`, `FORGEJO__database__SSL_MODE` |
| SMTP server (optional) | `FORGEJO__mailer__ENABLED`, `FORGEJO__mailer__PROTOCOL`, `FORGEJO__mailer__SMTP_ADDR`, `FORGEJO__mailer__SMTP_PORT` |

`FORGEJO__database__SSL_MODE` is `prefer`: the connection uses TLS when the server offers it, as managed databases do, and plain TCP to the bundled PostgreSQL service. The certificate is not verified.

Mail goes to the linked mail service without authentication or TLS. The mail variables are present only while the link exists. The sender is `noreply@<site domain>` unless `FORGEJO__mailer__FROM` is set on the service.

## What the manifest sets

- `FORGEJO__server__DOMAIN`, `FORGEJO__server__SSH_DOMAIN` and `FORGEJO__server__ROOT_URL` follow the service's primary route. A route change marks the service for redeploy.
- `FORGEJO____APP_NAME` is the app's title.
- `FORGEJO__security__INSTALL_LOCK` is `true`, so the web installer never runs.
- Generated tokens provide `FORGEJO__security__SECRET_KEY`, `FORGEJO__security__INTERNAL_TOKEN`, `FORGEJO__oauth2__JWT_SECRET` and `FORGEJO__server__LFS_JWT_SECRET`. They are stable across deployments. Changing `SECRET_KEY` makes stored two-factor secrets and other encrypted values unreadable.
- `FORGEJO__service__DISABLE_REGISTRATION` is `true`: accounts are created by an administrator. Set it to `false` to open sign-up.
- `FORGEJO__security__REVERSE_PROXY_TRUSTED_PROXIES` lists private networks, so Forgejo takes client addresses from the cluster gateway's `X-Forwarded-For` header.
- Git LFS is on (`FORGEJO__server__LFS_START_SERVER`).
- The image keeps `app.ini` at `/var/lib/gitea/custom/conf/app.ini` (its `GITEA_APP_INI` variable), on the data volume.

## Administrator

The first start creates an administrator from the service settings Admin username (default `forgejo_admin`), Admin password (required, at least 8 characters) and Admin email (default `<username>@<site domain>`), only when Forgejo has no administrator. The container stops with an error when the password is missing or too short. Later changes to these settings are not applied; change the password in Forgejo.

## Git over SSH

The `forgejo` endpoint has an `http` port (3000) and an `ssh` port (2222, Forgejo's built-in SSH server). Git over SSH needs the `ssh` port published on the app's Ports page. Publishing redeploys the service: `FORGEJO_SSH_PUBLIC_PORT` gets the public port, the startup script sets `FORGEJO__server__SSH_PORT` to it and turns SSH on. Until then `FORGEJO__server__DISABLE_SSH` is `true` and Forgejo shows only HTTPS clone addresses. The SSH user is `git`.

## Startup

The container runs a script after the image's entrypoint has written `app.ini`: it applies the SSH and mail sender values, runs `forgejo migrate`, creates the administrator when needed, then starts `forgejo web`. The health endpoint is `/api/healthz`; the startup probe allows 15 minutes for database migrations after an upgrade.

## Changing configuration

Change Forgejo settings by adding or editing `FORGEJO__<section>__<KEY>` environment variables on the service, then deploy. Removing a variable doesn't remove its key from `app.ini`; set the value you want instead. Settings in Forgejo's admin panel that are stored in the database, such as users and organizations, are not affected.

## Data

One `data` volume is mounted at `/var/lib/gitea`. It holds repositories (`git/repositories`), LFS objects, packages, attachments, avatars, Actions logs and artifacts, the SSH host keys and `app.ini`. Everything else is in the linked PostgreSQL database. Back up the volume and the database together. The manifest declares no backup, import or action for this service.

Forgejo Actions is available, but no runner is included: register an external Forgejo Runner to run workflows. The service runs one replica and can't be scaled.
