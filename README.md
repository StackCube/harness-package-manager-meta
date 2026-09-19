# Harness Package Manager

Versioned, composable distribution of skills, commands, agents, hooks, scripts and MCP configs across Claude Code, Kiro, GitHub Copilot and Codex, from a registry deployed per client into repos we do not own.

This repo holds the product requirements and design decisions. Implementation lives elsewhere.

- [Product requirements document](docs/prd.md) — the full PRD, currently draft v0.5
- [Decision log](docs/prd.md#22-decision-log) — every settled question and where it is written up
- [Iteration 1 prototype](docs/prd.md#20-iteration-1-prototype) — the cut we are building first
- [Open questions](docs/prd.md#23-open-questions)

## The idea in three lines

- Every component is its own package with its own semver. Meta packages compose them into toolkits.
- Assets are vendored into the project where the harness expects them. A lockfile records what came from where, at which version, with which hash.
- A registry instance per client, a Cloudflare Worker behind Access with R2 for archives, owns the distributables. It never owns source.

## Status

Draft. The CLI name `hpm` is a placeholder.
