---
name: best-github-workflows
description: >-
  GitHub Actions 规范目录。创建、修改或审查 workflow 时先匹配已登记的 profile，再按该 profile 全文实施。
  当前只有 go-single-binary-docker：Go 单静态二进制，同时把镜像发到 GHCR，并用 v 前缀 semver tag 打 GitHub Release。
  Use when adding, editing, or reviewing GitHub Actions for a Go single static binary that also publishes a Docker image (tests, GHCR tag policy, semver releases).
  Future workflow specs are additional profiles registered here; do not stretch an existing profile onto another project shape.
---

# Best GitHub Workflows

这个 skill 是 GitHub Actions workflow 规范的目录。每一份规范是一个 **profile**：一种项目形态下的一整套 workflow，包含事件划分、权限、并发、发布物和明确拒绝的做法。

实施时以 profile 全文为准。目录表只负责选择，不是规范本身。

## 何时读取

用户要新建、修改或审查 GitHub Actions，并且仓库的形态能对上下面某一行时，先读本文件，再把对应 profile **从头读到尾**，然后才改 YAML。

审查既有 workflow 时用同一份 profile。对得上的差异按 profile 改回去；profile 写成「不要做」的方案，不要当成可以顺手补上的增强。

## Profile 目录

| id | 适用 | 文档 |
| --- | --- | --- |
| `go-single-binary-docker` | 仓库根目录一个 `package main`，`CGO_ENABLED=0` 的静态二进制，同时发布一份 Docker 镜像到 GHCR | [profiles/go-single-binary-docker.md](profiles/go-single-binary-docker.md) |

没有登记的 profile 就不存在。不要根据 id 的命名方式提前发明一份来用。

## Profile 之间的边界

- 一个仓库只用一份 profile。Profile 之间不继承、不覆盖、不互相引用规则。
- 新的项目形态是一份新 profile，不是写进现有 profile 的新小节。
- 在第二份 profile 确实需要同一条规则之前，不要抽 `shared/`。那之前的重复是刻意的。

## 每份 profile 都要有的内容

新增 profile 时，文档里这些节都要在，名字可以按该形态调整：

1. **适用与不适用。** 写清哪种仓库用它，以及邻近但必须排除的形态。
2. **仓库参数。** 实施时替换的名字（二进制名、缓存 scope、并发组前缀等），并给出一套参考值。
3. **Workflow 文件与事件。** 每个文件拥有哪些事件；同一事件只跑一次测试；测试失败不能发布。
4. **权限。** Workflow 级是上限；job 级整组替换、不合并；被调用的 workflow 不能把调用方的权限放大。
5. **并发。** 组名、谁可以取消谁、调用方和 `workflow_call` 不能共用同一个 group。
6. **明确拒绝的做法。** 写上容易被当成简化或补全、但必须留住的决定，以及原因。
7. **要同步的说明。** 除了 YAML 以外，哪些 README 或 Release 正文必须和规范一起改。
8. **文件头注释。** 只打开 YAML 的人必须能看到不可回退的理由。不写「见 skill」就结束。

## 新增一份 profile

1. id 使用小写字母和连字符，描述项目形态，不描述某一个仓库。`go-single-binary-docker` 是这个形式。
2. 添加 `profiles/<id>.md`，按上一节写完。
3. 在上面的目录表加一行。
4. 不要为了容纳新 profile 去改旧 profile 的规则。

## 没有 profile 能覆盖时

说明目录里有哪些 profile、当前仓库哪一条对不上，然后停。不要放宽 `go-single-binary-docker` 去套这些仓库：

- 需要 cgo，或必须在 macOS / Windows runner 上编译
- 仓库根目录不是唯一的 `package main`
- 纯库：没有要发布的二进制，也没有镜像
- 一次要推多份镜像，或 monorepo 里有多条独立的发布线
- 默认分支的触发不能落成一个字面量分支名，同时又要求从非默认分支推 `:latest`
