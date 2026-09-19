# M1-A: Contract v0.3 and the Archive Format Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Clear the M0 carry-over from the contract, build the Go archive packer that is the reference implementation of `spec/archive.md`, and publish golden archive vectors that the registry will unpack in M1-B.

**Architecture:** Two contract releases bracket one CLI package. v0.3.0 fixes what the M0 reviews deferred. The CLI's `internal/archive` package then implements hashing, deterministic packing and safe unpacking. A small generator in the CLI repo writes archive vectors into the contract repo, which ships them as v0.4.0. Vectors pin the uncompressed tar bytes, not the gzip bytes, because DEFLATE output legitimately differs between implementations.

**Tech Stack:** Bun, Ajv, Redocly (contract) · Go 1.26 standard library `archive/tar`, `compress/gzip`, `crypto/sha256` (CLI).

**Spec:** `docs/superpowers/specs/2026-09-19-iteration-1-technical-design.md` §2 and §7.1. Carry-over list: `docs/superpowers/plans/2026-09-19-m0-carryover.md`. This is part A of milestone M1; parts B (registry) and C (CLI and end to end) follow in their own files.

## Global Constraints

- Workspace root is `harness-package-manager-meta/`. Sibling repos are separate git repos in `repos/<name>/`.
- In every repo, work on a branch named `m1a` and open one pull request per repo. The contract repo releases by tag from `main`, so its PR must be merged before tagging. Merging and tagging are ⚠ steps.
- ⚠ marks an outward-facing step (push, PR, merge, tag). Confirm with Rick before the first one in a run unless he has already authorized the run.
- Hash format: `sha256-<lowercase hex>`, regex `^sha256-[0-9a-f]{64}$`.
- Schema `pattern`s must be valid in ECMAScript and Go RE2. No lookahead, lookbehind or backreferences.
- `dist/openapi.bundled.yaml` changes only via `bun run bundle`. Generated code changes only via its generator.
- The contract is imported into consumers only with `scripts/contract-pull.sh vX.Y.Z`. Nobody edits `contract/` in place. The two scripts stay byte-identical in `cli` and `registry`.
- A vector disagreement between validators or implementations is a contract defect. Stop and report it. Never special-case it in a consumer.
- Commit messages end with a real trailer: a blank line, then `Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>`. Use `git commit -F` or several `-m` flags.
- No force-push, no amending pushed commits, no moving or deleting tags.

## File map

```
repos/contract/
  openapi.yaml                       400/403 declarations, PackageInfo.versions pattern
  schemas/*.json                     harness enums gain "type": "string"
  spec/archive.md                    Trigger wording, directory entries, gzip header, ustar path limit
  spec/changelog.md                  (unchanged; gains vectors)
  vectors/changelog/duplicate.md  subheading.md  cases.json
  vectors/manifests/cases.json       `reason` → `code`
  vectors/archives/                  NEW in v0.4.0 (Task 4)
    cases.json
    valid/<name>/src/**  valid/<name>/archive.tgz
    invalid/<name>.tgz
  test/schemas.test.ts  consistency.test.ts  vectors.test.ts  archives.test.ts (new)
  CHANGELOG.md  VERSION

repos/cli/
  scripts/contract-pull.sh  check-contract.sh      carry-over fixes (mirrored to registry)
  internal/manifest/validate.go                    error wrapping
  internal/archive/hash.go  pack.go  unpack.go  rules.go  *_test.go     NEW
  tools/genvectors/main.go                         NEW, writes contract archive vectors

repos/registry/
  scripts/*.sh                                     mirrored from cli
```

---

### Task 1: Contract v0.3.0, the M0 carry-over

**Repo:** `repos/contract`, branch `m1a`

**Files:**
- Modify: `openapi.yaml`, `schemas/index.json`, `schemas/lockfile.json`, `schemas/package-manifest.json`, `schemas/project-manifest.json`, `spec/archive.md`, `vectors/changelog/cases.json`, `vectors/manifests/cases.json`, `test/schemas.test.ts`, `test/consistency.test.ts`, `CHANGELOG.md`, `VERSION`
- Create: `vectors/changelog/duplicate.md`, `vectors/changelog/subheading.md`

**Interfaces:**
- Produces: tag `v0.3.0`. Breaking for consumers' tests: `vectors/manifests/cases.json` entries become `{ "file", "schema", "valid", "code" }`, where `code` is `"manifest_invalid"` for invalid cases and `null` for valid ones. The `reason` member is gone.
- Produces: harness enum items are typed strings, so generated Go becomes `Harnesses []string` (it is `[]interface{}` today) and TypeScript becomes a string-literal union array.

- [ ] **Step 1: Failing tests first**

In `test/consistency.test.ts` add a harness-enum test. It walks the four parsed schema files and collects every `enum` array whose first element is `"claude-code"`.

```ts
const HARNESS_IDS = ["claude-code", "copilot", "kiro", "codex"];

test("the harness id enum is identical everywhere and typed as string", () => {
  const found: { where: string; node: any }[] = [];
  const walk = (node: any, where: string): void => {
    if (Array.isArray(node)) return node.forEach((n, i) => walk(n, `${where}/${i}`));
    if (node === null || typeof node !== "object") return;
    if (Array.isArray(node.enum) && node.enum[0] === "claude-code") found.push({ where, node });
    for (const [k, v] of Object.entries(node)) walk(v, `${where}/${k}`);
  };
  for (const name of ["index", "lockfile", "package-manifest", "project-manifest"]) {
    walk(JSON.parse(readFileSync(join(root, `schemas/${name}.json`), "utf8")), name);
  }
  expect(found.length).toBeGreaterThanOrEqual(5);
  for (const f of found) {
    expect(f.node.enum, f.where).toEqual(HARNESS_IDS);
    expect(f.node.type, f.where).toBe("string");
  }
});
```

Use whatever `root`, `readFileSync` and `join` bindings the file already has.

In `test/schemas.test.ts`, change the manifest-vector test to read `code`, to assert it, and to print real titles:

```ts
const cases: { file: string; schema: string; valid: boolean; code: string | null }[] =
  load("vectors/manifests/cases.json");

test.each(cases.map((c) => [c.file, c] as const))("manifest vector: %s", (_file, c) => {
  expect(validators[c.schema](load(`vectors/manifests/${c.file}`))).toBe(c.valid);
  expect(c.code).toBe(c.valid ? null : "manifest_invalid");
});
```

Add two changelog cases to `vectors/changelog/cases.json` and their files:

`vectors/changelog/duplicate.md`:

```markdown
# Changelog

## [1.1.0] - 2026-09-22
- Added: the first section wins

## [1.1.0] - 2026-09-21
- Added: this second section is ignored
```

`vectors/changelog/subheading.md`:

```markdown
# Changelog

## [1.2.0] - 2026-09-23
### Added
- A sub-heading counts as a line
```

```json
{ "file": "duplicate.md", "version": "1.1.0", "hasSection": true, "nonEmpty": true, "hasTriggerLine": false,
  "lines": ["- Added: the first section wins"] },
{ "file": "subheading.md", "version": "1.2.0", "hasSection": true, "nonEmpty": true, "hasTriggerLine": false,
  "lines": ["### Added", "- A sub-heading counts as a line"] }
```

Run: `bun test`
Expected: the harness test FAILS on `type`, the manifest test FAILS because `code` is undefined. The two changelog cases should PASS against the existing reference parser. If `subheading.md` fails because the parser ends a section at `### `, the parser is wrong: a section ends at a line starting with `## ` followed by a space, and `###` does not match `^## `. Fix the parser in `test/vectors.test.ts`, not the vector.

- [ ] **Step 2: Schemas and vectors**

- Every harness enum (the five places the test finds) gains `"type": "string"` next to `"enum"`. For `propertyNames` uses, that means `"propertyNames": { "type": "string", "enum": [...], "description": "..." }`.
- `vectors/manifests/cases.json`: replace each `"reason": "..."` with `"code": null` for valid cases and `"code": "manifest_invalid"` for invalid ones. A `jq` one-liner does it: `jq 'map(del(.reason) + {code: (if .valid then null else "manifest_invalid" end)})' cases.json`.

- [ ] **Step 3: OpenAPI**

- `getPackage` and `getArchive`: add `"400": { $ref: "#/components/responses/Error" }`. They carry the patterned path parameters that produce `bad_request`.
- Reads are per instance (PRD §9: anyone admitted may read every package), so `scope_forbidden` never applies to reads. Do not add `403` to the read operations. Instead change `spec/errors.md`: the `scope_forbidden` meaning becomes "The identity may not publish to this scope."
- `PackageInfo.versions.items` gains the version pattern. The consistency test's expected occurrence count for the version pattern rises by one; update the threshold.

- [ ] **Step 4: `spec/archive.md`**

- The `trigger_line_missing` row: `Trigger:` becomes `- Trigger:`, matching `spec/changelog.md` and `spec/errors.md`.
- Under "A packer MUST", replace the mode bullet's directory clause and add three bullets:
  - "emit no directory entries. Directories are implied by file paths. An unpacker MUST accept directory entries and ignore them."
  - "write a gzip header with mtime 0, no file name, no comment and no extra field."
  - "use the ustar format. A path that does not fit ustar's name and prefix fields (100 and 155 bytes) MUST be refused by the packer. A registry MUST answer `archive_invalid` to a pax extended header (`x` or `g` typeflag) or a GNU long-name entry (`L` or `K`)."
- In the refusal table, the `archive_too_large` row becomes two conditions with the same code: "Compressed body larger than 10 MB (10 × 1024 × 1024 bytes), or uncompressed content larger than 64 MB (64 × 1024 × 1024 bytes)". Without the second bound a small body can expand without limit inside the registry.
- Replace the sentence "Packing the same directory twice MUST produce identical bytes." with: "Packing the same directory twice MUST produce an identical tar stream. The gzip bytes MAY differ between implementations of DEFLATE, so vectors pin the hash of the uncompressed tar."

- [ ] **Step 5: Release notes and local check**

`VERSION` → `0.3.0`. `CHANGELOG.md` gains:

```markdown
## [0.3.0] - 2026-09-19
- Changed: manifest vector cases carry `code` instead of `reason` (breaking for consumers' vector tests)
- Changed: harness id enums are typed `string`, so generated code gets string arrays
- Changed: `scope_forbidden` applies to publishing only; reads are per instance
- Changed: `archive_too_large` also covers uncompressed content over 64 MB
- Changed: a packer emits no directory entries, writes a fixed gzip header and uses ustar; vectors will pin the tar hash, not the gzip hash
- Fixed: `getPackage` and `getArchive` declare 400; `PackageInfo.versions` items carry the version pattern
- Fixed: `trigger_line_missing` wording says `- Trigger:`
- Fixed: 0.2.0 added ten invalid manifest vectors, not nine
- Added: changelog vectors for duplicate sections and sub-headings; a consistency test for the harness id enum
```

Run: `bun run lint && bun run bundle && bun test`
Expected: zero lint warnings, all tests PASS.

- [ ] **Step 6: ⚠ Commit, PR, merge, tag**

```bash
git add -A && git commit -m "Contract 0.3.0: M0 carry-over" -m "Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
bun run check
git push -u origin m1a
gh pr create --title "Contract 0.3.0: M0 carry-over" --body "Clears the contract items in the M0 carry-over list. See CHANGELOG.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
gh pr checks --watch
gh pr merge --merge --delete-branch
git checkout main && git pull --ff-only
git tag -a v0.3.0 -m "contract 0.3.0" && git push origin v0.3.0
gh run watch --exit-status "$(gh run list --branch v0.3.0 --limit 1 --json databaseId --jq '.[0].databaseId')"
```

Tag only after the merge commit's CI on `main` is green. Record `git rev-parse 'v0.3.0^{tree}'` in your report.

---

### Task 2: Consumer script fixes and the v0.3.0 import

**Repos:** `repos/cli` then `repos/registry`, branch `m1a` in each

**Files:**
- Modify (both): `scripts/contract-pull.sh`, `scripts/check-contract.sh`, `contract/` and `contract.lock` via the script, generated code
- Modify (cli): `internal/manifest/validate.go`, `internal/manifest/validate_test.go`, `internal/api/api_test.go`
- Modify (registry): `test/validate.test.ts`

**Interfaces:**
- Consumes: tag `v0.3.0`; `cases.json` entries `{ file, schema, valid, code }`.
- Produces (cli): `api.IndexIndexVersion.Harnesses` is `[]string`. Later tasks rely on that.

- [ ] **Step 1: `scripts/contract-pull.sh`**

Three changes. Write them in `cli`, copy to `registry`, confirm with `cmp`.

1. The commit the script makes is made by whoever runs it, so drop the hard-coded co-author trailer:

```bash
git commit -q -m "contract: $TAG"
```

2. Give the unguarded `read-tree` the same recovery hint the two explicit checks have:

```bash
if ! git read-tree --prefix=contract/ -u "$TREE"; then
  echo "Importing $TAG failed. Restore the previous state with: git reset --hard HEAD" >&2
  exit 1
fi
```

3. Nothing else changes.

- [ ] **Step 2: `scripts/check-contract.sh`**

After the missing-lock guard and the `. ./contract.lock` line, add:

```bash
if [ -z "${tag:-}" ] || [ -z "${tree:-}" ]; then
  echo "contract.lock must define tag and tree. Run scripts/contract-pull.sh vX.Y.Z." >&2
  exit 1
fi
if ! git rev-parse -q --verify HEAD:contract >/dev/null; then
  echo "contract/ is not committed. Run scripts/contract-pull.sh $tag." >&2
  exit 1
fi
```

Because the script runs under `set -u`, the `${tag:-}` form is required. Prove both messages in a throwaway clone under your scratch directory, not in the real checkout, and put the output in your report.

- [ ] **Step 3: Commit the scripts, then import**

```bash
git add scripts && git commit -m "Contract scripts: no hard-coded trailer, clearer failures" -m "Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
scripts/contract-pull.sh v0.3.0 && scripts/check-contract.sh
```

- [ ] **Step 4: Adapt the cli**

- `internal/manifest/validate_test.go`: the `vectorCase` struct replaces `Reason string` with `Code *string \`json:"code"\``. For an invalid case assert `c.Code != nil && *c.Code == "manifest_invalid"`. For a valid case assert `c.Code == nil`.
- `internal/manifest/validate.go`: wrap the two bare errors in `setup()` the same way as their neighbours: `fmt.Errorf("read %s: %w", k, err)` and `fmt.Errorf("add %s: %w", k, err)`.
- `go generate ./...`. In `internal/api/api_test.go` add an assertion that proves the new typing:

```go
if got := p.Versions[1].Harnesses; len(got) != 2 || got[0] != "claude-code" {
	t.Fatalf("harnesses = %v, want [claude-code kiro]", got)
}
```

If this does not compile because `Harnesses` is still `[]interface{}`, the contract change did not reach the generator. Stop and report.

Run: `make check && make build`. Commit.

- [ ] **Step 5: Adapt the registry**

- Copy both scripts from `cli`, `cmp` them, commit, import v0.3.0.
- `test/validate.test.ts`: titles come from `c.file` via the tuple form, and the test asserts `c.code` like the contract's own test does.
- `bun run check && bunx wrangler deploy --dry-run --outdir dist`. Commit.

- [ ] **Step 6: ⚠ Push both branches and open one PR in each**

Do not merge yet. Tasks 3 and 4 add to the same branches.

---

### Task 3: `internal/archive` in the CLI

**Repo:** `repos/cli`, branch `m1a`

**Files:**
- Create: `internal/archive/hash.go`, `internal/archive/rules.go`, `internal/archive/pack.go`, `internal/archive/unpack.go`, and a `_test.go` beside each

**Interfaces:**
- Produces:

```go
package archive

// Hash returns "sha256-<hex>" of b.
func Hash(b []byte) string

// File is one regular file in a package.
type File struct {
	Path string // slash-separated, relative, no ".." segment
	Mode int64  // 0644 or 0755
	Data []byte
}

// Archive is the result of packing.
type Archive struct {
	Gzip      []byte // what is uploaded; Hash(Gzip) is the integrity
	Tar       []byte // the deterministic tar stream
	Files     []File // sorted by Path, bytewise
}

// ReadDir loads every regular file under dir. It refuses symlinks and other irregular entries.
func ReadDir(dir string) ([]File, error)

// Pack builds a deterministic archive from files.
func Pack(files []File) (*Archive, error)

// Unpack verifies and reads an archive. maxBytes bounds the uncompressed size.
func Unpack(gz []byte, maxBytes int64) ([]File, error)

// A RuleError is a refusal that maps to a contract error code.
type RuleError struct {
	Code string // "archive_invalid", "path_forbidden", "env_file_present"
	Path string
	Msg  string
}
func (e *RuleError) Error() string

// CheckPath applies the path rules of spec/archive.md to one slash-separated path.
func CheckPath(p string) *RuleError

// IsEnvFile reports whether the base name of p is an environment file.
func IsEnvFile(p string) bool

const MaxCompressed = 10 * 1024 * 1024
const MaxUncompressed = 64 * 1024 * 1024
```

- [ ] **Step 1: Failing tests for hashing and rules**

`internal/archive/hash_test.go`:

```go
package archive

import "testing"

func TestHashMatchesContractVector(t *testing.T) {
	// From contract/vectors/canonical/cases.json: the canonical string `"npx"`.
	got := Hash([]byte(`"npx"`))
	want := "sha256-9d141730c5ec2795cac99c12762aa305d81be46e8a684f574ba9c5f5facb7f15"
	if got != want {
		t.Fatalf("got %s, want %s", got, want)
	}
}
```

`internal/archive/rules_test.go`:

```go
package archive

import "testing"

func TestCheckPath(t *testing.T) {
	ok := []string{"hpm.json", "claude/skills/tdd-loop/SKILL.md", "a/b..c/d", ".env.example"}
	for _, p := range ok {
		if err := CheckPath(p); err != nil {
			t.Errorf("CheckPath(%q) = %v, want nil", p, err)
		}
	}
	bad := map[string]string{
		"/etc/passwd":      "path_forbidden",
		"../x":             "path_forbidden",
		"a/../b":           "path_forbidden",
		"a/..":             "path_forbidden",
		"":                 "path_forbidden",
		"a//b":             "path_forbidden",
		`a\b`:              "path_forbidden",
		".env":             "env_file_present",
		"claude/.env.prod": "env_file_present",
		"scripts/prod.env": "env_file_present",
	}
	for p, code := range bad {
		err := CheckPath(p)
		if err == nil || err.Code != code {
			t.Errorf("CheckPath(%q) = %v, want code %s", p, err, code)
		}
	}
}

func TestIsEnvFile(t *testing.T) {
	for p, want := range map[string]bool{
		".env": true, "x/.env.local": true, "x/prod.env": true,
		".env.example": false, "x/.env.example": false, "environment.md": false, "x/env": false,
	} {
		if got := IsEnvFile(p); got != want {
			t.Errorf("IsEnvFile(%q) = %v, want %v", p, got, want)
		}
	}
}
```

Run: `go test ./internal/archive/` — Expected: FAIL, undefined symbols.

- [ ] **Step 2: Implement hashing and rules**

`internal/archive/hash.go`:

```go
// Package archive is the reference implementation of contract/spec/archive.md.
package archive

import (
	"crypto/sha256"
	"encoding/hex"
)

// Hash returns "sha256-<hex>" of b.
func Hash(b []byte) string {
	sum := sha256.Sum256(b)
	return "sha256-" + hex.EncodeToString(sum[:])
}
```

`internal/archive/rules.go`:

```go
package archive

import (
	"fmt"
	"path"
	"strings"
)

// MaxCompressed and MaxUncompressed are the archive size limits in spec/archive.md.
const (
	MaxCompressed   = 10 * 1024 * 1024
	MaxUncompressed = 64 * 1024 * 1024
)

// A RuleError is a refusal that maps to a contract error code.
type RuleError struct {
	Code string
	Path string
	Msg  string
}

func (e *RuleError) Error() string { return fmt.Sprintf("%s: %s (%s)", e.Code, e.Msg, e.Path) }

// IsEnvFile reports whether the base name of p is an environment file.
func IsEnvFile(p string) bool {
	base := path.Base(p)
	if base == ".env.example" {
		return false
	}
	return base == ".env" || strings.HasPrefix(base, ".env.") || strings.HasSuffix(base, ".env")
}

// CheckPath applies the path rules of spec/archive.md to one slash-separated path.
func CheckPath(p string) *RuleError {
	forbid := func(msg string) *RuleError { return &RuleError{Code: "path_forbidden", Path: p, Msg: msg} }
	switch {
	case p == "":
		return forbid("empty path")
	case strings.HasPrefix(p, "/"):
		return forbid("absolute path")
	case strings.Contains(p, `\`):
		return forbid("backslash in path")
	}
	for _, seg := range strings.Split(p, "/") {
		switch seg {
		case "":
			return forbid("empty path segment")
		case "..":
			return forbid("parent segment")
		case ".":
			return forbid("dot segment")
		}
	}
	if IsEnvFile(p) {
		return &RuleError{Code: "env_file_present", Path: p, Msg: "environment files never ship"}
	}
	return nil
}
```

Run: `go test ./internal/archive/` — Expected: PASS. Commit: "Archive hashing and path rules".

- [ ] **Step 3: Failing tests for pack and unpack**

`internal/archive/pack_test.go`:

```go
package archive

import (
	"archive/tar"
	"bytes"
	"compress/gzip"
	"errors"
	"io"
	"os"
	"path/filepath"
	"testing"
)

func sample() []File {
	return []File{
		{Path: "hpm.json", Mode: 0o644, Data: []byte(`{"name":"@t/x"}`)},
		{Path: "claude/skills/x/SKILL.md", Mode: 0o644, Data: []byte("# x\n")},
		{Path: "claude/scripts/run.sh", Mode: 0o755, Data: []byte("#!/bin/sh\n")},
	}
}

func TestPackIsDeterministicAndSorted(t *testing.T) {
	a, err := Pack(sample())
	if err != nil {
		t.Fatal(err)
	}
	reversed := sample()
	reversed[0], reversed[2] = reversed[2], reversed[0]
	b, err := Pack(reversed)
	if err != nil {
		t.Fatal(err)
	}
	if !bytes.Equal(a.Tar, b.Tar) || !bytes.Equal(a.Gzip, b.Gzip) {
		t.Fatal("packing the same files in a different order changed the bytes")
	}
	want := []string{"claude/scripts/run.sh", "claude/skills/x/SKILL.md", "hpm.json"}
	for i, f := range a.Files {
		if f.Path != want[i] {
			t.Fatalf("file %d is %q, want %q", i, f.Path, want[i])
		}
	}
}

func TestPackHeadersAreNormalised(t *testing.T) {
	a, err := Pack(sample())
	if err != nil {
		t.Fatal(err)
	}
	zr, err := gzip.NewReader(bytes.NewReader(a.Gzip))
	if err != nil {
		t.Fatal(err)
	}
	if !zr.ModTime.IsZero() || zr.Name != "" || zr.Comment != "" || len(zr.Extra) != 0 {
		t.Fatalf("gzip header is not fixed: %+v", zr.Header)
	}
	tr := tar.NewReader(zr)
	for {
		h, err := tr.Next()
		if errors.Is(err, io.EOF) {
			break
		}
		if err != nil {
			t.Fatal(err)
		}
		if h.Typeflag != tar.TypeReg {
			t.Errorf("%s: typeflag %c, want regular file only", h.Name, h.Typeflag)
		}
		if !h.ModTime.IsZero() && h.ModTime.Unix() != 0 {
			t.Errorf("%s: mtime %v", h.Name, h.ModTime)
		}
		if h.Uid != 0 || h.Gid != 0 || h.Uname != "" || h.Gname != "" {
			t.Errorf("%s: owner fields not zero", h.Name)
		}
		if h.Mode != 0o644 && h.Mode != 0o755 {
			t.Errorf("%s: mode %o", h.Name, h.Mode)
		}
		if h.Format != tar.FormatUSTAR {
			t.Errorf("%s: format %v, want ustar", h.Name, h.Format)
		}
	}
}

func TestPackRefusesBadInput(t *testing.T) {
	cases := map[string][]File{
		"path rule":  {{Path: "../x", Mode: 0o644}},
		"env file":   {{Path: ".env", Mode: 0o644}},
		"duplicate":  {{Path: "a", Mode: 0o644}, {Path: "a", Mode: 0o644}},
		"odd mode":   {{Path: "a", Mode: 0o600}},
		"empty list": {},
	}
	for name, files := range cases {
		if _, err := Pack(files); err == nil {
			t.Errorf("%s: expected an error", name)
		}
	}
}

func TestReadDirNormalisesModesAndRefusesSymlinks(t *testing.T) {
	dir := t.TempDir()
	must := func(err error) {
		t.Helper()
		if err != nil {
			t.Fatal(err)
		}
	}
	must(os.MkdirAll(filepath.Join(dir, "claude"), 0o755))
	must(os.WriteFile(filepath.Join(dir, "hpm.json"), []byte("{}"), 0o600))
	must(os.WriteFile(filepath.Join(dir, "claude", "run.sh"), []byte("x"), 0o700))
	files, err := ReadDir(dir)
	must(err)
	got := map[string]int64{}
	for _, f := range files {
		got[f.Path] = f.Mode
	}
	if got["hpm.json"] != 0o644 || got["claude/run.sh"] != 0o755 {
		t.Fatalf("modes = %v", got)
	}
	must(os.Symlink("hpm.json", filepath.Join(dir, "link")))
	if _, err := ReadDir(dir); err == nil {
		t.Fatal("expected ReadDir to refuse a symlink")
	}
}
```

`internal/archive/unpack_test.go`:

```go
package archive

import (
	"archive/tar"
	"bytes"
	"compress/gzip"
	"errors"
	"testing"
)

func TestUnpackRoundTrip(t *testing.T) {
	a, err := Pack(sample())
	if err != nil {
		t.Fatal(err)
	}
	files, err := Unpack(a.Gzip, 1<<20)
	if err != nil {
		t.Fatal(err)
	}
	if len(files) != 3 || files[2].Path != "hpm.json" || string(files[2].Data) != `{"name":"@t/x"}` {
		t.Fatalf("unexpected files: %+v", files)
	}
	if files[0].Mode != 0o755 {
		t.Fatalf("run.sh mode = %o", files[0].Mode)
	}
}

// raw builds an archive without Pack's checks, to feed Unpack hostile input.
func raw(t *testing.T, hdrs ...*tar.Header) []byte {
	t.Helper()
	var buf bytes.Buffer
	zw := gzip.NewWriter(&buf)
	tw := tar.NewWriter(zw)
	for _, h := range hdrs {
		body := []byte("x")
		if h.Typeflag != tar.TypeReg {
			body = nil
		}
		h.Size = int64(len(body))
		if err := tw.WriteHeader(h); err != nil {
			t.Fatal(err)
		}
		if _, err := tw.Write(body); err != nil {
			t.Fatal(err)
		}
	}
	tw.Close()
	zw.Close()
	return buf.Bytes()
}

func TestUnpackRefusals(t *testing.T) {
	reg := func(name string) *tar.Header { return &tar.Header{Name: name, Mode: 0o644, Typeflag: tar.TypeReg} }
	cases := map[string]struct {
		gz   []byte
		code string
	}{
		"not gzip":   {[]byte("plain"), "archive_invalid"},
		"symlink":    {raw(t, &tar.Header{Name: "l", Linkname: "hpm.json", Typeflag: tar.TypeSymlink}), "path_forbidden"},
		"hard link":  {raw(t, &tar.Header{Name: "l", Linkname: "hpm.json", Typeflag: tar.TypeLink}), "path_forbidden"},
		"absolute":   {raw(t, reg("/etc/x")), "path_forbidden"},
		"dot dot":    {raw(t, reg("a/../../x")), "path_forbidden"},
		"env file":   {raw(t, reg("claude/.env")), "env_file_present"},
		"duplicate":  {raw(t, reg("a"), reg("a")), "archive_invalid"},
	}
	for name, c := range cases {
		_, err := Unpack(c.gz, 1<<20)
		var re *RuleError
		if !errors.As(err, &re) || re.Code != c.code {
			t.Errorf("%s: got %v, want code %s", name, err, c.code)
		}
	}
}

func TestUnpackIgnoresDirectoryEntries(t *testing.T) {
	gz := raw(t, &tar.Header{Name: "claude/", Mode: 0o755, Typeflag: tar.TypeDir},
		&tar.Header{Name: "claude/a.md", Mode: 0o644, Typeflag: tar.TypeReg})
	files, err := Unpack(gz, 1<<20)
	if err != nil || len(files) != 1 || files[0].Path != "claude/a.md" {
		t.Fatalf("files=%v err=%v", files, err)
	}
}

func TestUnpackBoundsUncompressedSize(t *testing.T) {
	a, err := Pack([]File{{Path: "big", Mode: 0o644, Data: bytes.Repeat([]byte("a"), 4096)}})
	if err != nil {
		t.Fatal(err)
	}
	_, err = Unpack(a.Gzip, 1024)
	var re *RuleError
	if !errors.As(err, &re) || re.Code != "archive_too_large" {
		t.Fatalf("got %v, want archive_too_large", err)
	}
}
```

Run: `go test ./internal/archive/` — Expected: FAIL, undefined `Pack`, `Unpack`, `ReadDir`.

- [ ] **Step 4: Implement pack**

`internal/archive/pack.go`:

```go
package archive

import (
	"archive/tar"
	"bytes"
	"compress/gzip"
	"fmt"
	"io/fs"
	"os"
	"path/filepath"
	"sort"
	"time"
)

// File is one regular file in a package.
type File struct {
	Path string
	Mode int64
	Data []byte
}

// Archive is the result of packing.
type Archive struct {
	Gzip  []byte
	Tar   []byte
	Files []File
}

// ReadDir loads every regular file under dir. It refuses symlinks and other irregular entries.
func ReadDir(dir string) ([]File, error) {
	var files []File
	err := filepath.WalkDir(dir, func(p string, d fs.DirEntry, err error) error {
		if err != nil {
			return err
		}
		if d.IsDir() {
			return nil
		}
		rel, err := filepath.Rel(dir, p)
		if err != nil {
			return err
		}
		rel = filepath.ToSlash(rel)
		info, err := d.Info()
		if err != nil {
			return err
		}
		if !info.Mode().IsRegular() {
			return &RuleError{Code: "path_forbidden", Path: rel, Msg: "only regular files can be packed"}
		}
		data, err := os.ReadFile(p)
		if err != nil {
			return err
		}
		mode := int64(0o644)
		if info.Mode().Perm()&0o111 != 0 {
			mode = 0o755
		}
		files = append(files, File{Path: rel, Mode: mode, Data: data})
		return nil
	})
	return files, err
}

// Pack builds a deterministic archive from files.
func Pack(files []File) (*Archive, error) {
	if len(files) == 0 {
		return nil, fmt.Errorf("nothing to pack")
	}
	sorted := append([]File(nil), files...)
	sort.Slice(sorted, func(i, j int) bool { return sorted[i].Path < sorted[j].Path })

	var tarBuf bytes.Buffer
	tw := tar.NewWriter(&tarBuf)
	for i, f := range sorted {
		if err := CheckPath(f.Path); err != nil {
			return nil, err
		}
		if i > 0 && sorted[i-1].Path == f.Path {
			return nil, fmt.Errorf("duplicate path %q", f.Path)
		}
		if f.Mode != 0o644 && f.Mode != 0o755 {
			return nil, fmt.Errorf("%s: mode %o, want 0644 or 0755", f.Path, f.Mode)
		}
		hdr := &tar.Header{
			Typeflag: tar.TypeReg,
			Name:     f.Path,
			Mode:     f.Mode,
			Size:     int64(len(f.Data)),
			ModTime:  time.Unix(0, 0),
			Format:   tar.FormatUSTAR,
		}
		if err := tw.WriteHeader(hdr); err != nil {
			return nil, fmt.Errorf("%s: %w", f.Path, err)
		}
		if _, err := tw.Write(f.Data); err != nil {
			return nil, err
		}
	}
	if err := tw.Close(); err != nil {
		return nil, err
	}

	var gzBuf bytes.Buffer
	zw, err := gzip.NewWriterLevel(&gzBuf, gzip.BestCompression)
	if err != nil {
		return nil, err
	}
	// The zero Header already has no name, comment, extra or mtime.
	if _, err := zw.Write(tarBuf.Bytes()); err != nil {
		return nil, err
	}
	if err := zw.Close(); err != nil {
		return nil, err
	}
	if gzBuf.Len() > MaxCompressed {
		return nil, &RuleError{Code: "archive_too_large", Msg: fmt.Sprintf("%d bytes compressed, limit %d", gzBuf.Len(), MaxCompressed)}
	}
	return &Archive{Gzip: gzBuf.Bytes(), Tar: tarBuf.Bytes(), Files: sorted}, nil
}
```

`tar.FormatUSTAR` makes `WriteHeader` fail for a path that does not fit ustar, which is the refusal the spec asks the packer for.

- [ ] **Step 5: Implement unpack**

`internal/archive/unpack.go`:

```go
package archive

import (
	"archive/tar"
	"bytes"
	"compress/gzip"
	"errors"
	"io"
	"sort"
)

// Unpack verifies and reads an archive. maxBytes bounds the uncompressed size.
func Unpack(gz []byte, maxBytes int64) ([]File, error) {
	invalid := func(msg string) error { return &RuleError{Code: "archive_invalid", Msg: msg} }
	if len(gz) > MaxCompressed {
		return nil, &RuleError{Code: "archive_too_large", Msg: "compressed body over the limit"}
	}
	zr, err := gzip.NewReader(bytes.NewReader(gz))
	if err != nil {
		return nil, invalid("not gzip: " + err.Error())
	}
	tr := tar.NewReader(zr)
	seen := map[string]bool{}
	var files []File
	var total int64
	for {
		h, err := tr.Next()
		if errors.Is(err, io.EOF) {
			break
		}
		if err != nil {
			return nil, invalid("not tar: " + err.Error())
		}
		switch h.Typeflag {
		case tar.TypeDir:
			continue
		case tar.TypeReg:
		case tar.TypeXHeader, tar.TypeXGlobalHeader, tar.TypeGNULongName, tar.TypeGNULongLink:
			return nil, invalid("extended headers are not allowed")
		default:
			return nil, &RuleError{Code: "path_forbidden", Path: h.Name, Msg: "only files and directories are allowed"}
		}
		if re := CheckPath(h.Name); re != nil {
			return nil, re
		}
		if seen[h.Name] {
			return nil, invalid("duplicate entry " + h.Name)
		}
		seen[h.Name] = true
		total += h.Size
		if total > maxBytes {
			return nil, &RuleError{Code: "archive_too_large", Msg: "uncompressed content over the limit"}
		}
		data, err := io.ReadAll(io.LimitReader(tr, h.Size+1))
		if err != nil {
			return nil, invalid("truncated entry " + h.Name)
		}
		mode := int64(0o644)
		if h.Mode&0o111 != 0 {
			mode = 0o755
		}
		files = append(files, File{Path: h.Name, Mode: mode, Data: data})
	}
	if len(files) == 0 {
		return nil, invalid("archive has no files")
	}
	sort.Slice(files, func(i, j int) bool { return files[i].Path < files[j].Path })
	return files, nil
}
```

Go's `archive/tar` reader transparently consumes pax and GNU long-name headers before returning the file they describe, so the `TypeXHeader` branch rarely fires on the Go side. That is acceptable for the CLI, which only unpacks archives whose integrity it has verified against the index. The registry's parser in M1-B sees raw headers and enforces the rule.

Run: `go test ./internal/archive/ -v` — Expected: PASS.
Run: `make check`. Commit: "Deterministic pack and safe unpack".

---

### Task 4: Archive vectors, contract v0.4.0

**Repos:** `repos/cli` (generator), `repos/contract` (vectors), then both consumers import

**Files:**
- Create (cli): `tools/genvectors/main.go`
- Create (contract): `vectors/archives/cases.json`, `vectors/archives/valid/skill-only/src/**`, `vectors/archives/valid/skill-only/archive.tgz`, `vectors/archives/valid/exec-bit/...`, `vectors/archives/invalid/*.tgz`, `test/archives.test.ts`
- Modify (contract): `README.md` (vectors row), `CHANGELOG.md`, `VERSION`

**Interfaces:**
- Consumes: `archive.ReadDir`, `archive.Pack`, `archive.Hash` from Task 3.
- Produces: `vectors/archives/cases.json`:

```json
{
  "valid": [
    { "name": "skill-only", "src": "valid/skill-only/src", "archive": "valid/skill-only/archive.tgz",
      "tarSha256": "sha256-…",
      "files": [ { "path": "claude/skills/ping/SKILL.md", "mode": "0644", "sha256": "sha256-…" } ] }
  ],
  "invalid": [
    { "name": "symlink", "archive": "invalid/symlink.tgz", "code": "path_forbidden" }
  ]
}
```

`tarSha256` is the hash of the uncompressed tar stream. `files` is sorted by path. M1-B's registry tests read this file.

- [ ] **Step 1: Source fixtures in the contract repo**

On a contract branch `m1a-vectors`, create:

`vectors/archives/valid/skill-only/src/hpm.json`:

```json
{ "name": "@smoke/ping", "version": "0.1.0", "description": "Answers pong. Used by tests.",
  "keywords": ["test"], "owners": ["ci@ourorg.example"] }
```

`vectors/archives/valid/skill-only/src/README.md`:

```markdown
# ping
Answers pong. Used by tests.
```

`vectors/archives/valid/skill-only/src/CHANGELOG.md`:

```markdown
# Changelog

## [0.1.0] - 2026-09-19
- Added: the ping skill
```

`vectors/archives/valid/skill-only/src/claude/skills/ping/SKILL.md`:

```markdown
---
name: ping
description: Use when asked to ping. Answers pong.
---
Answer "pong".
```

`vectors/archives/valid/exec-bit/src/`: the same four files with the name `@smoke/runner`, the description "Carries an executable script. Used by tests.", plus `claude/scripts/run.sh` containing `#!/bin/sh\necho pong\n`, committed with the executable bit (`git update-index --chmod=+x`).

- [ ] **Step 2: The generator**

`repos/cli/tools/genvectors/main.go`:

```go
// Command genvectors writes the contract's archive vectors from the reference packer.
// Usage: go run ./tools/genvectors <path to contract/vectors/archives>
// Vectors are generated once, reviewed by a person, and committed. After that they are the authority.
package main

import (
	"archive/tar"
	"bytes"
	"compress/gzip"
	"encoding/json"
	"fmt"
	"os"
	"path/filepath"
	"sort"

	"github.com/StackCube/harness-package-manager-cli/internal/archive"
)

type fileEntry struct {
	Path   string `json:"path"`
	Mode   string `json:"mode"`
	Sha256 string `json:"sha256"`
}
type validCase struct {
	Name      string      `json:"name"`
	Src       string      `json:"src"`
	Archive   string      `json:"archive"`
	TarSha256 string      `json:"tarSha256"`
	Files     []fileEntry `json:"files"`
}
type invalidCase struct {
	Name    string `json:"name"`
	Archive string `json:"archive"`
	Code    string `json:"code"`
}

func main() {
	if len(os.Args) != 2 {
		fail(fmt.Errorf("usage: genvectors <vectors/archives dir>"))
	}
	root := os.Args[1]
	out := struct {
		Valid   []validCase   `json:"valid"`
		Invalid []invalidCase `json:"invalid"`
	}{}

	names, err := filepath.Glob(filepath.Join(root, "valid", "*"))
	if err != nil {
		fail(err)
	}
	sort.Strings(names)
	for _, dir := range names {
		name := filepath.Base(dir)
		files, err := archive.ReadDir(filepath.Join(dir, "src"))
		if err != nil {
			fail(err)
		}
		a, err := archive.Pack(files)
		if err != nil {
			fail(err)
		}
		write(filepath.Join(dir, "archive.tgz"), a.Gzip)
		c := validCase{Name: name, Src: "valid/" + name + "/src", Archive: "valid/" + name + "/archive.tgz", TarSha256: archive.Hash(a.Tar)}
		for _, f := range a.Files {
			c.Files = append(c.Files, fileEntry{Path: f.Path, Mode: fmt.Sprintf("%04o", f.Mode), Sha256: archive.Hash(f.Data)})
		}
		out.Valid = append(out.Valid, c)
	}

	reg := func(name string) *tar.Header { return &tar.Header{Name: name, Mode: 0o644, Typeflag: tar.TypeReg} }
	hostile := []struct {
		name, code string
		gz         []byte
	}{
		{"not-gzip", "archive_invalid", []byte("this is not gzip")},
		{"gzip-not-tar", "archive_invalid", gz([]byte("this is gzip but not a tar stream"))},
		{"symlink", "path_forbidden", raw(&tar.Header{Name: "link", Linkname: "hpm.json", Typeflag: tar.TypeSymlink})},
		{"hard-link", "path_forbidden", raw(&tar.Header{Name: "link", Linkname: "hpm.json", Typeflag: tar.TypeLink})},
		{"absolute-path", "path_forbidden", raw(reg("/etc/passwd"))},
		{"dot-dot", "path_forbidden", raw(reg("claude/../../escape.md"))},
		{"env-file", "env_file_present", raw(reg("hpm.json"), reg("claude/.env"))},
		{"env-suffix", "env_file_present", raw(reg("hpm.json"), reg("scripts/prod.env"))},
		{"pax-header", "archive_invalid", raw(&tar.Header{Name: "hpm.json", Mode: 0o644, Typeflag: tar.TypeReg, Format: tar.FormatPAX, PAXRecords: map[string]string{"comment": "x"}})},
	}
	for _, h := range hostile {
		write(filepath.Join(root, "invalid", h.name+".tgz"), h.gz)
		out.Invalid = append(out.Invalid, invalidCase{Name: h.name, Archive: "invalid/" + h.name + ".tgz", Code: h.code})
	}

	data, err := json.MarshalIndent(out, "", "  ")
	if err != nil {
		fail(err)
	}
	write(filepath.Join(root, "cases.json"), append(data, '\n'))
}

func gz(b []byte) []byte {
	var buf bytes.Buffer
	zw := gzip.NewWriter(&buf)
	zw.Write(b)
	zw.Close()
	return buf.Bytes()
}

func raw(hdrs ...*tar.Header) []byte {
	var tb bytes.Buffer
	tw := tar.NewWriter(&tb)
	for _, h := range hdrs {
		body := []byte("x")
		if h.Typeflag != tar.TypeReg {
			body = nil
		}
		h.Size = int64(len(body))
		if err := tw.WriteHeader(h); err != nil {
			fail(err)
		}
		tw.Write(body)
	}
	tw.Close()
	return gz(tb.Bytes())
}

func write(path string, b []byte) {
	if err := os.MkdirAll(filepath.Dir(path), 0o755); err != nil {
		fail(err)
	}
	if err := os.WriteFile(path, b, 0o644); err != nil {
		fail(err)
	}
}

func fail(err error) {
	fmt.Fprintln(os.Stderr, "genvectors:", err)
	os.Exit(1)
}
```

Run from `repos/cli`: `go run ./tools/genvectors ../contract/vectors/archives`

Run it twice and confirm `git -C ../contract status` shows no change after the second run. That is the determinism check across runs.

- [ ] **Step 3: Review the output by hand**

This is the step that makes the vectors an authority rather than a copy of the implementation. For `skill-only`:

```bash
cd ../contract/vectors/archives
gunzip -c valid/skill-only/archive.tgz | shasum -a 256          # equals tarSha256 without the prefix
gunzip -c valid/skill-only/archive.tgz | tar -tvf -             # four files, sorted, no directories, uid/gid 0, 1970 dates
shasum -a 256 valid/skill-only/src/hpm.json                     # equals that file's sha256 in cases.json
gunzip -c valid/exec-bit/archive.tgz | tar -tvf - | grep run.sh # mode -rwxr-xr-x
```

Put all four outputs in your report. If any disagrees with `cases.json`, the packer is wrong: stop and report.

- [ ] **Step 4: The contract's own test**

`repos/contract/test/archives.test.ts`. Bun has `Bun.gunzipSync`. The test checks everything it can without a tar parser: the tar hash, that every listed source file exists with the listed hash, and that every invalid archive exists.

```ts
import { expect, test } from "bun:test";
import { createHash } from "node:crypto";
import { readFileSync, statSync } from "node:fs";
import { join } from "node:path";

const root = join(import.meta.dir, "..", "vectors", "archives");
const cases = JSON.parse(readFileSync(join(root, "cases.json"), "utf8"));
const sha = (b: Uint8Array) => "sha256-" + createHash("sha256").update(b).digest("hex");
const CODES = ["archive_invalid", "path_forbidden", "env_file_present", "archive_too_large"];

test("there are valid and invalid archive vectors", () => {
  expect(cases.valid.length).toBeGreaterThanOrEqual(2);
  expect(cases.invalid.length).toBeGreaterThanOrEqual(8);
});

test.each(cases.valid.map((c: any) => [c.name, c] as const))("archive vector: %s", (_n, c: any) => {
  const tar = Bun.gunzipSync(readFileSync(join(root, c.archive)));
  expect(sha(tar)).toBe(c.tarSha256);
  expect(tar.length % 512).toBe(0);
  const paths = c.files.map((f: any) => f.path);
  expect(paths).toEqual([...paths].sort());
  for (const f of c.files) {
    expect(sha(readFileSync(join(root, c.src, f.path))), f.path).toBe(f.sha256);
    expect(["0644", "0755"]).toContain(f.mode);
    const exec = (statSync(join(root, c.src, f.path)).mode & 0o111) !== 0;
    expect(f.mode, f.path).toBe(exec ? "0755" : "0644");
  }
});

test.each(cases.invalid.map((c: any) => [c.name, c] as const))("hostile archive: %s", (_n, c: any) => {
  expect(statSync(join(root, c.archive)).size).toBeGreaterThan(0);
  expect(CODES).toContain(c.code);
});
```

Write this test before running the generator if you are following the order strictly: it fails with `ENOENT … cases.json`, then passes once Step 2 has run.

- [ ] **Step 5: ⚠ Release v0.4.0**

`VERSION` → `0.4.0`. `CHANGELOG.md`:

```markdown
## [0.4.0] - 2026-09-19
- Added: archive vectors. Valid cases pin the hash of the uncompressed tar and every file; hostile cases pin the error code
```

README vectors row: mention `vectors/archives/`. Then the same PR, merge, tag sequence as Task 1 Step 6, with `v0.4.0`. `.tgz` files are binary: add a `.gitattributes` with `*.tgz binary` so no line-ending conversion can ever touch them.

- [ ] **Step 6: Import v0.4.0 and close out**

In `cli`: commit `tools/genvectors`, then `scripts/contract-pull.sh v0.4.0`. Add one test that closes the loop, `internal/archive/vectors_test.go`:

```go
package archive

import (
	"encoding/json"
	"errors"
	"os"
	"path/filepath"
	"testing"
)

const vectorRoot = "../../contract/vectors/archives"

func TestArchiveVectors(t *testing.T) {
	raw, err := os.ReadFile(filepath.Join(vectorRoot, "cases.json"))
	if err != nil {
		t.Fatal(err)
	}
	var cases struct {
		Valid []struct {
			Name, Src, Archive, TarSha256 string
			Files                         []struct{ Path, Mode, Sha256 string }
		}
		Invalid []struct{ Name, Archive, Code string }
	}
	if err := json.Unmarshal(raw, &cases); err != nil {
		t.Fatal(err)
	}
	for _, c := range cases.Valid {
		t.Run(c.Name, func(t *testing.T) {
			files, err := ReadDir(filepath.Join(vectorRoot, c.Src))
			if err != nil {
				t.Fatal(err)
			}
			a, err := Pack(files)
			if err != nil {
				t.Fatal(err)
			}
			if got := Hash(a.Tar); got != c.TarSha256 {
				t.Fatalf("tar hash %s, want %s", got, c.TarSha256)
			}
			stored, err := os.ReadFile(filepath.Join(vectorRoot, c.Archive))
			if err != nil {
				t.Fatal(err)
			}
			unpacked, err := Unpack(stored, 1<<24)
			if err != nil {
				t.Fatal(err)
			}
			if len(unpacked) != len(c.Files) {
				t.Fatalf("%d files, want %d", len(unpacked), len(c.Files))
			}
			for i, f := range unpacked {
				if f.Path != c.Files[i].Path || Hash(f.Data) != c.Files[i].Sha256 {
					t.Errorf("file %d: %s %s", i, f.Path, Hash(f.Data))
				}
			}
		})
	}
	for _, c := range cases.Invalid {
		t.Run(c.Name, func(t *testing.T) {
			if c.Name == "pax-header" {
				t.Skip("Go's tar reader consumes pax headers before we see them; the registry enforces this one")
			}
			stored, err := os.ReadFile(filepath.Join(vectorRoot, c.Archive))
			if err != nil {
				t.Fatal(err)
			}
			_, err = Unpack(stored, 1<<24)
			var re *RuleError
			if !errors.As(err, &re) || re.Code != c.Code {
				t.Fatalf("got %v, want code %s", err, c.Code)
			}
		})
	}
}
```

Git does not preserve file modes beyond the executable bit, and a checkout on a system with a restrictive umask still yields 0644 or 0755 after `ReadDir` normalises, so the tar hash is stable across machines. If this test fails in CI but passes locally, that assumption is wrong: report it.

In `registry`: `scripts/contract-pull.sh v0.4.0`, `bun run check`. No code uses the vectors until M1-B.

⚠ Push both `m1a` branches, watch both PRs' checks to green, and merge both with merge commits once Rick agrees.

---

## Exit check for M1-A

- [ ] Contract tags `v0.3.0` and `v0.4.0` are published with green CI. `v0.1.0` and `v0.2.0` are untouched.
- [ ] Every contract item in `2026-09-19-m0-carryover.md` under "Contract (next release)" is ticked, except the canonical JSON vectors, which stay for M4, and the `bundle.ts` reminder.
- [ ] Every item under "Both consumers" is ticked, except `CODEOWNERS`, which is Rick's decision, and shipping the check script inside the contract.
- [ ] `repos/cli`: `make check` passes, including `TestArchiveVectors`. `api.IndexIndexVersion.Harnesses` is `[]string`.
- [ ] `repos/registry`: `bun run check` passes at contract v0.4.0.
- [ ] The two pinning scripts are byte-identical across the consumers.
- [ ] The hand review of the archive vectors (Task 4 Step 3) is in the report, with real command output.
