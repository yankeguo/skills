# Profile: go-single-binary-docker

id: `go-single-binary-docker`

Go 单静态二进制，同时发布一份 Docker 镜像。四个 workflow 文件把「测试」「默认分支的 `:latest`」「semver tag 的镜像与 GitHub Release」拆开，测试失败就不能发布。

这份文档自洽。实施时按这里写 YAML，把下面各节的理由压缩进对应文件的开头注释。不要写一句「见 skill」就结束，下次只打开 YAML 的人需要能看见为什么不能改回去。

参考实现是 [yankeguo/airmx](https://github.com/yankeguo/airmx) 的 `.github/workflows/`。那里的仓库名、二进制名和 `-X` 符号是参数，不是要抄进别的仓库的字面量。

## 适用

同时满足这些条件才用这份 profile：

- 仓库根目录有且只有一个 `package main`，`go build .` 得到发布用的二进制
- `CGO_ENABLED=0` 能从 `ubuntu-latest` 交叉编译 linux、darwin、windows
- 一份镜像，推到 `ghcr.io/<owner>/<repo>`，运行时是 Alpine 上的这个二进制
- 默认分支用字面量写进触发器（参考实现是 `main`）
- 版本 tag 带前缀 `v`，形式是 semver，预发布后缀用 `-`（`v1.2.3-rc.1`），不用 `+` 构建元数据

这些仓库不用这份 profile：需要 cgo；要在 macOS 或 Windows runner 上编译；根目录不是唯一的 main 包；不发镜像或不发二进制；一次推多份镜像；monorepo 里有多条发布线。

## 仓库参数

| 参数 | 参考值 | 写到哪里 |
| --- | --- | --- |
| 二进制名 | `airmx` | 归档名、归档内文件名、artifact 名前缀 |
| 版本 ldflag | `main.assetVersion` | 二进制和镜像的 `-X`。程序没有这个变量就去掉 `-X`，其余构建参数不动 |
| 缓存 scope | `airmx` | `cache-from` / `cache-to` 的固定字符串，用仓库短名，不用 git ref |
| 测试并发组前缀 | `airmx-test` | 必须和 `docker-`、`release-` 不同 |
| 默认分支 | `main` | `docker.yml` 的 `on.push.branches`，以及 `test.yml` 的 `branches-ignore`，两处同一个字面量 |
| 镜像 | `ghcr.io/${{ github.repository }}` | metadata-action 会把镜像名转成小写 |

下面用「二进制名」指第一行。示例里出现 `airmx` 时，换成这个参数。

## 四个文件，每种事件只走一条路

| 文件 | 谁触发它 | 做什么 |
| --- | --- | --- |
| `test.yml` | `pull_request`；push 到除默认分支以外的分支；另外两个文件用 `workflow_call` 调用 | `go test ./...` |
| `docker.yml` | push 到默认分支 | 先调用 `test.yml`，再调用 `docker-publish.yml` |
| `release.yml` | push 一个 `v*` semver tag | 校验版本，调用 `test.yml`，然后并行做镜像和二进制，最后创建 GitHub Release |
| `docker-publish.yml` | 只接受 `workflow_call` | 镜像 tag 和构建只写在这一份里 |

`workflow_call` 专用的文件不会在 Actions 页上变成单独的按钮。

同一事件只跑一次测试：

- 默认分支不写进 `test.yml` 的 `push`。push 到默认分支已经会跑 `docker.yml`，那边会先调用测试。再列一次就会每次跑两遍。
- tag 不是 branch，`on.push.branches` 匹配不到。tag 不会直接启动 `test.yml`，由 `release.yml` 调用。
- `on.workflow_call` 不使用调用方的事件过滤。`branches-ignore` 挡不住 `docker.yml` 和 `release.yml`。

任何目标分支的 pull request 都跑测试，不限于合进默认分支的 PR。默认不做 `paths` 过滤。Dockerfile、`go.mod`、workflow 本身，以及非 main 包的改动，都可能改变发布物。

发布和测试在同一次 run 里：`docker.yml` / `release.yml` 用 `needs: test` 调用。不用 `workflow_run` 来挡住发布。`workflow_run` 只读默认分支上的 workflow 文件，checkout 的 SHA 是 `github.event.workflow_run.head_sha` 而不是 `github.sha`，token 权限也不好放。`workflow_call` 把测试门禁留在同一次 run 里。

四个文件都不要 `workflow_dispatch`。手动跑会从触发时选定的 ref 再发一次 `:latest` 或 Release，很容易打到错误的提交。

## 权限

Workflow 级 `permissions` 是这一个文件里的上限。Job 不能授予 workflow 没写的权限。被调用的 workflow 只能把调用方的 `GITHUB_TOKEN` 收窄，不能放宽，所以权限要写在调用 job 上。

Job 级 `permissions` 整组替换 workflow 级，不合并。测试 job 要再次写上 `contents: read`，不能把 `packages: write` 留在测试 job 上。

`test.yml` 只有：

```yaml
permissions:
  contents: read
```

够 checkout 和跑测试。发布权限不放在这里。`secrets.GITHUB_TOKEN` 在被调用的 workflow 里已经可用，不需要 `secrets: inherit`。

`docker.yml` 的上限：

```yaml
permissions:
  contents: read
  packages: write
  attestations: write
  id-token: write
```

- 测试 job：`contents: read`
- 发布 job：上面四项都要

`packages: write` 用来推 GHCR。`id-token: write` 和 `attestations: write` 要留着，因为 `docker/build-push-action` 的 provenance attestation 默认开启。去掉这两项，失败发生在 attestation，不是发生在镜像构建。

`release.yml` 的上限把 `contents: read` 换成 `contents: write`（创建 Release 需要它），其余三项与 `docker.yml` 相同。各 job 再收窄：

| job | permissions |
| --- | --- |
| `version` | 参考实现不写 job 级 permissions。这个 job 不 checkout，步骤也不使用 token；省略时它继承本文件的 workflow 上限 |
| `test` | `contents: read` |
| `docker` | `contents: read`、`packages: write`、`attestations: write`、`id-token: write` |
| `binaries` | `contents: read` |
| `release` | `contents: write` |

`docker-publish.yml` 在自己的 workflow 级再声明与调用 job 相同的四项（`contents: read`、`packages: write`、`attestations: write`、`id-token: write`）。被调用文件不能把调用方的权限放大；调用 job 和 `docker-publish.yml` 两边都写上这四项。

## 并发

组名是固定前缀加 `${{ github.ref }}`。不要在可复用 workflow 的 `concurrency.group` 里用 `github.workflow`。`workflow_call` 里面这个上下文是**调用方**的 workflow 名。和调用方的 group 撞车时，GitHub 会用「deadlock for concurrency group」取消这次 run。

| 文件 | group | cancel-in-progress |
| --- | --- | --- |
| `test.yml` | `<测试并发组前缀>-${{ github.ref }}` | `true` |
| `docker.yml` | `docker-${{ github.ref }}` | `true` |
| `release.yml` | `release-${{ github.ref }}` | `false` |

`cancel-in-progress: true` 用于 PR synchronize，以及同一分支上的新 push，包括默认分支上还在跑的 `:latest`。

被调用 workflow 的并发作用在调用方这次 run 上，于是这次 run 同时占着两个 group。默认分支上这是需要的：新的 push 取消正在跑的 docker run。副作用是 tag：`release.yml` 自己的 group 不取消，但 `test.yml` 的 group 会取消。force-push 同一个 tag 仍可能通过测试 group 取消正在跑的 release。普通 tag 的 ref 各不相同，不会撞车。把这个副作用写进 `test.yml` 的文件头。不要靠删掉测试并发，或把测试改成 `cancel-in-progress: false`，来「修掉」它。

## Action 版本

2026-09-23 起 GitHub hosted runner 不再提供 Node 20，更早的 major 跑不起来。Major 保持浮动：action 的 patch 应该生效。不要 pin 到 commit SHA。

| 用途 | 版本 |
| --- | --- |
| checkout | `actions/checkout@v7` |
| setup-go | `actions/setup-go@v7` |
| QEMU | `docker/setup-qemu-action@v4` |
| Buildx | `docker/setup-buildx-action@v4` |
| 登录 GHCR | `docker/login-action@v4` |
| 镜像 metadata | `docker/metadata-action@v6` |
| 构建并推送 | `docker/build-push-action@v7` |
| 上传 artifact | `actions/upload-artifact@v7` |
| 下载 artifact | `actions/download-artifact@v8` |
| GitHub Release | `softprops/action-gh-release@v3` |

`upload-artifact` 和 `download-artifact` 不共用同一个 major。`action-gh-release` 的 v2 是最后一版 Node 20，并且不再维护。

以后因为 Node 大版本被移除而升级时，相关 major 一起换，在文件头写下日期和原因。

Go 工具链用 `go-version-file: go.mod`，读 `go` 那一行。不要在 YAML 里再 pin 一次 `go-version`。`setup-go` 的 module 和 build cache 保持默认开启。

## 超时

| job | `timeout-minutes` |
| --- | --- |
| test | 15 |
| docker publish | 45 |
| version | 5 |
| binaries | 15 |
| release | 10 |

发布 job 给 45 分钟，是因为 arm64 在 QEMU 下跑，这是慢的那一段。测试本身是秒级，15 分钟用来拦住挂住的测试。

Shell 步骤用 `set -euo pipefail`。要传进脚本的 ref、tag 列表放进环境变量，不要插值进脚本正文，避免多行 tag 被 shell 引号弄坏。

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

并发组前缀和 `branches-ignore` 里的分支名换成仓库参数。测试命令就是 `go test ./...`。套用这份 profile 时不要另加 `-race`、`golangci-lint` 或 `go vet`。

## docker.yml

触发器是默认分支的字面量，不是「当前默认分支是哪个」。GitHub 在事件 payload 出现之前就计算 `on.push.branches`，触发器里不能用 `github.event.repository.default_branch`。也不要改成所有分支都触发、再用 `if` 跳过：功能分支仍会被排进队列。

第二道保险在 `docker-publish.yml` 里（`enable={{is_default_branch}}`）。如果字面量分支不再是默认分支，这个表达式为 false，不会产生 tag，发布 workflow 故意失败，而不是从非默认分支推 `:latest`。

这个文件不跑 pull request，不跑其他分支，也不跑 tag。tag 即使指向默认分支，也不会因此移动 `:latest`。`:latest` 只在默认分支移动时移动。发一个 release tag 和更新默认分支是两次 push，两次各自的结果。

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

## 镜像 tag

tag 规则只放在 `docker-publish.yml`。`release.yml` 不复制一份。两处入口必须走同一个文件，避免漂移。改规则时同时改：这个文件的头注释、Release 正文、README 里的两张表。

`docker/metadata-action` 的 `{{version}}` 会去掉前缀 `v`，并保留预发布后缀。`v1.2.3-rc.1` 变成镜像 tag `1.2.3-rc.1`，不会变成 `1.2.3`。

| 事件 | 镜像 tag |
| --- | --- |
| push 到默认分支 | `latest` |
| tag `v1.2.3` | `1.2.3`、`1.2`、`1` |
| tag `v1.2.3-rc.1`（`-rc`、`-beta`、`-alpha`、`-rc1`、`-beta.2`、`-alpha.1`，以及其他 semver 预发布） | `1.2.3-rc.1` |
| tag `v0.2.0` | `0.2.0`、`0.2` |
| tag `v0.0.1` | `0.0.1` |

没有 commit SHA tag，也没有分支名 tag。

`:latest` 只跟踪默认分支。稳定版 `v1.2.3` 和预发布 tag 都不移动它。两处设置一起完成这件事：

1. `flavor: latest=false`。默认 flavor 是 `latest=auto`，而 auto 会给 `type=semver`（以及 `type=ref,event=tag`、`type=pep440`、`type=match`）加上 `:latest`。留着默认值，每次 semver push 都会把 `:latest` 改掉。auto 不会仅仅因为这次是分支 push 就加 `:latest`，所以只写 `latest=false` 的话，默认分支也不会再发 `:latest`。
2. `type=raw,value=latest,enable={{is_default_branch}}`。这是显式的 `:latest`。`{{is_default_branch}}` 是 metadata-action 的 Handlebars 表达式，由 action 计算。push 到仓库默认分支时为 true，tag 事件为 false。GitHub 的 `${{ }}` 和 action 的 `{{ }}` 是两种语言。`tags:` 里两处都出现是故意的。

`{{major}}.{{minor}}` 和 `{{major}}` 是会移动的通道（`1.2` 和 `1`）。通道本身会变成只有一个前导零时，把这一条关掉：

| tag | major 通道 | minor 通道 | 仍保留 |
| --- | --- | --- | --- |
| `v0.2.0` | `0`，关闭 | `0.2`，保留 | `0.2.0` |
| `v0.0.1` | `0`，关闭 | `0.0`，关闭 | `0.0.1`（来自 `{{version}}`） |
| `v0.10.0` | `0`，关闭 | `0.10`，保留。ref 是 `refs/tags/v0.10.0`，没有前缀 `refs/tags/v0.0.` | `0.10.0`、`0.10` |

这是 metadata-action 文档里「Major version zero」的做法（semver `0.y.z` 不稳定，Docker tag `0` 不应该存在），再往下延伸到 `0.0`。GitHub 表达式先算完，action 看到的是字面量 `true` 或 `false`：

```yaml
enable=${{ !startsWith(github.ref, 'refs/tags/v0.') }}
enable=${{ !startsWith(github.ref, 'refs/tags/v0.0.') }}
```

前缀里带着 `v`，因为这份 profile 的 tag 都有 `v`。分支 push 的 ref 是 `refs/heads/<branch>`，两个 enable 都是 true，这没有害处：`type=semver` 只在 tag 事件才发出 tag。

预发布还有第二层原因不会移动浮动通道。metadata-action 遇到预发布时，不会把 `{{major}}` 渲染成 `1`，而是渲染成完整的预发布字符串（`1.2.3-rc.1`），再和 `{{version}}` 去重。结果只剩完整版本这一个 tag。前导零通道仍然要关：稳定的 `0.x` 上，major / minor 真的会是 `0` 和 `0.0`。

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

不要把 `latest` 简化回 `auto`，不要去掉 `v0` / `v0.0` 的 enable，不要加回 `type=sha` 或 `type=ref`。

推送之前先要求 tag 列表非空。metadata-action 给不出 tag 时（tag 形状解析不了，或分支不是默认分支），`build-push-action` 要么报一个看不懂的错，要么推一个没有 tag 的 manifest。Tag 经环境变量传入。这一步把 tag 打到日志里。

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

## 镜像构建

平台只有 `linux/amd64` 和 `linux/arm64`。镜像是 Alpine。macOS 和 Windows 的二进制是 Release 归档，不是镜像。

Hosted runner 是 amd64，没有 binfmt 就执行不了 arm64。`setup-qemu-action` 放在 Buildx 之前。步骤顺序：checkout、算 revision、QEMU、Buildx、登录、metadata、要求 tag 非空、构建推送。

登录 `ghcr.io`，用户名 `github.actor`，密码 `secrets.GITHUB_TOKEN`。

Provenance attestation 保持 `build-push-action` 的默认（开着）。这就是调用方必须给 `id-token: write` 和 `attestations: write` 的原因。SBOM 关掉：这份 profile 不生成 SBOM，它会加长构建。`labels` 和 `annotations` 都传下去，仓库里的记录和 metadata-action 的 OCI 输出一致。

缓存 scope 是仓库参数里的固定字符串。gha 的默认 scope 是 git ref，tag 构建就读不到默认分支的缓存，写出的缓存也没有下次会用。一个共享 scope 让 tag 构建复用默认分支的层。默认分支和 tag 可能同时写这块缓存，后写的赢。对层缓存可以接受。`cache-to` 使用 `mode=max`。

```yaml
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

`scope=` 换成缓存 scope 参数。

Revision 打进二进制，也作为 Docker build-arg `REVISION`：

- tag 构建：去掉 `v` 的版本（`1.2.3-rc.1`），一次 release 的资源 URL 因此是稳定的
- 分支构建：完整 commit SHA。不用短 SHA，完整 SHA 不会撞车，默认分支上也没有版本号可以代替它

`.dockerignore` 里要有 `.git`。SHA 读不到镜像构建内部，只能从 build-arg 进去。

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
```

镜像构建和 Release 二进制使用同一组编译参数：`CGO_ENABLED=0`、`-trimpath`、`-ldflags "-s -w"`。仓库有版本 ldflag 时再加上 `-X <符号>=${REVISION}`。Dockerfile 声明 `ARG REVISION=dev`，并在 `go build` 里用它。这份 profile 不在 PR 上构建镜像。

## release.yml 的顺序

```text
version -> test -> (docker-publish 与 binaries 并行) -> GitHub Release
```

`version` 最先跑，并且不 checkout。glob 放过但 semver 拒绝的 tag（`v01.2.3`、`v1.2.3-`、`v1.2.3-rc.01`）在几秒内失败，发生在测试、多架构镜像和六次交叉编译之前。

`test` 在任何发布之前。`docker` 和 `binaries` 都 `needs: test`，所以它们并行。`release` 等这两边都成功。镜像推送失败时不创建 GitHub Release，Release 页面上就不会出现一个没推上去的镜像 tag。`action-gh-release` 在重跑时会更新这个 tag 已有的 Release，因此资源上传失败可以修好，不用删 tag。

Job output 只对 `needs` 了生产者的 job 可见，不会经 `test` 传递。`binaries` 和 `release` 都要直接 `needs: version`，即使 `test` 已经依赖 `version`。`release` 要用版本字符串和 prerelease 标志；`binaries` 要用打进归档名和 ldflag 的版本。

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

Matrix 的 `fail-fast` 保持默认 `true`。一个目标编译失败就停掉其余五个，不要上传半套归档。

## Tag 过滤和版本 job

触发器用 GitHub glob，不是正则。模式匹配整个 tag。`+` 表示前一个字符或字符类出现一次或更多，`[]` 是字符类，`*` 匹配除 `/` 以外的任意字符串。第一条匹配不到 `v1.2.3-rc.1`，后缀靠第二条。YAML 里要加引号，因为 `[` 有特殊含义。

```yaml
on:
  push:
    tags:
      - "v[0-9]+.[0-9]+.[0-9]+"
      - "v[0-9]+.[0-9]+.[0-9]+-*"
```

这个 glob 故意比 semver 宽，它只是便宜的事件过滤。它会放过 `v01.2.3` 和 `v1.2.3-`。真正的检查是 `version` job。不带 `v` 的 `1.2.3` 匹配不到。构建元数据 `v1.2.3+build` 也匹配不到。`+` 在 git ref 里麻烦，在 Docker tag 里也不合法。预发布用 `-`。

`version` 用 bash 和 `grep -Eq`，不 checkout，也不安装 Go。`${TAG#v}` 是 bash 去掉字面量前缀 `v`，不是正则。tag 没有前导 `v` 时 `TAG` 和 `ver` 相等，job 失败；以后就算 glob 放宽了，裸的 `1.2.3` 仍会被拒绝。

表达式是 semver 里 `MAJOR.MINOR.PATCH` 加可选预发布的语法：

- 数字标识符是 `0`，或没有前导零的数字（`0` 合法，`01` 不合法）
- 其他标识符必须含有字母或连字符，所以 `rc`、`rc1`、`alpha`、`beta.2`、`0rc` 通过，`01` 和 `rc.01` 失败

metadata-action 用 semver 包解析 tag，同样拒绝前导零。在这里拦住，避免 `docker-publish.yml` 里出现空 tag 列表和更晚、更难看懂的失败。`set -euo pipefail` 下必须写成 `if ! grep`：grep 不匹配时退出码是 1，没有 `if` 的话脚本会在打印错误之前就停。

`prerelease` 在版本含有 `-` 时为 true。稳定版本 `1.2.3` 没有 `-`。这个标志同时驱动 GitHub 的 pre-release 勾选和 `make_latest`。Job output 是字符串，所以写 `"true"` / `"false"`。

```yaml
version:
  runs-on: ubuntu-latest
  timeout-minutes: 5
  outputs:
    version: ${{ steps.meta.outputs.version }}
    prerelease: ${{ steps.meta.outputs.prerelease }}
  steps:
    - name: Validate semver tag
      id: meta
      env:
        TAG: ${{ github.ref_name }}
      run: |
        set -euo pipefail
        ver="${TAG#v}"
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

## 二进制归档

六个目标，都在 `ubuntu-latest` 上构建：

| GOOS | GOARCH |
| --- | --- |
| linux | amd64 |
| linux | arm64 |
| darwin | amd64 |
| darwin | arm64 |
| windows | amd64 |
| windows | arm64 |

这是常见的桌面和服务器组合。不包含 linux/arm v7、386、riscv64、freebsd。不开 macOS 或 Windows runner。模块不用 cgo，所以 `CGO_ENABLED=0` 从 Linux 交叉编译。Windows arm64 在 cgo 关闭时不需要 Windows runner。

编译参数和镜像一致：`-trimpath`、`-ldflags "-s -w"`。有版本 ldflag 时，`-X` 打去掉 `v` 的 release 版本（`1.2.3-rc.1`），不打 git SHA。默认分支上的镜像仍然打完整 SHA，见上一节。

归档名是 Go release 常见的形状：`<二进制名>_<version>_<os>_<arch>.tar.gz`。Windows 用 `.zip`，因为 Windows 上解 zip 不需要额外工具。归档根目录只有二进制本身（`tar -C`，zip 在暂存目录里执行）。二进制叫 `<二进制名>`，Windows 上是 `<二进制名>.exe`。`zip -X` 去掉额外的 zip 属性。暂存目录用 `mktemp`，工作区里不要留下一个叫二进制名的文件。

不用 goreleaser。打包是一段短 shell，留在这个文件里，别的项目复制 workflow 时不需要第二种配置语言。归档名和单独一份 `SHA256SUMS` 就是人们已经从 goreleaser 习惯的布局。

每个 matrix 分支上传自己的 artifact。`upload-artifact` v4 起不能从并行 job 往同一个 artifact 名后面追加文件，共用一个名字时第二次上传会失败。`download-artifact` 再用 `merge-multiple` 把 `<二进制名>-*` 收进一个目录。这个 pattern 是 artifact 名，不是里面的文件名。

`retention-days: 1`，因为 artifact 只需要跨过 job 边界。持久的那份是 GitHub Release 的附件。`if-no-files-found: error`，Release action 上再开 `fail_on_unmatched_files`，缺一份归档就让 job 失败，而不是发出半套 Release。

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
    - uses: actions/checkout@v7
    - uses: actions/setup-go@v7
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
        name="<二进制名>_${VERSION}_${GOOS}_${GOARCH}"
        stage="$(mktemp -d)"
        if [ "${GOOS}" = "windows" ]; then
          bin="<二进制名>.exe"
          archive="${name}.zip"
        else
          bin="<二进制名>"
          archive="${name}.tar.gz"
        fi
        go build -trimpath -ldflags="-s -w -X <版本 ldflag>=${VERSION}" -o "${stage}/${bin}" .
        if [ "${GOOS}" = "windows" ]; then
          (cd "${stage}" && zip -q -X "${GITHUB_WORKSPACE}/${archive}" "${bin}")
        else
          tar -C "${stage}" -czf "${archive}" "${bin}"
        fi
    - name: Upload archive
      uses: actions/upload-artifact@v7
      with:
        name: <二进制名>-${{ matrix.os }}-${{ matrix.arch }}
        path: <二进制名>_${{ needs.version.outputs.version }}_${{ matrix.os }}_${{ matrix.arch }}.*
        if-no-files-found: error
        retention-days: 1
```

没有版本 ldflag 时，`-ldflags` 只留 `-s -w`。Checkout 不改 `fetch-depth`，保持默认的 1。Release notes 来自 GitHub API，不来自本地 git 历史；metadata-action 从 `GITHUB_REF` 读 tag。

## 校验和与 GitHub Release

`SHA256SUMS` 在合并之后、在 `dist/` 里生成一次。每一行都是 basename（`sha256sum` 的两个空格格式），校验文件不包含它自己。不为每个归档再加 `.sha256`。人们核对的是这一份 SUMS。

```yaml
- name: Download archives
  uses: actions/download-artifact@v8
  with:
    pattern: <二进制名>-*
    merge-multiple: true
    path: dist

- name: Checksums
  working-directory: dist
  run: |
    set -euo pipefail
    sha256sum <二进制名>_* | tee SHA256SUMS
```

`working-directory: dist` 让校验列表里的名字是 basename。shell 先展开 glob，再由 `tee` 写出 `SHA256SUMS`，所以文件不会把自己算进去。

Release 使用 `softprops/action-gh-release@v3`。

- 标题是 git tag（`v1.2.3-rc.1`），不是去掉 `v` 的版本，这样标题和被 push 的 ref 一致
- `generate_release_notes: true`。同时设置 `body` 时，GitHub 把 body 放在自动生成的 notes 前面。body 写上镜像 tag，以及浮动 tag 的规则，让 Release 页面和 `docker-publish.yml` 一致
- `prerelease` 是布尔输入，表达式用比较：`needs.version.outputs.prerelease == 'true'`
- `make_latest` 是字符串枚举（`true` / `false` / `legacy`），不是布尔。GitHub 表达式没有三元运算符。`cond && 'false' || 'true'` 能用，是因为任意非空字符串都为真，包括字符串 `false`：
  - 预发布：`true && 'false'` 得到 `'false'`，`'false' || 'true'` 留下 `'false'`（字符串为真，`||` 不继续）
  - 稳定版：`false && 'false'` 得到布尔 `false`，`false || 'true'` 得到 `'true'`
- 预发布不能成为仓库在 GitHub API 上的 latest release。把 `make_latest` 显式设上，action 才不会送出 `true`
- `draft` 不设置。`action-gh-release` v3 说明：仓库开了 immutable releases 时，预发布应先以 draft 上传再发布。这份 profile 不用 immutable releases，附件挂上就发布，`release.prereleased` 仍会触发
- `fail_on_unmatched_files: true`
- `release` job 自己不 checkout

```yaml
- name: GitHub Release
  uses: softprops/action-gh-release@v3
  with:
    name: ${{ github.ref_name }}
    prerelease: ${{ needs.version.outputs.prerelease == 'true' }}
    make_latest: ${{ needs.version.outputs.prerelease == 'true' && 'false' || 'true' }}
    generate_release_notes: true
    fail_on_unmatched_files: true
    body: |
      Docker image: `ghcr.io/${{ github.repository }}:${{ needs.version.outputs.version }}`

      Stable tags also publish `major.minor` and `major` image tags. Floating tags that are only a leading zero (`0`, `0.0`) are omitted. Pre-release tags (for example `-rc.1`, `-beta.2`, `-alpha`) publish the full version only and are marked as GitHub pre-releases.
    files: |
      dist/<二进制名>_*
      dist/SHA256SUMS
```

`release.yml` 的并发是 `release-${{ github.ref }}`，`cancel-in-progress: false`。不要靠这个文件自己的 group 取消一次 tag 构建。

## README

仓库 README 保留一节 Continuous integration，和 tag 规则一起改。两张表：

事件：

| 事件 | 跑什么 |
| --- | --- |
| Pull request，或 push 到除默认分支以外的分支 | `go test ./...` |
| Push 到默认分支 | 同一套测试，然后 `ghcr.io/<owner>/<repo>:latest` |
| Push 一个 semver tag（`v1.2.3`、`v1.2.3-rc.1`，以及其他 `-` 预发布） | 同一套测试、semver 镜像 tag，以及 GitHub Release |

镜像 tag：

| Git tag | 镜像 tag |
| --- | --- |
| `v1.2.3` | `1.2.3`、`1.2`、`1` |
| `v1.2.3-rc.1` | `1.2.3-rc.1` |
| `v0.2.0` | `0.2.0`、`0.2` |
| `v0.0.1` | `0.0.1` |

正文说明这三件事：Docker tag 去掉前缀 `v`；没有 commit SHA tag；预发布只发完整版本，前导零浮动 tag（`0`、`0.0`）不发；预发布 tag 标成 GitHub pre-release，并且不会成为仓库的 latest release。再说明每个 Release 附带 `SHA256SUMS`，以及 linux / darwin / windows × amd64 / arm64 的归档（Windows 是 `.zip`，其余是 `.tar.gz`），归档里是二进制名（Windows 上加 `.exe`）。

## 文件头要写上的决定

| 文件 | 注释里要能找到 |
| --- | --- |
| `test.yml` | 四个文件如何分事件；为什么默认分支被 `branches-ignore`；为什么不用 `workflow_run`；`workflow_call` 不吃这里的过滤；PR 不限目标分支、没有 paths 过滤；权限只有 `contents: read`，调用方必须自己授予；不需要 `secrets: inherit`；并发组前缀以及和调用方撞车会死锁；同一 tag 被 force-push 时测试 group 仍可能取消 release；测试命令只有 `go test ./...`；major 浮动的原因；`go-version-file`；超时 15 |
| `docker.yml` | 先测试再发布，tag 规则不复制；不跑 PR、其他分支和 tag；触发器为什么是字面量分支；`is_default_branch` 是第二道保险；没有 SHA tag 和分支 tag；权限上限，以及 job 级是替换不是合并；provenance 为什么需要 `id-token` 和 `attestations`；并发组 `docker-<ref>`；没有 `workflow_dispatch` |
| `docker-publish.yml` | 整张 tag 表；`latest=false` 加上 raw tag 的原因；`v0` / `v0.0` 的 enable；预发布为什么不移动浮动通道；`${{ }}` 和 `{{ }}` 是两种语言；没有 `type=sha` 和 `type=ref`；revision 在 tag 与分支上分别是什么；平台、QEMU 顺序；登录方式；provenance 开、SBOM 关；labels 和 annotations 都传；缓存 scope 为什么固定；推送前要求 tag 非空；超时 45 |
| `release.yml` | job 顺序，以及 `version` 为什么不 checkout；output 不穿过 `test`；`fail-fast` 保持默认；glob 与 semver 的分工、引号；预发布标志；六个目标、cgo 关闭、不用 goreleaser；归档布局；artifact 不能追加、名字必须唯一、保留 1 天；upload v7 与 download v8；一份 `SHA256SUMS`；`make_latest` 的表达式；不设 draft；checkout 深度保持默认；`cancel-in-progress: false`；没有 `workflow_dispatch` |

## 不要改回去的决定

下面这些是这份 profile 的形状，不是还没做完的清单：

- 用 `workflow_call` 把测试和发布放在同一次 run，不用 `workflow_run`
- 默认分支的测试只经由 `docker.yml` 跑一次
- `:latest` 用 `latest=false` 加 `enable={{is_default_branch}}` 的 raw tag，不用 `latest=auto`
- 不发 commit SHA 镜像 tag，不发分支名镜像 tag
- 关闭会变成 `0` 或 `0.0` 的浮动通道
- 预发布只发完整版本这一个镜像 tag，并标成 GitHub pre-release，且 `make_latest` 为字符串 `false`
- 镜像平台只有 `linux/amd64,linux/arm64`，先 QEMU 再 Buildx
- 缓存 scope 是固定字符串，`mode=max`
- provenance 保持默认开启，不生成 SBOM
- tag 不合法时在 checkout 之前失败
- 六个目标、`CGO_ENABLED=0`、不用 goreleaser
- 一份 `SHA256SUMS`，没有每个归档的 sidecar
- Action major 浮动，不 pin commit SHA
- 测试命令只有 `go test ./...`
- 四个文件都没有 `workflow_dispatch`
- PR 只跑测试，不构建镜像
