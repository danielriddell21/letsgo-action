# letsgo-action

Run [letsgo](https://github.com/danielriddell21/letsgo) in GitHub Actions.

letsgo builds a Go project, refuses to ship it if it's wrong, publishes it
reproducibly, and lets anyone prove afterwards that the binaries match the
source. This action installs it and runs it.

## Usage

```yaml
name: release
on:
  push:
    tags: ["v*"]

permissions:
  contents: write      # create the release and upload assets
  id-token: write      # provenance attestation
  attestations: write
  packages: write      # only if you publish a container image

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with:
          go-version-file: go.mod

      - uses: danielriddell21/letsgo-action@v1
```

A shallow clone is fine. letsgo falls back to the GitHub compare API when the
changelog needs history the runner does not have.

### Checking a pull request

`plan` resolves and gates a release without building one, in about two seconds:

```yaml
      - uses: danielriddell21/letsgo-action@v1
        with:
          command: plan
          args: --explain
```

### Approving a release before it publishes

`plan` reads the forge and saves what a release would change; `apply`
publishes exactly that and nothing else. Put an
[environment](https://docs.github.com/actions/deployment/targeting-different-environments)
with required reviewers between them and the reviewer approves the plan
itself, shown in the job summary:

```yaml
on:
  push:
    tags: ["v*"]

jobs:
  plan:
    runs-on: ubuntu-latest
    permissions: { contents: read }
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0 }
      - uses: actions/setup-go@v7
        with: { go-version-file: go.mod }
      - uses: danielriddell21/letsgo-action@v1
        with: { command: plan }   # uploads release.plan, writes the job summary

  apply:
    needs: plan
    environment: release          # required reviewers approve here
    runs-on: ubuntu-latest        # a different runner from plan, on purpose
    permissions: { contents: write, id-token: write, attestations: write, packages: write }
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0 }
      - uses: actions/setup-go@v7
        with: { go-version-file: go.mod }
      - uses: danielriddell21/letsgo-action@v1
        with: { command: apply }  # downloads release.plan
```

The plan job needs only read access, so the approval gate is also a privilege
boundary. `apply` rebuilds the release and refuses to publish unless the
rebuild is the one the plan agreed, and unless the forge is still as the plan
found it.

Give `plan` or `apply` any `args` and the action runs them as written, with no
plan saved, uploaded or downloaded: `args: --explain` is still the two-second
pull request check above.

### Verifying a published release

```yaml
      - uses: danielriddell21/letsgo-action@v1
        with:
          command: verify
          args: v1.3.0
```

### Releasing modules in a monorepo

A module nested in a repository releases from tags named `<dir>/vX.Y.Z`. Run
the action from the module's directory and it scopes itself to that module:

```yaml
on:
  push:
    tags: ["*/v*"]

jobs:
  release:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        module: [services/api, services/worker]
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - uses: actions/setup-go@v7
        with:
          go-version-file: ${{ matrix.module }}/go.mod
      - uses: danielriddell21/letsgo-action@v1
        id: release
        # A tag names one module; every other matrix entry finds no release
        # of its own at this commit, so it fails and is skipped by the guard.
        if: startsWith(github.ref_name, matrix.module)
        with:
          working-directory: ${{ matrix.module }}
      - run: echo "released ${{ steps.release.outputs.tag }}"
        if: steps.release.outputs.tag != ''
```

### Promoting a prerelease

Flipping an RC from pre-release to release in the GitHub UI fires
`release: released`. This workflow turns that into a promotion: it rebuilds
the RC at its own commit and publishes it as the stable release.

```yaml
on:
  release:
    types: [released]

jobs:
  promote:
    # released also fires for ordinary stable releases; only a prerelease tag promotes.
    if: contains(github.event.release.tag_name, '-')
    runs-on: ubuntu-latest
    permissions: { contents: write, id-token: write, attestations: write }
    steps:
      - uses: actions/checkout@v7
        with: { ref: '${{ github.event.release.tag_name }}', fetch-tags: true }
      - uses: actions/setup-go@v7
        with: { go-version-file: go.mod }
      - uses: danielriddell21/letsgo-action@v1
        with:
          command: promote
          args: ${{ github.event.release.tag_name }}
```

Editing the release with the default `github.token` doesn't start workflows;
use an App token if the flip should trigger this one automatically. Either
way a re-trigger is a no-op: promote refuses once the stable tag already
exists.

### Installing letsgo without running it

```yaml
      - uses: danielriddell21/letsgo-action@v1
        with:
          command: ""
      - run: letsgo plan && letsgo build
```

## Inputs

| Input | Default | Description |
|---|---|---|
| `version` | `latest` | The letsgo release to install, or a tag such as `v0.8.0`. |
| `command` | `release` | `release`, `plan`, `apply`, `build`, `verify`, `diff`, `tag`, `promote`, `yank`, `audit`, `doctor`, `features`, or empty to install only. |
| `args` | `""` | Extra arguments, split on whitespace. |
| `working-directory` | `.` | Where to run. |
| `plan-file` | `release.plan` | Where `plan` saves the plan and `apply` reads it, relative to `working-directory`. |
| `plan-artifact` | `release.plan` | The artifact name `plan` uploads the plan under and `apply` downloads it from. |
| `token` | `github.token` | The forge token letsgo publishes with. |
| `tap-token` | `""` | The token a Homebrew formula is published with. Empty falls back to `token`. |

## Outputs

| Output | Description |
|---|---|
| `letsgo-version` | The letsgo version that ran. |
| `version` | The project version that was built or published. |
| `release-url` | The release page, when a release was published. |
| `manifest` | Path to the `letsgo.json` letsgo wrote. |
| `plan-file` | Path to the plan `plan` saved. |
| `tag` | The release's tag, with the module directory prefix for a nested module (`services/api/v1.2.3`). |

```yaml
      - uses: danielriddell21/letsgo-action@v1
        id: release
      - run: echo "published ${{ steps.release.outputs.release-url }}"
```

## Pinning the letsgo version

The default is `latest`, because the alternative was a release of this action
every time letsgo published one — and an action whose only change is a version
number teaches everyone to ignore its releases.

That default is a convenience, not a guarantee. letsgo decides the archive
layout and the linker flags, so it is a build input exactly as the compiler is,
and a release built by a version you did not choose is a release you cannot
reproduce. A repository that cares about that pins the tag:

```yaml
      - uses: danielriddell21/letsgo-action@v1
        with:
          version: v0.8.0
```

Bump that pin deliberately. `letsgo verify` replays a release from the manifest,
which records the version that built it, so a release stays checkable either
way — pinning is what makes the *next* one come out the same.

## How letsgo is installed

`go install github.com/danielriddell21/letsgo/cmd/letsgo@<version>`.

The module proxy checks what it fetches against the public checksum database,
which is a stronger guarantee than any digest this action could carry, and it
works on every runner without a platform matrix. The cost is a short compile;
letsgo has no dependencies, so it is short.

Go itself is a prerequisite rather than something this action installs: the Go
version changes the bytes of the binaries you ship, so choosing it belongs to
the workflow that knows which one your project releases with.

## Permissions

`contents: write` is the only one a plain release needs. Add `id-token` and
`attestations` for build provenance, and `packages: write` for a container
image.

A Homebrew tap lives in another repository, and the workflow token cannot write
to one. That needs an App token, passed as `tap-token` rather than as `token`:

```yaml
- uses: actions/create-github-app-token@v3
  id: tap-token
  with:
    app-id: ${{ vars.TAP_APP_ID }}
    private-key: ${{ secrets.TAP_APP_PRIVATE_KEY }}
    owner: ${{ github.repository_owner }}
    repositories: homebrew-tap

- uses: danielriddell21/letsgo-action@v1
  with:
    tap-token: ${{ steps.tap-token.outputs.token }}
```

Note what `repositories` does *not* list: the repository being released. Its
own release is published with the workflow token, so the App needs no
installation there — which also means a new repository cannot fail its first
release for having been left out of one. Passing the App token as `token`
instead works, and makes it a credential that can rewrite this repository's
releases too.
