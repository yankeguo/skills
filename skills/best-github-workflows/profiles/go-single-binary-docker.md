# Profile: go-single-binary-docker

id: `go-single-binary-docker`

One Go static binary, and one Docker image. Four workflow files split tests, `:latest` on the default branch, and the semver image plus GitHub Release. A failing test cannot publish.

This document is enough to implement the workflows. When writing them, compress the reason from the matching section into that file's header. A header that only says "see the skill" is not enough: the next person who opens the YAML has to see why the choice cannot be reverted.

The reference implementation is [yankeguo/airmx](https://github.com/yankeguo/airmx) `.github/workflows/`. The repository name, binary name, and `-X` symbol there are parameters, not literals to copy into another repository.

## Applicability

Use this profile only when all of these are true:

- The repository root has one `package main`, and `go build .` produces the release binary.
- `CGO_ENABLED=0` cross-compiles linux, darwin, and windows from `ubuntu-latest`.
- One image is pushed to `ghcr.io/<owner>/<repo>`. The runtime image is Alpine and contains that binary.
- The default branch is a literal name in the trigger. The reference uses `main`.
- Version tags are `v`-prefixed semver. Pre-release suffixes use `-` (`v1.2.3-rc.1`). Build metadata (`+`) is not used.

Do not use this profile when the module needs cgo, when macOS or Windows runners are required, when the root is not the only main package, when there is no image or no release binary, when more than one image is pushed, or when a monorepo has more than one release line.

## Parameters

| Parameter | Reference value | Where it is written |
| --- | --- | --- |
| Binary name | `airmx` | Archive name, the file inside the archive, artifact-name prefix |
| Version ldflag | `main.assetVersion` | `-X` on the binary and in the image. In the reference this value is the cache-bust query on static asset URLs. If the program has no such variable, drop `-X` and leave the other build flags |
| Cache scope | `airmx` | The fixed string on `cache-from` / `cache-to`. Use the repository's short name, not the git ref |
| Test concurrency prefix | `airmx-test` | Must be distinct from `docker-` and `release-` |
| Default branch | `main` | The same literal in `docker.yml` `on.push.branches` and in `test.yml` `branches-ignore` |
| Image | `ghcr.io/${{ github.repository }}` | `docker/metadata-action` lowercases the image name |

Examples below use `{binary}` and `{ldflag}` for the first two rows. Replace them. Where a block is copied from the reference and still says `airmx` or `main`, replace those with the parameters too.

## Four files, one path per event

| File | What starts it | What it does |
| --- | --- | --- |
| `test.yml` | `pull_request`; push to any branch except the default; `workflow_call` from the other two files | `go test ./...` |
| `docker.yml` | Push to the default branch | Call `test.yml`, then `docker-publish.yml` |
| `release.yml` | Push of a `v*` semver tag | Validate the version, call `test.yml`, then run the image and the binaries in parallel, then create the GitHub Release |
| `docker-publish.yml` | `workflow_call` only | The only place image tags and the image build are defined |

`docker-publish.yml` is called by `docker.yml` (push to the default branch) and by `release.yml` (semver tag). It publishes `ghcr.io/<owner>/<repo>` for whichever ref called it. A `workflow_call`-only file does not appear as its own button in the Actions tab. The tag rules are not copied into `release.yml`. The two publish entry points must not drift.

One test run per event, and a failing suite cannot publish:

- The default branch is `branches-ignore`'d in `test.yml` on purpose. A push there already runs `docker.yml`, which calls `test.yml` before it publishes. Listing the default branch here too would run the suite twice on every default-branch push.
- Tag pushes do not match `on.push.branches` (a tag ref is not a branch), so they do not start `test.yml` directly. `release.yml` calls it.
- `on.workflow_call` filters are not applied when another workflow calls this file. `branches-ignore` does not block `docker.yml` or `release.yml`.

Pull requests into any base branch are tested, not only pull requests into the default branch. There is no `paths` filter. The reference repository is small, and a change under an internal package or a workflow edit can both affect what ships.

Publish stays in the same run as the test: `docker.yml` and `release.yml` call `test.yml` with `needs`. `workflow_run` was rejected as the gate. `workflow_run` reads the workflow file from the default branch only, the checkout SHA is `github.event.workflow_run.head_sha` rather than `github.sha`, and the token permissions are awkward. `workflow_call` keeps the gate in the same run as the publish.

None of the four files defines `workflow_dispatch`.

- A manual docker run would republish `:latest` from whatever ref the dispatch used, which is easy to fire against the wrong commit.
- A manual release run would publish whatever tag filter happened to match, from an arbitrary ref.

## Permissions

The workflow-level `permissions` block is the ceiling for that file. A job cannot grant a permission the workflow omitted, and a reusable workflow cannot exceed the caller's grant. List every permission either job needs at workflow level, and narrow on the job.

Job-level `permissions` replace the workflow-level set for that job. They do not merge. Repeat `contents: read` on the test job even though the workflow already has it, and do not leave `packages: write` on the test job.

`test.yml` grants only:

```yaml
permissions:
  contents: read
```

That is enough to check out the repository and run tests. Publish permissions are not granted here. A reusable workflow can only narrow the caller's `GITHUB_TOKEN` permissions, never widen them, so `docker.yml` and `release.yml` must grant `contents: read` on the job that calls this file. `secrets.GITHUB_TOKEN` is already available inside a called workflow. `secrets: inherit` is not required.

`docker.yml` ceiling:

```yaml
permissions:
  contents: read
  packages: write
  attestations: write
  id-token: write
```

- test job: `contents: read`
- publish job: `contents: read`, `packages: write`, `id-token: write`, `attestations: write`

`packages: write` is the GHCR push. `id-token: write` and `attestations: write` are required because `docker/build-push-action` provenance attestations are on by default. Dropping those two permissions fails the push at attestation time, not at image build time.

`release.yml` uses the same ceiling with `contents: write` instead of `contents: read` (`contents: write` is what creating a release needs; `packages`, `id-token`, and `attestations` are what `docker-publish.yml` needs). Each job narrows:

| Job | Permissions |
| --- | --- |
| `version` | The reference sets no job-level `permissions`. The job does not check out and its steps do not use the token. Omitting the key leaves this job on the workflow ceiling |
| `test` | `contents: read` |
| `docker` | `contents: read`, `packages: write`, `attestations: write`, `id-token: write` |
| `binaries` | `contents: read` |
| `release` | `contents: write` |

`docker-publish.yml` declares the same four permissions at workflow level (`contents: read`, `packages: write`, `attestations: write`, `id-token: write`). A called workflow cannot widen the caller. Write the four permissions on the calling job and again on `docker-publish.yml`.

## Concurrency

The group name is a hardcoded prefix plus the ref. Do not use `github.workflow` in a reusable workflow's concurrency group. Inside `workflow_call` that context is the caller's workflow name. If the resulting string equals the caller's own concurrency group, GitHub cancels the run with `deadlock for concurrency group ... between top level workflow and ...`. The caller's groups are `docker-<ref>` and `release-<ref>`, so `<test-prefix>-<ref>` does not collide.

| File | Group | `cancel-in-progress` |
| --- | --- | --- |
| `test.yml` | `<test-prefix>-${{ github.ref }}` | `true` |
| `docker.yml` | `docker-${{ github.ref }}` | `true` |
| `release.yml` | `release-${{ github.ref }}` | `false` |

`cancel-in-progress: true` drops a stale run when a new commit lands on the same ref (a pull request synchronize, or another push to the same branch). On the default branch, a newer commit replaces an in-progress `:latest` build. `docker.yml`'s group is deliberately not the same string as `test.yml`'s group.

A called workflow's concurrency applies to the caller run, which then occupies both groups. That is wanted for the default branch: a newer push cancels the in-progress docker run. It has a side effect for tags. `release.yml` sets `cancel-in-progress` false so this file's own group does not cancel a tag build, but `test.yml` sets it true, so force-pushing the same tag can still cancel the in-progress release via the test group. Normal tag pushes use a unique ref and do not collide. Document that caveat in the `test.yml` header. Do not remove the test concurrency, and do not set the test group to `cancel-in-progress: false`, to "fix" it.

## Action versions

On 2026-09-23 GitHub removed Node 20 from hosted runners, so older majors of these actions no longer run. Majors float on purpose: the reference workflows used floating majors, and patch releases of an action should apply. Full commit SHAs are not pinned.

| Use | Version |
| --- | --- |
| checkout | `actions/checkout@v7` |
| setup-go | `actions/setup-go@v7` |
| QEMU | `docker/setup-qemu-action@v4` |
| Buildx | `docker/setup-buildx-action@v4` |
| GHCR login | `docker/login-action@v4` |
| Image metadata | `docker/metadata-action@v6` |
| Build and push | `docker/build-push-action@v7` |
| Upload artifact | `actions/upload-artifact@v7` |
| Download artifact | `actions/download-artifact@v8` |
| GitHub Release | `softprops/action-gh-release@v3` |

`actions/checkout@v7` and `actions/setup-go@v7` are the Node 24 majors. `docker/metadata-action` v5 and `docker/build-push-action` v6 predate the Node 20 removal; this profile uses metadata-action v6 and build-push-action v7. `upload-artifact` and `download-artifact` do not share a major version. Both are the Node 24 line. `softprops/action-gh-release` v2 is the last Node 20 release and is unmaintained.

When a later Node line is removed, bump the affected majors together and record the date and the reason in the file header.

`go-version-file: go.mod` reads the `go` line (the reference is `1.26.x`) so the toolchain is not pinned a second time in YAML. Do not also set `go-version`. Module and build caching stay at setup-go's default, which is on.

## Timeouts and shell

| Job | `timeout-minutes` |
| --- | --- |
| test | 15 |
| docker publish | 45 |
| version | 5 |
| binaries | 15 |
| release | 10 |

`timeout-minutes: 45` on the publish job because the arm64 leg runs under QEMU and is the slow part. Test and release jobs use the shorter limits. The test suite itself is seconds; 15 minutes bounds a hung test.

Shell steps use `set -euo pipefail`. Pass the ref and the tag list in through the environment rather than interpolating them into the script, so a multiline tag list is not shell-quoted by accident.

## test.yml

```yaml
name: test

on:
  workflow_call:
  pull_request:
  push:
    branches-ignore:
      - main

permissions:
  contents: read

# The prefix must stay distinct from the docker.yml and release.yml groups.
concurrency:
  group: airmx-test-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - name: Checkout
        uses: actions/checkout@v7

      - name: Setup Go
        uses: actions/setup-go@v7
        with:
          go-version-file: go.mod

      - name: Test
        run: go test ./...
```

Replace the concurrency prefix and the `branches-ignore` branch with the repository parameters.

The suite is `go test ./...` with the race detector left off. `golangci-lint` and `go vet` are not separate steps. The CI this profile defines is the test suite. In the reference repository, `-race` stays off because `internal/push` `TestNotifyDeliversAndPrunes` already fails under `-race` (a data race in webpush-go's send buffer, and the test handler's unsynchronized counter). Turning on `-race` would make CI red without a product change. That failure is specific to the reference; it is not a dependency of this profile. Do not add `-race` while applying the profile. Add it only after the suite is clean under it.

## docker.yml

The trigger is the literal default-branch name, not "whatever the default branch is". GitHub evaluates `on.push.branches` before the event payload exists, so `github.event.repository.default_branch` cannot be used in the trigger. Running on every branch and skipping with an `if` was rejected: the workflow would still be scheduled for feature branches.

The second guard is inside `docker-publish.yml` (`enable={{is_default_branch}}`). If the literal branch is no longer the default branch, that expression is false, no tag is produced, and the publish workflow fails on purpose instead of pushing `:latest` from a non-default branch.

This file does not run on pull requests or on other branches. Those only run tests (`test.yml`). It does not run on tags either. `release.yml` is the tag entry point and calls the same publish workflow. A tag pointing at the default branch therefore does not publish `:latest` by itself. `:latest` moves when the default branch moves. Shipping a release tag and updating the default branch are two pushes and two intended results.

There is no commit-SHA tag and no per-branch tag. The reference's previous single workflow used `type=sha` and `type=ref,event=branch` on every push. Those were removed because the default branch publishes only `:latest`, and tags publish only semver.

```yaml
name: docker

on:
  push:
    branches:
      - main

permissions:
  contents: read
  packages: write
  attestations: write
  id-token: write

concurrency:
  group: docker-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # Job-level permissions replace the workflow-level set for this job; they
  # do not merge. Repeat contents: read even though the workflow already has
  # it, and do not inherit packages: write into the test job.
  test:
    permissions:
      contents: read
    uses: ./.github/workflows/test.yml

  publish:
    needs: test
    permissions:
      contents: read
      packages: write
      attestations: write
      id-token: write
    uses: ./.github/workflows/docker-publish.yml
```

## Image tags

Tag policy lives only in `docker-publish.yml`. Keep the README "Continuous integration" table in sync, and keep the GitHub Release body in sync with the same rules.

`metadata-action`'s `{{version}}` expression strips the leading `v` and keeps a pre-release suffix, so `v1.2.3-rc.1` becomes the image tag `1.2.3-rc.1` rather than `1.2.3`.

| Event | Image tags |
| --- | --- |
| Push to the default branch | `latest` |
| Tag `v1.2.3` | `1.2.3`, `1.2`, `1` |
| Tag `v1.2.3-rc.1` (any pre-release, including `-rc`, `-beta`, `-alpha`, `-rc1`, `-beta.2`, `-alpha.1`) | `1.2.3-rc.1` |
| Tag `v0.2.0` | `0.2.0`, `0.2` |
| Tag `v0.0.1` | `0.0.1` |

No `type=sha` and no `type=ref`. Commit SHAs and branch names are not image tags.

`:latest` tracks the default branch only. A release tag must not move it, including a stable `v1.2.3` and a pre-release. Two settings together do that:

1. `flavor: latest=false`. The default flavor is `latest=auto`, and auto adds `:latest` for `type=semver` (and for `type=ref,event=tag`, `type=pep440`, `type=match`). Leaving the default on would retag `:latest` on every semver push. Auto does not add `:latest` just because the event is a branch push, so `latest=false` alone would also stop the default branch from publishing `:latest`.
2. `type=raw,value=latest,enable={{is_default_branch}}`. This is the explicit `:latest`. `{{is_default_branch}}` is a metadata-action Handlebars expression, evaluated by the action. It is true for a push to the repository's default branch and false for a tag event, so a tag push does not take this raw tag either. GitHub's `${{ }}` expressions and the action's `{{ }}` expressions are different languages. Both appear in `tags:` on purpose.

`{{major}}.{{minor}}` and `{{major}}` are moving channels (`1.2` and `1`). They are omitted when the channel itself would be only a leading zero:

| Tag | Major channel | Minor channel | Still published |
| --- | --- | --- | --- |
| `v0.2.0` | `0`, disabled | `0.2`, kept | `0.2.0` |
| `v0.0.1` | `0`, disabled | `0.0`, disabled | `0.0.1`, from `{{version}}` |
| `v0.10.0` | `0`, disabled | `0.10`, kept. The ref is `refs/tags/v0.10.0`, which does not have the prefix `refs/tags/v0.0.` | `0.10.0`, `0.10` |

The enable flags are the pattern from metadata-action's "Major version zero" note (semver `0.y.z` is unstable, so the Docker tag `0` should not exist), extended one level to `0.0`. The GitHub expressions are evaluated before the action sees the input, and the action expects the literal strings `true` or `false`:

```yaml
enable=${{ !startsWith(github.ref, 'refs/tags/v0.') }}
enable=${{ !startsWith(github.ref, 'refs/tags/v0.0.') }}
```

The prefix includes the `v` because this profile's tags are `v`-prefixed (`release.yml`'s filter rejects a bare `0.1.2`). On a branch push the ref is `refs/heads/<branch>`, both enables are true, and that is harmless: `type=semver` emits nothing unless the event is a tag.

Pre-release tags do not move the floating channels for a second reason. metadata-action, given a pre-release, does not render `{{major}}` as `1`. It renders the full pre-release string (`1.2.3-rc.1`) and then dedupes that with the `{{version}}` tag. The result is a single image tag, the full version. Disabling the zero channels is still required for stable `0.x` versions, where major and minor really would be `0` and `0.0`.

```yaml
- name: Docker metadata
  id: meta
  uses: docker/metadata-action@v6
  with:
    images: ghcr.io/${{ github.repository }}
    flavor: |
      latest=false
    tags: |
      type=raw,value=latest,enable={{is_default_branch}}
      type=semver,pattern={{version}}
      type=semver,pattern={{major}}.{{minor}},enable=${{ !startsWith(github.ref, 'refs/tags/v0.0.') }}
      type=semver,pattern={{major}},enable=${{ !startsWith(github.ref, 'refs/tags/v0.') }}
```

`latest=false` plus the raw tag, the semver patterns, and the two enable flags are the whole tag policy. Do not "simplify" `latest` back to `auto`, and do not drop the `v0` / `v0.0` enables. Do not add `type=sha` or `type=ref` back.

The "Require image tags" step runs before the push. If metadata-action emits an empty tag list (a tag shape it cannot parse, or a branch that is not the default), `build-push-action` either errors opaquely or pushes an untagged manifest. The step prints the tags so the log shows what was pushed.

```yaml
- name: Require image tags
  env:
    TAGS: ${{ steps.meta.outputs.tags }}
  run: |
    set -euo pipefail
    if [ -z "${TAGS}" ]; then
      echo "::error::no Docker tags generated for ${GITHUB_REF}"
      exit 1
    fi
    printf '%s\n' "${TAGS}"
```

## Image build

Platforms are `linux/amd64` and `linux/arm64` only. The image is Alpine. macOS and Windows binaries are release archives, not images.

`setup-qemu-action` runs before Buildx. Hosted runners are amd64 and cannot execute arm64 without binfmt. The reference's previous workflow omitted QEMU, so the arm64 half of a multi-arch build was not reliable.

Login is `ghcr.io` with `github.actor` and `secrets.GITHUB_TOKEN`. metadata-action lowercases the image name. `ghcr.io/${{ github.repository }}` is already lowercase for the reference repository; still pass that expression and let the action lowercase it.

Provenance attestations stay at `build-push-action`'s default (on). That is why the caller must grant `id-token: write` and `attestations: write`. SBOM generation is left off. It was not part of this profile, and it adds build time. `labels` and `annotations` are both passed through so the registry record matches the metadata action's OCI output.

Cache scope is the fixed string from the parameters (the reference uses `airmx`). The default gha scope is the git ref, which means a tag build would never read the default branch's cache and would write a cache nobody reuses. One shared scope lets tag builds reuse the default branch's layers. A default-branch push and a tag push can write that cache at the same time; last write wins, which is acceptable for a layer cache. `cache-to` uses `mode=max`.

The publish job's header summarizes setup as checkout, QEMU, Buildx, login, metadata, then build. The job also computes the revision after checkout and refuses an empty tag list after metadata. Write the steps in this order: checkout, image revision, QEMU, Buildx, login, metadata, require image tags, build and push.

```yaml
- name: Image revision
  id: rev
  run: |
    set -euo pipefail
    if [ "${GITHUB_REF_TYPE}" = "tag" ]; then
      printf 'value=%s\n' "${GITHUB_REF_NAME#v}" >> "${GITHUB_OUTPUT}"
    else
      printf 'value=%s\n' "${GITHUB_SHA}" >> "${GITHUB_OUTPUT}"
    fi

- name: Set up QEMU
  uses: docker/setup-qemu-action@v4

- name: Set up Buildx
  uses: docker/setup-buildx-action@v4

- name: Login to GHCR
  uses: docker/login-action@v4
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Build and push
  uses: docker/build-push-action@v7
  with:
    push: true
    platforms: linux/amd64,linux/arm64
    build-args: REVISION=${{ steps.rev.outputs.value }}
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
    annotations: ${{ steps.meta.outputs.annotations }}
    cache-from: type=gha,scope=airmx
    cache-to: type=gha,mode=max,scope=airmx
```

Replace `scope=airmx` with the cache-scope parameter. The metadata step from the previous section sits between login and "Require image tags".

The revision stamped into the binary, and passed as the Docker build-arg `REVISION`:

- Tag build: the version with `v` removed (`1.2.3-rc.1`), so a release has a stable asset URL.
- Branch build: the full commit SHA, the same value the reference's previous workflow passed as the `REVISION` build-arg. A short SHA was not used. The full SHA cannot collide, and the default branch has no version number to stamp instead.

`.git` is in `.dockerignore`, so the SHA cannot be read inside the image build. It has to arrive as a build-arg. The Dockerfile declares `ARG REVISION=dev` and consumes it in `go build`.

The image build and the release binaries use the same flags: `CGO_ENABLED=0`, `-trimpath`, and `-ldflags "-s -w"`. When the project has a version ldflag, add `-X {ldflag}=${REVISION}` in the image (the release archives pass the stripped version, not the SHA; see below). The Dockerfile uses the same `CGO_ENABLED=0`. This profile does not build an image on pull requests.

## release.yml order

```text
version -> test -> (docker-publish and binaries in parallel) -> GitHub Release
```

Accepted tag shapes include:

```text
v1.2.3
v1.2.3-rc.1
v1.2.3-rc1
v1.2.3-alpha
v1.2.3-alpha.1
v1.2.3-beta.2
```

`version` runs first and does not check out the repository, so a tag the glob accepts but semver rejects (`v01.2.3`, `v1.2.3-`, `v1.2.3-rc.01`, `v1.2.3-01`) fails in seconds, before tests, the multi-arch image, or six cross-compiles.

`test.yml` is called after `version` and before any publish. Tag pushes do not start `test.yml` on their own.

`docker` and `binaries` both need `test`, so they run together. The release job needs both. The GitHub Release is created only after the image push has succeeded, so the release page does not advertise an image tag that failed to push. `action-gh-release` updates the existing release for this tag on a re-run, which is how a failed asset upload is repaired without deleting the tag.

`binaries` needs `version` directly, not only via `test`. Job outputs are visible to jobs that `needs` the producer. They are not inherited transitively through `test`, so `needs: [version, test]` is required even though `test` already needs `version`. The release job needs `version` for the same reason (the prerelease flag and the version string in the body).

```yaml
jobs:
  version: { ... }
  test:
    needs: version
    permissions:
      contents: read
    uses: ./.github/workflows/test.yml
  docker:
    needs: test
    permissions:
      contents: read
      packages: write
      attestations: write
      id-token: write
    uses: ./.github/workflows/docker-publish.yml
  binaries:
    needs: [version, test]
    # ...
  release:
    needs: [version, docker, binaries]
    # ...
```

Matrix `fail-fast` stays at the default (`true`). One target that does not compile should stop the other five rather than upload a partial set.

## Tag filter and the version job

The trigger filter is a GitHub glob, not a regex. The pattern is matched against the whole tag. In a workflow filter, `+` means one or more of the preceding character or class, `[]` is a character class, and `*` matches any string except `/`. The first pattern does not match `v1.2.3-rc.1`; the extra suffix needs the second pattern. Quotes are required because `[` is special in YAML.

```yaml
on:
  push:
    tags:
      # Whole-tag globs. The second pattern admits -rc / -beta / -alpha
      # and any other hyphenated semver pre-release.
      - "v[0-9]+.[0-9]+.[0-9]+"
      - "v[0-9]+.[0-9]+.[0-9]+-*"
```

The glob is wider than semver on purpose. It is only a cheap event filter. It accepts `v01.2.3` and `v1.2.3-`. The version job is the real check. A bare `1.2.3` with no `v` does not match, and neither does build metadata (`v1.2.3+build`). `+` is awkward in git refs and illegal in a Docker tag. Pre-release suffixes use `-`.

The version job is bash and `grep -Eq`, with no checkout and no Go, so invalid tags stay cheap. `${TAG#v}` is bash prefix removal of a literal `v`, not a regex. `TAG` is an environment variable so the ref is not interpolated into the script. If the tag had no leading `v`, `TAG` and `ver` compare equal and the job fails. That still rejects `1.2.3` if the glob is widened later.

The expression is semver's grammar for `MAJOR.MINOR.PATCH` and an optional pre-release:

- A numeric identifier is `0` or a number without a leading zero (`0` is legal, `01` is not).
- Any other identifier must contain a letter or a hyphen, so `rc`, `rc1`, `alpha`, `beta.2`, and `0rc` pass, while `01` and `rc.01` fail.

metadata-action parses tags with the semver package, which rejects those same leading-zero forms. Catching them here avoids an empty Docker tag list and a later, less obvious failure in `docker-publish.yml`. `if ! grep` is required under `set -euo pipefail`: a non-matching grep exits 1, and without the `if` the script would die before the error line.

`prerelease` is true when the version contains `-`. A stable core (`1.2.3`) has none. That flag drives both the GitHub pre-release checkbox and `make_latest`. It is a string `"true"` / `"false"` because job outputs are strings.

```yaml
version:
  runs-on: ubuntu-latest
  timeout-minutes: 5
  outputs:
    version: ${{ steps.meta.outputs.version }}
    prerelease: ${{ steps.meta.outputs.prerelease }}
  steps:
    # No checkout. Invalid tags should die before any clone.
    - name: Validate semver tag
      id: meta
      env:
        TAG: ${{ github.ref_name }}
      run: |
        set -euo pipefail
        ver="${TAG#v}"
        # Numeric identifiers: 0 or [1-9][0-9]*. Other pre-release
        # identifiers must contain a letter or hyphen (rc, rc.1, alpha).
        # Rejects v01.2.3, v1.2.3-01, v1.2.3-rc.01, v1.2.3+build.
        semver='^(0|[1-9][0-9]*)\.(0|[1-9][0-9]*)\.(0|[1-9][0-9]*)(-((0|[1-9][0-9]*)|[0-9A-Za-z-]*[A-Za-z-][0-9A-Za-z-]*)(\.((0|[1-9][0-9]*)|[0-9A-Za-z-]*[A-Za-z-][0-9A-Za-z-]*))*)?$'
        if [ "${TAG}" = "${ver}" ] || ! printf '%s\n' "${ver}" | grep -Eq "${semver}"; then
          echo "::error::${TAG} is not a supported semver tag (example: v1.2.3 or v1.2.3-rc.1)"
          exit 1
        fi
        pre=false
        case "${ver}" in
          *-*) pre=true ;;
        esac
        printf 'version=%s\n' "${ver}" >> "${GITHUB_OUTPUT}"
        printf 'prerelease=%s\n' "${pre}" >> "${GITHUB_OUTPUT}"
```

## Binary archives

Six targets, built on `ubuntu-latest`:

| GOOS | GOARCH |
| --- | --- |
| linux | amd64 |
| linux | arm64 |
| darwin | amd64 |
| darwin | arm64 |
| windows | amd64 |
| windows | arm64 |

That is the mainstream desktop and server set. `linux/arm` v7, `386`, `riscv64`, and `freebsd` are not in the matrix. There are no macOS or Windows runners. The module does not use cgo, so `CGO_ENABLED=0` cross-compiles from Linux. The Dockerfile uses the same `CGO` flag. A Windows arm64 binary does not need a Windows runner when cgo is off.

`-trimpath` and `-ldflags "-s -w"` match the image build (smaller binary, paths independent of the runner checkout). `-X {ldflag}` stamps the release version without `v` (`1.2.3-rc.1`), not the git SHA, so asset URLs for a release stay on that version. The image on the default branch still stamps the full SHA. See the image-build section.

Archive names follow the usual Go release shape: `{binary}_{version}_{os}_{arch}.tar.gz`, with `.zip` on Windows because a zip is what Windows users unpack without extra tools. The archive contains only the binary at its root (`tar -C`, and `zip` from inside the stage directory). The binary is `{binary}`, or `{binary}.exe` on Windows. `zip -X` strips extra zip attributes so the archive is the file itself. The stage directory is `mktemp` so the workspace is not left holding a binary named `{binary}` (the reference also gitignores that path).

goreleaser is not used. The packaging is a short shell script, and keeping it in this file means another project can copy the workflow without a second config language. The archive names and the single `SHA256SUMS` file are the layout people already expect from goreleaser.

Each matrix leg uploads its own artifact. `upload-artifact` v4 and later cannot append files to one artifact name from parallel jobs; a shared name fails the second upload. `download-artifact` then merges `{binary}-*` into one directory (`merge-multiple`). The pattern `{binary}-*` is the artifact name, not the filename inside it.

`retention-days: 1` because the artifact only has to cross the job boundary. The GitHub Release asset is the durable copy. `if-no-files-found: error`, and `fail_on_unmatched_files` on the release action, so a missing archive fails the job instead of publishing a partial release.

`upload-artifact` is v7 and `download-artifact` is v8. Those two actions do not share a major version. Both are the Node 24 line.

```yaml
binaries:
  needs: [version, test]
  runs-on: ubuntu-latest
  timeout-minutes: 15
  permissions:
    contents: read
  strategy:
    matrix:
      include:
        - os: linux
          arch: amd64
        - os: linux
          arch: arm64
        - os: darwin
          arch: amd64
        - os: darwin
          arch: arm64
        - os: windows
          arch: amd64
        - os: windows
          arch: arm64
  steps:
    - name: Checkout
      uses: actions/checkout@v7

    - name: Setup Go
      uses: actions/setup-go@v7
      with:
        go-version-file: go.mod

    - name: Build archive
      env:
        GOOS: ${{ matrix.os }}
        GOARCH: ${{ matrix.arch }}
        CGO_ENABLED: "0"
        VERSION: ${{ needs.version.outputs.version }}
      run: |
        set -euo pipefail
        name="{binary}_${VERSION}_${GOOS}_${GOARCH}"
        stage="$(mktemp -d)"
        if [ "${GOOS}" = "windows" ]; then
          bin="{binary}.exe"
          archive="${name}.zip"
        else
          bin="{binary}"
          archive="${name}.tar.gz"
        fi
        go build -trimpath -ldflags="-s -w -X {ldflag}=${VERSION}" -o "${stage}/${bin}" .
        if [ "${GOOS}" = "windows" ]; then
          (cd "${stage}" && zip -q -X "${GITHUB_WORKSPACE}/${archive}" "${bin}")
        else
          tar -C "${stage}" -czf "${archive}" "${bin}"
        fi

    - name: Upload archive
      uses: actions/upload-artifact@v7
      with:
        name: {binary}-${{ matrix.os }}-${{ matrix.arch }}
        path: {binary}_${{ needs.version.outputs.version }}_${{ matrix.os }}_${{ matrix.arch }}.*
        if-no-files-found: error
        retention-days: 1
```

When there is no version ldflag, `-ldflags` is `-s -w` only. Checkout depth stays at the default (`1`). Do not set `fetch-depth: 0`. Release notes come from the GitHub API, not from local git history, and metadata-action reads the tag from `GITHUB_REF`.

## Checksums and the GitHub Release

`SHA256SUMS` is generated once, after the merge, in `dist/`, so every line is a basename (`sha256sum`'s two-space format) and the checksum file does not include itself. Per-archive `.sha256` sidecars are not added. One SUMS file is the thing people verify.

```yaml
- name: Download archives
  uses: actions/download-artifact@v8
  with:
    pattern: {binary}-*
    merge-multiple: true
    path: dist

- name: Checksums
  working-directory: dist
  run: |
    set -euo pipefail
    sha256sum {binary}_* | tee SHA256SUMS
```

`working-directory: dist` keeps the names in the checksum list as basenames. The shell expands `{binary}_*` before `sha256sum` runs, and `tee` writes `SHA256SUMS` after that expansion, so the file does not checksum itself.

The release uses `softprops/action-gh-release@v3`.

- The release name is the git tag (`v1.2.3-rc.1`), not the stripped version, so the title matches the ref that was pushed.
- `generate_release_notes` is on. When `body` is also set, GitHub prepends `body` to the generated notes. The body states the image tag and the floating-tag rules so the release page matches `docker-publish.yml`.
- `prerelease` is a boolean input, so the expression is a comparison (`== 'true'`).
- `make_latest` is a string enum (`true` / `false` / `legacy`), not a boolean, and GitHub expressions have no ternary. `cond && 'false' || 'true'` works because any non-empty string is truthy, including the string `false`:
  - Pre-release: `true && 'false'` yields `'false'`, and `'false' || 'true'` keeps `'false'` (the string is truthy, so `||` does not run).
  - Stable: `false && 'false'` yields boolean `false`, and `false || 'true'` yields `'true'`.
- A pre-release cannot be the repository's latest release on the GitHub API. Setting `make_latest` explicitly keeps the action from sending `true`.
- `draft` is left unset. `action-gh-release` v3 notes that a repository with immutable releases should upload a pre-release as a draft and publish afterwards. This profile does not use immutable releases, so the release is published as soon as the assets are attached and `release.prereleased` still fires.
- `fail_on_unmatched_files: true`.
- The release job does not check out the repository.

```yaml
release:
  needs: [version, docker, binaries]
  runs-on: ubuntu-latest
  timeout-minutes: 10
  permissions:
    contents: write
  steps:
    - name: GitHub Release
      uses: softprops/action-gh-release@v3
      with:
        name: ${{ github.ref_name }}
        prerelease: ${{ needs.version.outputs.prerelease == 'true' }}
        # String enum, not a boolean. Yields the string false for a pre-release.
        make_latest: ${{ needs.version.outputs.prerelease == 'true' && 'false' || 'true' }}
        generate_release_notes: true
        fail_on_unmatched_files: true
        body: |
          Docker image: `ghcr.io/${{ github.repository }}:${{ needs.version.outputs.version }}`

          Stable tags also publish `major.minor` and `major` image tags. Floating tags that are only a leading zero (`0`, `0.0`) are omitted. Pre-release tags (for example `-rc.1`, `-beta.2`, `-alpha`) publish the full version only and are marked as GitHub pre-releases.
        files: |
          dist/{binary}_*
          dist/SHA256SUMS
```

`release.yml` concurrency is `release-${{ github.ref }}` with `cancel-in-progress: false`.

## README

Keep a README section named "Continuous integration" in sync with the tag policy. Two tables, with the repository's image name, default branch, and binary name filled in.

Events:

| Event | What runs |
| --- | --- |
| Pull request, or a push to any branch other than the default branch | `go test ./...` |
| Push to the default branch | the same tests, then `ghcr.io/<owner>/<repo>:latest` |
| Push of a semver tag (`v1.2.3`, `v1.2.3-rc.1`, and any other `-` pre-release) | the same tests, semver image tags, and a GitHub Release |

Image tags:

| Git tag | Image tags |
| --- | --- |
| `v1.2.3` | `1.2.3`, `1.2`, `1` |
| `v1.2.3-rc.1` | `1.2.3-rc.1` |
| `v0.2.0` | `0.2.0`, `0.2` |
| `v0.0.1` | `0.0.1` |

The prose under the tables says: Docker tags drop the leading `v`. There is no commit-SHA tag. Pre-release suffixes (`-rc`, `-beta`, `-alpha`, and any other semver pre-release) publish the full version only. Floating tags that would be only a leading zero (`0`, `0.0`) are not published. A pre-release tag is marked as a GitHub pre-release and is not made the repository's latest release. Each GitHub Release attaches `SHA256SUMS` and an archive per mainstream OS and architecture: Linux, macOS, and Windows, on amd64 and arm64 (`.tar.gz`, or `.zip` on Windows). The binary inside is `{binary}` (`{binary}.exe` on Windows).

When the tag policy changes, update the `docker-publish.yml` header, the release `body`, and both README tables together.

## What each file header has to say

| File | The header has to cover |
| --- | --- |
| `test.yml` | How the four files divide events. Why the default branch is `branches-ignore`'d. Why `workflow_run` was rejected. `workflow_call` filters are not applied when this file is called. Pull requests into any base, and no `paths` filter. `contents: read` only, and the caller must grant it. `secrets: inherit` is not required. The concurrency prefix and the deadlock when it matches the caller. Force-pushing the same tag can still cancel a release through this group. The suite is `go test ./...` with `-race` left off. Why majors float. `go-version-file`. Timeout 15 |
| `docker.yml` | Test, then publish, so the tag rules are not copied. Not on pull requests, other branches, or tags. Why the trigger is a literal branch name, and why an `if` on every branch was rejected. `is_default_branch` is the second guard and fails closed. No SHA tag and no per-branch tag (`type=sha` and `type=ref,event=branch` were removed). The permission ceiling, and that job permissions replace rather than merge. Why provenance needs `id-token` and `attestations`. Group `docker-<ref>`. No `workflow_dispatch` |
| `docker-publish.yml` | The whole tag table. Why `latest=false` and the raw tag are both required. The `v0` and `v0.0` enables, including `v0.10.0`. Why a pre-release does not move floating channels. `${{ }}` and `{{ }}` are different languages. No `type=sha` and no `type=ref`. What the revision is on a tag and on a branch, and why `.git` in `.dockerignore` forces a build-arg. Platforms, and QEMU before Buildx. Login. Provenance on, SBOM off. Both `labels` and `annotations`. Why the cache scope is a fixed string. Require tags before the push. Timeout 45. metadata-action v5 and build-push-action v6 predate the Node 20 removal |
| `release.yml` | Job order, and why `version` does not check out. Outputs do not pass through `test`. `fail-fast` stays at the default. Glob versus semver, and why the patterns are quoted. The prerelease flag. Six targets, cgo off, no goreleaser. Archive layout, `zip -X`, and `mktemp`. Artifacts cannot be appended, names are unique per leg, retention is 1 day. upload v7 and download v8. One `SHA256SUMS`. The `make_latest` expression. `draft` left unset, and `release.prereleased` still fires. Checkout depth stays at the default. `cancel-in-progress: false`. No `workflow_dispatch` |

## Choices that stay

These are the shape of the profile, not a backlog:

- Gate publish with `workflow_call` in the same run. Do not switch the gate to `workflow_run`.
- The default branch runs tests once, through `docker.yml`.
- `:latest` is `latest=false` plus a raw tag with `enable={{is_default_branch}}`. Do not set flavor back to `latest=auto`.
- Do not publish a commit-SHA image tag or a branch-name image tag.
- Disable floating channels that would be only `0` or `0.0`.
- A pre-release publishes the full version as its only image tag, is a GitHub pre-release, and sets `make_latest` to the string `false`.
- Image platforms are `linux/amd64,linux/arm64`, with QEMU before Buildx.
- Cache scope is one fixed string, `mode=max`.
- Provenance stays at the build-push default (on). Do not generate an SBOM.
- An invalid tag fails before checkout.
- Six targets, `CGO_ENABLED=0`, no goreleaser.
- One `SHA256SUMS`, no per-archive sidecar.
- Action majors float. Do not pin commit SHAs.
- The test command is `go test ./...`.
- None of the four files defines `workflow_dispatch`.
- Pull requests run tests only. They do not build an image.
