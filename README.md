# Harness Package Manager

Versioned, composable distribution of skills, commands, agents, hooks, scripts and MCP configs across Claude Code, Kiro, GitHub Copilot and Codex, from a registry deployed per client into repos we do not own.

This is the workspace root, managed by [metastack](https://github.com/StackCube/metastack). It holds the product requirements and the list of sibling repos the product is built across. The sibling repos are checked out under `./repos/` and are not tracked here.

## Quick start

```bash
brew install stackcube/tap/metastack   # once
metastack clone     # fetch every repo listed in metastack.yaml into ./repos/
metastack doctor    # verify the toolchain
metastack status    # branch + working-tree state per repo
```

## Repos

| Name | Repo | What it is |
|---|---|---|
| `registry` | StackCube/harness-package-manager-registry | Registry instance: Cloudflare Worker behind Access, R2 for archives, D1 for the index. One deploy config per client. |
| `cli` | StackCube/harness-package-manager-cli | The `hpm` CLI and the harness adapters. |
| `packages` | StackCube/harness-package-manager-packages | Our own package source, one directory per package with per-harness folders. |
| `tap` | StackCube/homebrew-tap | Shared Homebrew tap for the StackCube suite. The `hpm` release publishes its formula here. |
| `docs` | StackCube/docs.stackcube.dev | Source for docs.stackcube.dev. User-facing `hpm` documentation lives here. |

`metastack.yaml` is the source of truth for this list. Add a repo with `metastack add StackCube/<repo>`.

## Documents

- [Product requirements document](docs/prd.md) — the full PRD, currently draft v0.7
- [Iteration 1 technical design](docs/superpowers/specs/2026-09-19-iteration-1-technical-design.md) — contract, architecture, testing and the eight milestones
- [Iteration 1 prototype](docs/prd.md#20-iteration-1-prototype) — what the repos above build first
- [Decision log](docs/prd.md#22-decision-log) — every settled question and where it is written up

## The idea in three lines

- Every component is its own package with its own semver. Meta packages compose them into toolkits.
- Assets are vendored into the project where the harness expects them. A lockfile records what came from where, at which version, with which hash.
- A registry instance per client, a Cloudflare Worker behind Access with R2 for archives, owns the distributables. It never owns source.

## Status

Draft. The CLI name `hpm` is a placeholder. The `registry`, `cli` and `packages` repos exist but are empty. A `contract` repo (OpenAPI, JSON Schema, golden vectors) is created in milestone 0 and added to `metastack.yaml` then.
