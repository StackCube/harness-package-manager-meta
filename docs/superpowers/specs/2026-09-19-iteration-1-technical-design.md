# Iteration 1 technical design

**Status: approved design · 19 Sep 2026 · implements [PRD](../../prd.md) §20**
Owner: Rick Whalley

This is the umbrella design for iteration 1. It fixes the cross-cutting decisions: the wire contract, how the repos fit together, the registry and CLI architecture, the merge engine, the testing strategy and the milestone order. Each milestone in section 8 gets its own implementation plan, written when we reach it.

The PRD says what the product does and why. This document says how iteration 1 is built. Where the two disagree, section 9 lists the amendment, and the PRD has been updated to match (v0.7).

## 1. Repos

| Name | Repo | Language | Role |
|---|---|---|---|
| `contract` | StackCube/harness-package-manager-contract (created in M0) | OpenAPI, JSON Schema | The wire contract and golden vectors. Imported into `cli` and `registry`. |
| `registry` | StackCube/harness-package-manager-registry | TypeScript | Worker, D1 migrations, Pulumi program, deploy script. |
| `cli` | StackCube/harness-package-manager-cli | Go | The `hpm` binary, adapters, end-to-end tests. |
| `packages` | StackCube/harness-package-manager-packages | Markdown, scripts | Our own package source. |
| `tap` | StackCube/homebrew-tap | Ruby | Shared Homebrew tap. Receives `Formula/hpm.rb`. |
| `docs` | StackCube/docs.stackcube.dev | Static HTML | Receives an `/hpm/` section. |

`contract` is imported into `cli` and `registry` at `contract/` as the exact tree of a tag, by each repo's `scripts/contract-pull.sh`. This is a plain tree import, not `git subtree`: consumers never edit `contract/`, so there is nothing to merge, and it does not depend on how the commit that last imported the contract was merged — squash, rebase or merge commit all work the same. `contract.lock` records the tag and the tree hash, and `scripts/check-contract.sh` verifies both offline in CI, along with `contract/VERSION`. It also runs the vector tests.

## 2. Contract

```
openapi.yaml            OpenAPI 3.1: the four iteration 1 routes, error model, auth schemes
schemas/                JSON Schema 2020-12, referenced from openapi.yaml
  package-manifest.json   hpm.json in a package
  project-manifest.json   hpm.json in a project
  lockfile.json           hpm.lock
  index.json              GET /v1/index response
spec/                   normative prose
  archive.md              archive format, hashing, canonical JSON, path rules, refusal order
  errors.md               the one error shape, every code with its HTTP status and meaning
  harnesses.md            harness ids and the id → package folder table
  semver.md               versions, ranges and the pick rule
  changelog.md            CHANGELOG.md section parsing
vectors/                golden fixtures both implementations must pass
  archives/   directory → expected tar.gz sha256 and per-file hashes
  manifests/  valid/ and invalid/, each invalid case with its expected error code
  semver/     ranges × versions → expected pick
  changelog/  CHANGELOG.md → parsed versions, trigger-line cases
  canonical/  JSON value → canonical bytes and hash
CHANGELOG.md            the contract is semver-tagged
```

### Decisions

- **Routes are prefixed `/v1`.** `GET /v1/index`, `GET /v1/pkg/:scope/:name`, `GET /v1/archive/:scope/:name/:version`, `PUT /v1/pkg/:scope/:name/:version`.
- **One error shape.** `{ "code": string, "message": string, "details": object }`. Codes are stable and enumerated in the OpenAPI spec: `unauthenticated`, `scope_forbidden`, `version_exists`, `version_not_higher`, `manifest_invalid`, `name_mismatch`, `readme_missing`, `changelog_missing`, `trigger_line_missing`, `scaffold_marker_present`, `env_file_present`, `path_forbidden`, `archive_too_large`, `archive_invalid`, `not_found`, `bad_request`, `internal_error`. The CLI switches on `code`, never on message text, and treats an unrecognised code as a generic failure, so adding a code is a minor change. `spec/errors.md` maps every code to its HTTP status. Infrastructure in front of the registry may answer with something that is not JSON at all, so the CLI must handle a non-JSON error body.
- **Archive format.** `tar.gz`. The root of the archive is the package directory's content, with `hpm.json` at the top level. The CLI packs deterministically: paths sorted bytewise, mtime, uid and gid zeroed, modes limited to 0644 and 0755, no extended headers. The registry stores the bytes it receives, so determinism is for reproducibility, not correctness.
- **Refused in an archive.** Symlinks and hard links, absolute paths, any path containing `..`, files matching env-file patterns (`.env`, `.env.*` except `.env.example`, `*.env`) at any depth, and archives over 10 MB compressed. The order of the checks is normative: `spec/archive.md` lists them in order and the first failure wins, so both implementations refuse with the same code.
- **Harness ids everywhere.** The contract names a harness by id — `claude-code`, `copilot`, `kiro`, `codex` — in the project manifest, in `requires.harness`, in the lockfile and in the index. A package folder name such as `claude/` is not an id; `spec/harnesses.md` holds the mapping, and the registry derives a version's `harnesses` from the archive's top-level folders using it.
- **URIs are a pattern, not `format: uri`.** `^https?://[!-~]+$`. Format assertions differ between validators: the Go one accepted `https://exämple.com/` where Ajv and `@cfworker/json-schema` refused it. A pattern is the same rule in all three. `http` stays allowed so `wrangler dev` and the end-to-end harness can use `http://127.0.0.1`.
- **Hashes.** `sha256-<lowercase hex>` everywhere. Archive integrity is the hash of the stored archive bytes. File hashes are over raw file bytes with no line-ending normalisation. We never rewrite a client's file in order to compare it.
- **Canonical JSON.** Used to hash array elements and scalar values in merge files. Object keys sorted bytewise, no insignificant whitespace, strings and numbers serialised as in RFC 8785. Defined in `spec/archive.md` with vectors.
- **Codegen.** Go types and an HTTP client for the CLI. TypeScript types from `openapi-typescript`. The Worker validates at runtime against the same schema files with a validator that does not use `eval`, since Workers forbid it. M0 picks the Go generator: whichever of `oapi-codegen` and `ogen` handles our 3.1 spec without workarounds. The vectors are the real guard against drift. Generated types are a convenience.
- **Editor support.** `hpm.json` and `hpm.lock` carry a `$schema` URL.

## 3. Registry

TypeScript Worker. Hono for routing, `jose` for JWT verification.

```
src/
  auth/        verify JWT → Identity; scope authorisation
  publish/     untar, validate, changelog and trigger checks
  routes/      index, pkg, archive, publish
  store/       D1 queries and R2 access. The only modules that touch bindings.
migrations/    D1 SQL
infra/         Pulumi program, one stack per client
scripts/deploy.ts
contract/      tree import of a contract tag, pinned in contract.lock
```

### Identity seam

Auth middleware is driven by configuration: `issuer`, `jwksUrl`, `audience`, `tokenHeader`, and a claim mapping. Cloudflare Access is one configuration, with `tokenHeader` set to `Cf-Access-Jwt-Assertion`. A direct OIDC issuer with `Authorization: Bearer` is another, later, with no code change in the routes.

The middleware produces a neutral identity:

```ts
type Identity = { kind: "person" | "service"; id: string; groups: string[] }
```

A person's `id` is the email claim. An Access service token has no email, so its `id` is the `common_name` claim and its kind is `service`. The Worker has no bypass path. Local development and tests point the same configuration at a test issuer that serves a local JWKS.

Access federates to each client's identity provider, whether Entra, Google, Okta or generic SAML and OIDC, so enterprise login works in iteration 1. The seam exists so that Cloudflare can be replaced, not because Access blocks anything.

### Storage

D1, three tables:

- `packages(scope, name, created_at)`, primary key `(scope, name)`.
- `versions(scope, name, version, integrity, size, manifest_json, harnesses, readme, changelog_entry, triggers_json, published_by, published_at)`, primary key `(scope, name, version)`.
- `scope_members(scope, principal, kind)`, where kind is `email`, `group` or `service`. Seeded idempotently by the deploy script from the client configuration.

R2 key: `archives/@<scope>/<name>/<version>.tgz`.

### Publish pipeline

In order, stopping at the first refusal:

1. Identity is a member of the scope, directly or through a group.
2. Body is within the size cap.
3. Gunzip and untar in memory. Path rules and env-file patterns.
4. `hpm.json` validates against the schema. Name and version match the URL.
5. `README.md` is present.
6. `CHANGELOG.md` has a non-empty section for this version. Always, including the first version.
7. The version is higher than every published version of the package: equal to a published version is `version_exists`, lower than the highest published version is `version_not_higher`.
8. Trigger rule: for each `SKILL.md`, hash the frontmatter `description`. If any hash differs from the previous version's `triggers_json`, the changelog section must contain a line starting `- Trigger:`.
9. No file contains the scaffold marker `HPM-TODO`.
10. Conditional put to R2, then insert into D1.

Anything that throws rather than refusing is a `500 internal_error`. A Hono `onError` handler answering in the contract's error shape lands in M1, with the first route that can throw; M0 only adds the code to the contract.

The D1 primary key is the immutability guard. A duplicate insert returns `409 version_exists`. If the insert fails after the R2 put, the orphaned object is harmless, and a retry may overwrite an object that has no row.

`triggers_json` is stored at publish time so that the next publish compares against a row and never re-reads an old archive.

### Reads

`GET /v1/index` is one query over `versions`, optionally filtered by scope, returned with an `ETag`. Latest means the highest semver. Dist-tags come later. `GET /v1/pkg/...` returns the latest version's metadata and README. `GET /v1/archive/...` streams from R2.

### Deploy

Pulumi, in TypeScript, owns infrastructure. One stack per client, configured in `Pulumi.<client>.yaml`. It creates the R2 bucket, the D1 database, the DNS record, the Access application with its policy, and a CI service token, and exports their identifiers.

Wrangler owns code. `scripts/deploy.ts <client>` runs `pulumi up`, renders the wrangler configuration from the stack outputs, applies D1 migrations, runs `wrangler deploy`, seeds `scope_members`, and finishes with the smoke test from section 7. Our own instance is the first stack.

## 4. CLI

Go. `cobra` for commands, `Masterminds/semver` for ranges. `hpm` never runs git. It reads and writes files.

```
cmd/hpm/             cobra wiring only: flags → call into internal → render
contract/            tree import of a contract tag, pinned in contract.lock
internal/
  api/               generated client and types
  manifest/          package manifest, project manifest, lockfile: load, validate, deterministic write
  auth/              AuthProvider: cloudflare-access, service-token
  registry/          Client interface: Index, Package, Archive, Publish.
                     ETag index cache. Archive cache in the user cache dir, keyed by integrity.
  resolve/           pure: root deps + indexes + existing lock → resolved set or conflict
  archive/           deterministic pack, safe unpack, hashing
  tree/              read-only Snapshot of the project. Real filesystem or in-memory.
  adapter/           Adapter interface and the claude implementation
  merge/             Markdown blocks, JSON leaf merge
  plan/              Plan types, builder, conflict checks
  apply/             executes a Plan
  status/ adopt/ pack/ scaffold/ changelog/
  ui/                prompts, tables, diff display
```

### Plan, then apply

Every mutating command has the same shape:

1. Load the manifest and lockfile.
2. Fetch indexes for the registries involved.
3. Resolve.
4. Snapshot the tree.
5. Build a `Plan`: file writes, file deletes, merge-file edits, `.hpm/` writes, the new lockfile. Every operation names its owning package.
6. Check the plan. Path ownership conflicts, short-name collisions, scalar merge conflicts, and any write that would overwrite a `modified` package all fail here, before anything touches disk.
7. Apply. Each write goes to a temp file and is renamed into place. The lockfile is written last.

`--dry-run` stops after step 6 and prints the plan. If apply fails midway the lockfile is untouched, and because every write is idempotent, running the command again converges.

Rejected: imperative install with a rollback journal, which turns "fail before touching disk" into "undo after touching disk" in a repo we do not own. Also rejected: staging the desired harness directory and syncing it, which cannot express merge files and so needs a second mechanism anyway.

### Resolver

Packages are grouped by registry through the project's scope map. A dependency whose scope routes to a different registry is an error, per PRD §7.

Within a registry the resolver iterates to a fixpoint. It collects every range on each package, picks the highest version satisfying all of them, adds that version's dependencies, and repeats until nothing changes. If no version satisfies the ranges, it fails and names every requester with its range. `add` and `install` prefer versions already in the lockfile. `update [pkg]` releases the lock on its targets.

There is no backtracking in iteration 1. With caret ranges and one version per package, a fixpoint covers the realistic cases. The resolver is one pure function, so a full solver can replace it behind the same signature.

### Auth

```go
type AuthProvider interface {
    Login(ctx context.Context, registryURL string) error
    Authorize(ctx context.Context, req *http.Request) error
}
```

- `cloudflare-access`: `Login` runs `cloudflared access login <url>`. `Authorize` obtains the token from `cloudflared access token` and sets it on the request. `cloudflared` is a prerequisite for people, and `hpm login` says how to install it when it is missing.
- `service-token`: reads `HPM_ACCESS_CLIENT_ID` and `HPM_ACCESS_CLIENT_SECRET` and sets the Access service-token headers. When both variables are set this provider wins, which is how CI works.

The project manifest's `registries` map accepts a URL string, or an object `{ "url": "...", "auth": "cloudflare-access" }`. The string is shorthand for the object with the default auth. A later `oidc` provider slots in here.

### Commands

| Command | Path through the modules |
|---|---|
| `login` | auth |
| `init` | adapter detect → write project manifest |
| `add`, `install`, `update` | the full plan-then-apply shape |
| `status [--ci]` | lockfile + snapshot, plus the index for outdated |
| `adopt` | adapter reverse listing + indexes → a plan that writes only `.hpm/` and the lockfile |
| `new`, `version` | scaffold, changelog |
| `publish <dir>` | archive → registry |
| `publish <pkg>` | pack from tree → archive → registry → lockfile bump |
| `search`, `info` | registry |

`install` is strict. It installs exactly what the lockfile says, verifies integrity, and fails if the lockfile is missing or does not satisfy the manifest.

`publish` runs the same checks as the registry locally first, so that the common refusals appear without a round trip. The registry remains the authority.

### Distribution

The same pattern as `metastack`: cross-compiled binaries uploaded to `updates.stackcube.dev/hpm/vX.Y.Z/`, and `Formula/hpm.rb` in the tap pointing at them. Self-update through the `StackCube/update` library is a later addition.

## 5. Claude Code adapter and merge engine

### Adapter

```go
type Adapter interface {
    ID() string                              // "claude-code"
    PackageDir() string                      // "claude"
    Detect(tree.Snapshot) bool
    MapIn(pkgPath string) (Target, bool)     // forward: install
    MapOut(projPath string) (string, bool)   // reverse: adopt, pack
    ListAssets(tree.Snapshot) []Candidate    // adopt
}
```

For Claude Code, `claude/**` maps to `.claude/**`, with three reserved fragments:

| Fragment | Target | Kind |
|---|---|---|
| `claude/CLAUDE.append.md` | `CLAUDE.md` | Markdown block |
| `claude/settings.merge.json` | `.claude/settings.json` | JSON merge |
| `claude/mcp.merge.json` | `.mcp.json` | JSON merge |

Any other file matching `*.merge.*` or `*.append.*` is refused at pack time and at publish. The adapter never reads or writes `.claude/settings.local.json`. The executable bit comes from the archive mode.

### Markdown blocks

```
<!-- hpm:begin @org/conventions 5.0.1 -->
...
<!-- hpm:end @org/conventions -->
```

Install appends the block at the end of the file, using the file's existing line-ending style, and creates the file if it does not exist. Update replaces the block in place. Remove deletes it. The lockfile stores the hash of the content between the markers. Missing markers or a different hash make the package `modified`.

### JSON merge

Library: `tailscale/hujson`, which parses JSON with comments into a syntax tree that keeps comments and whitespace, and serialises back byte for byte outside edited nodes.

Rules, refining PRD §8:

- **Containers are never owned. Leaves are.** Objects and arrays are created when missing and recorded as `createdContainers`. Owned leaves are scalars, identified by JSON Pointer, and array elements, identified by the hash of their canonical JSON. An array element is atomic. The merge never recurses into one.
- A scalar already present with an equal value is fine and is not owned. A different value is a conflict that fails the plan and shows both values.
- An array element already present with identical content is not inserted and is not owned.
- Remove deletes the owned leaves, then prunes only the containers this package created that are now empty.
- Update is remove then insert, computed in the plan so that the file is written once.
- Inserted text matches the indentation detected in the target. The file is never reformatted as a whole.

Leaf ownership is what lets package B append a hook to `hooks.PostToolUse` after package A created that array, and lets either be removed without disturbing the other.

Lockfile shape per package:

```jsonc
"managedEntries": {
  ".claude/settings.json": {
    "leaves": [
      { "ptr": "/hooks/PostToolUse", "element": "sha256-…", "indexHint": 0 }
    ],
    "createdContainers": ["/hooks", "/hooks/PostToolUse"]
  },
  ".mcp.json": {
    "leaves": [
      { "ptr": "/mcpServers/jira/command", "value": "sha256-…" },
      { "ptr": "/mcpServers/jira/args", "element": "sha256-…", "indexHint": 0 }
    ],
    "createdContainers": ["/mcpServers/jira", "/mcpServers/jira/args"]
  }
},
"managedBlocks": { "CLAUDE.md": "sha256-…" }
```

Drift for a scalar is a changed value hash. Drift for an array element is the recorded hash no longer being present in the array.

### Reverse direction

Pack, used by publish from the tree, maps plain files back through `MapOut`. A Markdown fragment is the block's current content. A JSON fragment is rebuilt from the current values of the owned leaves.

One limit. A hand-edited array element no longer matches its recorded hash, so it cannot be found by hash. Pack falls back to `indexHint`. If the element at that index is not owned by another package, pack shows it and asks for confirmation. Otherwise publish stops and names the entry. Edited scalars and edited Markdown blocks round-trip without help.

### Risk handling

The merge milestone starts from a golden corpus: real `settings.json` and `.mcp.json` files, including ones with comments, tabs, CRLF line endings and trailing commas, each with the exact expected bytes after install, update and remove. If hujson cannot make minimal edits against that corpus, the fallback is our own text splicer working from hujson's node offsets, behind the same `merge` interface.

### Aliases

Deferred to iteration 2. Short-name collisions are still detected in iteration 1, because they are path ownership conflicts, and the error names both packages. A non-empty `aliases` field in the project manifest is rejected as not yet supported.

### States in iteration 1

`managed`, `modified` and `unmatched`, plus the derived `outdated` and the recorded `missingHarness`. `status` also reports a `.hpm/` entry with no lockfile package, or the reverse, as corrupt.

## 6. `.hpm/` directory

As PRD §10. Install writes each package's `hpm.json`, `README.md` and `CHANGELOG.md` to `.hpm/packages/@scope/name/`. These writes are part of the plan. Publish from the tree reads the manifest from there and refuses if its name or scope differs from the lockfile's.

## 7. Testing

Test-first throughout. Most logic is pure, so most tests need no filesystem or network.

1. **Contract repo.** Lint `openapi.yaml`. Validate every example and every `vectors/manifests/valid` file against the schemas, and assert every `invalid` file fails with its stated code. Vectors are generated once by the Go implementation, reviewed by hand, and committed. After that they are the authority.
2. **CLI unit tests.** Table-driven over the in-memory snapshot.
   - Resolver and semver against the contract vectors.
   - `plan`: golden files of the rendered plan per scenario. Every conflict type asserts zero writes.
   - `merge`: the golden corpus, byte exact. One property test: install then remove restores the original bytes.
   - `archive`: packing twice gives the same hash. Unpack refuses symlinks, `..` and absolute paths.
   - `apply`: against a temp directory, with a failure injected midway. The lockfile is untouched and a re-run converges.
3. **Registry tests.** Vitest with `@cloudflare/vitest-pool-workers`, which runs real `workerd` with local D1 and R2. A test issuer generates a keypair and serves JWKS. One test per error code. Immutability under concurrent publish of the same version. Scope authorisation for person, group and service identities. Index `ETag` behaviour.
4. **End to end.** In the `cli` repo, against the real Worker under `wrangler dev`, from `../registry` locally and a pinned ref in CI. The only test double is a small fake Access proxy that turns service-token headers into a signed `Cf-Access-Jwt-Assertion` header, so the CLI's real auth path and the Worker's real verification path are both exercised. The script:
   1. Publish from a directory.
   2. `add` a meta package. Status is clean.
   3. Hand-edit a file. Status reports `modified`, and `--ci` exits non-zero.
   4. Publish from the tree. The package returns to managed at the new version.
   5. In a second project, `update` prints the changelog lines crossed.
   6. `adopt` on a tree of hand-copied files finds exact matches, near matches and unmatched files.
   7. Republish and a forbidden scope are refused with the right codes.
5. **Deployed smoke.** The last step of `deploy <client>`. Without a token, Access stops the request. With the CI service token, `GET /v1/index` returns 200. A publish and fetch round trip uses `@smoke/ping` at `0.0.<epoch>`, in a scope whose only member is that service token.
6. **Real-tree acceptance.** Manual. `adopt` and `add` on a branch of each of the four projects. Measure the match rate against the 80% target and review the diff for noise. Every surprising diff becomes a corpus case.

CI is GitHub Actions in each repo: lint, unit tests, the offline `contract/` tag and tree check, and the vector tests. The `cli` repo also runs the end-to-end suite.

## 8. Milestones

Each milestone gets its own implementation plan and ends in something that can be demonstrated.

| # | Milestone | Delivers | Exit criterion |
|---|---|---|---|
| M0 | Contract and scaffolding | The contract repo with `openapi.yaml`, schemas, `spec/` and first vectors. Tree imports of a contract tag in `cli` and `registry`. Go module and Worker skeleton with codegen and CI. `metastack.yaml` gains `contract` and checks for `go` and `pulumi`. | CI green in three repos. Generated types compile on both sides. |
| M1 | Walking skeleton | Registry: all four routes, the auth middleware against the test issuer, core publish checks (scope, schema, version exists, README). CLI: `init`, `publish <dir>`, `add`, `install`, `status`, for one package with no dependencies and plain files only, with the lockfile and plan-then-apply in place. The end-to-end harness with the fake Access proxy. | End-to-end steps 1 to 3 pass for a skill-only package. |
| M2 | Deploy and real identity | The Pulumi stack and `deploy <client>`. `hpm login` through cloudflared. The service-token provider. The smoke test. The first `hpm.rb` in the tap. Getting-started page in docs. | Our own instance is live. A person and the CI token can each publish and install against it. |
| M3 | Package model | The resolver with meta packages, ranges and registry routing. `update` with changelog output. `.hpm/` metadata. `search`, `info`, `new`, `version`. The full publish checks: changelog, trigger rule, `HPM-TODO`, env patterns. | End to end: a toolkit with a shared dependency installs, and `update` crosses versions and prints `Trigger:` lines. |
| M4 | Merge engine | Markdown blocks, JSON leaf merge, drift for managed entries, conflicts that fail before any write. | The golden corpus passes byte for byte. End to end: a toolkit with a hook, an MCP entry and a `CLAUDE.md` block. |
| M5 | Tree as source | `publish <pkg>` from the tree with fragment reconstruction. `new --from`. The `.hpm/` identity guard. Authoring guide in docs. | End-to-end step 4 passes, including an edited hook. |
| M6 | Adopt | Reverse listing, exact match, near match by diff size, prompts, non-interactive mode, the unmatched list. | End-to-end step 6 passes. |
| M7 | Content and rollout | The central repo restructured into `packages/` with one meta toolkit, published. `adopt` on the four projects. The `/hpm/` docs section complete. A second client instance deployed and timed. | The PRD §19 baselines are recorded: match rate, deploy time, time to first install. |

```
M0 → M1 ┬→ M2 ────────────┐
        ├→ M3 ┬→ M5 ──────┼→ M7
        └→ M4 ┘└→ M6 ─────┘
```

M2 comes straight after the skeleton because Access and cloudflared are the largest external unknown. M2, M3 and M4 depend only on M1 and can overlap. M5 needs M3 and M4. M6 needs M3, and adopts managed entries too once M4 exists.

### Inputs needed

- Before M2: the Cloudflare account, zone and identity provider for our own instance.
- Before M7: the location of the existing central harness repo, and the four project repos.

## 9. PRD amendments

Written back to the PRD as v0.7.

| Topic | Amendment |
|---|---|
| CLI language | Go, distributed through `updates.stackcube.dev` and the StackCube Homebrew tap. |
| Contract | A separate repo holding OpenAPI 3.1, JSON Schema and golden vectors, imported at a tagged version into `cli` and `registry`. |
| Index harness vocabulary | Harness ids everywhere; PRD §8 said folders. The id → folder table lives in `spec/harnesses.md`. |
| Contract import | A plain tree import pinned by tree hash in `contract.lock`, not `git subtree`. |
| Routes | Prefixed `/v1`. |
| Identity | Access stays, per §9. Worker verification and CLI login sit behind a provider seam so that a direct OIDC issuer can replace Access later. |
| Project manifest | A `registries` entry may be a URL string or an object with `url` and `auth`. |
| Changelog | An entry is required on every publish, including the first. Resolves the difference between §9 and §17. |
| Scaffold marker | `HPM-TODO`, not a bare `TODO`. Checked by the CLI and the Worker. |
| Merge ownership | Containers are never owned. Scalars are owned by JSON Pointer, array elements by content hash. Created containers are recorded and pruned when empty. |
| Aliases | Deferred to iteration 2. Collisions are still detected and refused. |
| Deploy | Pulumi owns infrastructure, one stack per client. Wrangler owns code and migrations. |
| Resolver | Fixpoint without backtracking in iteration 1. |
