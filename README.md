# Home Assistant workflows

Reusable GitHub Actions workflows for Home Assistant repositories.

## App workflows

The official Home Assistant apps, one repository per app, build and publish their images with two workflows:

- [`app-ci.yaml`](.github/workflows/app-ci.yaml) lints the app and builds its images without pushing them.
- [`app-deploy.yaml`](.github/workflows/app-deploy.yaml) builds, signs and pushes the images, then notifies the app index repository.

They build with the actions from [home-assistant/builder](https://github.com/home-assistant/builder), pinned to a builder release.

`app-deploy.yaml` only runs for repositories of the `home-assistant` organization.
Other projects can use the builder actions in their own workflows.

```yaml
jobs:
  ci:
    permissions:
      contents: read
      security-events: write
    uses: home-assistant/workflows/.github/workflows/app-ci.yaml@[version]

  deploy:
    needs: ci
    permissions:
      contents: read
      id-token: write
      packages: write
    uses: home-assistant/workflows/.github/workflows/app-deploy.yaml@[version]
    with:
      repository: apps
    secrets:
      DISPATCH_TOKEN: ${{ secrets.DISPATCH_TOKEN }}
```

- A push to `main` publishes images tagged `dev` and with the short commit SHA.
- A GitHub pre-release publishes images tagged with the version and `beta`.
- A GitHub release publishes images tagged with the version and `stable`.
  It then sends an `update` dispatch event to the app index repository, `home-assistant/apps` by default.
- `repository_edge` and `repository_beta` name optional edge and beta index repositories.
  Set `repository_edge: apps-dev` to notify the development index; its updater channel and GitHub environment remain `edge`, while the image tag is `dev`.

Rerunning deployment jobs rebuilds and republishes their images and signs them again, even when the tags already exist.
This allows recovery from a signing failure after an image was pushed.
Reruns also update the channel tags (`dev`, `beta` or `stable`), including when rerunning an older release.

## Release helpers

[`app-release-drafter.yaml`](.github/workflows/app-release-drafter.yaml) drafts app releases using the caller's `.github/release-drafter.yml`:

```yaml
on:
  push:
    branches: [main]

jobs:
  release-drafter:
    permissions:
      contents: write
      pull-requests: read
    uses: home-assistant/workflows/.github/workflows/app-release-drafter.yaml@[version]
```

The separate `release-drafter.yaml` workflow drafts this repository's own releases, tagged `v<major>.<minor>.<patch>`.

[`repository-updater.yaml`](.github/workflows/repository-updater.yaml) updates an app index using `hassio-addons/repository-updater` and the index's `.apps.yml`:

```yaml
on:
  repository_dispatch:
    types: [update]

permissions:
  contents: read

jobs:
  update:
    uses: home-assistant/workflows/.github/workflows/repository-updater.yaml@[version]
    secrets:
      UPDATER_TOKEN: ${{ secrets.UPDATER_TOKEN }}
```

The dispatch's `app` field selects an index entry by its key or source repository name.
`UPDATER_TOKEN` must belong to a user with a profile name, public email and permission to push to the index's default branch.
The updater selects the latest release or commit for the configured channel; it does not consume the dispatch's version field.
The wrapper retains upstream's concurrency behavior, which replaces an existing pending update when another arrives.

Both helpers call pinned actions directly; they do not call `hassio-addons/workflows`.

## Signature verification

The signature identity names `app-deploy.yaml` in this repository.
Any repository can call the workflow, so always check the app repository as well:

```sh
cosign verify ghcr.io/home-assistant/app-configurator:6.2.1 \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp '^https://github\.com/home-assistant/workflows/' \
  --certificate-github-workflow-repository home-assistant/app-configurator
```
