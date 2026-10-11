# download-go-module-action

A GitHub Action and reusable workflow to download Go modules and upload them as artifacts, separating the download job that requires `id-token: write` from the build job.

[Reusable Workflow](.github/workflows/download.yaml), [install action](install/action.yaml), [download action](download/action.yaml)

## Motivation

We use [flatt-security/setup-takumi-guard-golang](https://github.com/flatt-security/setup-takumi-guard-golang), a GitHub Action that routes `go mod download` through a security proxy.
setup-takumi-guard-golang requires the `id-token: write` permission for authentication, but this permission should not be granted to the build job.
If it were, malicious code running in the build job could abuse it to sign malicious artifacts or assume AWS IAM roles or Google Cloud service accounts via OIDC.
To avoid this, this action downloads Go modules in a separate job and uploads them as artifacts, which the build job then downloads.

## Usage

### GitHub Action

```yaml
jobs:
  gomodule:
    timeout-minutes: 30
    runs-on: ubuntu-26.04
    permissions:
      contents: read
      id-token: write
    steps:
      - name: Checkout the repository
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false

      - name: Download Go modules and upload them as artifacts
        uses: suzuki-shunsuke/download-go-module-action@main
        with:
          takumi_guard_bot_id: ${{secrets.TAKUMI_GUARD_BOT_ID}}

  test:
    timeout-minutes: 30
    runs-on: ubuntu-26.04
    needs: gomodule
    permissions:
      contents: read
    env:
      # Use only the modules downloaded in the gomodule job.
      # Fail instead of silently downloading modules without Takumi Guard.
      GOPROXY: "off"
    steps:
      - name: Checkout the repository
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false

      - name: Set up Go
        uses: actions/setup-go@b7ad1dad31e06c5925ef5d2fc7ad053ef454303e # v7.0.0
        with:
          go-version-file: go.mod
          cache: false
          
      - name: Download Go modules
        uses: suzuki-shunsuke/download-go-module-action/download@main

      - name: go test
        run: go test -v ./... -race -covermode=atomic
```

### Reusable Workflow

```yaml
jobs:
  gomodule:
    uses: suzuki-shunsuke/download-go-module-action/.github/workflows/download.yaml@main
    permissions:
      contents: read
      id-token: write
    secrets:
      TAKUMI_GUARD_BOT_ID: ${{secrets.TAKUMI_GUARD_BOT_ID}}
```

### Install go-licenses with aqua

This action can also install tools such as [google/go-licenses](install/aqua/imports) via `go install`.

1. Set `aqua_version` and `aqua_packages` to [suzuki-shunsuke/download-go-module-action/.github/workflows/download.yaml](.github/workflows/download.yaml) or [suzuki-shunsuke/download-go-module-action](action.yaml)
2. Set `aqua_version` to [suzuki-shunsuke/download-go-module-action/download](download/action.yaml)

```yaml
jobs:
  gomodule:
    uses: suzuki-shunsuke/download-go-module-action/.github/workflows/download.yaml@main
    permissions:
      contents: read
      id-token: write
    with:
      aqua_version: v2.64.0
      aqua_packages: go-licenses
    secrets:
      TAKUMI_GUARD_BOT_ID: ${{secrets.TAKUMI_GUARD_BOT_ID}}

  test:
    timeout-minutes: 30
    runs-on: ubuntu-26.04
    needs: gomodule
    permissions:
      contents: read
    env:
      GOPROXY: "off"
    steps:
      - name: Checkout the repository
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false

      - name: Set up Go
        uses: actions/setup-go@b7ad1dad31e06c5925ef5d2fc7ad053ef454303e # v7.0.0
        with:
          go-version-file: go.mod
          cache: false
          
      - name: Download Go modules
        uses: suzuki-shunsuke/download-go-module-action/download@main
        with:
          aqua_version: v2.64.0

      - name: go test
        run: go test -v ./... -race -covermode=atomic

      - name: Validate Go licenses
        run: go-licenses check --disallowed_types=forbidden,restricted,unknown ./...
```
