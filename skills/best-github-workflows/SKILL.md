---
name: best-github-workflows
description: >-
  Catalog of GitHub Actions workflow profiles. When creating, editing, or reviewing workflows, match a registered profile and follow that document in full.
  The current profile is go-single-binary-docker: one Go static binary, a GHCR image, and a GitHub Release from a v-prefixed semver tag.
  Use when adding or changing CI for a Go single static binary that also publishes a Docker image (tests, GHCR tag policy, semver releases).
  Future workflow specs are additional profiles registered here. Do not stretch an existing profile onto another project shape.
---

# Best GitHub Workflows

This skill is a catalog of GitHub Actions workflow specifications. Each specification is a **profile**: the full workflow set for one project shape, including which events each file owns, permissions, concurrency, what gets published, and the choices that look like simplifications but have to stay.

Follow the profile document, not the catalog row. The row only chooses which document to read.

## When to read this

Read this file when the user wants GitHub Actions created, edited, or reviewed and the repository matches a row below. Then read that profile from start to finish before changing YAML.

Use the same profile to review an existing workflow. Where the repository differs, change it to match the profile. A choice the profile rejects is not an enhancement to add back.

## Profile catalog

| id | Applies to | Document |
| --- | --- | --- |
| `go-single-binary-docker` | One `package main` at the repository root, a `CGO_ENABLED=0` static binary, and one Docker image published to GHCR | [profiles/go-single-binary-docker.md](profiles/go-single-binary-docker.md) |

A profile that is not in this table does not exist. Do not invent one from the shape of an id.

## How profiles relate

- A repository uses one profile. Profiles do not inherit, override, or cite each other's rules.
- A new project shape is a new profile, not a new section inside an existing one.
- Do not extract a `shared/` document until a second profile needs the same rule verbatim. Until then, duplication is intentional.

## What every profile contains

A new profile document needs these sections. Names can follow the shape being specified.

1. **Applicability.** Which repositories use it, and the nearby shapes that are out of scope.
2. **Parameters.** Names the implementing repository fills in (binary name, cache scope, concurrency prefix, and so on), with one set of reference values.
3. **Workflow files and events.** Which file owns which events, one test run per event, and a failing test cannot publish.
4. **Permissions.** The workflow-level block is the ceiling. Job-level permissions replace that set; they do not merge. A called workflow cannot widen the caller's grant.
5. **Concurrency.** Group names, what may cancel what, and why a caller and its `workflow_call` callee must not share a group.
6. **Rejected choices.** Decisions that are easy to "simplify" or complete, and why they stay.
7. **Text that must move with the workflows.** Which README section or release body stays in sync with the YAML.
8. **File headers.** Someone opening only the YAML has to see why a decision cannot be reverted. A header that says "see the skill" is not enough.

## Adding a profile

1. Use a lowercase hyphenated id that names the project shape, not a repository. `go-single-binary-docker` is that form.
2. Add `profiles/<id>.md` with the sections above.
3. Add one row to the catalog.
4. Do not change an older profile's rules to make room for the new one.

## When no profile fits

Say which profiles exist and which condition of the current repository fails, then stop. Do not widen `go-single-binary-docker` to cover:

- cgo, or a binary that has to be compiled on a macOS or Windows runner
- a repository whose only `package main` is not at the root
- a library with no release binary and no image
- more than one image, or a monorepo with several independent release lines
- a default branch that cannot be named literally in the trigger while still publishing `:latest` from a non-default branch
