# download-go-module-action

A GitHub Action and reusable workflow to download Go modules and upload them as artifacts, separating the download job that requires `id-token: write` from the build job.

## Motivation

We use [flatt-security/setup-takumi-guard-golang](https://github.com/flatt-security/setup-takumi-guard-golang), a GitHub Action that routes go mod download through a security proxy.
setup-takumi-guard-golang requires the `id-token: write` permission for authentication, but we should not grant the permission to the build job to prevent malicious codes from abusing the permission.
Otherwise, the malicious code can sign malicious tools and assume AWS IAM Roles or Google Service Accounts via OIDC.
To prevent this, this action downloads Go Modules and upload them as GitHub Artifacts.
Then the build job can download them.

## Example

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

      - name: Download Go Modules and upload them as GitHub Artifacts
        uses: suzuki-shunsuke/download-go-mdoule-action@main
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

      - name: Download Go modules and go-install packages (go-licenses)
        uses: actions/download-artifact@9000827ccba6bdab643e8b6fd33ac0654aef8333 # v8.0.2
        with:
          name: gomodule
          path: ${{ runner.temp }}/gomodule

      - name: Extract Go modules
        run: |
          gomodcache=$(go env GOMODCACHE)
          mkdir -p "$gomodcache"
          tar -C "$gomodcache" -xzf "$RUNNER_TEMP/gomodule/gomodcache.tar.gz"

      - name: go test
        run: go test -v ./... -race -covermode=atomic
```

### Reusable Workflow

```yaml
jobs:
  gomodule:
    uses: suzuki-shunsuke/download-go-mdoule-action/.github/workflows/download.yaml@main
    permissions:
      contents: read
      id-token: write
    with:
      takumi_guard_bot_id: ${{secrets.TAKUMI_GUARD_BOT_ID}}
```

### Install go-licenses with aqua

This action can install [some tools like google/go-liceses](actions/install/aqua/imports) via `go install`.

```yaml
jobs:
  gomodule:
    uses: suzuki-shunsuke/download-go-mdoule-action/.github/workflows/download.yaml@main
    permissions:
      contents: read
      id-token: write
    with:
      takumi_guard_bot_id: ${{secrets.TAKUMI_GUARD_BOT_ID}}
      aqua_version: v2.64.0
      aqua_packages: go-licenses

  test:
    timeout-minutes: 30
    runs-on: ubuntu-26.04
    needs: gomodule
    permissions:
      pull-requests: write
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

      - name: Download Go modules and go-install packages (go-licenses)
        uses: actions/download-artifact@9000827ccba6bdab643e8b6fd33ac0654aef8333 # v8.0.2
        with:
          name: gomodule
          path: ${{ runner.temp }}/gomodule

      - name: Extract Go modules
        run: |
          gomodcache=$(go env GOMODCACHE)
          mkdir -p "$gomodcache"
          tar -C "$gomodcache" -xzf "$RUNNER_TEMP/gomodule/gomodcache.tar.gz"

      - name: Extract Go install packages (go-licenses)
        run: |
          root_dir=$(aqua root-dir)
          mkdir -p "$root_dir"
          tar -C "$root_dir" -xzf "$RUNNER_TEMP/gomodule/go_install.tar.gz"
          
      - name: Install tools
        uses: aquaproj/aqua-installer@v4.1.0
        with:
          aqua_version: v2.64.0

      - name: go test
        run: go test -v ./... -race -covermode=atomic
```
