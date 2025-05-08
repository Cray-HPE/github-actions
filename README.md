# Reusable Github Actions and Workflows

## Build Sign Scan Reusable Workflow

### Location

Build, Sign and Scan reusable workflow is located in `.github/workflows//build-sign-scan.yaml` file.

### Features

- Container image builds via provided `Makefile` or without it (i.e. only `Dockerfle` is present).
- Multi-platform builds (via `docker_platform` parameter).
- Push into generic private registry (basic authentication) or Google Artifact Registry (OIDC auth).
- Scan with Snyk.
- Scan with Trivy.
- Signing with Sigstore Cosign using key stored in Google KMS.

### Requirements
Reusable workflow follows dry-out configuration approach. All input parameters are optional and have reasonable defaults.
The only mandatory parameter is `docker_tag` (if no `Makefile` is provided) or `make_target` (if `Makefile` is provided).
Also, access to `secrets` context is required. Please refer to workflow file for detailed description of input parameters.

### Minimal Configuration

- If `Makefile` is provided:

      jobs:
        build-sign-scan:
          uses: Cray-HPE/.github/workflows//build-sign-scan.yaml@build-sign-scan-workflow/v1
          with:
            make_target: ${{ github.ref_type == 'tag' && 'stable' || 'unstable' }}
          secrets: inherit

    NOTE: `Makefle` must honor `DOCKER_BUILD_ARGS` environment variable.

- If `Makefile` is not provided:

      jobs:
        build-sign-scan:
          uses: Cray-HPE/.github/workflows//build-sign-scan.yaml@build-sign-scan-workflow/v1
          with:
            docker_tag: artifactory.algol60.net/csm-docker/${{ github.ref_type == 'tag' && 'stable' || 'unstable' }}/my-image:tag
          secrets: inherit

In these examples, `Makefile` target `stable` is called when build is invoked on git tag, otherwise `unstable` target is called (i.e. tag-based release strategy is used).

### Secrets in Build Context

Workflow accepts `docker_secrets` input secret as new-line separated list of `key=value` secret pairs, to be made available during build time. Each `key=value` pair is injected into build as environment variable named as `<key>`, with value set to `<value>`.

- If `Makefile` is used, it should provide `--secret id=<key>,env=<key>` command line option(s) for `docker buildx build` command to mount secrets during build time.
- If `Makefile` is not used, `--secret id=<key>,env=<key>` command line option is added to `docker buildx build` command automatically for each secret.

Each secret will be available in build context as a file named `/run/secrets/<key>`.

#### NOTE 1: 
Multiline secrets are not supported (will be converted to space-separated secrets).

#### NOTE 2:
Reusable workflow syntax does not allow mixing implicitly specified secrets (`inherit` keyword) and exlicitly specified secrets. Therefore all secrets needed by image build, push, sign and scan will need to be provided, if `docker_secrets` is used.

#### Example:

    jobs:
      build-sign-scan:
        uses: Cray-HPE/.github/workflows//build-sign-scan.yaml@build-sign-scan-workflow/v1
        with:
          docker_tag: artifactory.algol60.net/csm-docker/${{ github.ref_type == 'tag' && 'stable' || 'unstable' }}/my-image:tag
        secrets:
          docker_username: ${{ secrets.ARTIFACTORY_ALGOL60_USERNAME }}
          docker_password: ${{ secrets.ARTIFACTORY_ALGOL60_TOKEN }}
          gcp_workload_identity_provider: ${{ secrets.COSIGN_GCP_WORKLOAD_IDENTITY_PROVIDER_RSA }}
          gcp_service_account: ${{ secrets.COSIGN_GCP_SERVICE_ACCOUNT_RSA }}
          gcp_cosign_key: ${{ secrets.COSIGN_KEY_RSA }}
          snyk_token: ${{ secrets.SNYK_TOKEN }}
          docker_secrets: |
            SECRET_1="${{ secrets.ORG_SECRET_1 }}"
            SECRET_2="${{ secrets.ORG_SECRET_2 }}"
