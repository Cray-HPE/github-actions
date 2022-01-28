# Container Image Build Action

GitHub Action to build and push container images with [docker/metadata-action]
and [docker/build-push-action].

_Stable_ builds are triggered by a [push tag
event](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#push)
with a valid [SemVer] Git tag, e.g., `vX.Y.Z` which corresponds to version
`X.Y.Z`. All other builds are considered _unstable_. Note that:

* _Stable_ builds use image repositories specified in `images` and
  `stable-images` to generate tags. _Unstable_ builds use image repositories
  specified in `images` and `unstable-images` to generate tags.

* The default `tags` settings generate image tags for:

    * _Stable_ images based on corresponding SemVer Git tags, e.g., `X.Y.Z`,
      `X.Y`, and `X` (unless `X` is `0`) as well as `latest`.

    * _Unstable_ images according to the event type, e.g.:

        * A [pull request event] for `refs/pull/2/merge` gets tag `pr-2`
        * A [push event] to `refs/heads/feature/branch` gets tag
          `feature-branch`

    * All images based on Git commit, e.g., `sha-`.

* Distinct image references pinned to digests are available in the `refs`
  output. They are suitable image references for signing and scanning as shown
  in the example.


## Example

```yaml
name: Build, sign, and scanimage

on:
  push:
    branches:
      - '**'
    tags:
      - 'v*.*.*'
  pull_request:
    branches:
      - main
      - release/**
  workflow_dispatch:

env:
  registry: artifactory.algol60.net/csm-docker
  image-name: my-image

jobs:
  build:
    name: Build and push images

    runs-on: ubuntu-latest

    permissions:
      contents: read

    outputs:
      refs: ${{ steps.build.outputs.refs }}

    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Login to Registry
        uses: docker/login-action@v1
        with:
          registry: ${{ env.registry }}
          username: ${{ secrets.ARTIFACTORY_ALGOL60_USERNAME }}
          password: ${{ secrets.ARTIFACTORY_ALGOL60_TOKEN }}

      - uses: Cray-HPE/github-actions/setup-docker-buildx

      - name: Build image
        id: build
        uses: Cray-HPE/github-actions/container-images/build@main
        with:
          stable-images: ${{ env.registry }}/stable/${{ env.image-name }}
          unstable-images: ${{ env.registry }}/unstable/${{ env.image-name }}

  sign:
    name: Sign images

    needs: build
    runs-on: ubuntu-latest

    permissions:
      id-token: write

    steps:
      - uses: Cray-HPE/github-actions/setup-cosign

      - name: Sign image
        uses: Cray-HPE/github-actions/container-images/sign
        with:
          images: ${{ join(fromJSON(needs.build.outputs.refs), '\n') }}

  trivy:
    name: Scan image for fixable vulnerabilities

    needs: build
    runs-on: ubuntu-latest

    permissions:
      id-token: write
      security-events: write

    steps:
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ fromJSON(needs.build.outputs.refs)[0] }}
          exit-code: '1'
          ignore-unfixed: true
          format: template
          template: '@/contrib/sarif.tpl'
          output: trivy-results.sarif

      - name: Upload results to GitHub Code Scanning
        uses: github/codeql-action/upload-sarif@v1
        if: ${{ always() && hashFiles('trivy-results.sarif') != ''}}
        with:
          sarif_file: trivy-results.sarif
```


## Inputs

> `List` type is a newline-delimited string

> `CSV` type is a comma-delimited string

| Name         | Type      | Default                                     | Description                                                                                                                                |
| ------------ | --------- | --------------------------------------      | --------------                                                                                                                             |
| `images`     | List      |                                             | List of image repositories to use as base name for generated tags                                                                          |
| `registry`   | String    | `artifactory.algol60.net/csm-docker`        | Image registry                                                                                                                             |
| `stable`     | Boolean   | _auto_                                      | Indicates image _stability_; by default stable images require [SemVer] Git tag                                                             |
| `name`       | String    |                                             | Image name, if given generate image repository and append to `images`                                                                      |
| `tags`       | List      | (See [action.yml])                          | List of [tags](https://github.com/docker/metadata-action#tags-input) as key-value pair attributes, always generates `type=sha,format=long` |
| `flavor`     | List      |                                             | [Flavor](https://github.com/docker/metadata-action#flavor-input) to apply                                                                  |
| `vendor`     | String    | `Hewlett Packard Enterprise Development LP` | Value of `org.opencontainers.image.vendor` label                                                                                           |

Inputs passed through to [docker/build-push-action]:

| Name              | Type       | Description                                                                                                                                                                       |
| ----------------- | ---------- | --------------                                                                                                                                                                    |
| `allow`           | List/CSV   | List of [extra privileged entitlement](https://github.com/docker/buildx/blob/master/docs/reference/buildx_build.md#allow) (e.g., `network.host,security.insecure`)                |
| `builder`         | String     | Builder instance (see [setup-buildx](https://github.com/docker/setup-buildx-action) action)                                                                                       |
| `build-args`      | List       | List of build-time variables                                                                                                                                                      |
| `cache-from`      | List       | List of [external cache sources](https://github.com/docker/buildx/blob/master/docs/reference/buildx_build.md#cache-from) (e.g., `type=local,src=path/to/dir`)                     |
| `cache-to`        | List       | List of [cache export destinations](https://github.com/docker/buildx/blob/master/docs/reference/buildx_build.md#cache-to) (e.g., `type=local,dest=path/to/dir`)                   |
| `cgroup-parent`   | String     | Optional [parent cgroup](https://docs.docker.com/engine/reference/commandline/build/#use-a-custom-parent-cgroup---cgroup-parent) for the container used in the build              |
| `context`         | String     | Build's context is the set of files located in the specified [`PATH` or `URL`](https://docs.docker.com/engine/reference/commandline/build/) (default [Git context](#git-context)) |
| `file`            | String     | Path to the Dockerfile. (default `{context}/Dockerfile`)                                                                                                                          |
| `labels`          | List       | List of metadata for an image                                                                                                                                                     |
| `load`            | Boolean    | [Load](https://github.com/docker/buildx/blob/master/docs/reference/buildx_build.md#load) is a shorthand for `--output=type=docker` (default `false`)                              |
| `network`         | String     | Set the networking mode for the `RUN` instructions during build                                                                                                                   |
| `no-cache`        | Boolean    | Do not use cache when building the image (default `false`)                                                                                                                        |
| `outputs`         | List       | List of [output destinations](https://github.com/docker/buildx/blob/master/docs/reference/buildx_build.md#output) (format: `type=local,dest=path`)                                |
| `platforms`       | List/CSV   | List of [target platforms](https://github.com/docker/buildx/blob/master/docs/reference/buildx_build.md#platform) for build                                                        |
| `pull`            | Boolean    | Always attempt to pull a newer version of the image (default `false`)                                                                                                             |
| `push`            | Boolean    | [Push](https://github.com/docker/buildx/blob/master/docs/reference/buildx_build.md#push) is a shorthand for `--output=type=registry` (default `false`)                            |
| `secrets`         | List       | List of secrets to expose to the build (e.g., `key=string`, `GIT_AUTH_TOKEN=mytoken`)                                                                                             |
| `secret-files`    | List       | List of secret files to expose to the build (e.g., `key=filename`, `MY_SECRET=./secret.txt`)                                                                                      |
| `shm-size`        | String     | Size of [`/dev/shm`](https://github.com/docker/buildx/blob/master/docs/reference/buildx_build.md#-size-of-devshm---shm-size) (e.g., `2g`)                                         |
| `ssh`             | List       | List of SSH agent socket or keys to expose to the build                                                                                                                           |
| `target`          | String     | Sets the target stage to build                                                                                                                                                    |
| `ulimit`          | List       | [Ulimit](https://github.com/docker/buildx/blob/master/docs/reference/buildx_build.md#-set-ulimits---ulimit) options (e.g., `nofile=1024:1024`)                                    |
| `github-token`    | String     | GitHub Token used to authenticate against a repository for Git context (default `${{ github.token }}`)                                                                            |


## Outputs

> `List` type is a newline-delimited string

> `JSON` type is a JSON-encoded object that can be decoded with `fromJSON()`

| Name            | Type      | Description                                                     |
| --------------- | --------- | --------------------------------------------------------------- |
| `stable`        | Boolean   | Indicates if build was stable                                   |
| `images`        | List      | List of image repositories used as base name for generated tags |
| `refs`          | JSON      | Array of image refs pinned to digest                            |

Outputs from [docker/metadata-action]:

| Name            | Type      | Description                     |
| --------------- | --------- | ------------------------------- |
| `version`       | String    | Generated image version         |
| `tags`          | List      | List of generated image tags    |
| `labels`        | List      | List of generated image labels  |
| `json`          | JSON      | JSON output of tags and labels  |

Outputs from [docker/build-push-action]:

| Name            | Type      | Description                                               |
| --------------- | --------- | --------------------------------------------------------- |
| `digest`        | String    | Image content-addressable identifier also called a digest |
| `metadata`      | JSON      | Build result metadata                                     |


[docker/metadata-action]: https://github.com/docker/metadata-action/
[docker/build-push-action]: https://github.com/docker/build-push-action/
[SemVer]: https://semver.org/
[pull request event]: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#pull_request
[push event]: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#push
[schedule event]: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule
