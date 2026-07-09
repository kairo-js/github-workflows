# github-workflows

Reusable GitHub Actions workflows for Kairo.js and related Minecraft Bedrock packs.

## Docker build & push

Use `.github/workflows/docker-build-push.yml` to build and push one or more Docker images to Docker Hub. Each service is pushed as `<DOCKERHUB_USERNAME>/<image-prefix>-<name>:<tag>`.

```yaml
jobs:
  build-and-push:
    uses: kairo-js/github-workflows/.github/workflows/docker-build-push.yml@main
    with:
      services: '[{"name":"backend","context":"./backend"},{"name":"frontend","context":"./frontend"}]'
      image-prefix: mc-werewolf
      tag: dev
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
```

## PostgreSQL backup

Use `.github/workflows/postgres-backup.yml` to `pg_dump` a PostgreSQL container running on a remote host over SSH, then upload the dump as both a workflow artifact and to Google Drive via `rclone`.

```yaml
jobs:
  backup:
    uses: kairo-js/github-workflows/.github/workflows/postgres-backup.yml@main
    with:
      container-name: ww-prod-postgres-1
      db-user: werewolf
      db-name: werewolf
      backup-prefix: werewolf-prod
      gdrive-destination: minecraft/werewolf/server/backups
    secrets:
      DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
      DEPLOY_USER: ${{ secrets.DEPLOY_USER }}
      DEPLOY_SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
      RCLONE_CONFIG: ${{ secrets.RCLONE_CONFIG }}
```

`RCLONE_CONFIG` must contain a full `rclone.conf` with a remote named `gdrive` (OAuth-authorized; service accounts can't write to a personal Drive due to `storageQuotaExceeded`).

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
    uses: kairo-js/github-workflows/.github/workflows/minecraft-pack-release.yml@main
    permissions:
      contents: write
    with:
      artifact-name: kairo
```
