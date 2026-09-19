# M0: Contract and Scaffolding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the contract repo (OpenAPI 3.1, JSON Schemas, specs, first golden vectors), subtree it into the `cli` and `registry` repos, and scaffold both so that CI is green in all three and generated types compile on both sides.

**Architecture:** The contract repo is language-neutral and carries its own self-tests (Bun + Ajv + Redocly). It publishes a bundled OpenAPI file in `dist/` so that consumers never resolve external `$ref`s. `cli` (Go) and `registry` (TypeScript Worker) each hold the contract as a squashed `git subtree` at `contract/`, pinned by a `contract.lock` file that records the tag and the git tree hash, which CI verifies without network access. Both consumers prove agreement by running the manifest vectors through their own JSON Schema validator.

**Tech Stack:** OpenAPI 3.1, JSON Schema 2020-12, Bun 1.2, Ajv, Redocly CLI · Go 1.26, cobra, `santhosh-tekuri/jsonschema/v6`, `oapi-codegen` or `ogen` · TypeScript, Hono, `@cfworker/json-schema`, `openapi-typescript`, Vitest with `@cloudflare/vitest-pool-workers`, wrangler · GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-09-19-iteration-1-technical-design.md` (sections 1, 2, 7 and the M0 row of section 8). Product context: `docs/prd.md` v0.7.

## Global Constraints

- Workspace root is `harness-package-manager-meta/`. Sibling repos live in `repos/<name>/` and are separate git repos. Every path below is relative to the workspace root unless it starts inside a task's stated repo.
- Hash format everywhere: `sha256-<lowercase hex>`, regex `^sha256-[0-9a-f]{64}$`.
- Package names: `^@[a-z0-9][a-z0-9-]*/[a-z0-9][a-z0-9-]*$`.
- Versions: `^(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)(-[0-9A-Za-z.-]+)?$`. No build metadata.
- All routes are prefixed `/v1`.
- Schema `pattern`s must be valid in both ECMAScript and Go RE2, because three validators read them. No lookahead, lookbehind or backreferences. Express exclusions with `not`.
- One error shape: `{ "code": string, "message": string, "details": object }`. Codes, exactly: `unauthenticated`, `scope_forbidden`, `version_exists`, `version_not_higher`, `manifest_invalid`, `name_mismatch`, `readme_missing`, `changelog_missing`, `trigger_line_missing`, `scaffold_marker_present`, `env_file_present`, `path_forbidden`, `archive_too_large`, `archive_invalid`, `not_found`.
- Archive cap: 10 MB compressed. Scaffold marker: `HPM-TODO`.
- The contract is consumed only at a tag, only through `scripts/contract-pull.sh`. Nobody edits `contract/` inside a consumer.
- Go module path: `github.com/StackCube/harness-package-manager-cli`. Go 1.26.
- In `cli` and `registry`, work on a branch named `m0-scaffolding` and open a PR. The contract repo is brand new, so its first content goes to `main`.
- Commit messages end with `Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>`.
- **Outward-facing steps** (creating the GitHub repo, pushing, tagging, opening PRs) are marked ⚠. Confirm with Rick before running each one.
- Out of scope for M0: any route behaviour, D1, R2, auth, Pulumi, archive packing, archive vectors (they arrive in M1 with the Go packer, per spec section 7.1).

## File map

```
repos/contract/                       NEW REPO
  package.json  tsconfig.json  .gitignore  README.md  CHANGELOG.md  VERSION
  openapi.yaml
  redocly.yaml
  schemas/{package-manifest,project-manifest,lockfile,index}.json
  spec/{archive,semver,changelog}.md
  vectors/manifests/{valid,invalid}/*.json   vectors/manifests/cases.json
  vectors/index/example.json
  vectors/canonical/cases.json
  vectors/semver/cases.json
  vectors/changelog/*.md  vectors/changelog/cases.json
  scripts/bundle.ts                   redocly bundle + strip $schema/$id
  dist/openapi.bundled.yaml           generated, committed
  test/{schemas,vectors,openapi}.test.ts
  .github/workflows/ci.yml

repos/cli/
  go.mod  Makefile  .gitignore
  contractfs.go                       embeds contract/schemas
  contract/  contract.lock            subtree + pin
  scripts/{contract-pull.sh,check-contract.sh}
  cmd/hpm/main.go
  internal/cli/root.go  root_test.go
  internal/manifest/validate.go  validate_test.go
  internal/api/generate.go  api.gen.go (generated)  api_test.go
  docs/decisions/0001-go-openapi-generator.md
  .github/workflows/ci.yml

repos/registry/
  package.json  tsconfig.json  wrangler.jsonc  vitest.config.ts  .gitignore
  contract/  contract.lock
  scripts/{contract-pull.sh,check-contract.sh}
  src/index.ts  src/errors.ts  src/validate.ts
  src/generated/api.ts (generated)
  test/{app,validate}.test.ts
  .github/workflows/ci.yml

metastack.yaml  README.md             meta repo
```

---

### Task 1: Contract repo, schemas and manifest vectors

**Repo:** `repos/contract` (new)

**Files:**
- Create: everything under `repos/contract/` listed for schemas, `vectors/manifests`, `vectors/index`, `test/schemas.test.ts`, `package.json`, `tsconfig.json`, `.gitignore`, `VERSION`
- Modify: `metastack.yaml` (meta repo) to register the repo

**Interfaces:**
- Produces: four schema files with these `$id`s, which later tasks compile by `$id`:
  - `https://stackcube.dev/hpm/schemas/v1/package-manifest.json`
  - `https://stackcube.dev/hpm/schemas/v1/project-manifest.json`
  - `https://stackcube.dev/hpm/schemas/v1/lockfile.json`
  - `https://stackcube.dev/hpm/schemas/v1/index.json`
- Produces: `vectors/manifests/cases.json`, an array of `{ "file": string, "schema": "package-manifest" | "project-manifest" | "lockfile", "valid": boolean, "reason": string }`. `file` is relative to `vectors/manifests/`. Every invalid case maps to error code `manifest_invalid`.
- Produces: `vectors/index/example.json`, valid against the index schema.

- [ ] **Step 1: ⚠ Create the GitHub repo and register it**

```bash
gh repo create StackCube/harness-package-manager-contract --private \
  --description "Wire contract for harness-package-manager: OpenAPI 3.1, JSON Schema, specs and golden vectors. Subtreed into cli and registry." \
  --add-readme
```

In the meta repo, add to `metastack.yaml` under `repos:`, before the `registry` entry:

```yaml
  # Wire contract: OpenAPI 3.1, JSON Schema, archive and hashing spec,
  # golden vectors. Subtreed into cli and registry at a tag. Design §2.
  - name: contract
    url: StackCube/harness-package-manager-contract

```

Then run `metastack clone` from the workspace root. Expected: `clone  contract`.

- [ ] **Step 2: Tooling**

`repos/contract/package.json`:

```json
{
  "name": "harness-package-manager-contract",
  "private": true,
  "type": "module",
  "scripts": {
    "test": "bun test",
    "lint": "redocly lint openapi.yaml",
    "bundle": "bun scripts/bundle.ts",
    "check": "bun run lint && bun run bundle && git diff --exit-code dist/ && bun test"
  },
  "devDependencies": {
    "@redocly/cli": "latest",
    "@types/bun": "latest",
    "ajv": "^8",
    "ajv-formats": "^3",
    "semver": "^7",
    "@types/semver": "^7",
    "yaml": "^2"
  }
}
```

`repos/contract/tsconfig.json`:

```json
{ "compilerOptions": { "target": "ES2022", "module": "ESNext", "moduleResolution": "bundler",
  "strict": true, "resolveJsonModule": true, "types": ["bun"], "noEmit": true } }
```

`repos/contract/.gitignore`: `node_modules/`
`repos/contract/VERSION`: `0.1.0`

Run `bun install`. Replace each `"latest"` in `package.json` with the caret version bun resolved (read them from `bun pm ls`), so the file pins real versions.

- [ ] **Step 3: Write the failing test**

`repos/contract/test/schemas.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import Ajv2020 from "ajv/dist/2020";
import addFormats from "ajv-formats";
import { readFileSync } from "node:fs";
import { join } from "node:path";

const root = join(import.meta.dir, "..");
const load = (p: string) => JSON.parse(readFileSync(join(root, p), "utf8"));
const names = ["package-manifest", "project-manifest", "lockfile", "index"] as const;

const ajv = new Ajv2020({ strict: true, allErrors: true });
addFormats(ajv);
const validators = Object.fromEntries(
  names.map((n) => [n, ajv.compile(load(`schemas/${n}.json`))]),
);

describe("schemas", () => {
  test.each(names)("%s has the v1 $id", (n) => {
    expect(load(`schemas/${n}.json`).$id).toBe(`https://stackcube.dev/hpm/schemas/v1/${n}.json`);
  });
});

describe("manifest vectors", () => {
  const cases: { file: string; schema: string; valid: boolean; reason: string }[] =
    load("vectors/manifests/cases.json");

  test("there are valid and invalid cases for every file schema", () => {
    for (const s of ["package-manifest", "project-manifest", "lockfile"]) {
      expect(cases.some((c) => c.schema === s && c.valid)).toBe(true);
      expect(cases.some((c) => c.schema === s && !c.valid)).toBe(true);
    }
  });

  test.each(cases)("$file → valid=$valid ($reason)", (c) => {
    expect(validators[c.schema](load(`vectors/manifests/${c.file}`))).toBe(c.valid);
  });
});

test("index example validates", () => {
  expect(validators["index"](load("vectors/index/example.json"))).toBe(true);
});
```

- [ ] **Step 4: Run it and see it fail**

Run: `cd repos/contract && bun test`
Expected: FAIL, `ENOENT ... schemas/package-manifest.json`.

- [ ] **Step 5: Write the four schemas**

`schemas/package-manifest.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://stackcube.dev/hpm/schemas/v1/package-manifest.json",
  "title": "hpm package manifest",
  "type": "object",
  "required": ["name", "version", "description", "keywords", "owners"],
  "additionalProperties": false,
  "properties": {
    "$schema": { "type": "string" },
    "name": { "$ref": "#/$defs/packageName" },
    "version": { "$ref": "#/$defs/version" },
    "description": { "type": "string", "minLength": 1, "maxLength": 280 },
    "keywords": { "type": "array", "items": { "type": "string", "minLength": 1 }, "uniqueItems": true },
    "owners": { "type": "array", "minItems": 1, "items": { "type": "string", "minLength": 1 }, "uniqueItems": true },
    "source": { "type": "string", "format": "uri" },
    "dependencies": {
      "type": "object",
      "propertyNames": { "$ref": "#/$defs/packageName" },
      "additionalProperties": { "$ref": "#/$defs/range" }
    },
    "slots": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["path", "required", "description"],
        "additionalProperties": false,
        "properties": {
          "path": { "$ref": "#/$defs/relativePath" },
          "required": { "type": "boolean" },
          "description": { "type": "string", "minLength": 1 }
        }
      }
    },
    "requires": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "bin": { "type": "array", "items": { "type": "string", "minLength": 1 } },
        "env": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["name", "required", "secret", "description"],
            "additionalProperties": false,
            "properties": {
              "name": { "type": "string", "pattern": "^[A-Z_][A-Z0-9_]*$" },
              "required": { "type": "boolean" },
              "secret": { "type": "boolean" },
              "description": { "type": "string", "minLength": 1 }
            }
          }
        },
        "harness": { "type": "object", "additionalProperties": { "$ref": "#/$defs/range" } }
      }
    }
  },
  "$defs": {
    "packageName": { "type": "string", "pattern": "^@[a-z0-9][a-z0-9-]*/[a-z0-9][a-z0-9-]*$" },
    "version": { "type": "string", "pattern": "^(0|[1-9]\\d*)\\.(0|[1-9]\\d*)\\.(0|[1-9]\\d*)(-[0-9A-Za-z.-]+)?$" },
    "range": { "type": "string", "minLength": 1 },
    "relativePath": {
      "type": "string", "minLength": 1,
      "not": { "anyOf": [{ "pattern": "^/" }, { "pattern": "(^|/)\\.\\.(/|$)" }] }
    }
  }
}
```

`schemas/project-manifest.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://stackcube.dev/hpm/schemas/v1/project-manifest.json",
  "title": "hpm project manifest",
  "type": "object",
  "required": ["harnesses", "registries", "dependencies"],
  "additionalProperties": false,
  "properties": {
    "$schema": { "type": "string" },
    "harnesses": {
      "type": "array", "minItems": 1, "uniqueItems": true,
      "items": { "enum": ["claude-code", "copilot", "kiro", "codex"] }
    },
    "registries": {
      "type": "object",
      "propertyNames": { "pattern": "^@[a-z0-9][a-z0-9-]*$" },
      "additionalProperties": {
        "oneOf": [
          { "type": "string", "format": "uri" },
          {
            "type": "object",
            "required": ["url"],
            "additionalProperties": false,
            "properties": {
              "url": { "type": "string", "format": "uri" },
              "auth": { "enum": ["cloudflare-access"] }
            }
          }
        ]
      }
    },
    "dependencies": {
      "type": "object",
      "propertyNames": { "pattern": "^@[a-z0-9][a-z0-9-]*/[a-z0-9][a-z0-9-]*$" },
      "additionalProperties": { "type": "string", "minLength": 1 }
    },
    "aliases": { "type": "object", "additionalProperties": { "type": "string", "minLength": 1 } },
    "unsupported": { "enum": ["warn", "fail"] }
  }
}
```

`schemas/lockfile.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://stackcube.dev/hpm/schemas/v1/lockfile.json",
  "title": "hpm lockfile",
  "type": "object",
  "required": ["lockfileVersion", "packages"],
  "additionalProperties": false,
  "properties": {
    "$schema": { "type": "string" },
    "lockfileVersion": { "const": 1 },
    "packages": {
      "type": "object",
      "propertyNames": { "pattern": "^@[a-z0-9][a-z0-9-]*/[a-z0-9][a-z0-9-]*$" },
      "additionalProperties": { "$ref": "#/$defs/package" }
    },
    "unmatched": { "type": "array", "items": { "type": "string", "minLength": 1 }, "uniqueItems": true }
  },
  "$defs": {
    "hash": { "type": "string", "pattern": "^sha256-[0-9a-f]{64}$" },
    "pointer": { "type": "string", "pattern": "^(/[^/]*)+$" },
    "package": {
      "type": "object",
      "required": ["version", "registry", "integrity", "state"],
      "additionalProperties": false,
      "properties": {
        "version": { "type": "string", "pattern": "^(0|[1-9]\\d*)\\.(0|[1-9]\\d*)\\.(0|[1-9]\\d*)(-[0-9A-Za-z.-]+)?$" },
        "registry": { "type": "string", "format": "uri" },
        "integrity": { "$ref": "#/$defs/hash" },
        "requestedBy": { "type": "array", "items": { "type": "string", "minLength": 1 } },
        "state": { "enum": ["managed", "modified"] },
        "adopted": { "type": "boolean" },
        "missingHarness": { "type": "array", "items": { "type": "string" }, "uniqueItems": true },
        "files": {
          "type": "object",
          "additionalProperties": { "type": "object", "additionalProperties": { "$ref": "#/$defs/hash" } }
        },
        "managedBlocks": { "type": "object", "additionalProperties": { "$ref": "#/$defs/hash" } },
        "managedEntries": {
          "type": "object",
          "additionalProperties": {
            "type": "object",
            "required": ["leaves", "createdContainers"],
            "additionalProperties": false,
            "properties": {
              "createdContainers": { "type": "array", "items": { "$ref": "#/$defs/pointer" } },
              "leaves": {
                "type": "array",
                "items": {
                  "oneOf": [
                    {
                      "type": "object", "required": ["ptr", "value"], "additionalProperties": false,
                      "properties": { "ptr": { "$ref": "#/$defs/pointer" }, "value": { "$ref": "#/$defs/hash" } }
                    },
                    {
                      "type": "object", "required": ["ptr", "element", "indexHint"], "additionalProperties": false,
                      "properties": {
                        "ptr": { "$ref": "#/$defs/pointer" },
                        "element": { "$ref": "#/$defs/hash" },
                        "indexHint": { "type": "integer", "minimum": 0 }
                      }
                    }
                  ]
                }
              }
            }
          }
        }
      }
    }
  }
}
```

`schemas/index.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://stackcube.dev/hpm/schemas/v1/index.json",
  "title": "hpm registry index",
  "type": "object",
  "required": ["packages"],
  "additionalProperties": false,
  "properties": {
    "packages": { "type": "array", "items": { "$ref": "#/$defs/indexPackage" } }
  },
  "$defs": {
    "indexPackage": {
      "type": "object",
      "required": ["name", "description", "keywords", "versions"],
      "additionalProperties": false,
      "properties": {
        "name": { "type": "string", "pattern": "^@[a-z0-9][a-z0-9-]*/[a-z0-9][a-z0-9-]*$" },
        "description": { "type": "string" },
        "keywords": { "type": "array", "items": { "type": "string" } },
        "versions": { "type": "array", "minItems": 1, "items": { "$ref": "#/$defs/indexVersion" } }
      }
    },
    "indexVersion": {
      "type": "object",
      "required": ["version", "integrity", "size", "dependencies", "harnesses", "publishedAt", "publishedBy"],
      "additionalProperties": false,
      "properties": {
        "version": { "type": "string", "pattern": "^(0|[1-9]\\d*)\\.(0|[1-9]\\d*)\\.(0|[1-9]\\d*)(-[0-9A-Za-z.-]+)?$" },
        "integrity": { "type": "string", "pattern": "^sha256-[0-9a-f]{64}$" },
        "size": { "type": "integer", "minimum": 1 },
        "dependencies": { "type": "object", "additionalProperties": { "type": "string", "minLength": 1 } },
        "harnesses": { "type": "array", "uniqueItems": true, "items": { "enum": ["claude", "copilot", "kiro", "codex"] } },
        "publishedAt": { "type": "string", "format": "date-time" },
        "publishedBy": { "type": "string", "minLength": 1 }
      }
    }
  }
}
```

Note that `harnesses` in the index lists package folder names (`claude`), while the project manifest lists harness ids (`claude-code`). That is deliberate: the index reports which folders an archive ships (PRD §8).

- [ ] **Step 6: Write the vectors**

`vectors/manifests/valid/package-minimal.json`:

```json
{ "name": "@ourorg/tdd-loop", "version": "1.0.0", "description": "Red-green-refactor loop",
  "keywords": [], "owners": ["rick@ourorg.example"] }
```

`vectors/manifests/valid/package-full.json`:

```json
{
  "name": "@ourorg/jira-helper", "version": "1.4.2-next.1",
  "description": "Jira lookups from a session", "keywords": ["jira", "tickets"],
  "owners": ["rick@ourorg.example"], "source": "https://github.com/ourorg/harness",
  "dependencies": { "@ourorg/gh-cli-helpers": "^1.2" },
  "slots": [{ "path": "reference/jira-projects.md", "required": false, "description": "Project keys this repo uses" }],
  "requires": {
    "bin": ["jq", "gh>=2.40"],
    "env": [{ "name": "JIRA_TOKEN", "required": true, "secret": true, "description": "API token, read scope" }],
    "harness": { "claude-code": ">=2.0" }
  }
}
```

`vectors/manifests/valid/package-meta.json`:

```json
{ "name": "@ourorg/backend-toolkit", "version": "3.1.0", "description": "Toolkit for backend services",
  "keywords": ["toolkit"], "owners": ["rick@ourorg.example"],
  "dependencies": { "@ourorg/tdd-loop": "^1.4", "@ourorg/jira-helper": "^1.0" } }
```

`vectors/manifests/valid/project.json`:

```json
{
  "harnesses": ["claude-code"],
  "registries": {
    "@ourorg": "https://hpm.ourorg.example",
    "@acme": { "url": "https://hpm.acme.example", "auth": "cloudflare-access" }
  },
  "dependencies": { "@ourorg/backend-toolkit": "^3.1", "@acme/adr-writer": "~0.4" },
  "aliases": {}, "unsupported": "warn"
}
```

`vectors/manifests/valid/lockfile.json`:

```json
{
  "lockfileVersion": 1,
  "packages": {
    "@ourorg/tdd-loop": {
      "version": "1.4.2", "registry": "https://hpm.ourorg.example",
      "integrity": "sha256-cdab067e9f3beb32d1252cfd63e492592fecbf591b0d08cadb24bb17f3864246",
      "requestedBy": ["@ourorg/backend-toolkit@3.1.0"], "state": "managed",
      "files": { "claude-code": { ".claude/skills/tdd-loop/SKILL.md": "sha256-9d141730c5ec2795cac99c12762aa305d81be46e8a684f574ba9c5f5facb7f15" } }
    },
    "@ourorg/formatter": {
      "version": "1.2.0", "registry": "https://hpm.ourorg.example",
      "integrity": "sha256-6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b",
      "state": "modified", "missingHarness": ["kiro"],
      "managedBlocks": { "CLAUDE.md": "sha256-b5bea41b6c623f7c09f1bf24dcae58ebab3c0cdd90ad966bc43a45b44867e12b" },
      "managedEntries": {
        ".claude/settings.json": {
          "leaves": [
            { "ptr": "/hooks/PostToolUse", "element": "sha256-8bd9f442dfdfd76a25cb9b04d8a305c576cf889a96227a9f169bfe51c57ac943", "indexHint": 0 },
            { "ptr": "/env/FORMAT_ON_SAVE", "value": "sha256-b5bea41b6c623f7c09f1bf24dcae58ebab3c0cdd90ad966bc43a45b44867e12b" }
          ],
          "createdContainers": ["/hooks/PostToolUse", "/env"]
        }
      }
    }
  },
  "unmatched": [".claude/skills/legacy-deploy/SKILL.md"]
}
```

Invalid cases, one defect each. Create each by copying the matching valid file and making exactly the stated change:

| File under `vectors/manifests/invalid/` | Base | Change |
|---|---|---|
| `package-no-scope.json` | package-minimal | `"name": "tdd-loop"` |
| `package-uppercase-name.json` | package-minimal | `"name": "@ourorg/TDD-Loop"` |
| `package-bad-version.json` | package-minimal | `"version": "1.0"` |
| `package-build-metadata.json` | package-minimal | `"version": "1.0.0+build5"` |
| `package-no-owners.json` | package-minimal | `"owners": []` |
| `package-missing-description.json` | package-minimal | remove `description` |
| `package-unknown-field.json` | package-minimal | add `"scripts": {}` |
| `package-slot-escapes.json` | package-full | slot `"path": "../secrets.md"` |
| `package-env-lowercase.json` | package-full | env `"name": "jira_token"` |
| `project-no-harness.json` | project | `"harnesses": []` |
| `project-unknown-harness.json` | project | `"harnesses": ["cursor"]` |
| `project-bad-scope-key.json` | project | rename key `@ourorg` to `ourorg` in `registries` |
| `project-unknown-auth.json` | project | `"auth": "basic"` |
| `project-bad-unsupported.json` | project | `"unsupported": "ignore"` |
| `lockfile-wrong-version.json` | lockfile | `"lockfileVersion": 2` |
| `lockfile-base64-hash.json` | lockfile | tdd-loop `"integrity": "sha256-9f3a=="` |
| `lockfile-unknown-state.json` | lockfile | tdd-loop `"state": "patched"` |
| `lockfile-leaf-both-kinds.json` | lockfile | add `"value": "sha256-b5be…"` (the full hash used above) to the `element` leaf |

`vectors/manifests/cases.json` lists all 5 valid and 18 invalid files in this form. `reason` is the file's base name without the extension, which already states the defect. `schema` follows the file name prefix: `package-` is `package-manifest`, `project` is `project-manifest`, `lockfile` is `lockfile`.

```json
[
  { "file": "valid/package-minimal.json", "schema": "package-manifest", "valid": true, "reason": "package-minimal" },
  { "file": "invalid/package-no-scope.json", "schema": "package-manifest", "valid": false, "reason": "package-no-scope" }
]
```

`vectors/index/example.json`:

```json
{
  "packages": [
    {
      "name": "@ourorg/tdd-loop", "description": "Red-green-refactor loop", "keywords": ["testing", "tdd"],
      "versions": [
        { "version": "1.4.1", "integrity": "sha256-6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b",
          "size": 2048, "dependencies": {}, "harnesses": ["claude"],
          "publishedAt": "2026-09-02T10:00:00Z", "publishedBy": "rick@ourorg.example" },
        { "version": "1.4.2", "integrity": "sha256-cdab067e9f3beb32d1252cfd63e492592fecbf591b0d08cadb24bb17f3864246",
          "size": 2101, "dependencies": { "@ourorg/gh-cli-helpers": "^1.2" }, "harnesses": ["claude", "kiro"],
          "publishedAt": "2026-09-19T09:30:00Z", "publishedBy": "ci-publisher" }
      ]
    }
  ]
}
```

- [ ] **Step 7: Run the tests and see them pass**

Run: `cd repos/contract && bun test`
Expected: PASS, 4 `$id` tests, 1 coverage test, 23 vector cases, 1 index test. If an invalid vector passes validation, fix the schema, not the vector.

- [ ] **Step 8: Commit**

```bash
cd repos/contract && git add -A && git commit -m "Schemas for manifests, lockfile and index, with manifest vectors"
cd ../.. && git add metastack.yaml && git commit -m "Register the contract repo"
```

---

### Task 2: OpenAPI description and bundle

**Repo:** `repos/contract`

**Files:**
- Create: `openapi.yaml`, `redocly.yaml`, `scripts/bundle.ts`, `dist/openapi.bundled.yaml`, `test/openapi.test.ts`

**Interfaces:**
- Consumes: the four schemas from Task 1.
- Produces: `dist/openapi.bundled.yaml`, self-contained, with no `$schema` or `$id` keys inside `components`. Component schema names that consumers' generated types rely on: `Error`, `ErrorCode`, `Index`, `PackageInfo`, `PackageManifest`, `PublishResult`. Operation ids: `getIndex`, `getPackage`, `getArchive`, `publishVersion`.

- [ ] **Step 1: Write the failing test**

`test/openapi.test.ts`:

```ts
import { expect, test } from "bun:test";
import { readFileSync } from "node:fs";
import { join } from "node:path";
import { parse } from "yaml";

const doc = parse(readFileSync(join(import.meta.dir, "..", "dist/openapi.bundled.yaml"), "utf8"));

const CODES = ["unauthenticated", "scope_forbidden", "version_exists", "version_not_higher",
  "manifest_invalid", "name_mismatch", "readme_missing", "changelog_missing", "trigger_line_missing",
  "scaffold_marker_present", "env_file_present", "path_forbidden", "archive_too_large",
  "archive_invalid", "not_found"];

test("is OpenAPI 3.1 with exactly the four v1 routes", () => {
  expect(doc.openapi).toStartWith("3.1");
  expect(Object.keys(doc.paths).sort()).toEqual([
    "/v1/archive/{scope}/{name}/{version}",
    "/v1/index",
    "/v1/pkg/{scope}/{name}",
    "/v1/pkg/{scope}/{name}/{version}",
  ]);
});

test("operation ids are stable", () => {
  const ids = Object.values(doc.paths).flatMap((p: any) => Object.values(p).map((o: any) => o.operationId));
  expect(ids.sort()).toEqual(["getArchive", "getIndex", "getPackage", "publishVersion"]);
});

test("error codes match the design exactly", () => {
  expect(doc.components.schemas.ErrorCode.enum).toEqual(CODES);
});

test("bundle is self-contained and generator-friendly", () => {
  const text = JSON.stringify(doc.components);
  expect(text).not.toContain('"$id"');
  expect(text).not.toContain('"$schema":"https://json-schema.org');
  expect(JSON.stringify(doc)).not.toMatch(/"\$ref":"(?!#)/);
});

test("every $ref resolves inside the bundle", () => {
  const refs: string[] = [];
  const walk = (n: any): void => {
    if (Array.isArray(n)) return n.forEach(walk);
    if (n === null || typeof n !== "object") return;
    if (typeof n.$ref === "string") refs.push(n.$ref);
    Object.values(n).forEach(walk);
  };
  walk(doc);
  expect(refs.length).toBeGreaterThan(0);
  for (const ref of refs) {
    const target = ref.slice(2).split("/").reduce((node: any, key) => node?.[key.replace(/~1/g, "/").replace(/~0/g, "~")], doc);
    expect(target, ref).toBeDefined();
  }
});
```

- [ ] **Step 2: Run it and see it fail**

Run: `bun test test/openapi.test.ts`
Expected: FAIL, `ENOENT ... dist/openapi.bundled.yaml`.

- [ ] **Step 3: Write `openapi.yaml`**

```yaml
openapi: 3.1.0
info:
  title: hpm registry
  version: 0.1.0
  description: Registry instance API for harness-package-manager. Every route sits behind the instance's identity layer.
  license: { name: Proprietary, identifier: LicenseRef-Proprietary }
servers:
  - url: https://{host}
    variables: { host: { default: hpm.example.com } }
security:
  - accessJwt: []
tags:
  - name: read
  - name: publish
paths:
  /v1/index:
    get:
      operationId: getIndex
      tags: [read]
      summary: Every package with its versions
      parameters:
        - name: scope
          in: query
          required: false
          description: Limit to one scope, with the leading at sign, for example @ourorg
          schema: { type: string, pattern: "^@[a-z0-9][a-z0-9-]*$" }
        - name: If-None-Match
          in: header
          required: false
          schema: { type: string }
      responses:
        "200":
          description: The index
          headers:
            ETag: { schema: { type: string } }
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Index" }
        "304": { description: Not modified }
        "401": { $ref: "#/components/responses/Error" }
  /v1/pkg/{scope}/{name}:
    get:
      operationId: getPackage
      tags: [read]
      summary: Metadata and README for the latest version
      parameters:
        - $ref: "#/components/parameters/scope"
        - $ref: "#/components/parameters/name"
      responses:
        "200":
          description: Package info
          content:
            application/json:
              schema: { $ref: "#/components/schemas/PackageInfo" }
        "401": { $ref: "#/components/responses/Error" }
        "404": { $ref: "#/components/responses/Error" }
  /v1/archive/{scope}/{name}/{version}:
    get:
      operationId: getArchive
      tags: [read]
      summary: The archive bytes. Verify against the index integrity before unpacking.
      parameters:
        - $ref: "#/components/parameters/scope"
        - $ref: "#/components/parameters/name"
        - $ref: "#/components/parameters/version"
      responses:
        "200":
          description: tar.gz archive
          content:
            application/gzip: {}
        "401": { $ref: "#/components/responses/Error" }
        "404": { $ref: "#/components/responses/Error" }
  /v1/pkg/{scope}/{name}/{version}:
    put:
      operationId: publishVersion
      tags: [publish]
      summary: Publish a version. The body is the archive. Refused if the version exists.
      parameters:
        - $ref: "#/components/parameters/scope"
        - $ref: "#/components/parameters/name"
        - $ref: "#/components/parameters/version"
      requestBody:
        required: true
        content:
          application/gzip: {}
      responses:
        "201":
          description: Published
          content:
            application/json:
              schema: { $ref: "#/components/schemas/PublishResult" }
        "400": { $ref: "#/components/responses/Error" }
        "401": { $ref: "#/components/responses/Error" }
        "403": { $ref: "#/components/responses/Error" }
        "409": { $ref: "#/components/responses/Error" }
        "413": { $ref: "#/components/responses/Error" }
components:
  securitySchemes:
    accessJwt:
      type: apiKey
      in: header
      name: Cf-Access-Jwt-Assertion
      description: >
        Signed JWT attached by the identity layer. With Cloudflare Access the edge sets this header
        after a person logs in or a service token is presented. A later OIDC provider uses
        Authorization Bearer instead; the Worker reads the header named in its configuration.
  parameters:
    scope:
      name: scope
      in: path
      required: true
      description: Scope with the leading at sign
      schema: { type: string, pattern: "^@[a-z0-9][a-z0-9-]*$" }
    name:
      name: name
      in: path
      required: true
      schema: { type: string, pattern: "^[a-z0-9][a-z0-9-]*$" }
    version:
      name: version
      in: path
      required: true
      schema: { type: string }
  responses:
    Error:
      description: Refusal or failure
      content:
        application/json:
          schema: { $ref: "#/components/schemas/Error" }
  schemas:
    ErrorCode:
      type: string
      enum:
        - unauthenticated
        - scope_forbidden
        - version_exists
        - version_not_higher
        - manifest_invalid
        - name_mismatch
        - readme_missing
        - changelog_missing
        - trigger_line_missing
        - scaffold_marker_present
        - env_file_present
        - path_forbidden
        - archive_too_large
        - archive_invalid
        - not_found
    Error:
      type: object
      required: [code, message, details]
      properties:
        code: { $ref: "#/components/schemas/ErrorCode" }
        message: { type: string }
        details: { type: object, additionalProperties: true }
    Index:
      $ref: "./schemas/index.json"
    PackageManifest:
      $ref: "./schemas/package-manifest.json"
    PackageInfo:
      type: object
      required: [name, latest, manifest, readme, versions]
      properties:
        name: { type: string }
        latest: { type: string }
        manifest: { $ref: "#/components/schemas/PackageManifest" }
        readme: { type: string }
        versions:
          type: array
          items: { type: string }
    PublishResult:
      type: object
      required: [name, version, integrity, size, publishedAt, publishedBy]
      properties:
        name: { type: string }
        version: { type: string }
        integrity: { type: string, pattern: "^sha256-[0-9a-f]{64}$" }
        size: { type: integer }
        publishedAt: { type: string, format: date-time }
        publishedBy: { type: string }
```

`redocly.yaml`:

```yaml
extends: [recommended]
rules:
  no-server-example.com: off
  operation-4xx-response: error
  security-defined: error
```

- [ ] **Step 4: Write the bundler**

`scripts/bundle.ts`. Redocly inlines the external schema files. The script then strips `$schema` and `$id` from everything under `components` and rewrites `#/$defs/x` references, because code generators for OpenAPI do not understand JSON Schema `$defs` inside a component.

```ts
import { $ } from "bun";
import { readFileSync, writeFileSync, mkdirSync } from "node:fs";
import { parse, stringify } from "yaml";

mkdirSync("dist", { recursive: true });
await $`bunx redocly bundle openapi.yaml --output dist/openapi.bundled.yaml`.quiet();

const doc = parse(readFileSync("dist/openapi.bundled.yaml", "utf8"));
const schemas: Record<string, any> = doc.components.schemas;

// Hoist each component's $defs to top-level components, prefixed with the component name.
for (const [name, schema] of Object.entries({ ...schemas })) {
  const defs = schema.$defs ?? {};
  for (const [defName, def] of Object.entries(defs)) {
    schemas[`${name}_${defName}`] = def;
  }
  delete schema.$defs;
  rewrite(schema, name);
  for (const defName of Object.keys(defs)) rewrite(schemas[`${name}_${defName}`], name);
}

function rewrite(node: any, owner: string): void {
  if (Array.isArray(node)) return node.forEach((n) => rewrite(n, owner));
  if (node === null || typeof node !== "object") return;
  delete node.$schema;
  delete node.$id;
  if (typeof node.$ref === "string" && node.$ref.startsWith("#/$defs/")) {
    node.$ref = `#/components/schemas/${owner}_${node.$ref.slice("#/$defs/".length)}`;
  }
  for (const v of Object.values(node)) rewrite(v, owner);
}

writeFileSync("dist/openapi.bundled.yaml", stringify(doc, { lineWidth: 0 }));
console.log("wrote dist/openapi.bundled.yaml");
```

If Redocly emits a `$ref` that still points inside an inlined schema using a path other than `#/$defs/`, the "every $ref resolves" test fails and names it. In that case print the offending `$ref` values and extend `rewrite` to map them the same way. Do not hand-edit `dist/`.

- [ ] **Step 5: Lint, bundle, test**

Run: `bun run lint` — Expected: `Woohoo! Your API description is valid.`
Run: `bun run bundle && bunx redocly lint dist/openapi.bundled.yaml` — Expected: valid.
Run: `bun test` — Expected: all PASS.

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "OpenAPI 3.1 description of the four v1 routes, with a generator-friendly bundle"
```

---

### Task 3: Normative specs and the remaining vectors

**Repo:** `repos/contract`

**Files:**
- Create: `spec/archive.md`, `spec/semver.md`, `spec/changelog.md`, `vectors/canonical/cases.json`, `vectors/semver/cases.json`, `vectors/changelog/basic.md`, `vectors/changelog/trigger.md`, `vectors/changelog/empty-section.md`, `vectors/changelog/cases.json`, `test/vectors.test.ts`

**Interfaces:**
- Produces, for M1 onward (no consumer reads these in M0):
  - `vectors/canonical/cases.json`: `[{ "name": string, "value": any, "canonical": string, "hash": string }]`
  - `vectors/semver/cases.json`: `[{ "name": string, "ranges": string[], "versions": string[], "pick": string | null }]`
  - `vectors/changelog/cases.json`: `[{ "file": string, "version": string, "hasSection": boolean, "nonEmpty": boolean, "hasTriggerLine": boolean, "lines": string[] }]`

- [ ] **Step 1: Write the failing test**

`test/vectors.test.ts`. The contract repo has no canonicaliser or changelog parser of its own, so these tests check that the vectors are internally consistent and agree with a reference semver implementation.

```ts
import { expect, test } from "bun:test";
import { createHash } from "node:crypto";
import { existsSync, readFileSync } from "node:fs";
import { join } from "node:path";
import semver from "semver";

const root = join(import.meta.dir, "..", "vectors");
const load = (p: string) => JSON.parse(readFileSync(join(root, p), "utf8"));

test.each(load("canonical/cases.json"))("canonical: $name", (c: any) => {
  expect(JSON.parse(c.canonical)).toEqual(c.value);
  expect(c.canonical).not.toMatch(/[\n\r\t]| :|: |, /);
  expect(c.hash).toBe("sha256-" + createHash("sha256").update(c.canonical, "utf8").digest("hex"));
});

test.each(load("semver/cases.json"))("semver: $name", (c: any) => {
  const ok = c.versions.filter((v: string) => c.ranges.every((r: string) => semver.satisfies(v, r)));
  const pick = ok.sort(semver.rcompare)[0] ?? null;
  expect(pick).toBe(c.pick);
});

test.each(load("changelog/cases.json"))("changelog: $file @ $version", (c: any) => {
  expect(existsSync(join(root, "changelog", c.file))).toBe(true);
  expect(c.nonEmpty).toBe(c.lines.length > 0);
  expect(c.hasTriggerLine).toBe(c.lines.some((l: string) => l.startsWith("- Trigger:")));
  if (!c.hasSection) expect(c.lines).toEqual([]);
});
```

- [ ] **Step 2: Run it and see it fail**

Run: `bun test test/vectors.test.ts`
Expected: FAIL, `ENOENT ... canonical/cases.json`.

- [ ] **Step 3: Write the vectors**

`vectors/canonical/cases.json`:

```json
[
  { "name": "object keys sort bytewise", "value": { "b": 1, "a": "x" },
    "canonical": "{\"a\":\"x\",\"b\":1}",
    "hash": "sha256-cdab067e9f3beb32d1252cfd63e492592fecbf591b0d08cadb24bb17f3864246" },
  { "name": "hook element, nested keys sort, array order kept",
    "value": { "matcher": "Edit|Write", "hooks": [{ "type": "command", "command": "scripts/format.sh" }] },
    "canonical": "{\"hooks\":[{\"command\":\"scripts/format.sh\",\"type\":\"command\"}],\"matcher\":\"Edit|Write\"}",
    "hash": "sha256-8bd9f442dfdfd76a25cb9b04d8a305c576cf889a96227a9f169bfe51c57ac943" },
  { "name": "string scalar", "value": "npx", "canonical": "\"npx\"",
    "hash": "sha256-9d141730c5ec2795cac99c12762aa305d81be46e8a684f574ba9c5f5facb7f15" },
  { "name": "integer-valued number has no fraction", "value": 1.0, "canonical": "1",
    "hash": "sha256-6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b" },
  { "name": "non-ASCII is raw UTF-8, not escaped", "value": "é", "canonical": "\"é\"",
    "hash": "sha256-f2886017e9c7abacf804b54d64787dce2b611c9544ba21f3affdd126a6e50086" },
  { "name": "boolean", "value": true, "canonical": "true",
    "hash": "sha256-b5bea41b6c623f7c09f1bf24dcae58ebab3c0cdd90ad966bc43a45b44867e12b" },
  { "name": "empty array", "value": [], "canonical": "[]",
    "hash": "sha256-4f53cda18c2baa0c0354bb5f9a3ecbe5ed12ab4d8e11ba873c2f11161202b945" }
]
```

`vectors/semver/cases.json`:

```json
[
  { "name": "single caret picks highest minor", "ranges": ["^1.2"], "versions": ["1.1.9", "1.2.0", "1.4.2", "2.0.0"], "pick": "1.4.2" },
  { "name": "two carets intersect", "ranges": ["^1.1", "^1.2"], "versions": ["1.1.5", "1.2.3", "1.3.0"], "pick": "1.3.0" },
  { "name": "tilde on 0.x stays in the minor", "ranges": ["~0.4"], "versions": ["0.4.0", "0.4.1", "0.5.0"], "pick": "0.4.1" },
  { "name": "caret on 0.x stays in the minor", "ranges": ["^0.4.0"], "versions": ["0.4.1", "0.5.0", "1.0.0"], "pick": "0.4.1" },
  { "name": "exact pin", "ranges": ["1.2.3"], "versions": ["1.2.2", "1.2.3", "1.2.4"], "pick": "1.2.3" },
  { "name": "prereleases are skipped by a plain range", "ranges": ["^1.0"], "versions": ["1.0.0", "1.1.0-next.1"], "pick": "1.0.0" },
  { "name": "disjoint ranges have no pick", "ranges": ["^1.0", "^2.0"], "versions": ["1.5.0", "2.1.0"], "pick": null },
  { "name": "nothing published in range", "ranges": ["^3.0"], "versions": ["1.0.0", "2.0.0"], "pick": null }
]
```

`vectors/changelog/basic.md`:

```markdown
# Changelog

## [1.4.2] - 2026-09-19
- Fixed: stopped suggesting a test runner the project does not use

## [1.4.1] - 2026-09-02
- Fixed: Kiro steering file referenced the Claude slot path
```

`vectors/changelog/trigger.md`:

```markdown
# Changelog

## [1.5.0] - 2026-09-20
- Trigger: also fires when a test file is opened, not only when one is named
- Added: optional conventions slot
```

`vectors/changelog/empty-section.md`:

```markdown
# Changelog

## [2.0.0] - 2026-09-21

## [1.5.0] - 2026-09-20
- Added: optional conventions slot
```

`vectors/changelog/cases.json`:

```json
[
  { "file": "basic.md", "version": "1.4.2", "hasSection": true, "nonEmpty": true, "hasTriggerLine": false,
    "lines": ["- Fixed: stopped suggesting a test runner the project does not use"] },
  { "file": "basic.md", "version": "1.4.1", "hasSection": true, "nonEmpty": true, "hasTriggerLine": false,
    "lines": ["- Fixed: Kiro steering file referenced the Claude slot path"] },
  { "file": "basic.md", "version": "9.9.9", "hasSection": false, "nonEmpty": false, "hasTriggerLine": false, "lines": [] },
  { "file": "trigger.md", "version": "1.5.0", "hasSection": true, "nonEmpty": true, "hasTriggerLine": true,
    "lines": ["- Trigger: also fires when a test file is opened, not only when one is named", "- Added: optional conventions slot"] },
  { "file": "empty-section.md", "version": "2.0.0", "hasSection": true, "nonEmpty": false, "hasTriggerLine": false, "lines": [] }
]
```

- [ ] **Step 4: Write the specs**

`spec/archive.md`:

```markdown
# Archive, hashing and canonical JSON

Normative. Words like MUST are used in the RFC 2119 sense.

## Hash format

Every hash is `sha256-` followed by 64 lowercase hex digits.

- **Archive integrity** is the hash of the archive bytes exactly as stored by the registry.
- **File hash** is the hash of the file's raw bytes. Implementations MUST NOT normalise line endings, encodings or trailing whitespace before hashing.
- **Value hash** and **element hash**, used for entries in merge files, are the hash of the canonical JSON of the value.

## Archive format

An archive is a gzip-compressed POSIX tar (ustar or pax without extended headers). Its root is the content of the package directory: `hpm.json` is at the top level, not inside a folder.

A packer MUST:

- order entries by path, compared bytewise;
- set mtime, uid and gid to 0, and leave uname and gname empty;
- set mode to 0755 for files with any execute bit and 0644 otherwise, and 0755 for directories;
- emit only regular files and directories.

Packing the same directory twice MUST produce identical bytes. The registry stores what it receives, so this is a reproducibility rule, not a validity rule.

## What a registry MUST refuse

| Condition | Error code |
|---|---|
| Compressed body larger than 10 MB (10 × 1024 × 1024 bytes) | `archive_too_large` |
| Not valid gzip or tar | `archive_invalid` |
| A symlink, hard link, device or any entry type other than file or directory | `path_forbidden` |
| An absolute path, or a path with a `..` segment | `path_forbidden` |
| A file whose name is `.env`, starts with `.env.` (except `.env.example`), or ends with `.env` | `env_file_present` |
| A file matching `*.merge.*` or `*.append.*` that is not a known fragment of its harness folder | `path_forbidden` |
| No `hpm.json`, or one that fails the package manifest schema | `manifest_invalid` |
| `hpm.json` name or version differs from the URL | `name_mismatch` |
| No `README.md` at the top level | `readme_missing` |
| `CHANGELOG.md` has no non-empty section for the version, see changelog.md | `changelog_missing` |
| A skill trigger description changed and the section has no `Trigger:` line | `trigger_line_missing` |
| Any file contains the literal `HPM-TODO` | `scaffold_marker_present` |
| The version is not higher than every published version | `version_not_higher` |
| The version already exists | `version_exists` |

A CLI SHOULD run the same checks before upload. The registry is the authority.

Known fragments for the `claude/` folder: `claude/CLAUDE.append.md`, `claude/settings.merge.json`, `claude/mcp.merge.json`.

## Canonical JSON

Canonical JSON follows RFC 8785 (JCS):

- object members sorted by key, compared as UTF-16 code units, which equals bytewise order for ASCII keys;
- no whitespace between tokens;
- strings escaped minimally: `"` and `\` and control characters only. Non-ASCII characters are written as raw UTF-8;
- numbers in the shortest form that round-trips, so `1.0` is `1`;
- array order is preserved.

Vectors: `vectors/canonical/cases.json`.
```

`spec/semver.md`:

```markdown
# Versions and ranges

Versions are SemVer 2.0.0 without build metadata. Ranges use the npm grammar subset: exact (`1.2.3`), caret (`^1.2`), tilde (`~0.4`), comparators (`>=2.0`), and space-separated intersections (`>=1.2 <1.5`).

- `^1.2` means `>=1.2.0 <2.0.0`. `^0.4.0` means `>=0.4.0 <0.5.0`.
- `~0.4` means `>=0.4.0 <0.5.0`.
- A prerelease version satisfies a range only if the range itself names a prerelease on the same major, minor and patch.

**Pick rule.** Given every range that requests a package and the list of published versions, the pick is the highest version that satisfies all ranges, or none.

Vectors: `vectors/semver/cases.json`.
```

`spec/changelog.md`:

```markdown
# Changelog parsing

`CHANGELOG.md` follows Keep a Changelog. A version section starts at a line matching

    ^## \[(?<version>[^\]]+)\](?: - (?<date>\d{4}-\d{2}-\d{2}))?\s*$

and runs to the next line starting with `## ` or the end of the file.

- The section's **lines** are its lines with trailing whitespace removed and blank lines dropped.
- A section is **non-empty** if it has at least one line.
- A section **has a trigger line** if any line starts with `- Trigger:`.

Vectors: `vectors/changelog/cases.json`.
```

- [ ] **Step 5: Run the tests and see them pass**

Run: `bun test`
Expected: all PASS, including 7 canonical, 8 semver and 5 changelog cases.

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "Archive, semver and changelog specs with canonical, semver and changelog vectors"
```

---

### Task 4: Contract CI, README and the v0.1.0 tag

**Repo:** `repos/contract`

**Files:**
- Create: `.github/workflows/ci.yml`, `CHANGELOG.md`
- Modify: `README.md`

**Interfaces:**
- Produces: the annotated git tag `v0.1.0` on the remote, which Tasks 5 and 6 pull.

- [ ] **Step 1: CI workflow**

`.github/workflows/ci.yml`:

```yaml
name: ci
on:
  push: { branches: [main], tags: ["v*"] }
  pull_request:
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun install --frozen-lockfile
      - run: bun run lint
      - name: Bundle is fresh
        run: bun run bundle && git diff --exit-code dist/
      - run: bun test
      - name: VERSION matches the tag
        if: startsWith(github.ref, 'refs/tags/v')
        run: test "v$(cat VERSION)" = "${GITHUB_REF_NAME}"
```

- [ ] **Step 2: README and changelog**

`README.md`:

```markdown
# harness-package-manager-contract

The wire contract between the `hpm` CLI (Go) and the registry Worker (TypeScript).

| Path | What |
|---|---|
| `openapi.yaml` | OpenAPI 3.1 description of the `/v1` routes. Edit this. |
| `dist/openapi.bundled.yaml` | Generated by `bun run bundle`. Consumers generate code from this. Never edit. |
| `schemas/` | JSON Schema 2020-12 for `hpm.json` (package and project), `hpm.lock` and the index. |
| `spec/` | Normative prose: archive format, hashing, canonical JSON, semver, changelog parsing. |
| `vectors/` | Golden fixtures. Both implementations must pass them. |

## Changing the contract

1. Edit `openapi.yaml`, `schemas/`, `spec/` or `vectors/`. A new rule needs a vector.
2. `bun run check`.
3. Bump `VERSION` and add a `CHANGELOG.md` section. Removing or renaming anything is a major bump.
4. Merge, then tag `v$(cat VERSION)` and push the tag.
5. In each consumer run `scripts/contract-pull.sh vX.Y.Z`.

Consumers hold this repo as a squashed git subtree at `contract/` and never edit it in place.
```

`CHANGELOG.md`:

```markdown
# Changelog

## [0.1.0] - 2026-09-19
- Added: OpenAPI 3.1 description of the four v1 routes and the error model
- Added: schemas for the package manifest, project manifest, lockfile and index
- Added: archive, semver and changelog specs
- Added: manifest, index, canonical, semver and changelog vectors
```

- [ ] **Step 3: Full local check**

Run: `bun run check`
Expected: lint valid, no diff in `dist/`, all tests PASS.

- [ ] **Step 4: ⚠ Commit, push, tag**

```bash
git add -A && git commit -m "CI, README and changelog for 0.1.0"
git push origin main
git tag -a v0.1.0 -m "contract 0.1.0" && git push origin v0.1.0
```

Run: `gh run watch --repo StackCube/harness-package-manager-contract`
Expected: both the branch run and the tag run succeed.

---

### Task 5: CLI scaffold

**Repo:** `repos/cli`, branch `m0-scaffolding`

**Files:**
- Create: `go.mod`, `Makefile`, `.gitignore`, `contractfs.go`, `contract.lock`, `scripts/contract-pull.sh`, `scripts/check-contract.sh`, `cmd/hpm/main.go`, `internal/cli/root.go`, `internal/cli/root_test.go`, `internal/manifest/validate.go`, `internal/manifest/validate_test.go`, `internal/api/generate.go`, `internal/api/api_test.go`, `docs/decisions/0001-go-openapi-generator.md`, `.github/workflows/ci.yml`
- Generated: `contract/` (subtree), `internal/api/*.gen.go` or ogen's `oas_*.go`

**Interfaces:**
- Consumes: contract tag `v0.1.0`; schema `$id`s and `vectors/manifests/cases.json` from Task 1; `dist/openapi.bundled.yaml` and component name `Index` from Task 2.
- Produces:
  - `contractfs.Schemas` — an `embed.FS` rooted so that `contract/schemas/<name>.json` is readable.
  - `manifest.Kind` with constants `manifest.PackageManifest`, `manifest.ProjectManifest`, `manifest.Lockfile`.
  - `func manifest.Validate(kind manifest.Kind, data []byte) error` — nil when valid, otherwise an error whose message names the failing location.
  - `func cli.NewRootCmd(version string) *cobra.Command`.
  - Package `internal/api` with a generated type `api.Index`.

- [ ] **Step 1: Branch, module, subtree scripts**

```bash
cd repos/cli && git checkout -b m0-scaffolding
go mod init github.com/StackCube/harness-package-manager-cli
```

`.gitignore`:

```
/bin/
/dist/
```

`scripts/contract-pull.sh`:

```bash
#!/usr/bin/env bash
# Pull the contract subtree at a tag and pin it in contract.lock.
set -euo pipefail
TAG="${1:?usage: contract-pull.sh vX.Y.Z}"
URL="${CONTRACT_URL:-git@github.com:StackCube/harness-package-manager-contract.git}"
cd "$(git rev-parse --show-toplevel)"

git fetch "$URL" "refs/tags/$TAG"
EXPECTED="$(git rev-parse 'FETCH_HEAD^{tree}')"

if [ -d contract ]; then
  git subtree pull --prefix contract "$URL" "$TAG" --squash -m "contract: pull $TAG"
else
  git subtree add --prefix contract "$URL" "$TAG" --squash -m "contract: add $TAG"
fi

ACTUAL="$(git rev-parse HEAD:contract)"
if [ "$ACTUAL" != "$EXPECTED" ]; then
  echo "contract/ tree $ACTUAL does not match $TAG tree $EXPECTED" >&2
  exit 1
fi
printf 'tag=%s\ntree=%s\n' "$TAG" "$ACTUAL" > contract.lock
git add contract.lock
git commit -m "contract: lock $TAG"
```

`scripts/check-contract.sh`:

```bash
#!/usr/bin/env bash
# Fail if contract/ was edited in place or contract.lock is stale. Needs no network.
set -euo pipefail
cd "$(git rev-parse --show-toplevel)"
# shellcheck disable=SC1091
. ./contract.lock
ACTUAL="$(git rev-parse HEAD:contract)"
if [ "$ACTUAL" != "$tree" ]; then
  echo "contract/ is $ACTUAL but contract.lock pins $tag at $tree. Run scripts/contract-pull.sh." >&2
  exit 1
fi
if ! git diff --quiet HEAD -- contract; then
  echo "contract/ has uncommitted edits. It is read-only here." >&2
  exit 1
fi
echo "contract/ matches $tag"
```

```bash
chmod +x scripts/*.sh
git add -A && git commit -m "Go module and contract subtree scripts"
scripts/contract-pull.sh v0.1.0
scripts/check-contract.sh
```

Expected last line: `contract/ matches v0.1.0`. A git tree hash depends only on content, so the subtree's tree equals the tag's root tree. That is what makes the check work offline.

- [ ] **Step 2: Failing test for the root command**

`internal/cli/root_test.go`:

```go
package cli

import (
	"bytes"
	"testing"
)

func TestVersionFlag(t *testing.T) {
	cmd := NewRootCmd("1.2.3")
	var out bytes.Buffer
	cmd.SetOut(&out)
	cmd.SetArgs([]string{"--version"})
	if err := cmd.Execute(); err != nil {
		t.Fatal(err)
	}
	if got, want := out.String(), "hpm 1.2.3\n"; got != want {
		t.Fatalf("got %q, want %q", got, want)
	}
}

func TestUnknownCommandFails(t *testing.T) {
	cmd := NewRootCmd("dev")
	cmd.SetOut(&bytes.Buffer{})
	cmd.SetErr(&bytes.Buffer{})
	cmd.SetArgs([]string{"frobnicate"})
	if err := cmd.Execute(); err == nil {
		t.Fatal("expected an error for an unknown command")
	}
}
```

Run: `go test ./internal/cli/` — Expected: FAIL, `undefined: NewRootCmd`.

- [ ] **Step 3: Root command**

```bash
go get github.com/spf13/cobra@latest
```

`internal/cli/root.go`:

```go
// Package cli wires cobra commands. It holds no logic of its own.
package cli

import "github.com/spf13/cobra"

// NewRootCmd returns the hpm root command. Subcommands are added per milestone.
func NewRootCmd(version string) *cobra.Command {
	cmd := &cobra.Command{
		Use:           "hpm",
		Short:         "Versioned harness assets from a registry into your project",
		Version:       version,
		SilenceUsage:  true,
		SilenceErrors: true,
		Args:          cobra.NoArgs,
		RunE:          func(cmd *cobra.Command, _ []string) error { return cmd.Help() },
	}
	cmd.SetVersionTemplate("hpm {{.Version}}\n")
	return cmd
}
```

`cobra.NoArgs` on the root is what makes `hpm frobnicate` an error rather than help output.

`cmd/hpm/main.go`:

```go
package main

import (
	"fmt"
	"os"

	"github.com/StackCube/harness-package-manager-cli/internal/cli"
)

// version is set at build time with -ldflags "-X main.version=...".
var version = "dev"

func main() {
	if err := cli.NewRootCmd(version).Execute(); err != nil {
		fmt.Fprintln(os.Stderr, "hpm:", err)
		os.Exit(1)
	}
}
```

Run: `go test ./internal/cli/` — Expected: PASS.
Run: `go run ./cmd/hpm --version` — Expected: `hpm dev`.
Commit: `git add -A && git commit -m "Root command with version flag"`

- [ ] **Step 4: Failing test for manifest validation against the vectors**

`internal/manifest/validate_test.go`:

```go
package manifest

import (
	"encoding/json"
	"os"
	"path/filepath"
	"testing"
)

const vectors = "../../contract/vectors/manifests"

type vectorCase struct {
	File   string `json:"file"`
	Schema string `json:"schema"`
	Valid  bool   `json:"valid"`
	Reason string `json:"reason"`
}

func TestManifestVectors(t *testing.T) {
	raw, err := os.ReadFile(filepath.Join(vectors, "cases.json"))
	if err != nil {
		t.Fatal(err)
	}
	var cases []vectorCase
	if err := json.Unmarshal(raw, &cases); err != nil {
		t.Fatal(err)
	}
	if len(cases) == 0 {
		t.Fatal("no vector cases found")
	}
	kinds := map[string]Kind{
		"package-manifest": PackageManifest,
		"project-manifest": ProjectManifest,
		"lockfile":         Lockfile,
	}
	for _, c := range cases {
		t.Run(c.File, func(t *testing.T) {
			kind, ok := kinds[c.Schema]
			if !ok {
				t.Fatalf("unknown schema %q", c.Schema)
			}
			data, err := os.ReadFile(filepath.Join(vectors, c.File))
			if err != nil {
				t.Fatal(err)
			}
			err = Validate(kind, data)
			if c.Valid && err != nil {
				t.Fatalf("expected valid, got: %v", err)
			}
			if !c.Valid && err == nil {
				t.Fatalf("expected invalid (%s), got nil", c.Reason)
			}
		})
	}
}

func TestValidateRejectsNonJSON(t *testing.T) {
	if err := Validate(PackageManifest, []byte("{not json")); err == nil {
		t.Fatal("expected an error")
	}
}
```

Run: `go test ./internal/manifest/` — Expected: FAIL, `undefined: Kind`.

- [ ] **Step 5: Embed the schemas and implement `Validate`**

`contractfs.go` at the repo root. `go:embed` cannot reach a parent directory, and `contract/` is at the root, so the embedding package lives at the root too.

```go
// Package contractfs embeds the parts of the contract subtree the binary needs at run time.
package contractfs

import "embed"

// Schemas holds contract/schemas/*.json.
//
//go:embed contract/schemas/*.json
var Schemas embed.FS
```

```bash
go get github.com/santhosh-tekuri/jsonschema/v6@latest
```

`internal/manifest/validate.go`:

```go
// Package manifest loads and validates hpm.json and hpm.lock.
package manifest

import (
	"bytes"
	"fmt"
	"sync"

	"github.com/santhosh-tekuri/jsonschema/v6"

	contractfs "github.com/StackCube/harness-package-manager-cli"
)

// Kind names one of the contract's file schemas.
type Kind string

const (
	PackageManifest Kind = "package-manifest"
	ProjectManifest Kind = "project-manifest"
	Lockfile        Kind = "lockfile"
)

const idBase = "https://stackcube.dev/hpm/schemas/v1/"

var (
	once     sync.Once
	schemas  map[Kind]*jsonschema.Schema
	setupErr error
)

func setup() {
	c := jsonschema.NewCompiler()
	c.AssertFormat()
	kinds := []Kind{PackageManifest, ProjectManifest, Lockfile}
	for _, k := range kinds {
		raw, err := contractfs.Schemas.ReadFile("contract/schemas/" + string(k) + ".json")
		if err != nil {
			setupErr = err
			return
		}
		doc, err := jsonschema.UnmarshalJSON(bytes.NewReader(raw))
		if err != nil {
			setupErr = fmt.Errorf("%s: %w", k, err)
			return
		}
		if err := c.AddResource(idBase+string(k)+".json", doc); err != nil {
			setupErr = err
			return
		}
	}
	schemas = make(map[Kind]*jsonschema.Schema, len(kinds))
	for _, k := range kinds {
		s, err := c.Compile(idBase + string(k) + ".json")
		if err != nil {
			setupErr = fmt.Errorf("compile %s: %w", k, err)
			return
		}
		schemas[k] = s
	}
}

// Validate checks data against the contract schema for kind.
func Validate(kind Kind, data []byte) error {
	once.Do(setup)
	if setupErr != nil {
		return setupErr
	}
	s, ok := schemas[kind]
	if !ok {
		return fmt.Errorf("unknown manifest kind %q", kind)
	}
	inst, err := jsonschema.UnmarshalJSON(bytes.NewReader(data))
	if err != nil {
		return fmt.Errorf("not valid JSON: %w", err)
	}
	return s.Validate(inst)
}
```

Run: `go test ./internal/manifest/ -v` — Expected: PASS, 23 subtests.

If exactly the `format: uri` cases disagree with Ajv, the validators differ on format strictness. Report it and fix it in the contract (a vector plus a tighter pattern), then re-pull. Do not special-case it in Go.

Commit: `git add -A && git commit -m "Validate manifests against the embedded contract schemas, proven by the vectors"`

- [ ] **Step 6: Failing test for the generated API types**

`internal/api/api_test.go`:

```go
package api

import (
	"encoding/json"
	"os"
	"testing"
)

func TestIndexExampleDecodes(t *testing.T) {
	raw, err := os.ReadFile("../../contract/vectors/index/example.json")
	if err != nil {
		t.Fatal(err)
	}
	var idx Index
	if err := json.Unmarshal(raw, &idx); err != nil {
		t.Fatal(err)
	}
	if len(idx.Packages) != 1 {
		t.Fatalf("got %d packages, want 1", len(idx.Packages))
	}
	p := idx.Packages[0]
	if p.Name != "@ourorg/tdd-loop" || len(p.Versions) != 2 {
		t.Fatalf("unexpected package: %+v", p)
	}
	if got := p.Versions[1].Version; got != "1.4.2" {
		t.Fatalf("got version %q, want 1.4.2", got)
	}
}
```

Run: `go test ./internal/api/` — Expected: FAIL, `undefined: Index`.

- [ ] **Step 7: Choose the generator and generate**

Try `oapi-codegen` first. It is the smaller dependency and generates a plain `net/http` client.

```bash
go get -tool github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@latest
```

`internal/api/oapi-codegen.yaml`:

```yaml
package: api
output: api.gen.go
generate:
  models: true
  client: true
output-options:
  skip-prune: true
```

`internal/api/generate.go`:

```go
// Package api is generated from contract/dist/openapi.bundled.yaml. Do not edit the generated files.
package api

//go:generate go tool oapi-codegen -config oapi-codegen.yaml ../../contract/dist/openapi.bundled.yaml
```

Run: `go generate ./internal/api/ && go build ./... && go test ./internal/api/`

**Decision rule.** Keep `oapi-codegen` if generation succeeds with no edits to the bundle and no `x-go-*` extensions in the contract, and the test passes. Otherwise switch to `ogen`:

```bash
go get -tool github.com/ogen-go/ogen/cmd/ogen@latest
rm internal/api/oapi-codegen.yaml internal/api/api.gen.go
```

and change the directive in `generate.go` to:

```go
//go:generate go tool ogen --target . --package api --clean ../../contract/dist/openapi.bundled.yaml
```

then re-run the same three commands. If `ogen` rejects the spec as well, stop and report both error outputs. The fix then belongs in the contract's `scripts/bundle.ts`, released as contract `0.1.1`, not in either consumer.

Record the outcome in `docs/decisions/0001-go-openapi-generator.md`:

```markdown
# 0001: Go OpenAPI generator

Date: 2026-09-19. Status: accepted.

**Chosen:** <oapi-codegen | ogen>, version <from go.mod>.

**Rule applied:** the first of oapi-codegen, then ogen, that generates from `contract/dist/openapi.bundled.yaml` with no edits to the bundle and no generator-specific extensions in the contract.

**What happened:** <one paragraph: the command run, and either "generated cleanly" or the exact error that ruled the first option out>.

**Consequence:** `go generate ./internal/api/` is the only way the package changes. CI fails if the generated code is stale.
```

Fill the three angle-bracket fields with what actually happened. They are the record, not placeholders to leave.

Run: `go test ./...` — Expected: PASS.
Commit: `git add -A && git commit -m "Generated API types and client from the bundled contract"`

- [ ] **Step 8: Makefile and CI**

`Makefile`:

```make
VERSION ?= dev

.PHONY: build test generate check
build:
	go build -ldflags "-X main.version=$(VERSION)" -o bin/hpm ./cmd/hpm
test:
	go test ./...
generate:
	go generate ./...
check:
	scripts/check-contract.sh
	go generate ./...
	git diff --exit-code -- internal/api
	go vet ./...
	go test ./...
```

`.github/workflows/ci.yml`:

```yaml
name: ci
on:
  push: { branches: [main] }
  pull_request:
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version-file: go.mod }
      - run: make check
      - run: make build && ./bin/hpm --version
```

Run: `make check && make build VERSION=0.0.0 && ./bin/hpm --version`
Expected: `contract/ matches v0.1.0`, no diff, tests PASS, then `hpm 0.0.0`.

- [ ] **Step 9: ⚠ Commit, push, open the PR**

```bash
git add -A && git commit -m "Makefile and CI"
git push -u origin m0-scaffolding
gh pr create --title "M0: CLI scaffold on contract v0.1.0" --body "Go module, cobra root, contract subtree pinned by contract.lock, manifest validation proven by the contract vectors, generated API types. Plan: meta repo docs/superpowers/plans/2026-09-19-m0-contract-and-scaffolding.md, Task 5.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
gh pr checks --watch
```

Expected: the `check` job succeeds.

---

### Task 6: Registry scaffold

**Repo:** `repos/registry`, branch `m0-scaffolding`

**Files:**
- Create: `package.json`, `tsconfig.json`, `wrangler.jsonc`, `vitest.config.ts`, `.gitignore`, `contract.lock`, `scripts/contract-pull.sh`, `scripts/check-contract.sh`, `src/index.ts`, `src/errors.ts`, `src/validate.ts`, `test/app.test.ts`, `test/validate.test.ts`, `.github/workflows/ci.yml`
- Generated: `contract/` (subtree), `src/generated/api.ts`

**Interfaces:**
- Consumes: contract tag `v0.1.0`; `vectors/manifests/cases.json`; `dist/openapi.bundled.yaml` with components `Error` and `ErrorCode`.
- Produces:
  - `src/errors.ts`: `type ErrorCode`, `type ApiError`, `function errorResponse(c: Context, status: ContentfulStatusCode, code: ErrorCode, message: string, details?: Record<string, unknown>): Response`.
  - `src/validate.ts`: `type SchemaName = "package-manifest" | "project-manifest" | "lockfile"`, `function validate(schema: SchemaName, value: unknown): { valid: true } | { valid: false; errors: string[] }`.
  - `src/index.ts`: default export, a Hono app whose unknown routes answer 404 with the contract error shape.

- [ ] **Step 1: Branch, tooling, subtree**

```bash
cd repos/registry && git checkout -b m0-scaffolding
mkdir -p scripts && cp ../cli/scripts/contract-pull.sh ../cli/scripts/check-contract.sh scripts/
```

The two scripts are identical in both consumers on purpose. They have no repo-specific content. This needs Task 5 Step 1 to exist on the local `cli` branch. If it does not, create both files with the content shown there.

`package.json`:

```json
{
  "name": "harness-package-manager-registry",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "wrangler dev",
    "gen": "openapi-typescript contract/dist/openapi.bundled.yaml -o src/generated/api.ts",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "check": "scripts/check-contract.sh && bun run gen && git diff --exit-code -- src/generated && bun run typecheck && bun run test"
  },
  "dependencies": {
    "@cfworker/json-schema": "latest",
    "hono": "latest"
  },
  "devDependencies": {
    "@cloudflare/vitest-pool-workers": "latest",
    "@cloudflare/workers-types": "latest",
    "openapi-typescript": "latest",
    "typescript": "latest",
    "vitest": "latest",
    "wrangler": "latest"
  }
}
```

Run `bun install`. If bun reports a peer conflict between `vitest` and `@cloudflare/vitest-pool-workers`, install the vitest major that the pool's `peerDependencies` names. Then replace every `"latest"` with the resolved caret version.

`tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022", "module": "ESNext", "moduleResolution": "bundler",
    "lib": ["ES2022"], "strict": true, "noEmit": true, "skipLibCheck": true,
    "resolveJsonModule": true, "verbatimModuleSyntax": true,
    "types": ["@cloudflare/workers-types", "@cloudflare/vitest-pool-workers"]
  },
  "include": ["src", "test", "vitest.config.ts"]
}
```

`wrangler.jsonc`:

```jsonc
{
  // Bindings for D1 and R2 arrive in M1. Per-client values are rendered by the deploy script in M2.
  "name": "hpm-registry",
  "main": "src/index.ts",
  "compatibility_date": "2026-09-01",
  "compatibility_flags": ["nodejs_compat"]
}
```

`vitest.config.ts`:

```ts
import { defineWorkersConfig } from "@cloudflare/vitest-pool-workers/config";

export default defineWorkersConfig({
  test: {
    poolOptions: { workers: { wrangler: { configPath: "./wrangler.jsonc" } } },
  },
});
```

`.gitignore`:

```
node_modules/
.wrangler/
.dev.vars
```

```bash
chmod +x scripts/*.sh
git add -A && git commit -m "Worker tooling and contract subtree scripts"
scripts/contract-pull.sh v0.1.0 && scripts/check-contract.sh
bun run gen
```

Expected: `contract/ matches v0.1.0`, then `src/generated/api.ts` exists and contains `ErrorCode`.
Commit: `git add -A && git commit -m "Generated API types from the bundled contract"`

- [ ] **Step 2: Failing test for the error shape**

`test/app.test.ts`:

```ts
import { SELF } from "cloudflare:test";
import { expect, test } from "vitest";

test("an unknown route answers with the contract error shape", async () => {
  const res = await SELF.fetch("https://registry.test/v1/nope");
  expect(res.status).toBe(404);
  expect(res.headers.get("content-type")).toContain("application/json");
  expect(await res.json()).toEqual({ code: "not_found", message: "No such route: GET /v1/nope", details: {} });
});
```

Run: `bun run test` — Expected: FAIL, cannot resolve `src/index.ts`.

- [ ] **Step 3: Errors and the app**

`src/errors.ts`:

```ts
import type { Context } from "hono";
import type { ContentfulStatusCode } from "hono/utils/http-status";
import type { components } from "./generated/api";

export type ErrorCode = components["schemas"]["ErrorCode"];
export type ApiError = components["schemas"]["Error"];

/** The only way a route reports a refusal. The CLI switches on `code`. */
export function errorResponse(
  c: Context,
  status: ContentfulStatusCode,
  code: ErrorCode,
  message: string,
  details: Record<string, unknown> = {},
): Response {
  const body: ApiError = { code, message, details };
  return c.json(body, status);
}
```

`src/index.ts`:

```ts
import { Hono } from "hono";
import { errorResponse } from "./errors";

const app = new Hono();

// Routes are added in M1 under /v1, behind the auth middleware.

app.notFound((c) => errorResponse(c, 404, "not_found", `No such route: ${c.req.method} ${new URL(c.req.url).pathname}`));

export default app;
```

Run: `bun run test` — Expected: PASS, 1 test. An `onError` handler arrives in M1 with the first route that can throw.
Commit: `git add -A && git commit -m "Hono app answering unknown routes with the contract error shape"`

- [ ] **Step 4: Failing test for schema validation against the vectors**

Tests run inside `workerd`, which has no filesystem, so the vectors are pulled in at transform time with `import.meta.glob`.

`test/validate.test.ts`:

```ts
/// <reference types="vite/client" />
import { describe, expect, test } from "vitest";
import cases from "../contract/vectors/manifests/cases.json";
import { type SchemaName, validate } from "../src/validate";

const files = import.meta.glob("../contract/vectors/manifests/{valid,invalid}/*.json", {
  eager: true,
  import: "default",
}) as Record<string, unknown>;

describe("manifest vectors", () => {
  test("every case file was loaded", () => {
    expect(cases.length).toBeGreaterThan(0);
    for (const c of cases) expect(files).toHaveProperty(`../contract/vectors/manifests/${c.file}`);
  });

  test.each(cases)("$file → valid=$valid ($reason)", (c) => {
    const result = validate(c.schema as SchemaName, files[`../contract/vectors/manifests/${c.file}`]);
    expect(result.valid).toBe(c.valid);
    if (!result.valid) expect(result.errors.length).toBeGreaterThan(0);
  });
});
```

Run: `bun run test` — Expected: FAIL, cannot resolve `../src/validate`.

If the Workers pool rejects `import.meta.glob`, keep the test file as it is and split the Vitest config into two projects: the Workers project for `test/app.test.ts`, and a plain Node project for `test/validate.test.ts`. `validate` is pure and behaves the same in both. Note the change in the commit message.

- [ ] **Step 5: Implement `validate`**

`src/validate.ts`:

```ts
import { type Schema, Validator } from "@cfworker/json-schema";
import lockfile from "../contract/schemas/lockfile.json";
import packageManifest from "../contract/schemas/package-manifest.json";
import projectManifest from "../contract/schemas/project-manifest.json";

export type SchemaName = "package-manifest" | "project-manifest" | "lockfile";

// @cfworker/json-schema interprets schemas without eval or new Function, which Workers forbid.
const validators: Record<SchemaName, Validator> = {
  "package-manifest": new Validator(packageManifest as Schema, "2020-12", false),
  "project-manifest": new Validator(projectManifest as Schema, "2020-12", false),
  lockfile: new Validator(lockfile as Schema, "2020-12", false),
};

export function validate(
  schema: SchemaName,
  value: unknown,
): { valid: true } | { valid: false; errors: string[] } {
  const result = validators[schema].validate(value);
  if (result.valid) return { valid: true };
  return { valid: false, errors: result.errors.map((e) => `${e.instanceLocation}: ${e.error}`) };
}
```

The third constructor argument is `shortCircuit`. It is `false` so that every failure is reported, which publish will want in `details`.

Run: `bun run test` — Expected: PASS, 1 app test, 1 loader test, 23 vector cases.

As in the CLI: if only `format` cases disagree with the other validators, fix the contract, not this file.

Commit: `git add -A && git commit -m "Schema validation without eval, proven by the contract vectors"`

- [ ] **Step 6: CI**

`.github/workflows/ci.yml`:

```yaml
name: ci
on:
  push: { branches: [main] }
  pull_request:
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun install --frozen-lockfile
      - run: bun run check
      - name: Worker bundles
        run: bunx wrangler deploy --dry-run --outdir dist
```

Run: `bun run check && bunx wrangler deploy --dry-run --outdir dist`
Expected: `contract/ matches v0.1.0`, no diff in `src/generated`, typecheck clean, tests PASS, and wrangler prints a bundle size without uploading. Add `dist/` to `.gitignore`.

- [ ] **Step 7: ⚠ Commit, push, open the PR**

```bash
git add -A && git commit -m "CI"
git push -u origin m0-scaffolding
gh pr create --title "M0: Registry scaffold on contract v0.1.0" --body "Hono Worker skeleton, contract subtree pinned by contract.lock, generated API types, eval-free schema validation proven by the contract vectors, Vitest on workerd. Plan: meta repo docs/superpowers/plans/2026-09-19-m0-contract-and-scaffolding.md, Task 6.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
gh pr checks --watch
```

Expected: the `check` job succeeds.

---

### Task 7: Workspace checks and README

**Repo:** meta repo, on the current branch

**Files:**
- Modify: `metastack.yaml` (the `checks:` list), `README.md`

**Interfaces:**
- Consumes: the `contract` repo entry added in Task 1.

- [ ] **Step 1: Update the checks**

In `metastack.yaml`, remove the `wrangler` check and add `go` and `pulumi` after `git`. Wrangler is a dev dependency of the registry repo and is run with `bunx`, so a global binary is not required.

```yaml
  - name: go
    command: go version
    min_version: "1.26"
    required: true
  # Registry deploys, from M2. Not needed to build or test.
  - name: pulumi
    command: pulumi version
    min_version: "3.200.0"
    required: false
```

Remove the existing `wrangler` entry. Leave `gh`, `git`, `bun`, `cloudflared` and `jq` as they are.

- [ ] **Step 2: Verify**

Run: `metastack doctor`
Expected: `gh`, `git`, `go`, `bun` pass. `pulumi` warns if older than 3.200.0. Nothing required fails.

Run: `metastack status`
Expected: six repos listed: `contract`, `registry`, `cli`, `packages`, `tap`, `docs`.

- [ ] **Step 3: README**

In `README.md`, add as the first row of the Repos table:

```markdown
| `contract` | StackCube/harness-package-manager-contract | Wire contract: OpenAPI 3.1, JSON Schema, specs and golden vectors. Subtreed into `cli` and `registry` at a tag. |
```

Replace the Status paragraph with:

```markdown
Draft. The CLI name `hpm` is a placeholder. Milestone 0 is done: the contract is at v0.1.0, and `cli` and `registry` are scaffolded against it with CI. Next is M1, the walking skeleton.
```

- [ ] **Step 4: Commit**

```bash
git add metastack.yaml README.md && git commit -m "M0: workspace checks for go and pulumi, contract repo in the README"
```

---

## Exit check for M0

All of these hold:

- [ ] `repos/contract`: `bun run check` passes, tag `v0.1.0` is on the remote, and both CI runs are green.
- [ ] `repos/cli`: `make check` passes, `./bin/hpm --version` prints a version, and the PR's CI is green.
- [ ] `repos/registry`: `bun run check` passes, `wrangler deploy --dry-run` bundles, and the PR's CI is green.
- [ ] Both consumers pass all 23 manifest vector cases with their own validator.
- [ ] `docs/decisions/0001-go-openapi-generator.md` in `cli` records which generator was kept and why.
- [ ] `metastack doctor` has no required failures.
