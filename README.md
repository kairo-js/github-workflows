# github-workflows

Reusable GitHub Actions workflows for Kairo.js and related Minecraft Bedrock packs.

## Docker build & push

Use `.github/workflows/docker-build-push.yml` to build and push one or more Docker images to Docker Hub. Each service is pushed as `<DOCKERHUB_USERNAME>/<image-prefix>-<name>:<tag>`.

```yaml
jobs:
  build-and-push:
    uses: kairo-js/github-workflows/.github/workflows/docker-build-push.yml@v0.1.2
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
    uses: kairo-js/github-workflows/.github/workflows/app-deploy.yml@v0.1.2
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
2. **`deploy-snippet`**: writes the calling app's own routing config to `/opt/proxy/conf.d/<app-name>.caddy` and reloads Caddy.

For an app's containers to actually be reachable, they must join the `proxy` docker network (see "App deploy" above). If a snippet needs supported secret values substituted in, pass `snippet-template-path` and the corresponding secrets; the reusable workflow renders the file immediately before copying it, so rendered secrets do not have to pass through job outputs.

```yaml
jobs:
  deploy-caddy:
    needs: deploy
    uses: kairo-js/github-workflows/.github/workflows/caddy-snippet-deploy.yml@v0.1.2
    with:
      app-name: werewolf
      snippet-template-path: deploy/caddy/service.caddy
      acme-email: ${{ vars.ACME_EMAIL }}
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

## PostgreSQL backup

Use `.github/workflows/postgres-backup.yml` to `pg_dump` a PostgreSQL container running on a remote host over SSH, then upload the dump as both a workflow artifact and to Google Drive via `rclone`.

```yaml
jobs:
  backup:
    uses: kairo-js/github-workflows/.github/workflows/postgres-backup.yml@v0.1.2
    with:
      app-name: werewolf
      deploy-env-name: prod
      db-user: werewolf
      db-name: werewolf
      backup-prefix: werewolf-prod
      skip-if-missing: true
    secrets:
      DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
      DEPLOY_USER: ${{ secrets.DEPLOY_USER }}
      DEPLOY_SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
      RCLONE_CONFIG: ${{ secrets.RCLONE_CONFIG }}
      BACKUP_GDRIVE_DESTINATION: ${{ secrets.PROD_GDRIVE_DESTINATION }}
```

`RCLONE_CONFIG` must contain a full `rclone.conf` with a remote named `gdrive` (OAuth-authorized; service accounts can't write to a personal Drive due to `storageQuotaExceeded`).
`BACKUP_GDRIVE_DESTINATION` is the repository-specific path under that remote, e.g. `minecraft/werewolf/server/prod/backups`. Callers can map this from environment-specific secrets such as `PROD_GDRIVE_DESTINATION` or `DEV_GDRIVE_DESTINATION`.
Set `skip-if-missing: true` for pre-deploy backups that should pass on the first deploy before the prod database exists.

## Web app release

Use `.github/workflows/web-app-release.yml` when an app should use the standard full release flow: build Docker images, back up prod before tag deploys, deploy the app stack, and deploy/reload the shared Caddy snippet.

```yaml
jobs:
  release:
    uses: kairo-js/github-workflows/.github/workflows/web-app-release.yml@v0.1.2
    with:
      app-name: werewolf
      image-prefix: mc-werewolf
      services: '[{"name":"backend","context":"./backend"},{"name":"frontend","context":"./frontend"}]'
      postgres-user: werewolf
      postgres-db: werewolf
      caddy-snippet-template-path: deploy/caddy/service.caddy
      acme-email: ${{ vars.ACME_EMAIL }}
      prod-backup-prefix: werewolf-prod-before-deploy
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
      DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
      DEPLOY_USER: ${{ secrets.DEPLOY_USER }}
      DEPLOY_SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
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
    uses: kairo-js/github-workflows/.github/workflows/minecraft-pack-release.yml@v0.1.2
    permissions:
      contents: write
    with:
      artifact-name: kairo
```
