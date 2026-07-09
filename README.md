# github-workflows

Reusable GitHub Actions workflows for Kairo.js and related Minecraft Bedrock packs.

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
