# Composite Actions deploying Hugo sites to Azure

Composite GitHub Actions for Hugo projects deploying to Azure

1. generate-metadata
1. build
1. deploy

## Example workflow invoking these Acitons

I am utilizing [release-please-action](https://github.com/google-github-actions/release-please-action)
from Google to handle semantic versioning and cutting
releases via PRs in repos where these hugo/azure pipelines are used. It's also used within this repo `hugo-azure-actions`
to generate Releases that follow the traditional GitHub Actions semver format where you can specify
@major, @major.minor or @major.minor.patch when invoking each of them.
See available [releases](https://github.com/kevholmes/hugo-azure-actions/releases)
in this repo to see what the latest major semver to reference is when invoking these Actions in your own repos.

This is a relatively simple process that upon a PR being opened or having a `synchronize` event
which could be an event like push to an open PR will cause our CD process to trigger.

This process is based on GitHub PR "flow" and we will deploy to `dev` for a new PR being opened. When
at least one of these PRs (following conventional commits ideally) is then merged, Release Please
will generate a semantic version for us based on the PR title and a corresponding PR describing the included
changes through auto-generated release notes. That PR for our release will be built and deployed to the `stg`
or staging environment. This build is a combination of at least one PR that has been merged into the default branch.
Once the Release PR is merged we then auto-generate a Release in GitHub and corresponding Semantic Version tag(s).
This triggers a release to the `prod` environment.

The two main values to really tweak here if you just want to follow the above process are:

- `hugo-version`
- `site-base-tld`

```yaml
name: cd

on:
  pull_request:
    types: [ synchronize, opened ]
  release:
    types:
      - released

defaults:
  run:
    shell: bash

concurrency: cd

jobs:
  gen-metadata:
    runs-on: ubuntu-latest
    outputs:
      build-id: ${{ steps.build-metadata.outputs.artifact-id }}
      build-domain: ${{ steps.build-metadata.outputs.composite-domain }}
      env-target: ${{ steps.build-metadata.outputs.env-target }}
      hugo-version: ${{ steps.build-metadata.outputs.hugo-version }}
    steps:
      - name: generate build metadata
        id: build-metadata
        uses: kevholmes/hugo-azure-actions/.github/actions/generate-metadata@v1
        with:
          hugo-version: 0.118.2
          site-base-tld: cloverstack.dev
  build:
    runs-on: ubuntu-latest
    needs: gen-metadata
    environment:
      name: ${{ needs.gen-metadata.outputs.env-target }}
      url: ${{ needs.gen-metadata.outputs.build-domain }}
    steps:
      - name: checkout source
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - name: build hugo site
        uses: kevholmes/hugo-azure-actions/.github/actions/build@v1
        with:
          build-id: ${{ needs.gen-metadata.outputs.build-id }}
          build-domain: ${{ needs.gen-metadata.outputs.build-domain }}
          hugo-version: ${{ needs.gen-metadata.outputs.hugo-version }}
  deploy-azure:
    runs-on: ubuntu-latest
    needs: [gen-metadata, build]
    environment:
      name: ${{ needs.gen-metadata.outputs.env-target }}
      url: ${{ needs.gen-metadata.outputs.build-domain }}
    steps:
      - name: deploy hugo site to azure
        uses: kevholmes/hugo-azure-actions/.github/actions/deploy@v1
        with:
          build-id: ${{ needs.gen-metadata.outputs.build-id }}
          az-client-id: ${{ vars.CLIENT_ID }}
          az-client-secret: ${{ secrets.CLIENT_SECRET }}
          az-subscription-id: ${{ vars.SUBSCRIPTION_ID }}
          az-tenant-id: ${{ vars.TENANT_ID }}
          az-storage-acct: ${{ vars.AZ_STORAGE_ACCT }}
          az-cdn-profile-name: ${{ vars.AZ_CDN_PROFILE_NAME }}
          az-cdn-endpoint: ${{ vars.AZ_CDN_ENDPOINT }}
          az-resource-group: ${{ vars.AZ_RESOURCE_GROUP }}

```
