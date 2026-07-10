# github-workflows

Reusable GitHub Actions workflows for Kairo.js and related Minecraft Bedrock packs.

## Docker build & push

Use `.github/workflows/docker-build-push.yml` to build and push one or more Docker images to Docker Hub. Each service is pushed as `<DOCKERHUB_USERNAME>/<image-prefix>-<name>:<tag>`.

```yaml
jobs:
  build-and-push:
    uses: kairo-js/github-workflows/.github/workflows/docker-build-push.yml@v0.2.0
    with:
      services: '[{"name":"backend","context":"./backend"},{"name":"frontend","context":"./frontend"}]'
      image-prefix: mc-werewolf
      tag: dev
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
```

## App deploy

Use `.github/workflows/app-deploy.yml` to write a `.env` file and `docker compose pull && up -d` an app stack on a remote host over SSH. It expects a `deploy/docker-compose.yml` in the calling repo (see werewolf-server for an example) and deploys to `/opt/<app-name>/<deploy-env-name>` using compose project `<app-name>-<deploy-env-name>`, so multiple apps can share one host without container/network name collisions. Before `up`, it also ensures a docker network named `proxy` exists (creating it if missing) — see "Caddy snippet deploy" below. If a service needs to be reachable through the shared Caddy proxy, give it `networks: [default, proxy]` in your `deploy/docker-compose.yml` (see werewolf-server's `backend`/`frontend` for an example); services that don't need it (e.g. `postgres`) can omit `networks:` entirely and stay on `default` only.

```yaml
jobs:
  deploy:
    uses: kairo-js/github-workflows/.github/workflows/app-deploy.yml@v0.2.0
    with:
      app-name: werewolf
      image-prefix: mc-werewolf
      deploy-env-name: ${{ needs.vars.outputs.deploy-env-name }}
      image-tag: ${{ needs.vars.outputs.tag }}
      app-env: ${{ needs.vars.outputs.deploy-env-name }}
      postgres-user: werewolf
      postgres-db: werewolf
    secrets:
      DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
      DEPLOY_USER: ${{ secrets.DEPLOY_USER }}
      DEPLOY_SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      POSTGRES_PASSWORD: ${{ needs.vars.outputs.deploy-env-name == 'prod' && secrets.PROD_POSTGRES_PASSWORD || secrets.DEV_POSTGRES_PASSWORD }}
```

## Caddy snippet deploy

The reverse proxy / TLS termination for every app sharing a host is a single, fully self-contained Caddy stack that `.github/workflows/caddy-snippet-deploy.yml` owns end-to-end — there is no separate "proxy" repo. Calling this workflow does two things, in order:

1. **`deploy-proxy-body`**: idempotently ensures the shared `proxy-caddy` container is running on `PROXY_HOST` (creating the `proxy` docker network and `docker compose pull && up -d` if needed). The compose file and `Caddyfile` are generated inline inside the workflow — Caddy's own config never references any app by name, it just does `import /etc/caddy/conf.d/*.caddy` and joins the one generic `proxy` network. Safe to call from every app's deploy, every time; each app's own secrets (`PROXY_HOST`/`PROXY_USER`/`PROXY_SSH_KEY`) determine which host it lands on, so co-locating or splitting apps onto separate proxy hosts is just a matter of what each app repo's secrets point to — no shared file to edit either way.
2. **`deploy-snippet`**: writes the calling app's own routing config to `/opt/proxy/conf.d/<app-name>.caddy`, validates the full Caddy config, and reloads Caddy.

For an app's containers to actually be reachable, they must join the `proxy` docker network (see "App deploy" above). If a snippet needs supported secret values substituted in, pass `snippet-template-path` and the corresponding secrets; the reusable workflow renders the file immediately before copying it, so rendered secrets do not have to pass through job outputs.

```yaml
jobs:
  deploy-caddy:
    needs: deploy
    uses: kairo-js/github-workflows/.github/workflows/caddy-snippet-deploy.yml@v0.2.0
    with:
      app-name: werewolf
      snippet-template-path: deploy/caddy/service.caddy
      acme-email: ${{ vars.ACME_EMAIL }}
      prod-backend-upstream: ${{ vars.PROD_BACKEND_UPSTREAM }}
      prod-frontend-upstream: ${{ vars.PROD_FRONTEND_UPSTREAM }}
    secrets:
      PROXY_HOST: ${{ secrets.PROXY_HOST }}
      PROXY_USER: ${{ secrets.PROXY_USER }}
      PROXY_SSH_KEY: ${{ secrets.PROXY_SSH_KEY }}
      BASIC_AUTH_DEV_USER: ${{ secrets.BASIC_AUTH_DEV_USER }}
      BASIC_AUTH_DEV_HASH: ${{ secrets.BASIC_AUTH_DEV_HASH }}
      BASIC_AUTH_ADMIN_USER: ${{ secrets.BASIC_AUTH_ADMIN_USER }}
      BASIC_AUTH_ADMIN_HASH: ${{ secrets.BASIC_AUTH_ADMIN_HASH }}
```

`acme-email` is not a secret (it's just the address used for the Caddy instance's Let's Encrypt account) — set it as a repository **variable** (`Settings → Secrets and variables → Actions → Variables`), not a secret. All apps sharing one `PROXY_HOST` should set the same value.

For an app with no secret placeholders in its snippet, pass a fully rendered string as `snippet-content` instead.

Templates can also use `DEV_BACKEND_UPSTREAM`, `DEV_FRONTEND_UPSTREAM`, `PROD_BACKEND_UPSTREAM`, and `PROD_FRONTEND_UPSTREAM`. If the corresponding inputs are omitted, they default to `<app-name>-dev-backend:8000`, `<app-name>-dev-frontend:3000`, `<app-name>-prod-backend:8000`, and `<app-name>-prod-frontend:3000`.

## App undeploy

Use `.github/workflows/app-undeploy.yml` to manually withdraw an app from a host. It can remove the app's Caddy snippet, validate/reload Caddy, then run `docker compose down` for one or more deployment environments. `remove-volumes` defaults to `false`; set it to `true` only when the database volume should be deleted too. `deploy-env-names` accepts multiple space-separated environments in one call **only when they share a host** -- this workflow itself only ever talks to one `DEPLOY_HOST`. If dev and prod are on different hosts, call it once per environment instead, each with its own `DEPLOY_HOST`/`DEPLOY_USER`/`DEPLOY_SSH_KEY`, and only pass `remove-caddy-snippet: true` on the call that should actually remove the shared snippet (typically gated so it only fires once, not on both calls).

```yaml
jobs:
  undeploy-dev:
    if: ${{ inputs.target == 'all' || inputs.target == 'dev' }}
    uses: kairo-js/github-workflows/.github/workflows/app-undeploy.yml@v0.2.0
    with:
      app-name: werewolf
      deploy-env-names: dev
      remove-volumes: false
      remove-app-dirs: true
      remove-caddy-snippet: false
      confirm: undeploy werewolf
    secrets:
      DEPLOY_HOST: ${{ secrets.DEV_DEPLOY_HOST }}
      DEPLOY_USER: ${{ secrets.DEV_DEPLOY_USER }}
      DEPLOY_SSH_KEY: ${{ secrets.DEV_DEPLOY_SSH_KEY }}
      PROXY_HOST: ${{ secrets.PROXY_HOST }}
      PROXY_USER: ${{ secrets.PROXY_USER }}
      PROXY_SSH_KEY: ${{ secrets.PROXY_SSH_KEY }}

  undeploy-prod:
    if: ${{ inputs.target == 'all' || inputs.target == 'prod' }}
    uses: kairo-js/github-workflows/.github/workflows/app-undeploy.yml@v0.2.0
    with:
      app-name: werewolf
      deploy-env-names: prod
      remove-volumes: false
      remove-app-dirs: true
      remove-caddy-snippet: ${{ inputs.target == 'all' }}
      confirm: undeploy werewolf
    secrets:
      DEPLOY_HOST: ${{ secrets.PROD_DEPLOY_HOST }}
      DEPLOY_USER: ${{ secrets.PROD_DEPLOY_USER }}
      DEPLOY_SSH_KEY: ${{ secrets.PROD_DEPLOY_SSH_KEY }}
      PROXY_HOST: ${{ secrets.PROXY_HOST }}
      PROXY_USER: ${{ secrets.PROXY_USER }}
      PROXY_SSH_KEY: ${{ secrets.PROXY_SSH_KEY }}
```

If dev and prod are still co-located on one host, keep the original single-job form with `deploy-env-names: dev prod` and both `DEPLOY_HOST` secrets pointing at the same host.

## PostgreSQL backup

Use `.github/workflows/postgres-backup.yml` to `pg_dump` a PostgreSQL container running on a remote host over SSH, then upload the dump as both a workflow artifact and to Google Drive via `rclone`.

```yaml
jobs:
  backup:
    uses: kairo-js/github-workflows/.github/workflows/postgres-backup.yml@v0.2.0
    with:
      app-name: werewolf
      deploy-env-name: prod
      db-user: werewolf
      db-name: werewolf
      backup-prefix: werewolf-prod
      gdrive-destination-suffix: daily
      gdrive-retention-days: 30
      skip-if-missing: true
    secrets:
      DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
      DEPLOY_USER: ${{ secrets.DEPLOY_USER }}
      DEPLOY_SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
      RCLONE_CONFIG: ${{ secrets.RCLONE_CONFIG }}
      BACKUP_GDRIVE_DESTINATION: ${{ secrets.PROD_GDRIVE_DESTINATION }}
```

`RCLONE_CONFIG` must contain a full `rclone.conf` with a remote named `gdrive` (OAuth-authorized; service accounts can't write to a personal Drive due to `storageQuotaExceeded`).
`BACKUP_GDRIVE_DESTINATION` is the repository-specific path under that remote, e.g. `minecraft/werewolf/server/prod/backups`. Callers can map this from environment-specific secrets such as `PROD_GDRIVE_DESTINATION` or `DEV_GDRIVE_DESTINATION`. Set `gdrive-destination-suffix: daily` to upload into a child folder such as `minecraft/werewolf/server/prod/backups/daily`. Set `gdrive-retention-days: 30` to delete `.dump` files older than 30 days in the resolved Google Drive folder.
Set `skip-if-missing: true` for pre-deploy backups that should pass on the first deploy before the prod database exists.

## Web app release

Use `.github/workflows/web-app-release.yml` when an app should use the standard full release flow: build Docker images, back up prod before tag deploys, deploy the app stack, and deploy/reload the shared Caddy snippet.

```yaml
jobs:
  release:
    uses: kairo-js/github-workflows/.github/workflows/web-app-release.yml@v0.2.0
    with:
      app-name: werewolf
      image-prefix: mc-werewolf
      services: '[{"name":"backend","context":"./backend"},{"name":"frontend","context":"./frontend"}]'
      postgres-user: werewolf
      postgres-db: werewolf
      caddy-snippet-template-path: deploy/caddy/service.caddy
      acme-email: ${{ vars.ACME_EMAIL }}
      prod-backup-prefix: werewolf-prod-before-${{ github.ref_name }}
      prod-backend-upstream: ${{ vars.PROD_BACKEND_UPSTREAM }}
      prod-frontend-upstream: ${{ vars.PROD_FRONTEND_UPSTREAM }}
      prod-backend-port: ${{ vars.PROD_BACKEND_PORT }}
      prod-frontend-port: ${{ vars.PROD_FRONTEND_PORT }}
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
      DEV_DEPLOY_HOST: ${{ secrets.DEV_DEPLOY_HOST }}
      DEV_DEPLOY_USER: ${{ secrets.DEV_DEPLOY_USER }}
      DEV_DEPLOY_SSH_KEY: ${{ secrets.DEV_DEPLOY_SSH_KEY }}
      PROD_DEPLOY_HOST: ${{ secrets.PROD_DEPLOY_HOST }}
      PROD_DEPLOY_USER: ${{ secrets.PROD_DEPLOY_USER }}
      PROD_DEPLOY_SSH_KEY: ${{ secrets.PROD_DEPLOY_SSH_KEY }}
      DEV_POSTGRES_PASSWORD: ${{ secrets.DEV_POSTGRES_PASSWORD }}
      PROD_POSTGRES_PASSWORD: ${{ secrets.PROD_POSTGRES_PASSWORD }}
      PROXY_HOST: ${{ secrets.PROXY_HOST }}
      PROXY_USER: ${{ secrets.PROXY_USER }}
      PROXY_SSH_KEY: ${{ secrets.PROXY_SSH_KEY }}
      RCLONE_CONFIG: ${{ secrets.RCLONE_CONFIG }}
      PROD_GDRIVE_DESTINATION: ${{ secrets.PROD_GDRIVE_DESTINATION }}
      BASIC_AUTH_DEV_USER: ${{ secrets.BASIC_AUTH_DEV_USER }}
      BASIC_AUTH_DEV_HASH: ${{ secrets.BASIC_AUTH_DEV_HASH }}
      BASIC_AUTH_ADMIN_USER: ${{ secrets.BASIC_AUTH_ADMIN_USER }}
      BASIC_AUTH_ADMIN_HASH: ${{ secrets.BASIC_AUTH_ADMIN_HASH }}
```

`DEV_DEPLOY_*` and `PROD_DEPLOY_*` are separate secrets so dev and prod can live on different hosts. To keep them co-located on one host instead, just set both triples to the same `HOST`/`USER`/`SSH_KEY` values -- `app-deploy.yml`'s `/opt/<app-name>/<deploy-env-name>` directory and `<app-name>-<deploy-env-name>` compose project naming already avoid collisions either way.

When `dev-backend-port`, `dev-frontend-port`, `prod-backend-port`, or `prod-frontend-port` are set, app deploy adds a small Compose override that publishes the corresponding backend/frontend container port on the app host. Leave them empty for same-host Caddy via the Docker `proxy` network. If dev and prod are on different hosts, also set `dev-backend-upstream`/`dev-frontend-upstream`/`prod-backend-upstream`/`prod-frontend-upstream` (see "Caddy snippet deploy" below) to `<host>:<port>` so the shared Caddy proxy can actually reach them -- the default upstreams (`<app-name>-dev-backend:8000`, etc.) only resolve when Caddy shares the same Docker `proxy` network as the app.

## Minecraft pack release

Use `.github/workflows/minecraft-pack-release.yml` from a pack repository to build `BP` and `RP`, publish a Minecraft-friendly `.mcaddon`, and publish a `.zip` whose contents are `*-BP.zip` and `*-RP.zip`.

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    uses: kairo-js/github-workflows/.github/workflows/minecraft-pack-release.yml@v0.2.0
    permissions:
      contents: write
    with:
      artifact-name: kairo
```
