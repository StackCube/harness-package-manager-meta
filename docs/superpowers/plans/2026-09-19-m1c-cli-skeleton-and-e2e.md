# M1-C: CLI Walking Skeleton and End-to-End Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `hpm init`, `publish <dir>`, `add`, `install` and `status` working against the real registry Worker for one package with no dependencies and plain files, proven by an end-to-end test that publishes, installs, detects a hand edit and refuses to overwrite it.

**Architecture:** Every mutating command builds a pure `Plan` from the project manifest, the lockfile, a read-only snapshot of the tree and unpacked archives, checks it for conflicts, and only then applies it, lockfile last. The Claude Code adapter is data plus one mapping function. The registry client wraps the generated API client, adds identity through an `auth.Provider`, and turns every failure into one error type that carries the contract code when there is one and survives a non-JSON body when there is not. The end-to-end harness runs the real Worker under `wrangler dev` behind a fake Access proxy written in Go, so the CLI's auth path and the Worker's verification path are both real.

**Tech Stack:** Go 1.26, cobra, `Masterminds/semver/v3`, the generated `internal/api` client, `internal/archive` from M1-A, standard library `crypto/ecdsa` and `net/http/httputil` for the fake Access proxy.

**Spec:** `docs/superpowers/specs/2026-09-19-iteration-1-technical-design.md` §4, §5 (adapter only; merge is M4), §7.2, §7.4. Requires M1-A merged in `cli` and M1-B merged in `registry`.

## Global Constraints

- Repo: `repos/cli`, branch `m1c`, one pull request. The last task also edits the workspace root repo on a branch `m1-docs`. Push, PR and merge are ⚠ steps.
- `hpm` never runs git and never writes outside the project root it was pointed at.
- Nothing touches disk before the plan has been checked. A conflict means zero writes. The lockfile is written last.
- In M1 a package with dependencies, a package that ships a merge fragment, and a non-empty `aliases` are each refused with a message that names the milestone that adds them (M3, M4, iteration 2).
- M1 has one auth provider: the Access service token, from `HPM_ACCESS_CLIENT_ID` and `HPM_ACCESS_CLIENT_SECRET`. `hpm login` is M2. When the variables are unset the error says so.
- Files `hpm` writes are deterministic: two-space indented JSON, keys in a stable order, one trailing newline, no HTML escaping.
- Harness ids, never folder names, in the manifest and lockfile. The adapter owns the mapping `claude-code` → package folder `claude/` → project root `.claude/`.
- Hashes are `archive.Hash`: `sha256-<hex>` over raw bytes, no normalisation.
- The CLI switches on `registry.Error.Code`, never on message text, and treats an unknown or missing code as a generic failure that reports the HTTP status.
- Commit messages end with a real trailer: blank line, then `Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>`.
- Test-first throughout. Pure packages are tested against `tree.Mem`, never the real filesystem, except `apply`.

## File map

```
repos/cli/
  internal/manifest/project.go  lock.go  pkg.go  write.go  names.go  *_test.go
  internal/auth/auth.go  auth_test.go
  internal/registry/client.go  client_test.go
  internal/tree/tree.go  tree_test.go
  internal/adapter/adapter.go  claude.go  claude_test.go
  internal/plan/plan.go  build.go  plan_test.go
  internal/apply/apply.go  apply_test.go
  internal/status/status.go  status_test.go
  internal/resolve/pick.go  pick_test.go
  internal/cli/root.go  context.go  init.go  publish.go  add.go  install.go  status.go  flow_test.go
  e2e/fakeaccess/fakeaccess.go  fakeaccess_test.go
  e2e/e2e_test.go                 build tag: e2e
  e2e/registry.ref                pinned registry commit for CI
  Makefile  .github/workflows/ci.yml  README.md
```

---

### Task 1: Manifest, lockfile and names

**Files:**
- Create: `internal/manifest/names.go`, `project.go`, `lock.go`, `pkg.go`, `write.go`, `manifest_test.go`

**Interfaces:**
- Consumes: `manifest.Validate(kind, data)` and the `Kind` constants from M0.
- Produces:

```go
const ProjectFile, LockFile = "hpm.json", "hpm.lock"

func SplitName(full string) (scope, short string, err error) // "@ourorg/tdd-loop" → "@ourorg", "tdd-loop"

type RegistryRef struct{ URL, Auth string }   // JSON: a string, or {"url","auth"}
type Project struct {
	Schema       string                 `json:"$schema,omitempty"`
	Harnesses    []string               `json:"harnesses"`
	Registries   map[string]RegistryRef `json:"registries"`
	Dependencies map[string]string      `json:"dependencies"`
	Aliases      map[string]string      `json:"aliases,omitempty"`
	Unsupported  string                 `json:"unsupported,omitempty"`
}
func (p *Project) RegistryFor(pkg string) (RegistryRef, error)

type Locked struct {
	Version        string                       `json:"version"`
	Registry       string                       `json:"registry"`
	Integrity      string                       `json:"integrity"`
	RequestedBy    []string                     `json:"requestedBy,omitempty"`
	State          string                       `json:"state"`
	Files          map[string]map[string]string `json:"files,omitempty"` // harness id → path → hash
	MissingHarness []string                     `json:"missingHarness,omitempty"`
}
type Lock struct {
	LockfileVersion int                `json:"lockfileVersion"`
	Packages        map[string]*Locked `json:"packages"`
	Unmatched       []string           `json:"unmatched,omitempty"`
}
func NewLock() *Lock
func (l *Lock) Clone() *Lock
func (l *Lock) Owner(path string) (pkg string, ok bool)

type Package struct {
	Name         string            `json:"name"`
	Version      string            `json:"version"`
	Dependencies map[string]string `json:"dependencies"`
}

func LoadProject(dir string) (*Project, error)  // validates against the schema first
func LoadLock(dir string) (*Lock, error)        // errors.Is(err, fs.ErrNotExist) when absent
func ParsePackage(data []byte) (*Package, error) // validates against the schema first
func IsProjectManifest(data []byte) bool        // has a "harnesses" member
func Encode(v any) ([]byte, error)              // deterministic JSON with a trailing newline
```

- [ ] **Step 1: Failing tests**

`internal/manifest/manifest_test.go`:

```go
package manifest

import (
	"errors"
	"io/fs"
	"os"
	"path/filepath"
	"testing"
)

func TestSplitName(t *testing.T) {
	scope, short, err := SplitName("@ourorg/tdd-loop")
	if err != nil || scope != "@ourorg" || short != "tdd-loop" {
		t.Fatalf("got %q %q %v", scope, short, err)
	}
	for _, bad := range []string{"tdd-loop", "@ourorg", "@ourorg/", "@/x", "@a/b/c", ""} {
		if _, _, err := SplitName(bad); err == nil {
			t.Errorf("SplitName(%q) should fail", bad)
		}
	}
}

func TestProjectRoundTripKeepsBothRegistryForms(t *testing.T) {
	dir := t.TempDir()
	src, err := os.ReadFile("../../contract/vectors/manifests/valid/project.json")
	if err != nil {
		t.Fatal(err)
	}
	if err := os.WriteFile(filepath.Join(dir, ProjectFile), src, 0o644); err != nil {
		t.Fatal(err)
	}
	p, err := LoadProject(dir)
	if err != nil {
		t.Fatal(err)
	}
	if p.Registries["@ourorg"] != (RegistryRef{URL: "https://hpm.ourorg.example"}) {
		t.Fatalf("string form: %+v", p.Registries["@ourorg"])
	}
	if p.Registries["@acme"] != (RegistryRef{URL: "https://hpm.acme.example", Auth: "cloudflare-access"}) {
		t.Fatalf("object form: %+v", p.Registries["@acme"])
	}
	out, err := Encode(p)
	if err != nil {
		t.Fatal(err)
	}
	if err := Validate(ProjectManifest, out); err != nil {
		t.Fatalf("encoded project no longer validates: %v\n%s", err, out)
	}
	again, _ := Encode(p)
	if string(out) != string(again) || out[len(out)-1] != '\n' {
		t.Fatal("Encode is not deterministic or lacks a trailing newline")
	}
	ref, err := p.RegistryFor("@acme/adr-writer")
	if err != nil || ref.URL != "https://hpm.acme.example" {
		t.Fatalf("RegistryFor: %+v %v", ref, err)
	}
	if _, err := p.RegistryFor("@nobody/x"); err == nil {
		t.Fatal("expected an error for an unmapped scope")
	}
}

func TestLoadProjectRejectsInvalid(t *testing.T) {
	dir := t.TempDir()
	os.WriteFile(filepath.Join(dir, ProjectFile), []byte(`{"harnesses":[],"registries":{},"dependencies":{}}`), 0o644)
	if _, err := LoadProject(dir); err == nil {
		t.Fatal("expected a schema error")
	}
}

func TestLockLoadCloneOwner(t *testing.T) {
	dir := t.TempDir()
	if _, err := LoadLock(dir); !errors.Is(err, fs.ErrNotExist) {
		t.Fatalf("missing lock: got %v", err)
	}
	src, _ := os.ReadFile("../../contract/vectors/manifests/valid/lockfile.json")
	os.WriteFile(filepath.Join(dir, LockFile), src, 0o644)
	l, err := LoadLock(dir)
	if err != nil {
		t.Fatal(err)
	}
	if pkg, ok := l.Owner(".claude/skills/tdd-loop/SKILL.md"); !ok || pkg != "@ourorg/tdd-loop" {
		t.Fatalf("Owner: %q %v", pkg, ok)
	}
	c := l.Clone()
	c.Packages["@ourorg/tdd-loop"].Files["claude-code"]["x"] = "y"
	if _, leaked := l.Packages["@ourorg/tdd-loop"].Files["claude-code"]["x"]; leaked {
		t.Fatal("Clone shares maps with the original")
	}
}

func TestEncodeDoesNotEscapeHTML(t *testing.T) {
	out, _ := Encode(map[string]string{"bin": "gh>=2.40"})
	if string(out) != "{\n  \"bin\": \"gh>=2.40\"\n}\n" {
		t.Fatalf("got %q", out)
	}
}

func TestParsePackageAndKindSniffing(t *testing.T) {
	src, _ := os.ReadFile("../../contract/vectors/manifests/valid/package-meta.json")
	p, err := ParsePackage(src)
	if err != nil || p.Name != "@ourorg/backend-toolkit" || len(p.Dependencies) != 2 {
		t.Fatalf("%+v %v", p, err)
	}
	if IsProjectManifest(src) {
		t.Fatal("a package manifest was taken for a project manifest")
	}
	proj, _ := os.ReadFile("../../contract/vectors/manifests/valid/project.json")
	if !IsProjectManifest(proj) {
		t.Fatal("a project manifest was not recognised")
	}
}
```

The contract's valid lockfile vector carries `managedEntries` and `managedBlocks`, which `Locked` does not model until M4. `LoadLock` must therefore not silently drop them on a later save: decode unknown members into `Extra map[string]json.RawMessage` on `Locked` with custom `UnmarshalJSON` and `MarshalJSON`, and add a test that a load-then-`Encode` of the vector still validates and still contains `"managedBlocks"`.

Run: `go test ./internal/manifest/` — Expected: FAIL, undefined symbols.

- [ ] **Step 2: Implement**

`internal/manifest/names.go`:

```go
package manifest

import (
	"fmt"
	"strings"
)

// SplitName splits "@scope/name" into "@scope" and "name".
func SplitName(full string) (scope, short string, err error) {
	scope, short, ok := strings.Cut(full, "/")
	if !ok || len(scope) < 2 || scope[0] != '@' || short == "" || strings.Contains(short, "/") {
		return "", "", fmt.Errorf("%q is not a package name of the form @scope/name", full)
	}
	return scope, short, nil
}
```

`internal/manifest/write.go`:

```go
package manifest

import (
	"bytes"
	"encoding/json"
)

// Encode renders v the way hpm writes every JSON file: two-space indent, stable key order
// (struct order, then sorted map keys), no HTML escaping, one trailing newline.
func Encode(v any) ([]byte, error) {
	var buf bytes.Buffer
	enc := json.NewEncoder(&buf)
	enc.SetEscapeHTML(false)
	enc.SetIndent("", "  ")
	if err := enc.Encode(v); err != nil {
		return nil, err
	}
	return buf.Bytes(), nil
}
```

`internal/manifest/project.go`:

```go
package manifest

import (
	"encoding/json"
	"fmt"
	"os"
	"path/filepath"
)

const (
	ProjectFile = "hpm.json"
	LockFile    = "hpm.lock"
)

// RegistryRef is one entry of a project's registries map.
type RegistryRef struct {
	URL  string
	Auth string // "" means the default provider
}

func (r *RegistryRef) UnmarshalJSON(b []byte) error {
	var s string
	if json.Unmarshal(b, &s) == nil {
		*r = RegistryRef{URL: s}
		return nil
	}
	var o struct {
		URL  string `json:"url"`
		Auth string `json:"auth"`
	}
	if err := json.Unmarshal(b, &o); err != nil {
		return err
	}
	*r = RegistryRef{URL: o.URL, Auth: o.Auth}
	return nil
}

func (r RegistryRef) MarshalJSON() ([]byte, error) {
	if r.Auth == "" {
		return json.Marshal(r.URL)
	}
	return json.Marshal(struct {
		URL  string `json:"url"`
		Auth string `json:"auth"`
	}{r.URL, r.Auth})
}

// Project is hpm.json in a project.
type Project struct {
	Schema       string                 `json:"$schema,omitempty"`
	Harnesses    []string               `json:"harnesses"`
	Registries   map[string]RegistryRef `json:"registries"`
	Dependencies map[string]string      `json:"dependencies"`
	Aliases      map[string]string      `json:"aliases,omitempty"`
	Unsupported  string                 `json:"unsupported,omitempty"`
}

// RegistryFor returns the registry a package's scope routes to.
func (p *Project) RegistryFor(pkg string) (RegistryRef, error) {
	scope, _, err := SplitName(pkg)
	if err != nil {
		return RegistryRef{}, err
	}
	ref, ok := p.Registries[scope]
	if !ok {
		return RegistryRef{}, fmt.Errorf("no registry is mapped for scope %s in %s", scope, ProjectFile)
	}
	return ref, nil
}

// LoadProject reads and validates dir/hpm.json.
func LoadProject(dir string) (*Project, error) {
	data, err := os.ReadFile(filepath.Join(dir, ProjectFile))
	if err != nil {
		return nil, err
	}
	if err := Validate(ProjectManifest, data); err != nil {
		return nil, fmt.Errorf("%s: %w", ProjectFile, err)
	}
	var p Project
	if err := json.Unmarshal(data, &p); err != nil {
		return nil, fmt.Errorf("%s: %w", ProjectFile, err)
	}
	return &p, nil
}

// IsProjectManifest tells a project's hpm.json from a package's: only a project has "harnesses".
func IsProjectManifest(data []byte) bool {
	var probe map[string]json.RawMessage
	if json.Unmarshal(data, &probe) != nil {
		return false
	}
	_, ok := probe["harnesses"]
	return ok
}
```

`internal/manifest/pkg.go`:

```go
package manifest

import (
	"encoding/json"
	"fmt"
)

// Package is the part of a package's hpm.json the CLI acts on.
type Package struct {
	Name         string            `json:"name"`
	Version      string            `json:"version"`
	Dependencies map[string]string `json:"dependencies"`
}

// ParsePackage validates data against the package manifest schema and decodes it.
func ParsePackage(data []byte) (*Package, error) {
	if err := Validate(PackageManifest, data); err != nil {
		return nil, fmt.Errorf("package manifest: %w", err)
	}
	var p Package
	if err := json.Unmarshal(data, &p); err != nil {
		return nil, err
	}
	return &p, nil
}
```

`internal/manifest/lock.go`. `Locked` keeps members it does not model in `Extra`, so an M1 binary never destroys what a later version wrote.

```go
package manifest

import (
	"encoding/json"
	"fmt"
	"os"
	"path/filepath"
)

type Locked struct {
	Version        string
	Registry       string
	Integrity      string
	RequestedBy    []string
	State          string
	Files          map[string]map[string]string
	MissingHarness []string
	Extra          map[string]json.RawMessage
}

var lockedKnown = []string{"version", "registry", "integrity", "requestedBy", "state", "files", "missingHarness"}

func (l *Locked) UnmarshalJSON(b []byte) error {
	var raw map[string]json.RawMessage
	if err := json.Unmarshal(b, &raw); err != nil {
		return err
	}
	fields := map[string]any{
		"version": &l.Version, "registry": &l.Registry, "integrity": &l.Integrity,
		"requestedBy": &l.RequestedBy, "state": &l.State, "files": &l.Files, "missingHarness": &l.MissingHarness,
	}
	for _, k := range lockedKnown {
		if v, ok := raw[k]; ok {
			if err := json.Unmarshal(v, fields[k]); err != nil {
				return fmt.Errorf("%s: %w", k, err)
			}
			delete(raw, k)
		}
	}
	if len(raw) > 0 {
		l.Extra = raw
	}
	return nil
}

func (l Locked) MarshalJSON() ([]byte, error) {
	out := map[string]any{"version": l.Version, "registry": l.Registry, "integrity": l.Integrity, "state": l.State}
	if len(l.RequestedBy) > 0 {
		out["requestedBy"] = l.RequestedBy
	}
	if len(l.Files) > 0 {
		out["files"] = l.Files
	}
	if len(l.MissingHarness) > 0 {
		out["missingHarness"] = l.MissingHarness
	}
	for k, v := range l.Extra {
		out[k] = v
	}
	return json.Marshal(out)
}

type Lock struct {
	LockfileVersion int                `json:"lockfileVersion"`
	Packages        map[string]*Locked `json:"packages"`
	Unmatched       []string           `json:"unmatched,omitempty"`
}

func NewLock() *Lock { return &Lock{LockfileVersion: 1, Packages: map[string]*Locked{}} }

// Clone returns a deep copy.
func (l *Lock) Clone() *Lock {
	data, _ := json.Marshal(l)
	var c Lock
	_ = json.Unmarshal(data, &c)
	if c.Packages == nil {
		c.Packages = map[string]*Locked{}
	}
	return &c
}

// Owner reports which package owns a project path.
func (l *Lock) Owner(path string) (string, bool) {
	for name, p := range l.Packages {
		for _, files := range p.Files {
			if _, ok := files[path]; ok {
				return name, true
			}
		}
	}
	return "", false
}

// LoadLock reads and validates dir/hpm.lock.
func LoadLock(dir string) (*Lock, error) {
	data, err := os.ReadFile(filepath.Join(dir, LockFile))
	if err != nil {
		return nil, err
	}
	if err := Validate(Lockfile, data); err != nil {
		return nil, fmt.Errorf("%s: %w", LockFile, err)
	}
	var l Lock
	if err := json.Unmarshal(data, &l); err != nil {
		return nil, fmt.Errorf("%s: %w", LockFile, err)
	}
	if l.Packages == nil {
		l.Packages = map[string]*Locked{}
	}
	return &l, nil
}
```

A package's members marshal through a map, so they come out sorted by key. That is stable, which is what determinism needs; it is not the order in the PRD's example, and that is fine.

Run: `go test ./internal/manifest/` — Expected: PASS. Commit: "Project manifest, lockfile and package manifest types".

---

### Task 2: Identity and the registry client

**Files:**
- Create: `internal/auth/auth.go`, `internal/auth/auth_test.go`, `internal/registry/client.go`, `internal/registry/client_test.go`

**Interfaces:**
- Consumes: generated `api.NewClient`, `api.WithHTTPClient`, `api.WithRequestEditorFn`, `api.GetIndexParams`, `api.Index`, `api.PublishResult`, `api.Error`; `manifest.RegistryRef`, `manifest.SplitName`.
- Produces:

```go
// internal/auth
type Provider interface{ Authorize(req *http.Request) error }
func ForRegistry(ref manifest.RegistryRef, getenv func(string) string) (Provider, error)

// internal/registry
type Error struct {
	Status  int
	Code    string // a contract error code, or "" when the body was not a contract error
	Message string
}
func (e *Error) Error() string
type Client struct{ /* unexported */ }
func New(baseURL string, p auth.Provider, hc *http.Client) (*Client, error)
func (c *Client) Index(ctx context.Context, scope string) (*api.Index, error)
func (c *Client) Archive(ctx context.Context, pkg, version string) ([]byte, error)
func (c *Client) Publish(ctx context.Context, pkg, version string, gz []byte) (*api.PublishResult, error)
```

- [ ] **Step 1: Failing tests**

`internal/auth/auth_test.go`:

```go
package auth

import (
	"net/http"
	"strings"
	"testing"

	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
)

func env(m map[string]string) func(string) string { return func(k string) string { return m[k] } }

func TestServiceTokenFromEnvironment(t *testing.T) {
	p, err := ForRegistry(manifest.RegistryRef{URL: "https://hpm.test"}, env(map[string]string{
		"HPM_ACCESS_CLIENT_ID": "abc.access", "HPM_ACCESS_CLIENT_SECRET": "s3cret",
	}))
	if err != nil {
		t.Fatal(err)
	}
	req, _ := http.NewRequest("GET", "https://hpm.test/v1/index", nil)
	if err := p.Authorize(req); err != nil {
		t.Fatal(err)
	}
	if req.Header.Get("CF-Access-Client-Id") != "abc.access" || req.Header.Get("CF-Access-Client-Secret") != "s3cret" {
		t.Fatalf("headers: %v", req.Header)
	}
}

func TestMissingCredentialsExplainWhatToDo(t *testing.T) {
	_, err := ForRegistry(manifest.RegistryRef{URL: "https://hpm.test"}, env(nil))
	if err == nil || !strings.Contains(err.Error(), "HPM_ACCESS_CLIENT_ID") || !strings.Contains(err.Error(), "hpm login") {
		t.Fatalf("got %v", err)
	}
}

func TestUnknownAuthKind(t *testing.T) {
	_, err := ForRegistry(manifest.RegistryRef{URL: "https://hpm.test", Auth: "oidc"}, env(nil))
	if err == nil || !strings.Contains(err.Error(), "oidc") {
		t.Fatalf("got %v", err)
	}
}
```

`internal/registry/client_test.go`:

```go
package registry

import (
	"context"
	"errors"
	"io"
	"net/http"
	"net/http/httptest"
	"testing"
)

type stamp struct{}

func (stamp) Authorize(r *http.Request) error { r.Header.Set("X-Test-Auth", "yes"); return nil }

func serve(t *testing.T, h http.HandlerFunc) *Client {
	t.Helper()
	srv := httptest.NewServer(h)
	t.Cleanup(srv.Close)
	c, err := New(srv.URL, stamp{}, srv.Client())
	if err != nil {
		t.Fatal(err)
	}
	return c
}

func TestIndexSendsScopeAndIdentity(t *testing.T) {
	c := serve(t, func(w http.ResponseWriter, r *http.Request) {
		if r.URL.Path != "/v1/index" || r.URL.Query().Get("scope") != "@ourorg" || r.Header.Get("X-Test-Auth") != "yes" {
			t.Errorf("unexpected request: %s %v", r.URL, r.Header)
		}
		w.Header().Set("content-type", "application/json")
		io.WriteString(w, `{"packages":[{"name":"@ourorg/x","description":"d","keywords":[],"versions":[]}]}`)
	})
	idx, err := c.Index(context.Background(), "@ourorg")
	if err != nil || len(idx.Packages) != 1 || idx.Packages[0].Name != "@ourorg/x" {
		t.Fatalf("%+v %v", idx, err)
	}
}

func TestContractErrorsCarryTheirCode(t *testing.T) {
	c := serve(t, func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("content-type", "application/json")
		w.WriteHeader(409)
		io.WriteString(w, `{"code":"version_exists","message":"1.0.0 is already published","details":{}}`)
	})
	_, err := c.Publish(context.Background(), "@ourorg/x", "1.0.0", []byte("gz"))
	var re *Error
	if !errors.As(err, &re) || re.Code != "version_exists" || re.Status != 409 {
		t.Fatalf("got %v", err)
	}
}

func TestNonJSONErrorBodiesSurvive(t *testing.T) {
	c := serve(t, func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("content-type", "text/html")
		w.WriteHeader(403)
		io.WriteString(w, "<html><body>Forbidden by the identity layer</body></html>")
	})
	_, err := c.Index(context.Background(), "")
	var re *Error
	if !errors.As(err, &re) || re.Code != "" || re.Status != 403 {
		t.Fatalf("got %v", err)
	}
	if got := re.Error(); got == "" || len(got) > 300 {
		t.Fatalf("message should be short and non-empty, got %q", got)
	}
}

func TestPublishAndArchiveUseThePackagePath(t *testing.T) {
	c := serve(t, func(w http.ResponseWriter, r *http.Request) {
		switch {
		case r.Method == "PUT" && r.URL.Path == "/v1/pkg/@ourorg/x/1.0.0":
			body, _ := io.ReadAll(r.Body)
			if string(body) != "gz" || r.Header.Get("content-type") != "application/gzip" {
				t.Errorf("publish body %q type %q", body, r.Header.Get("content-type"))
			}
			w.Header().Set("content-type", "application/json")
			w.WriteHeader(201)
			io.WriteString(w, `{"name":"@ourorg/x","version":"1.0.0","integrity":"sha256-aa","size":2,"publishedAt":"2026-09-19T10:00:00Z","publishedBy":"ci"}`)
		case r.Method == "GET" && r.URL.Path == "/v1/archive/@ourorg/x/1.0.0":
			io.WriteString(w, "bytes")
		default:
			t.Errorf("unexpected %s %s", r.Method, r.URL.Path)
			w.WriteHeader(500)
		}
	})
	res, err := c.Publish(context.Background(), "@ourorg/x", "1.0.0", []byte("gz"))
	if err != nil || res.PublishedBy != "ci" {
		t.Fatalf("%+v %v", res, err)
	}
	got, err := c.Archive(context.Background(), "@ourorg/x", "1.0.0")
	if err != nil || string(got) != "bytes" {
		t.Fatalf("%q %v", got, err)
	}
}
```

`r.URL.Path` is the decoded path, so the test passes whether the generated client sends `@` or `%40`.

Run: `go test ./internal/auth/ ./internal/registry/` — Expected: FAIL, undefined symbols.

- [ ] **Step 2: Implement**

`internal/auth/auth.go`:

```go
// Package auth attaches identity to registry requests.
package auth

import (
	"fmt"
	"net/http"

	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
)

// Provider adds whatever a registry's identity layer needs to a request.
type Provider interface {
	Authorize(req *http.Request) error
}

type serviceToken struct{ id, secret string }

func (s serviceToken) Authorize(req *http.Request) error {
	req.Header.Set("CF-Access-Client-Id", s.id)
	req.Header.Set("CF-Access-Client-Secret", s.secret)
	return nil
}

// ForRegistry picks the provider for a registry entry. M1 has the Access service token only.
func ForRegistry(ref manifest.RegistryRef, getenv func(string) string) (Provider, error) {
	switch ref.Auth {
	case "", "cloudflare-access":
	default:
		return nil, fmt.Errorf("registry %s asks for auth %q, which this version of hpm does not support", ref.URL, ref.Auth)
	}
	id, secret := getenv("HPM_ACCESS_CLIENT_ID"), getenv("HPM_ACCESS_CLIENT_SECRET")
	if id == "" || secret == "" {
		return nil, fmt.Errorf("no identity for %s: set HPM_ACCESS_CLIENT_ID and HPM_ACCESS_CLIENT_SECRET to an Access service token (browser login with `hpm login` arrives in M2)", ref.URL)
	}
	return serviceToken{id, secret}, nil
}
```

`internal/registry/client.go`:

```go
// Package registry is the CLI's view of one registry instance.
package registry

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"strings"

	"github.com/StackCube/harness-package-manager-cli/internal/api"
	"github.com/StackCube/harness-package-manager-cli/internal/auth"
	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
)

// Error is any failure a registry, or whatever sits in front of it, answered with.
type Error struct {
	Status  int
	Code    string
	Message string
}

func (e *Error) Error() string {
	if e.Code != "" {
		return fmt.Sprintf("%s: %s (HTTP %d)", e.Code, e.Message, e.Status)
	}
	return fmt.Sprintf("the registry answered HTTP %d: %s", e.Status, e.Message)
}

type Client struct{ api *api.Client }

func New(baseURL string, p auth.Provider, hc *http.Client) (*Client, error) {
	if hc == nil {
		hc = http.DefaultClient
	}
	c, err := api.NewClient(strings.TrimRight(baseURL, "/"), api.WithHTTPClient(hc),
		api.WithRequestEditorFn(func(_ context.Context, req *http.Request) error { return p.Authorize(req) }))
	if err != nil {
		return nil, err
	}
	return &Client{api: c}, nil
}

// read returns the body of a successful response, or an *Error.
func read(res *http.Response, err error, want int) ([]byte, error) {
	if err != nil {
		return nil, err
	}
	defer res.Body.Close()
	body, err := io.ReadAll(io.LimitReader(res.Body, 64<<20))
	if err != nil {
		return nil, err
	}
	if res.StatusCode == want {
		return body, nil
	}
	var ce api.Error
	if json.Unmarshal(body, &ce) == nil && ce.Code != "" {
		return nil, &Error{Status: res.StatusCode, Code: string(ce.Code), Message: ce.Message}
	}
	msg := strings.Join(strings.Fields(string(body)), " ")
	if len(msg) > 160 {
		msg = msg[:160] + "…"
	}
	if msg == "" {
		msg = http.StatusText(res.StatusCode)
	}
	return nil, &Error{Status: res.StatusCode, Message: msg}
}

func (c *Client) Index(ctx context.Context, scope string) (*api.Index, error) {
	params := &api.GetIndexParams{}
	if scope != "" {
		params.Scope = &scope
	}
	res, err := c.api.GetIndex(ctx, params)
	body, err := read(res, err, http.StatusOK)
	if err != nil {
		return nil, err
	}
	var idx api.Index
	if err := json.Unmarshal(body, &idx); err != nil {
		return nil, fmt.Errorf("the index is not valid JSON: %w", err)
	}
	return &idx, nil
}

func (c *Client) Archive(ctx context.Context, pkg, version string) ([]byte, error) {
	scope, name, err := manifest.SplitName(pkg)
	if err != nil {
		return nil, err
	}
	res, err := c.api.GetArchive(ctx, scope, name, version)
	return read(res, err, http.StatusOK)
}

func (c *Client) Publish(ctx context.Context, pkg, version string, gz []byte) (*api.PublishResult, error) {
	scope, name, err := manifest.SplitName(pkg)
	if err != nil {
		return nil, err
	}
	res, err := c.api.PublishVersionWithBody(ctx, scope, name, version, "application/gzip", bytes.NewReader(gz))
	body, err := read(res, err, http.StatusCreated)
	if err != nil {
		return nil, err
	}
	var out api.PublishResult
	if err := json.Unmarshal(body, &out); err != nil {
		return nil, fmt.Errorf("the publish result is not valid JSON: %w", err)
	}
	return &out, nil
}
```

If `GetIndexParams.Scope` has a different type in the generated code, adapt the two lines that set it. Nothing else may be adapted around the generated client.

Run: `go test ./internal/auth/ ./internal/registry/` — Expected: PASS. Commit: "Service-token identity and the registry client".

---

### Task 3: Tree snapshot and the Claude Code adapter

**Files:**
- Create: `internal/tree/tree.go`, `internal/tree/tree_test.go`, `internal/adapter/adapter.go`, `internal/adapter/claude.go`, `internal/adapter/claude_test.go`

**Interfaces:**
- Produces:

```go
// internal/tree
type Snapshot interface {
	ReadFile(path string) ([]byte, error) // fs.ErrNotExist when absent; path is slash-separated, relative
	HasDir(path string) bool
}
type Dir string             // the real filesystem under a root
type Mem map[string][]byte  // tests

// internal/adapter
type Kind int
const ( File Kind = iota; Fragment )
type Target struct{ Path string; Kind Kind }
type Adapter interface {
	ID() string          // "claude-code"
	PackageDir() string  // "claude"
	Detect(tree.Snapshot) bool
	MapIn(pkgPath string) (Target, bool)
}
func ByID(id string) (Adapter, bool)
func All() []Adapter
```

- [ ] **Step 1: Failing tests**

`internal/tree/tree_test.go`:

```go
package tree

import (
	"errors"
	"io/fs"
	"os"
	"path/filepath"
	"testing"
)

func TestMem(t *testing.T) {
	m := Mem{".claude/skills/x/SKILL.md": []byte("x")}
	if b, err := m.ReadFile(".claude/skills/x/SKILL.md"); err != nil || string(b) != "x" {
		t.Fatal(b, err)
	}
	if _, err := m.ReadFile("nope"); !errors.Is(err, fs.ErrNotExist) {
		t.Fatal(err)
	}
	if !m.HasDir(".claude") || !m.HasDir(".claude/skills") || m.HasDir(".clau") || m.HasDir(".kiro") {
		t.Fatal("HasDir is wrong")
	}
}

func TestDirStaysInsideItsRoot(t *testing.T) {
	root := t.TempDir()
	os.MkdirAll(filepath.Join(root, ".claude"), 0o755)
	os.WriteFile(filepath.Join(root, ".claude", "a.md"), []byte("a"), 0o644)
	d := Dir(root)
	if b, err := d.ReadFile(".claude/a.md"); err != nil || string(b) != "a" {
		t.Fatal(b, err)
	}
	if !d.HasDir(".claude") || d.HasDir(".claude/a.md") {
		t.Fatal("HasDir is wrong")
	}
	for _, p := range []string{"../x", "/etc/passwd", "a/../../x"} {
		if _, err := d.ReadFile(p); err == nil {
			t.Errorf("ReadFile(%q) escaped the root", p)
		}
	}
}
```

`internal/adapter/claude_test.go`:

```go
package adapter

import (
	"testing"

	"github.com/StackCube/harness-package-manager-cli/internal/tree"
)

func TestClaudeMapIn(t *testing.T) {
	a, ok := ByID("claude-code")
	if !ok || a.PackageDir() != "claude" {
		t.Fatal("claude-code adapter missing")
	}
	files := map[string]string{
		"claude/skills/tdd-loop/SKILL.md": ".claude/skills/tdd-loop/SKILL.md",
		"claude/commands/pr-review.md":    ".claude/commands/pr-review.md",
		"claude/scripts/run.sh":           ".claude/scripts/run.sh",
	}
	for in, want := range files {
		got, ok := a.MapIn(in)
		if !ok || got.Path != want || got.Kind != File {
			t.Errorf("MapIn(%q) = %+v %v", in, got, ok)
		}
	}
	fragments := map[string]string{
		"claude/CLAUDE.append.md":    "CLAUDE.md",
		"claude/settings.merge.json": ".claude/settings.json",
		"claude/mcp.merge.json":      ".mcp.json",
	}
	for in, want := range fragments {
		got, ok := a.MapIn(in)
		if !ok || got.Path != want || got.Kind != Fragment {
			t.Errorf("MapIn(%q) = %+v %v", in, got, ok)
		}
	}
	for _, in := range []string{"hpm.json", "README.md", "kiro/steering/x.md", "claude", "claudex/a.md"} {
		if _, ok := a.MapIn(in); ok {
			t.Errorf("MapIn(%q) should not map", in)
		}
	}
	if got, _ := a.MapIn("claude/settings.local.json"); got.Path == ".claude/settings.local.json" {
		t.Error("settings.local.json must never be written")
	}
}

func TestDetectAndRegistry(t *testing.T) {
	a, _ := ByID("claude-code")
	if !a.Detect(tree.Mem{".claude/settings.json": nil}) || a.Detect(tree.Mem{"src/main.go": nil}) {
		t.Fatal("Detect is wrong")
	}
	if _, ok := ByID("kiro"); ok {
		t.Fatal("only claude-code exists in M1")
	}
	if len(All()) != 1 {
		t.Fatal("All should list one adapter")
	}
}
```

Run: `go test ./internal/tree/ ./internal/adapter/` — Expected: FAIL.

- [ ] **Step 2: Implement**

`internal/tree/tree.go`:

```go
// Package tree gives planning code a read-only view of a project.
package tree

import (
	"fmt"
	"io/fs"
	"os"
	"path/filepath"
	"strings"
)

type Snapshot interface {
	ReadFile(path string) ([]byte, error)
	HasDir(path string) bool
}

// Dir is the real filesystem under a root.
type Dir string

func (d Dir) resolve(p string) (string, error) {
	if p == "" || strings.HasPrefix(p, "/") || !filepath.IsLocal(filepath.FromSlash(p)) {
		return "", fmt.Errorf("%q is not a path inside the project", p)
	}
	return filepath.Join(string(d), filepath.FromSlash(p)), nil
}

func (d Dir) ReadFile(p string) ([]byte, error) {
	full, err := d.resolve(p)
	if err != nil {
		return nil, err
	}
	return os.ReadFile(full)
}

func (d Dir) HasDir(p string) bool {
	full, err := d.resolve(p)
	if err != nil {
		return false
	}
	info, err := os.Stat(full)
	return err == nil && info.IsDir()
}

// Mem is an in-memory tree for tests.
type Mem map[string][]byte

func (m Mem) ReadFile(p string) ([]byte, error) {
	if b, ok := m[p]; ok {
		return b, nil
	}
	return nil, &fs.PathError{Op: "open", Path: p, Err: fs.ErrNotExist}
}

func (m Mem) HasDir(p string) bool {
	prefix := strings.TrimSuffix(p, "/") + "/"
	for k := range m {
		if strings.HasPrefix(k, prefix) {
			return true
		}
	}
	return false
}
```

`internal/adapter/adapter.go`:

```go
// Package adapter maps package folders onto harness layouts. Authors port; adapters copy.
package adapter

import "github.com/StackCube/harness-package-manager-cli/internal/tree"

type Kind int

const (
	File Kind = iota
	Fragment
)

type Target struct {
	Path string
	Kind Kind
}

type Adapter interface {
	ID() string
	PackageDir() string
	Detect(tree.Snapshot) bool
	MapIn(pkgPath string) (Target, bool)
}

var registry = []Adapter{claude{}}

func All() []Adapter { return registry }

func ByID(id string) (Adapter, bool) {
	for _, a := range registry {
		if a.ID() == id {
			return a, true
		}
	}
	return nil, false
}
```

`internal/adapter/claude.go`:

```go
package adapter

import (
	"strings"

	"github.com/StackCube/harness-package-manager-cli/internal/tree"
)

type claude struct{}

func (claude) ID() string         { return "claude-code" }
func (claude) PackageDir() string { return "claude" }

func (claude) Detect(s tree.Snapshot) bool { return s.HasDir(".claude") }

// Fragments are merged into files the project also edits by hand. The merge engine is M4.
var claudeFragments = map[string]string{
	"claude/CLAUDE.append.md":    "CLAUDE.md",
	"claude/settings.merge.json": ".claude/settings.json",
	"claude/mcp.merge.json":      ".mcp.json",
}

// Never written: the project's personal, uncommitted settings.
var claudeNever = map[string]bool{"claude/settings.local.json": true}

func (claude) MapIn(pkgPath string) (Target, bool) {
	if target, ok := claudeFragments[pkgPath]; ok {
		return Target{Path: target, Kind: Fragment}, true
	}
	rest, ok := strings.CutPrefix(pkgPath, "claude/")
	if !ok || rest == "" || claudeNever[pkgPath] {
		return Target{}, false
	}
	return Target{Path: ".claude/" + rest, Kind: File}, true
}
```

Run: `go test ./internal/tree/ ./internal/adapter/` — Expected: PASS. Commit: "Tree snapshot and the Claude Code adapter's forward mapping".

---

### Task 4: Plan, apply and status

**Files:**
- Create: `internal/plan/plan.go`, `internal/plan/build.go`, `internal/plan/plan_test.go`, `internal/apply/apply.go`, `internal/apply/apply_test.go`, `internal/status/status.go`, `internal/status/status_test.go`

**Interfaces:**
- Consumes: `tree.Snapshot`, `adapter.ByID`, `manifest.Project`, `manifest.Lock`, `manifest.Locked`, `manifest.Encode`, `archive.File`, `archive.Hash`.
- Produces:

```go
// internal/plan
type Write struct{ Path string; Mode int64; Data []byte; Owner string }
type Delete struct{ Path, Owner string }
type Plan struct {
	Writes   []Write   // sorted by Path
	Deletes  []Delete  // sorted by Path
	Project  *manifest.Project // nil when hpm.json does not change
	Lock     *manifest.Lock
	Warnings []string
}
type Install struct {
	Name, Version, Registry, Integrity string
	RequestedBy []string
	Files       []archive.File // the unpacked archive
}
type Conflict struct{ Kind, Path, Package, Detail string }
// Kinds: "owned" (another package owns the path), "unmanaged" (a file the project wrote is in the way),
// "modified" (the package's installed files were edited), "unsupported" (needs a later milestone),
// "harness" (no folder for a declared harness, and the project says fail).
type ConflictError struct{ Conflicts []Conflict }
func (e *ConflictError) Error() string
func Build(snap tree.Snapshot, proj *manifest.Project, lock *manifest.Lock, installs []Install) (*Plan, error)
func (p *Plan) Summary() string

// internal/apply
func Apply(root string, p *plan.Plan) error

// internal/status
type Package struct{ Name, Version, State string; Modified, Missing, MissingHarness []string }
func Check(snap tree.Snapshot, lock *manifest.Lock) []Package // sorted by Name; State is "managed" or "modified"
func Stale(proj *manifest.Project, lock *manifest.Lock) []string // human-readable reasons; empty when the lock satisfies the manifest
```

- [ ] **Step 1: Failing tests for the planner**

`internal/plan/plan_test.go`:

```go
package plan

import (
	"errors"
	"strings"
	"testing"

	"github.com/StackCube/harness-package-manager-cli/internal/archive"
	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
	"github.com/StackCube/harness-package-manager-cli/internal/tree"
)

func project() *manifest.Project {
	return &manifest.Project{
		Harnesses:    []string{"claude-code"},
		Registries:   map[string]manifest.RegistryRef{"@t": {URL: "https://r.test"}},
		Dependencies: map[string]string{},
	}
}

func ping(version, body string, extra ...archive.File) Install {
	files := []archive.File{
		{Path: "hpm.json", Mode: 0o644, Data: []byte("{}")},
		{Path: "README.md", Mode: 0o644, Data: []byte("# ping")},
		{Path: "claude/skills/ping/SKILL.md", Mode: 0o644, Data: []byte(body)},
	}
	return Install{Name: "@t/ping", Version: version, Registry: "https://r.test",
		Integrity: "sha256-" + strings.Repeat("a", 64), Files: append(files, extra...)}
}

func conflicts(t *testing.T, err error) []Conflict {
	t.Helper()
	var ce *ConflictError
	if !errors.As(err, &ce) {
		t.Fatalf("expected a ConflictError, got %v", err)
	}
	return ce.Conflicts
}

func TestFreshInstall(t *testing.T) {
	p, err := Build(tree.Mem{}, project(), manifest.NewLock(), []Install{ping("1.0.0", "v1")})
	if err != nil {
		t.Fatal(err)
	}
	if len(p.Writes) != 1 || p.Writes[0].Path != ".claude/skills/ping/SKILL.md" || p.Writes[0].Owner != "@t/ping" {
		t.Fatalf("writes: %+v", p.Writes)
	}
	l := p.Lock.Packages["@t/ping"]
	if l.Version != "1.0.0" || l.State != "managed" || l.Registry != "https://r.test" {
		t.Fatalf("lock: %+v", l)
	}
	if l.Files["claude-code"][".claude/skills/ping/SKILL.md"] != archive.Hash([]byte("v1")) {
		t.Fatalf("file hash: %+v", l.Files)
	}
}

func TestOnlyMappedFilesAreInstalled(t *testing.T) {
	in := ping("1.0.0", "v1", archive.File{Path: "kiro/steering/ping.md", Mode: 0o644, Data: []byte("k")})
	p, err := Build(tree.Mem{}, project(), manifest.NewLock(), []Install{in})
	if err != nil || len(p.Writes) != 1 {
		t.Fatalf("%+v %v", p, err)
	}
}

func TestIdenticalFileAlreadyPresentIsAdoptedWithoutAWrite(t *testing.T) {
	snap := tree.Mem{".claude/skills/ping/SKILL.md": []byte("v1")}
	p, err := Build(snap, project(), manifest.NewLock(), []Install{ping("1.0.0", "v1")})
	if err != nil || len(p.Writes) != 0 || p.Lock.Packages["@t/ping"] == nil {
		t.Fatalf("%+v %v", p, err)
	}
}

func TestUnmanagedFileInTheWay(t *testing.T) {
	snap := tree.Mem{".claude/skills/ping/SKILL.md": []byte("the project's own")}
	_, err := Build(snap, project(), manifest.NewLock(), []Install{ping("1.0.0", "v1")})
	c := conflicts(t, err)
	if len(c) != 1 || c[0].Kind != "unmanaged" || c[0].Path != ".claude/skills/ping/SKILL.md" {
		t.Fatalf("%+v", c)
	}
}

func TestPathOwnedByAnotherPackage(t *testing.T) {
	first, _ := Build(tree.Mem{}, project(), manifest.NewLock(), []Install{ping("1.0.0", "v1")})
	other := ping("1.0.0", "v1")
	other.Name = "@t/pong"
	snap := tree.Mem{".claude/skills/ping/SKILL.md": []byte("v1")}
	_, err := Build(snap, project(), first.Lock, []Install{other})
	c := conflicts(t, err)
	if len(c) != 1 || c[0].Kind != "owned" || !strings.Contains(c[0].Detail, "@t/ping") {
		t.Fatalf("%+v", c)
	}
}

func TestTwoInstallsInOnePlanCannotShareAPath(t *testing.T) {
	a, b := ping("1.0.0", "v1"), ping("1.0.0", "v1")
	b.Name = "@t/pong"
	_, err := Build(tree.Mem{}, project(), manifest.NewLock(), []Install{a, b})
	if c := conflicts(t, err); len(c) != 1 || c[0].Kind != "owned" {
		t.Fatalf("%+v", c)
	}
}

func TestModifiedPackageIsNeverOverwritten(t *testing.T) {
	first, _ := Build(tree.Mem{}, project(), manifest.NewLock(), []Install{ping("1.0.0", "v1")})
	snap := tree.Mem{".claude/skills/ping/SKILL.md": []byte("hand edited")}
	_, err := Build(snap, project(), first.Lock, []Install{ping("1.0.1", "v2")})
	c := conflicts(t, err)
	if len(c) != 1 || c[0].Kind != "modified" || c[0].Package != "@t/ping" {
		t.Fatalf("%+v", c)
	}
}

func TestUpgradeReplacesAndDeletesWhatTheNewVersionDropped(t *testing.T) {
	extra := archive.File{Path: "claude/skills/ping/notes.md", Mode: 0o644, Data: []byte("n")}
	first, _ := Build(tree.Mem{}, project(), manifest.NewLock(), []Install{ping("1.0.0", "v1", extra)})
	snap := tree.Mem{".claude/skills/ping/SKILL.md": []byte("v1"), ".claude/skills/ping/notes.md": []byte("n")}
	p, err := Build(snap, project(), first.Lock, []Install{ping("1.1.0", "v2")})
	if err != nil {
		t.Fatal(err)
	}
	if len(p.Writes) != 1 || string(p.Writes[0].Data) != "v2" {
		t.Fatalf("writes: %+v", p.Writes)
	}
	if len(p.Deletes) != 1 || p.Deletes[0].Path != ".claude/skills/ping/notes.md" {
		t.Fatalf("deletes: %+v", p.Deletes)
	}
	if _, still := p.Lock.Packages["@t/ping"].Files["claude-code"][".claude/skills/ping/notes.md"]; still {
		t.Fatal("the dropped file is still in the lock")
	}
}

func TestReinstallOfACleanPackageRestoresMissingFiles(t *testing.T) {
	first, _ := Build(tree.Mem{}, project(), manifest.NewLock(), []Install{ping("1.0.0", "v1")})
	p, err := Build(tree.Mem{}, project(), first.Lock, []Install{ping("1.0.0", "v1")})
	if err != nil || len(p.Writes) != 1 {
		t.Fatalf("a missing file is restored, not treated as an edit: %+v %v", p, err)
	}
}

func TestFragmentsAreRefusedUntilM4(t *testing.T) {
	in := ping("1.0.0", "v1", archive.File{Path: "claude/settings.merge.json", Mode: 0o644, Data: []byte("{}")})
	_, err := Build(tree.Mem{}, project(), manifest.NewLock(), []Install{in})
	c := conflicts(t, err)
	if len(c) != 1 || c[0].Kind != "unsupported" || !strings.Contains(c[0].Detail, "M4") {
		t.Fatalf("%+v", c)
	}
}

func TestMissingHarnessFolderWarnsOrFails(t *testing.T) {
	bare := Install{Name: "@t/kiro-only", Version: "1.0.0", Registry: "https://r.test",
		Integrity: "sha256-" + strings.Repeat("b", 64),
		Files: []archive.File{{Path: "hpm.json", Mode: 0o644}, {Path: "kiro/steering/x.md", Mode: 0o644}}}
	p, err := Build(tree.Mem{}, project(), manifest.NewLock(), []Install{bare})
	if err != nil || len(p.Warnings) != 1 || p.Lock.Packages["@t/kiro-only"].MissingHarness[0] != "claude-code" {
		t.Fatalf("%+v %v", p, err)
	}
	strict := project()
	strict.Unsupported = "fail"
	_, err = Build(tree.Mem{}, strict, manifest.NewLock(), []Install{bare})
	if c := conflicts(t, err); c[0].Kind != "harness" {
		t.Fatalf("%+v", c)
	}
}

func TestEveryConflictIsReportedNotJustTheFirst(t *testing.T) {
	in := ping("1.0.0", "v1", archive.File{Path: "claude/commands/ping.md", Mode: 0o644, Data: []byte("c")})
	snap := tree.Mem{".claude/skills/ping/SKILL.md": []byte("x"), ".claude/commands/ping.md": []byte("y")}
	_, err := Build(snap, project(), manifest.NewLock(), []Install{in})
	if c := conflicts(t, err); len(c) != 2 {
		t.Fatalf("%+v", c)
	}
}

func TestBuildNeverMutatesItsInputs(t *testing.T) {
	lock := manifest.NewLock()
	if _, err := Build(tree.Mem{}, project(), lock, []Install{ping("1.0.0", "v1")}); err != nil {
		t.Fatal(err)
	}
	if len(lock.Packages) != 0 {
		t.Fatal("Build wrote into the lock it was given")
	}
}

func TestUnknownHarnessIsAnError(t *testing.T) {
	p := project()
	p.Harnesses = []string{"kiro"}
	if _, err := Build(tree.Mem{}, p, manifest.NewLock(), []Install{ping("1.0.0", "v1")}); err == nil {
		t.Fatal("expected an error: there is no kiro adapter in M1")
	}
}
```

Run: `go test ./internal/plan/` — Expected: FAIL, undefined symbols.

- [ ] **Step 2: Implement the planner**

`internal/plan/plan.go`:

```go
// Package plan computes what a command would do to a project, without doing it.
package plan

import (
	"fmt"
	"strings"

	"github.com/StackCube/harness-package-manager-cli/internal/archive"
	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
)

type Write struct {
	Path  string
	Mode  int64
	Data  []byte
	Owner string
}

type Delete struct{ Path, Owner string }

type Plan struct {
	Writes   []Write
	Deletes  []Delete
	Project  *manifest.Project
	Lock     *manifest.Lock
	Warnings []string
}

type Install struct {
	Name, Version, Registry, Integrity string
	RequestedBy                        []string
	Files                              []archive.File
}

type Conflict struct{ Kind, Path, Package, Detail string }

type ConflictError struct{ Conflicts []Conflict }

func (e *ConflictError) Error() string {
	var b strings.Builder
	fmt.Fprintf(&b, "nothing was changed: %d conflict(s)", len(e.Conflicts))
	for _, c := range e.Conflicts {
		fmt.Fprintf(&b, "\n  %s", c.Detail)
	}
	return b.String()
}

// Summary is what --dry-run prints.
func (p *Plan) Summary() string {
	var b strings.Builder
	for _, w := range p.Writes {
		fmt.Fprintf(&b, "write   %s  (%s)\n", w.Path, w.Owner)
	}
	for _, d := range p.Deletes {
		fmt.Fprintf(&b, "delete  %s  (%s)\n", d.Path, d.Owner)
	}
	for _, w := range p.Warnings {
		fmt.Fprintf(&b, "warning %s\n", w)
	}
	if b.Len() == 0 {
		return "nothing to do\n"
	}
	return b.String()
}
```

`internal/plan/build.go`:

```go
package plan

import (
	"errors"
	"fmt"
	"io/fs"
	"sort"

	"github.com/StackCube/harness-package-manager-cli/internal/adapter"
	"github.com/StackCube/harness-package-manager-cli/internal/archive"
	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
	"github.com/StackCube/harness-package-manager-cli/internal/tree"
)

// Build plans the given installs against a snapshot. It reads, and never writes.
func Build(snap tree.Snapshot, proj *manifest.Project, lock *manifest.Lock, installs []Install) (*Plan, error) {
	adapters := make([]adapter.Adapter, 0, len(proj.Harnesses))
	for _, id := range proj.Harnesses {
		a, ok := adapter.ByID(id)
		if !ok {
			return nil, fmt.Errorf("harness %q is declared in %s but this version of hpm has no adapter for it", id, manifest.ProjectFile)
		}
		adapters = append(adapters, a)
	}

	out := &Plan{Lock: lock.Clone()}
	var conflicts []Conflict
	claimed := map[string]string{} // path → package, within this plan

	onDisk := func(path string) ([]byte, bool, error) {
		data, err := snap.ReadFile(path)
		if errors.Is(err, fs.ErrNotExist) {
			return nil, false, nil
		}
		return data, err == nil, err
	}

	for _, in := range installs {
		prev := lock.Packages[in.Name]

		// A package whose installed files were edited is never overwritten.
		if prev != nil {
			for _, files := range prev.Files {
				for path, want := range files {
					data, exists, err := onDisk(path)
					if err != nil {
						return nil, err
					}
					if exists && archive.Hash(data) != want {
						conflicts = append(conflicts, Conflict{"modified", path, in.Name,
							fmt.Sprintf("%s was edited since %s@%s was installed. Revert it, or publish it as a new version.", path, in.Name, prev.Version)})
					}
				}
			}
		}

		next := &manifest.Locked{Version: in.Version, Registry: in.Registry, Integrity: in.Integrity,
			RequestedBy: in.RequestedBy, State: "managed", Files: map[string]map[string]string{}}
		if prev != nil {
			next.Extra = prev.Extra
		}

		for _, a := range adapters {
			files := map[string]string{}
			shipped := false
			for _, f := range in.Files {
				target, ok := a.MapIn(f.Path)
				if !ok {
					continue
				}
				shipped = true
				if target.Kind == adapter.Fragment {
					conflicts = append(conflicts, Conflict{"unsupported", target.Path, in.Name,
						fmt.Sprintf("%s ships %s, which is merged into %s. Merge files are supported from M4.", in.Name, f.Path, target.Path)})
					continue
				}
				if owner, taken := claimed[target.Path]; taken && owner != in.Name {
					conflicts = append(conflicts, Conflict{"owned", target.Path, in.Name,
						fmt.Sprintf("%s and %s both install %s.", owner, in.Name, target.Path)})
					continue
				}
				if owner, ok := lock.Owner(target.Path); ok && owner != in.Name {
					conflicts = append(conflicts, Conflict{"owned", target.Path, in.Name,
						fmt.Sprintf("%s is owned by %s, so %s cannot install it.", target.Path, owner, in.Name)})
					continue
				}
				claimed[target.Path] = in.Name
				files[target.Path] = archive.Hash(f.Data)

				data, exists, err := onDisk(target.Path)
				if err != nil {
					return nil, err
				}
				switch {
				case exists && archive.Hash(data) == files[target.Path]:
					// Already what we would write.
				case exists && (prev == nil || !ownedBy(prev, target.Path)):
					conflicts = append(conflicts, Conflict{"unmanaged", target.Path, in.Name,
						fmt.Sprintf("%s already exists and is not managed by hpm. %s will not overwrite it.", target.Path, in.Name)})
				default:
					out.Writes = append(out.Writes, Write{Path: target.Path, Mode: f.Mode, Data: f.Data, Owner: in.Name})
				}
			}
			if !shipped {
				msg := fmt.Sprintf("%s@%s has no %s/ folder, so nothing was installed for %s.", in.Name, in.Version, a.PackageDir(), a.ID())
				if proj.Unsupported == "fail" {
					conflicts = append(conflicts, Conflict{"harness", "", in.Name, msg})
				} else {
					out.Warnings = append(out.Warnings, msg)
					next.MissingHarness = append(next.MissingHarness, a.ID())
				}
				continue
			}
			next.Files[a.ID()] = files
		}

		// Files the previous version owned and this one does not.
		if prev != nil {
			for harness, files := range prev.Files {
				for path := range files {
					if _, kept := next.Files[harness][path]; !kept {
						if _, exists, _ := onDisk(path); exists {
							out.Deletes = append(out.Deletes, Delete{Path: path, Owner: in.Name})
						}
					}
				}
			}
		}
		out.Lock.Packages[in.Name] = next
	}

	if len(conflicts) > 0 {
		sort.SliceStable(conflicts, func(i, j int) bool { return conflicts[i].Path < conflicts[j].Path })
		return nil, &ConflictError{Conflicts: conflicts}
	}
	sort.Slice(out.Writes, func(i, j int) bool { return out.Writes[i].Path < out.Writes[j].Path })
	sort.Slice(out.Deletes, func(i, j int) bool { return out.Deletes[i].Path < out.Deletes[j].Path })
	return out, nil
}

func ownedBy(l *manifest.Locked, path string) bool {
	for _, files := range l.Files {
		if _, ok := files[path]; ok {
			return true
		}
	}
	return false
}
```

Run: `go test ./internal/plan/` — Expected: PASS. Commit: "Planner: ownership, unmanaged files, edits and fragments all refuse before any write".

- [ ] **Step 3: Failing tests for apply and status**

`internal/apply/apply_test.go`:

```go
package apply

import (
	"os"
	"path/filepath"
	"testing"

	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
	"github.com/StackCube/harness-package-manager-cli/internal/plan"
)

func read(t *testing.T, root, p string) string {
	t.Helper()
	b, err := os.ReadFile(filepath.Join(root, p))
	if err != nil {
		t.Fatal(err)
	}
	return string(b)
}

func TestApplyWritesFilesThenProjectThenLock(t *testing.T) {
	root := t.TempDir()
	os.MkdirAll(filepath.Join(root, ".claude/skills/old"), 0o755)
	os.WriteFile(filepath.Join(root, ".claude/skills/old/SKILL.md"), []byte("old"), 0o644)
	lock := manifest.NewLock()
	lock.Packages["@t/ping"] = &manifest.Locked{Version: "1.0.0", Registry: "https://r.test",
		Integrity: "sha256-aa", State: "managed"}
	p := &plan.Plan{
		Writes: []plan.Write{
			{Path: ".claude/skills/ping/SKILL.md", Mode: 0o644, Data: []byte("v1"), Owner: "@t/ping"},
			{Path: ".claude/scripts/run.sh", Mode: 0o755, Data: []byte("#!/bin/sh\n"), Owner: "@t/ping"},
		},
		Deletes: []plan.Delete{{Path: ".claude/skills/old/SKILL.md", Owner: "@t/ping"}},
		Project: &manifest.Project{Harnesses: []string{"claude-code"}, Registries: map[string]manifest.RegistryRef{},
			Dependencies: map[string]string{"@t/ping": "^1.0.0"}},
		Lock: lock,
	}
	if err := Apply(root, p); err != nil {
		t.Fatal(err)
	}
	if read(t, root, ".claude/skills/ping/SKILL.md") != "v1" {
		t.Fatal("file content")
	}
	info, _ := os.Stat(filepath.Join(root, ".claude/scripts/run.sh"))
	if info.Mode().Perm()&0o111 == 0 {
		t.Fatal("the executable bit was lost")
	}
	if _, err := os.Stat(filepath.Join(root, ".claude/skills/old/SKILL.md")); !os.IsNotExist(err) {
		t.Fatal("the delete did not happen")
	}
	if _, err := os.Stat(filepath.Join(root, ".claude/skills/old")); !os.IsNotExist(err) {
		t.Fatal("a directory hpm emptied should be removed")
	}
	if err := manifest.Validate(manifest.Lockfile, []byte(read(t, root, manifest.LockFile))); err != nil {
		t.Fatal(err)
	}
	if _, err := manifest.LoadProject(root); err != nil {
		t.Fatal(err)
	}
	matches, _ := filepath.Glob(filepath.Join(root, ".claude/skills/ping/.hpm-*"))
	if len(matches) != 0 {
		t.Fatalf("temp files left behind: %v", matches)
	}
}

func TestAFailedApplyLeavesTheLockfileAloneAndARerunConverges(t *testing.T) {
	root := t.TempDir()
	os.WriteFile(filepath.Join(root, manifest.LockFile), []byte("ORIGINAL"), 0o644)
	os.WriteFile(filepath.Join(root, "blocker"), []byte("a file where a directory is needed"), 0o644)
	p := &plan.Plan{
		Writes: []plan.Write{
			{Path: ".claude/a.md", Mode: 0o644, Data: []byte("a"), Owner: "@t/x"},
			{Path: "blocker/b.md", Mode: 0o644, Data: []byte("b"), Owner: "@t/x"},
		},
		Lock: manifest.NewLock(),
	}
	if err := Apply(root, p); err == nil {
		t.Fatal("expected the second write to fail")
	}
	if read(t, root, manifest.LockFile) != "ORIGINAL" {
		t.Fatal("the lockfile was touched by a failed apply")
	}
	os.Remove(filepath.Join(root, "blocker"))
	if err := Apply(root, p); err != nil {
		t.Fatalf("the rerun should converge: %v", err)
	}
	if read(t, root, "blocker/b.md") != "b" || read(t, root, manifest.LockFile) == "ORIGINAL" {
		t.Fatal("the rerun did not finish the job")
	}
}

func TestApplyRefusesPathsOutsideTheRoot(t *testing.T) {
	root := t.TempDir()
	p := &plan.Plan{Writes: []plan.Write{{Path: "../escape.md", Mode: 0o644, Data: []byte("x")}}, Lock: manifest.NewLock()}
	if err := Apply(root, p); err == nil {
		t.Fatal("expected a refusal")
	}
}
```

`internal/status/status_test.go`:

```go
package status

import (
	"testing"

	"github.com/StackCube/harness-package-manager-cli/internal/archive"
	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
	"github.com/StackCube/harness-package-manager-cli/internal/tree"
)

func lock() *manifest.Lock {
	l := manifest.NewLock()
	l.Packages["@t/ping"] = &manifest.Locked{Version: "1.0.0", State: "managed",
		Files: map[string]map[string]string{"claude-code": {
			".claude/skills/ping/SKILL.md": archive.Hash([]byte("v1")),
			".claude/skills/ping/notes.md": archive.Hash([]byte("n")),
		}}}
	l.Packages["@t/bare"] = &manifest.Locked{Version: "2.0.0", State: "managed", MissingHarness: []string{"claude-code"}}
	return l
}

func TestCheck(t *testing.T) {
	clean := tree.Mem{".claude/skills/ping/SKILL.md": []byte("v1"), ".claude/skills/ping/notes.md": []byte("n")}
	got := Check(clean, lock())
	if len(got) != 2 || got[0].Name != "@t/bare" || got[1].State != "managed" {
		t.Fatalf("%+v", got)
	}
	if got[0].MissingHarness[0] != "claude-code" {
		t.Fatalf("%+v", got[0])
	}
	dirty := tree.Mem{".claude/skills/ping/SKILL.md": []byte("edited")}
	p := Check(dirty, lock())[1]
	if p.State != "modified" || len(p.Modified) != 1 || len(p.Missing) != 1 {
		t.Fatalf("%+v", p)
	}
}

func TestStale(t *testing.T) {
	proj := &manifest.Project{Dependencies: map[string]string{"@t/ping": "^1.0.0", "@t/new": "^1.0.0"}}
	reasons := Stale(proj, lock())
	if len(reasons) != 1 {
		t.Fatalf("%v", reasons)
	}
	proj.Dependencies = map[string]string{"@t/ping": "^2.0.0"}
	if len(Stale(proj, lock())) != 1 {
		t.Fatal("a locked version outside the range is stale")
	}
	proj.Dependencies = map[string]string{"@t/ping": "^1.0.0"}
	if len(Stale(proj, lock())) != 0 {
		t.Fatal("a satisfied manifest is not stale")
	}
}
```

Run: `go test ./internal/apply/ ./internal/status/` — Expected: FAIL.

- [ ] **Step 4: Implement apply and status**

```bash
go get github.com/Masterminds/semver/v3@latest
```

`internal/apply/apply.go`:

```go
// Package apply carries out a checked plan. It is the only package that writes to a project.
package apply

import (
	"fmt"
	"os"
	"path/filepath"

	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
	"github.com/StackCube/harness-package-manager-cli/internal/plan"
)

func inside(root, p string) (string, error) {
	if p == "" || !filepath.IsLocal(filepath.FromSlash(p)) {
		return "", fmt.Errorf("%q is not a path inside the project", p)
	}
	return filepath.Join(root, filepath.FromSlash(p)), nil
}

// writeFile writes through a temp file in the same directory, so a reader never sees half a file.
func writeFile(full string, data []byte, mode os.FileMode) error {
	if err := os.MkdirAll(filepath.Dir(full), 0o755); err != nil {
		return err
	}
	tmp, err := os.CreateTemp(filepath.Dir(full), ".hpm-*")
	if err != nil {
		return err
	}
	defer os.Remove(tmp.Name())
	if _, err := tmp.Write(data); err != nil {
		tmp.Close()
		return err
	}
	if err := tmp.Close(); err != nil {
		return err
	}
	if err := os.Chmod(tmp.Name(), mode); err != nil {
		return err
	}
	return os.Rename(tmp.Name(), full)
}

// Apply performs the plan's writes and deletes, then hpm.json if it changed, then hpm.lock, last.
// Every step is idempotent, so running the same command again after a failure converges.
func Apply(root string, p *plan.Plan) error {
	for _, w := range p.Writes {
		full, err := inside(root, w.Path)
		if err != nil {
			return err
		}
		if err := writeFile(full, w.Data, os.FileMode(w.Mode)); err != nil {
			return fmt.Errorf("write %s: %w", w.Path, err)
		}
	}
	for _, d := range p.Deletes {
		full, err := inside(root, d.Path)
		if err != nil {
			return err
		}
		if err := os.Remove(full); err != nil && !os.IsNotExist(err) {
			return fmt.Errorf("delete %s: %w", d.Path, err)
		}
		// Remove directories hpm has just emptied. os.Remove refuses a non-empty directory.
		for dir := filepath.Dir(full); dir != root && os.Remove(dir) == nil; dir = filepath.Dir(dir) {
		}
	}
	if p.Project != nil {
		data, err := manifest.Encode(p.Project)
		if err != nil {
			return err
		}
		if err := writeFile(filepath.Join(root, manifest.ProjectFile), data, 0o644); err != nil {
			return err
		}
	}
	data, err := manifest.Encode(p.Lock)
	if err != nil {
		return err
	}
	return writeFile(filepath.Join(root, manifest.LockFile), data, 0o644)
}
```

`internal/status/status.go`:

```go
// Package status compares a project's tree with its lockfile. It needs no network.
package status

import (
	"errors"
	"fmt"
	"io/fs"
	"sort"

	"github.com/Masterminds/semver/v3"

	"github.com/StackCube/harness-package-manager-cli/internal/archive"
	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
	"github.com/StackCube/harness-package-manager-cli/internal/tree"
)

type Package struct {
	Name, Version, State              string
	Modified, Missing, MissingHarness []string
}

func Check(snap tree.Snapshot, lock *manifest.Lock) []Package {
	var out []Package
	for name, l := range lock.Packages {
		p := Package{Name: name, Version: l.Version, State: "managed", MissingHarness: l.MissingHarness}
		for _, files := range l.Files {
			for path, want := range files {
				data, err := snap.ReadFile(path)
				switch {
				case errors.Is(err, fs.ErrNotExist):
					p.Missing = append(p.Missing, path)
				case err != nil || archive.Hash(data) != want:
					p.Modified = append(p.Modified, path)
				}
			}
		}
		sort.Strings(p.Modified)
		sort.Strings(p.Missing)
		if len(p.Modified)+len(p.Missing) > 0 {
			p.State = "modified"
		}
		out = append(out, p)
	}
	sort.Slice(out, func(i, j int) bool { return out[i].Name < out[j].Name })
	return out
}

// Stale lists why the lockfile does not satisfy the manifest.
func Stale(proj *manifest.Project, lock *manifest.Lock) []string {
	var reasons []string
	names := make([]string, 0, len(proj.Dependencies))
	for n := range proj.Dependencies {
		names = append(names, n)
	}
	sort.Strings(names)
	for _, name := range names {
		rng := proj.Dependencies[name]
		l, ok := lock.Packages[name]
		if !ok {
			reasons = append(reasons, fmt.Sprintf("%s is in %s but not in %s", name, manifest.ProjectFile, manifest.LockFile))
			continue
		}
		c, err := semver.NewConstraint(rng)
		v, verr := semver.NewVersion(l.Version)
		if err != nil || verr != nil || !c.Check(v) {
			reasons = append(reasons, fmt.Sprintf("%s is locked at %s, which does not satisfy %s", name, l.Version, rng))
		}
	}
	return reasons
}
```

Run: `go test ./internal/...` — Expected: PASS. Commit: "Apply with the lockfile last, and offline status".

---

### Task 5: Version pick and the commands

**Files:**
- Create: `internal/resolve/pick.go`, `internal/resolve/pick_test.go`, `internal/cli/context.go`, `init.go`, `publish.go`, `add.go`, `install.go`, `status.go`, `flow_test.go`
- Modify: `internal/cli/root.go`

**Interfaces:**
- Consumes: everything from Tasks 1–4, `archive.ReadDir`, `archive.Pack`, `archive.Unpack`, `archive.Hash`, `archive.MaxUncompressed`.
- Produces:

```go
// internal/resolve
func Pick(versions []string, ranges []string) (string, error) // highest version satisfying every range

// internal/cli
type Deps struct {
	Getenv func(string) string
	HTTP   *http.Client
	Stdout, Stderr io.Writer
}
func NewRootCmd(version string) *cobra.Command             // unchanged signature; uses real Deps
func NewRootCmdWith(version string, d Deps) *cobra.Command // tests
type ExitError struct{ Code int; Msg string }              // main maps this to the process exit code
```

Commands: `hpm init`, `hpm publish [dir]`, `hpm add <pkg>[@range]`, `hpm install`, `hpm status`. Persistent flag `-C, --dir` (default `.`) on the root. `--dry-run` on `add` and `install`. `--ci` on `status`. `--registry` on `init` (repeatable, `@scope=url`) and on `publish` (a URL).

- [ ] **Step 1: Pick, test first**

`internal/resolve/pick_test.go` runs the contract's semver vectors, skipping by name the two prerelease cases that the carry-over list assigns to M3:

```go
package resolve

import (
	"encoding/json"
	"os"
	"testing"
)

var deferredToM3 = map[string]bool{
	"prerelease range admits only prereleases of the same version tuple": true,
}

func TestSemverVectors(t *testing.T) {
	raw, err := os.ReadFile("../../contract/vectors/semver/cases.json")
	if err != nil {
		t.Fatal(err)
	}
	var cases []struct {
		Name     string
		Ranges   []string
		Versions []string
		Pick     *string
	}
	if err := json.Unmarshal(raw, &cases); err != nil {
		t.Fatal(err)
	}
	for _, c := range cases {
		t.Run(c.Name, func(t *testing.T) {
			if deferredToM3[c.Name] {
				t.Skip("Masterminds/semver admits prereleases of later versions; the M3 resolver adds the filter")
			}
			got, err := Pick(c.Versions, c.Ranges)
			switch {
			case c.Pick == nil && err == nil:
				t.Fatalf("got %q, want no pick", got)
			case c.Pick != nil && (err != nil || got != *c.Pick):
				t.Fatalf("got %q %v, want %q", got, err, *c.Pick)
			}
		})
	}
}
```

Add one more test in the same file:

```go
func TestNoRangeMeansTheHighestStableVersion(t *testing.T) {
	got, err := Pick([]string{"1.0.0", "1.1.0-next.1", "0.9.0"}, nil)
	if err != nil || got != "1.0.0" {
		t.Fatalf("got %q %v", got, err)
	}
	if _, err := Pick([]string{"1.0.0-next.1"}, nil); err == nil {
		t.Fatal("only prereleases exist, so there is no latest; the caller must ask for a range")
	}
}
```

If a vector other than the named one fails, do not add it to the skip list. Report it: either the library disagrees with the contract somewhere new, or `Pick` is wrong.

`internal/resolve/pick.go`:

```go
// Package resolve chooses versions. M1 picks one package; the dependency resolver is M3.
package resolve

import (
	"fmt"
	"sort"
	"strings"

	"github.com/Masterminds/semver/v3"
)

// Pick returns the highest of versions that satisfies every range.
func Pick(versions, ranges []string) (string, error) {
	var cs []*semver.Constraints
	for _, r := range ranges {
		c, err := semver.NewConstraint(r)
		if err != nil {
			return "", fmt.Errorf("range %q: %w", r, err)
		}
		cs = append(cs, c)
	}
	var ok []*semver.Version
	for _, raw := range versions {
		v, err := semver.StrictNewVersion(raw)
		if err != nil {
			continue
		}
		if len(cs) == 0 && v.Prerelease() != "" {
			continue // "latest" never means a prerelease
		}
		all := true
		for _, c := range cs {
			all = all && c.Check(v)
		}
		if all {
			ok = append(ok, v)
		}
	}
	if len(ok) == 0 {
		return "", fmt.Errorf("no published version satisfies %s", strings.Join(ranges, " and "))
	}
	sort.Sort(sort.Reverse(semver.Collection(ok)))
	return ok[0].Original(), nil
}
```

- [ ] **Step 2: A failing flow test with a fake registry**

`internal/cli/flow_test.go` drives the real commands against an in-memory registry. It is the unit-level twin of the end-to-end test.

```go
package cli

import (
	"bytes"
	"encoding/json"
	"errors"
	"io"
	"net/http"
	"net/http/httptest"
	"os"
	"path/filepath"
	"strings"
	"sync"
	"testing"

	"github.com/StackCube/harness-package-manager-cli/internal/archive"
	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
)

type fakeRegistry struct {
	mu       sync.Mutex
	archives map[string][]byte // "@scope/name@version" → gz
}

func (f *fakeRegistry) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	f.mu.Lock()
	defer f.mu.Unlock()
	fail := func(status int, code, msg string) {
		w.Header().Set("content-type", "application/json")
		w.WriteHeader(status)
		json.NewEncoder(w).Encode(map[string]any{"code": code, "message": msg, "details": map[string]any{}})
	}
	if r.Header.Get("CF-Access-Client-Id") == "" {
		w.WriteHeader(403)
		io.WriteString(w, "<html>Forbidden</html>")
		return
	}
	parts := strings.Split(strings.TrimPrefix(r.URL.Path, "/v1/"), "/")
	switch {
	case r.Method == "PUT" && len(parts) == 4 && parts[0] == "pkg":
		key := parts[1] + "/" + parts[2] + "@" + parts[3]
		if _, dup := f.archives[key]; dup {
			fail(409, "version_exists", "already published")
			return
		}
		body, _ := io.ReadAll(r.Body)
		f.archives[key] = body
		w.Header().Set("content-type", "application/json")
		w.WriteHeader(201)
		json.NewEncoder(w).Encode(map[string]any{"name": parts[1] + "/" + parts[2], "version": parts[3],
			"integrity": archive.Hash(body), "size": len(body), "publishedAt": "2026-09-19T10:00:00Z", "publishedBy": "ci"})
	case r.Method == "GET" && parts[0] == "index":
		byPkg := map[string][]map[string]any{}
		for key, gz := range f.archives {
			i := strings.LastIndex(key, "@")
			name, version := key[:i], key[i+1:]
			byPkg[name] = append(byPkg[name], map[string]any{"version": version, "integrity": archive.Hash(gz), "size": len(gz),
				"dependencies": map[string]string{}, "harnesses": []string{"claude-code"},
				"publishedAt": "2026-09-19T10:00:00Z", "publishedBy": "ci"})
		}
		var pkgs []map[string]any
		for name, versions := range byPkg {
			pkgs = append(pkgs, map[string]any{"name": name, "description": "d", "keywords": []string{}, "versions": versions})
		}
		w.Header().Set("content-type", "application/json")
		json.NewEncoder(w).Encode(map[string]any{"packages": pkgs})
	case r.Method == "GET" && len(parts) == 4 && parts[0] == "archive":
		gz, ok := f.archives[parts[1]+"/"+parts[2]+"@"+parts[3]]
		if !ok {
			fail(404, "not_found", "no such version")
			return
		}
		w.Write(gz)
	default:
		fail(404, "not_found", "no such route")
	}
}

type world struct {
	t   *testing.T
	url string
	env map[string]string
}

func newWorld(t *testing.T) *world {
	srv := httptest.NewServer(&fakeRegistry{archives: map[string][]byte{}})
	t.Cleanup(srv.Close)
	return &world{t: t, url: srv.URL, env: map[string]string{"HPM_ACCESS_CLIENT_ID": "id", "HPM_ACCESS_CLIENT_SECRET": "secret"}}
}

func (w *world) run(args ...string) (string, error) {
	var out bytes.Buffer
	cmd := NewRootCmdWith("test", Deps{Getenv: func(k string) string { return w.env[k] }, HTTP: http.DefaultClient, Stdout: &out, Stderr: &out})
	cmd.SetArgs(args)
	err := cmd.Execute()
	return out.String(), err
}

func writeTree(t *testing.T, root string, files map[string]string) {
	t.Helper()
	for p, body := range files {
		full := filepath.Join(root, p)
		os.MkdirAll(filepath.Dir(full), 0o755)
		if err := os.WriteFile(full, []byte(body), 0o644); err != nil {
			t.Fatal(err)
		}
	}
}

func pingPackage(t *testing.T, version, skill string) string {
	dir := t.TempDir()
	writeTree(t, dir, map[string]string{
		"hpm.json":     `{"name":"@t/ping","version":"` + version + `","description":"d","keywords":[],"owners":["o@x.test"]}`,
		"README.md":    "# ping\n",
		"CHANGELOG.md": "# Changelog\n\n## [" + version + "] - 2026-09-19\n- Added: x\n",
		"claude/skills/ping/SKILL.md": skill,
	})
	return dir
}

func TestPublishAddStatusEditInstall(t *testing.T) {
	w := newWorld(t)
	pkg := pingPackage(t, "1.0.0", "v1")
	if out, err := w.run("publish", pkg, "--registry", w.url); err != nil {
		t.Fatalf("publish: %v\n%s", err, out)
	}
	out, err := w.run("publish", pkg, "--registry", w.url)
	if err == nil || !strings.Contains(out+err.Error(), "version_exists") {
		t.Fatalf("republish should be refused with the contract code: %v\n%s", err, out)
	}

	proj := t.TempDir()
	if out, err := w.run("-C", proj, "init", "--registry", "@t="+w.url); err != nil {
		t.Fatalf("init: %v\n%s", err, out)
	}
	if out, err := w.run("-C", proj, "add", "@t/ping", "--dry-run"); err != nil || !strings.Contains(out, ".claude/skills/ping/SKILL.md") {
		t.Fatalf("dry run: %v\n%s", err, out)
	}
	if _, err := os.Stat(filepath.Join(proj, "hpm.lock")); !os.IsNotExist(err) {
		t.Fatal("--dry-run wrote a lockfile")
	}
	if out, err := w.run("-C", proj, "add", "@t/ping"); err != nil {
		t.Fatalf("add: %v\n%s", err, out)
	}
	got, _ := os.ReadFile(filepath.Join(proj, ".claude/skills/ping/SKILL.md"))
	if string(got) != "v1" {
		t.Fatalf("installed %q", got)
	}
	manifestJSON, _ := os.ReadFile(filepath.Join(proj, "hpm.json"))
	if !strings.Contains(string(manifestJSON), `"@t/ping": "^1.0.0"`) {
		t.Fatalf("hpm.json: %s", manifestJSON)
	}
	if out, err := w.run("-C", proj, "status", "--ci"); err != nil {
		t.Fatalf("a clean project must pass --ci: %v\n%s", err, out)
	}

	// A second clone: hpm.json and hpm.lock only.
	clone := t.TempDir()
	for _, f := range []string{"hpm.json", "hpm.lock"} {
		b, _ := os.ReadFile(filepath.Join(proj, f))
		os.WriteFile(filepath.Join(clone, f), b, 0o644)
	}
	if out, err := w.run("-C", clone, "install"); err != nil {
		t.Fatalf("install: %v\n%s", err, out)
	}
	a, _ := os.ReadFile(filepath.Join(clone, ".claude/skills/ping/SKILL.md"))
	if string(a) != "v1" {
		t.Fatal("the clone differs")
	}

	// A hand edit.
	os.WriteFile(filepath.Join(proj, ".claude/skills/ping/SKILL.md"), []byte("edited"), 0o644)
	out, err = w.run("-C", proj, "status", "--ci")
	var exit *ExitError
	if !errors.As(err, &exit) || exit.Code != 1 || !strings.Contains(out, "modified") {
		t.Fatalf("status --ci on an edited project: %v\n%s", err, out)
	}
	if out, err := w.run("-C", proj, "status"); err != nil || !strings.Contains(out, "modified") {
		t.Fatalf("plain status reports and exits 0: %v\n%s", err, out)
	}
	if _, err := w.run("-C", proj, "install"); err == nil {
		t.Fatal("install must refuse to overwrite an edit")
	}
	kept, _ := os.ReadFile(filepath.Join(proj, ".claude/skills/ping/SKILL.md"))
	if string(kept) != "edited" {
		t.Fatal("the edit was overwritten")
	}
}

func TestRefusalsThatNameTheirMilestone(t *testing.T) {
	w := newWorld(t)
	withDeps := t.TempDir()
	writeTree(t, withDeps, map[string]string{
		"hpm.json":  `{"name":"@t/kit","version":"1.0.0","description":"d","keywords":[],"owners":["o@x.test"],"dependencies":{"@t/ping":"^1.0"}}`,
		"README.md": "# kit\n",
	})
	if out, err := w.run("publish", withDeps, "--registry", w.url); err != nil {
		t.Fatalf("publishing a meta package is fine: %v\n%s", err, out)
	}
	proj := t.TempDir()
	w.run("-C", proj, "init", "--registry", "@t="+w.url)
	out, err := w.run("-C", proj, "add", "@t/kit")
	if err == nil || !strings.Contains(out+err.Error(), "M3") {
		t.Fatalf("dependencies are M3: %v\n%s", err, out)
	}
}

func TestIdentityProblemsAreLegible(t *testing.T) {
	w := newWorld(t)
	w.env = map[string]string{}
	out, err := w.run("publish", pingPackage(t, "1.0.0", "v1"), "--registry", w.url)
	if err == nil || !strings.Contains(out+err.Error(), "HPM_ACCESS_CLIENT_ID") {
		t.Fatalf("%v\n%s", err, out)
	}
}

func TestIntegrityMismatchStopsTheInstall(t *testing.T) {
	w := newWorld(t)
	w.run("publish", pingPackage(t, "1.0.0", "v1"), "--registry", w.url)
	proj := t.TempDir()
	w.run("-C", proj, "init", "--registry", "@t="+w.url)
	w.run("-C", proj, "add", "@t/ping")
	lock, err := manifest.LoadLock(proj)
	if err != nil {
		t.Fatal(err)
	}
	lock.Packages["@t/ping"].Integrity = "sha256-" + strings.Repeat("0", 64)
	tampered, _ := manifest.Encode(lock)
	os.WriteFile(filepath.Join(proj, "hpm.lock"), tampered, 0o644)
	os.RemoveAll(filepath.Join(proj, ".claude"))
	out, err := w.run("-C", proj, "install")
	if err == nil || !strings.Contains(out+err.Error(), "integrity") {
		t.Fatalf("%v\n%s", err, out)
	}
	if _, statErr := os.Stat(filepath.Join(proj, ".claude")); !os.IsNotExist(statErr) {
		t.Fatal("files were written despite the integrity failure")
	}
}
```

Run: `go test ./internal/cli/` — Expected: FAIL, undefined `NewRootCmdWith`, `Deps`, `ExitError`.

- [ ] **Step 3: Shared command plumbing**

`internal/cli/context.go`:

```go
package cli

import (
	"context"
	"errors"
	"fmt"
	"io"
	"io/fs"
	"net/http"

	"github.com/StackCube/harness-package-manager-cli/internal/apply"
	"github.com/StackCube/harness-package-manager-cli/internal/archive"
	"github.com/StackCube/harness-package-manager-cli/internal/auth"
	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
	"github.com/StackCube/harness-package-manager-cli/internal/plan"
	"github.com/StackCube/harness-package-manager-cli/internal/registry"
)

// Deps is everything a command takes from the outside world.
type Deps struct {
	Getenv         func(string) string
	HTTP           *http.Client
	Stdout, Stderr io.Writer
}

// ExitError asks main for a specific exit code without printing a stack of wrapped errors.
type ExitError struct {
	Code int
	Msg  string
}

func (e *ExitError) Error() string { return e.Msg }

func (d Deps) client(ref manifest.RegistryRef) (*registry.Client, error) {
	p, err := auth.ForRegistry(ref, d.Getenv)
	if err != nil {
		return nil, err
	}
	return registry.New(ref.URL, p, d.HTTP)
}

// fetch downloads one version, verifies it against the expected integrity, and unpacks it.
func (d Deps) fetch(ctx context.Context, c *registry.Client, name, version, integrity string) ([]archive.File, error) {
	gz, err := c.Archive(ctx, name, version)
	if err != nil {
		return nil, fmt.Errorf("download %s@%s: %w", name, version, err)
	}
	if got := archive.Hash(gz); got != integrity {
		return nil, fmt.Errorf("integrity check failed for %s@%s: expected %s, the registry sent %s", name, version, integrity, got)
	}
	return archive.Unpack(gz, archive.MaxUncompressed)
}

// loadLock returns an empty lock when the project has none yet.
func loadLock(dir string) (*manifest.Lock, error) {
	l, err := manifest.LoadLock(dir)
	if errors.Is(err, fs.ErrNotExist) {
		return manifest.NewLock(), nil
	}
	return l, err
}

// refuseM1 rejects what M1 does not do yet, naming where it arrives.
func refuseM1(proj *manifest.Project, pkg *manifest.Package) error {
	if len(proj.Aliases) > 0 {
		return fmt.Errorf("aliases are not supported until iteration 2; remove them from %s", manifest.ProjectFile)
	}
	if pkg != nil && len(pkg.Dependencies) > 0 {
		return fmt.Errorf("%s has dependencies, and dependency resolution arrives in M3", pkg.Name)
	}
	return nil
}

// finish prints the plan and, unless this is a dry run, applies it.
func finish(d Deps, root string, p *plan.Plan, dryRun bool) error {
	fmt.Fprint(d.Stdout, p.Summary())
	if dryRun {
		return nil
	}
	return apply.Apply(root, p)
}
```

- [ ] **Step 4: The commands**

Each is one file with one constructor. They hold no logic beyond sequencing; everything decidable lives in the packages from Tasks 1–4.

`internal/cli/root.go`:

```go
// Package cli wires cobra commands. It holds no logic of its own.
package cli

import (
	"context"
	"net/http"
	"os"

	"github.com/spf13/cobra"
)

// NewRootCmd returns the hpm root command wired to the real world.
func NewRootCmd(version string) *cobra.Command {
	return NewRootCmdWith(version, Deps{Getenv: os.Getenv, HTTP: http.DefaultClient, Stdout: os.Stdout, Stderr: os.Stderr})
}

// NewRootCmdWith lets tests supply the environment, the HTTP client and the output streams.
func NewRootCmdWith(version string, d Deps) *cobra.Command {
	var dir string
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
	cmd.SetOut(d.Stdout)
	cmd.SetErr(d.Stderr)
	cmd.SetContext(context.Background()) // subcommands read cmd.Context(), which is nil without this
	cmd.PersistentFlags().StringVarP(&dir, "dir", "C", ".", "project directory")
	cmd.AddCommand(newInitCmd(d, &dir), newPublishCmd(d, &dir), newAddCmd(d, &dir), newInstallCmd(d, &dir), newStatusCmd(d, &dir))
	return cmd
}
```

The two existing root tests call `cmd.SetOut` themselves after construction, so they still pass unchanged.

`cmd/hpm/main.go`:

```go
func main() {
	err := cli.NewRootCmd(version).Execute()
	if err == nil {
		return
	}
	var exit *cli.ExitError
	if errors.As(err, &exit) {
		if exit.Msg != "" {
			fmt.Fprintln(os.Stderr, "hpm:", exit.Msg)
		}
		os.Exit(exit.Code)
	}
	fmt.Fprintln(os.Stderr, "hpm:", err)
	os.Exit(1)
}
```

`internal/cli/init.go`:

```go
package cli

import (
	"fmt"
	"net/url"
	"os"
	"path/filepath"
	"strings"

	"github.com/spf13/cobra"

	"github.com/StackCube/harness-package-manager-cli/internal/adapter"
	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
	"github.com/StackCube/harness-package-manager-cli/internal/tree"
)

func newInitCmd(d Deps, dir *string) *cobra.Command {
	var registries []string
	cmd := &cobra.Command{
		Use:   "init",
		Short: "Create hpm.json, detect harnesses, map scopes to registries",
		Args:  cobra.NoArgs,
		RunE: func(*cobra.Command, []string) error {
			path := filepath.Join(*dir, manifest.ProjectFile)
			if _, err := os.Stat(path); err == nil {
				return fmt.Errorf("%s already exists", path)
			}
			proj := &manifest.Project{Registries: map[string]manifest.RegistryRef{}, Dependencies: map[string]string{}}
			for _, a := range adapter.All() {
				if a.Detect(tree.Dir(*dir)) {
					proj.Harnesses = append(proj.Harnesses, a.ID())
				}
			}
			if len(proj.Harnesses) == 0 {
				proj.Harnesses = []string{"claude-code"}
			}
			for _, r := range registries {
				scope, raw, ok := strings.Cut(r, "=")
				u, err := url.Parse(raw)
				if !ok || !strings.HasPrefix(scope, "@") || err != nil || (u.Scheme != "http" && u.Scheme != "https") || u.Host == "" {
					return fmt.Errorf("--registry wants @scope=https://host, got %q", r)
				}
				proj.Registries[scope] = manifest.RegistryRef{URL: strings.TrimRight(raw, "/")}
			}
			data, err := manifest.Encode(proj)
			if err != nil {
				return err
			}
			if err := manifest.Validate(manifest.ProjectManifest, data); err != nil {
				return err
			}
			// The one write outside internal/apply: there is no plan before there is a project.
			if err := os.WriteFile(path, data, 0o644); err != nil {
				return err
			}
			fmt.Fprintf(d.Stdout, "created %s (harnesses: %s)\n", manifest.ProjectFile, strings.Join(proj.Harnesses, ", "))
			return nil
		},
	}
	cmd.Flags().StringArrayVar(&registries, "registry", nil, "map a scope to a registry: @scope=https://host (repeatable)")
	return cmd
}
```

`internal/cli/publish.go`:

```go
package cli

import (
	"errors"
	"fmt"
	"os"
	"path/filepath"

	"github.com/spf13/cobra"

	"github.com/StackCube/harness-package-manager-cli/internal/archive"
	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
)

func newPublishCmd(d Deps, dir *string) *cobra.Command {
	var registryURL string
	cmd := &cobra.Command{
		Use:   "publish [dir]",
		Short: "Pack a package directory and send it to its scope's registry",
		Args:  cobra.MaximumNArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			pkgDir := *dir
			if len(args) == 1 {
				pkgDir = args[0]
				if !filepath.IsAbs(pkgDir) {
					pkgDir = filepath.Join(*dir, pkgDir)
				}
			}
			data, err := os.ReadFile(filepath.Join(pkgDir, manifest.ProjectFile))
			if err != nil {
				return err
			}
			if manifest.IsProjectManifest(data) {
				return fmt.Errorf("%s is a project, not a package; publishing a package from a project tree arrives in M5", pkgDir)
			}
			pkg, err := manifest.ParsePackage(data)
			if err != nil {
				return err
			}
			if _, err := os.Stat(filepath.Join(pkgDir, "README.md")); err != nil {
				return errors.New("readme_missing: a package needs a README.md")
			}
			files, err := archive.ReadDir(pkgDir)
			if err != nil {
				return err
			}
			packed, err := archive.Pack(files)
			if err != nil {
				return err
			}
			ref := manifest.RegistryRef{URL: registryURL}
			if registryURL == "" {
				proj, perr := manifest.LoadProject(*dir)
				if perr != nil {
					return fmt.Errorf("no registry: pass --registry, or run from a project whose %s maps the scope", manifest.ProjectFile)
				}
				if ref, err = proj.RegistryFor(pkg.Name); err != nil {
					return err
				}
			}
			client, err := d.client(ref)
			if err != nil {
				return err
			}
			res, err := client.Publish(cmd.Context(), pkg.Name, pkg.Version, packed.Gzip)
			if err != nil {
				return err
			}
			fmt.Fprintf(d.Stdout, "published %s@%s  %s  (%d bytes) as %s\n", res.Name, res.Version, res.Integrity, res.Size, res.PublishedBy)
			return nil
		},
	}
	cmd.Flags().StringVar(&registryURL, "registry", "", "registry URL; defaults to the project's mapping for the package's scope")
	return cmd
}
```

`internal/cli/add.go`:

```go
package cli

import (
	"fmt"
	"strings"

	"github.com/spf13/cobra"

	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
	"github.com/StackCube/harness-package-manager-cli/internal/plan"
	"github.com/StackCube/harness-package-manager-cli/internal/resolve"
	"github.com/StackCube/harness-package-manager-cli/internal/tree"
)

func newAddCmd(d Deps, dir *string) *cobra.Command {
	var dryRun bool
	cmd := &cobra.Command{
		Use:   "add <@scope/name>[@range]",
		Short: "Add a dependency, install it, write the lockfile",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			name, rng := args[0], ""
			if i := strings.LastIndex(args[0], "@"); i > 0 {
				name, rng = args[0][:i], args[0][i+1:]
			}
			scope, _, err := manifest.SplitName(name)
			if err != nil {
				return err
			}
			proj, err := manifest.LoadProject(*dir)
			if err != nil {
				return err
			}
			if err := refuseM1(proj, nil); err != nil {
				return err
			}
			lock, err := loadLock(*dir)
			if err != nil {
				return err
			}
			ref, err := proj.RegistryFor(name)
			if err != nil {
				return err
			}
			client, err := d.client(ref)
			if err != nil {
				return err
			}
			idx, err := client.Index(cmd.Context(), scope)
			if err != nil {
				return err
			}
			var versions []string
			byVersion := map[string]struct {
				integrity string
				deps      int
			}{}
			for _, p := range idx.Packages {
				if p.Name != name {
					continue
				}
				for _, v := range p.Versions {
					versions = append(versions, v.Version)
					byVersion[v.Version] = struct {
						integrity string
						deps      int
					}{v.Integrity, len(v.Dependencies)}
				}
			}
			if len(versions) == 0 {
				return fmt.Errorf("%s was not found in %s", name, ref.URL)
			}
			var ranges []string
			if rng != "" {
				ranges = []string{rng}
			}
			picked, err := resolve.Pick(versions, ranges)
			if err != nil {
				return fmt.Errorf("%s: %w", name, err)
			}
			if rng == "" {
				rng = "^" + picked
			}
			if byVersion[picked].deps > 0 {
				return fmt.Errorf("%s@%s has dependencies, and dependency resolution arrives in M3", name, picked)
			}
			files, err := d.fetch(cmd.Context(), client, name, picked, byVersion[picked].integrity)
			if err != nil {
				return err
			}
			p, err := plan.Build(tree.Dir(*dir), proj, lock, []plan.Install{{
				Name: name, Version: picked, Registry: ref.URL, Integrity: byVersion[picked].integrity,
				RequestedBy: []string{manifest.ProjectFile}, Files: files,
			}})
			if err != nil {
				return err
			}
			next := *proj
			next.Dependencies = map[string]string{}
			for k, v := range proj.Dependencies {
				next.Dependencies[k] = v
			}
			next.Dependencies[name] = rng
			p.Project = &next
			if err := finish(d, *dir, p, dryRun); err != nil {
				return err
			}
			if !dryRun {
				fmt.Fprintf(d.Stdout, "added %s@%s\n", name, picked)
			}
			return nil
		},
	}
	cmd.Flags().BoolVar(&dryRun, "dry-run", false, "print the plan and change nothing")
	return cmd
}
```

`internal/cli/install.go`:

```go
package cli

import (
	"errors"
	"fmt"
	"io/fs"
	"sort"
	"strings"

	"github.com/spf13/cobra"

	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
	"github.com/StackCube/harness-package-manager-cli/internal/plan"
	"github.com/StackCube/harness-package-manager-cli/internal/status"
	"github.com/StackCube/harness-package-manager-cli/internal/tree"
)

func newInstallCmd(d Deps, dir *string) *cobra.Command {
	var dryRun bool
	cmd := &cobra.Command{
		Use:   "install",
		Short: "Install exactly what the lockfile says",
		Args:  cobra.NoArgs,
		RunE: func(cmd *cobra.Command, _ []string) error {
			proj, err := manifest.LoadProject(*dir)
			if err != nil {
				return err
			}
			if err := refuseM1(proj, nil); err != nil {
				return err
			}
			lock, err := manifest.LoadLock(*dir)
			if errors.Is(err, fs.ErrNotExist) {
				return fmt.Errorf("no %s; run `hpm add` first", manifest.LockFile)
			}
			if err != nil {
				return err
			}
			if reasons := status.Stale(proj, lock); len(reasons) > 0 {
				return fmt.Errorf("%s is stale:\n  %s", manifest.LockFile, strings.Join(reasons, "\n  "))
			}
			names := make([]string, 0, len(lock.Packages))
			for n := range lock.Packages {
				names = append(names, n)
			}
			sort.Strings(names)
			var installs []plan.Install
			for _, name := range names {
				l := lock.Packages[name]
				ref := manifest.RegistryRef{URL: l.Registry}
				for _, candidate := range proj.Registries {
					if candidate.URL == l.Registry {
						ref = candidate
					}
				}
				client, err := d.client(ref)
				if err != nil {
					return err
				}
				files, err := d.fetch(cmd.Context(), client, name, l.Version, l.Integrity)
				if err != nil {
					return err
				}
				installs = append(installs, plan.Install{Name: name, Version: l.Version, Registry: l.Registry,
					Integrity: l.Integrity, RequestedBy: l.RequestedBy, Files: files})
			}
			p, err := plan.Build(tree.Dir(*dir), proj, lock, installs)
			if err != nil {
				return err
			}
			return finish(d, *dir, p, dryRun)
		},
	}
	cmd.Flags().BoolVar(&dryRun, "dry-run", false, "print the plan and change nothing")
	return cmd
}
```

`internal/cli/status.go`:

```go
package cli

import (
	"errors"
	"fmt"
	"io/fs"

	"github.com/spf13/cobra"

	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
	"github.com/StackCube/harness-package-manager-cli/internal/status"
	"github.com/StackCube/harness-package-manager-cli/internal/tree"
)

func newStatusCmd(d Deps, dir *string) *cobra.Command {
	var ci bool
	cmd := &cobra.Command{
		Use:   "status",
		Short: "Compare the project with its lockfile",
		Args:  cobra.NoArgs,
		RunE: func(*cobra.Command, []string) error {
			proj, err := manifest.LoadProject(*dir)
			if err != nil {
				return err
			}
			problems := 0
			lock, err := manifest.LoadLock(*dir)
			switch {
			case errors.Is(err, fs.ErrNotExist):
				fmt.Fprintf(d.Stdout, "no %s\n", manifest.LockFile)
				lock, problems = manifest.NewLock(), 1
			case err != nil:
				return err
			}
			for _, p := range status.Check(tree.Dir(*dir), lock) {
				fmt.Fprintf(d.Stdout, "%-40s %-12s %s\n", p.Name, p.Version, p.State)
				for _, f := range p.Modified {
					fmt.Fprintf(d.Stdout, "    modified: %s\n", f)
				}
				for _, f := range p.Missing {
					fmt.Fprintf(d.Stdout, "    missing:  %s\n", f)
				}
				for _, h := range p.MissingHarness {
					fmt.Fprintf(d.Stdout, "    not ported to %s\n", h)
				}
				problems += len(p.Modified) + len(p.Missing)
			}
			for _, reason := range status.Stale(proj, lock) {
				fmt.Fprintf(d.Stdout, "stale: %s\n", reason)
				problems++
			}
			if ci && problems > 0 {
				return &ExitError{Code: 1}
			}
			return nil
		},
	}
	cmd.Flags().BoolVar(&ci, "ci", false, "exit 1 when anything is modified, missing or stale")
	return cmd
}
```

Run: `go test ./...` — Expected: PASS. Run `make check && make build`.
Commit: "init, publish, add, install and status".

---

### Task 6: End to end against the real Worker

**Files:**
- Create: `e2e/fakeaccess/fakeaccess.go`, `e2e/fakeaccess/fakeaccess_test.go`, `e2e/e2e_test.go`, `e2e/registry.ref`
- Modify: `Makefile`, `.github/workflows/ci.yml`, `README.md`

**Interfaces:**
- Consumes (from M1-B): in the registry repo, `bun run dev:migrate -- --persist-to <dir>`, `scripts/dev-seed.sh <dir> <scope> <principal> <kind>`, and `bunx wrangler dev --port … --persist-to … --var K:V`.
- Produces:

```go
// e2e/fakeaccess
type Proxy struct{ URL, Issuer, Audience string }
func Start(upstream string, tokens map[string]string) (*Proxy, func(), error) // tokens: client id → secret
```

- [ ] **Step 1: The fake Access proxy, test first**

`e2e/fakeaccess/fakeaccess_test.go`:

```go
package fakeaccess

import (
	"crypto/ecdsa"
	"crypto/elliptic"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"io"
	"math/big"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func TestProxy(t *testing.T) {
	var seen http.Header
	upstream := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		seen = r.Header.Clone()
		io.WriteString(w, "upstream says hi")
	}))
	defer upstream.Close()
	p, stop, err := Start(upstream.URL, map[string]string{"ci-publisher": "s3cret"})
	if err != nil {
		t.Fatal(err)
	}
	defer stop()

	get := func(id, secret, forged string) *http.Response {
		req, _ := http.NewRequest("GET", p.URL+"/v1/index", nil)
		if id != "" {
			req.Header.Set("CF-Access-Client-Id", id)
			req.Header.Set("CF-Access-Client-Secret", secret)
		}
		if forged != "" {
			req.Header.Set("Cf-Access-Jwt-Assertion", forged)
		}
		res, err := http.DefaultClient.Do(req)
		if err != nil {
			t.Fatal(err)
		}
		return res
	}

	for name, res := range map[string]*http.Response{
		"no credentials": get("", "", "forged.jwt.value"),
		"wrong secret":   get("ci-publisher", "nope", ""),
		"unknown id":     get("stranger", "s3cret", ""),
	} {
		body, _ := io.ReadAll(res.Body)
		if res.StatusCode != 403 || !strings.Contains(res.Header.Get("content-type"), "text/html") || !strings.Contains(string(body), "<html") {
			t.Errorf("%s: status %d type %q", name, res.StatusCode, res.Header.Get("content-type"))
		}
	}
	if seen != nil {
		t.Fatal("a refused request reached the upstream")
	}

	res := get("ci-publisher", "s3cret", "forged.jwt.value")
	if body, _ := io.ReadAll(res.Body); res.StatusCode != 200 || string(body) != "upstream says hi" {
		t.Fatalf("status %d body %q", res.StatusCode, body)
	}
	if seen.Get("CF-Access-Client-Id") != "" || seen.Get("CF-Access-Client-Secret") != "" {
		t.Fatal("client credentials leaked to the upstream")
	}
	token := seen.Get("Cf-Access-Jwt-Assertion")
	if token == "" || token == "forged.jwt.value" {
		t.Fatalf("assertion header: %q", token)
	}

	// Verify the token against the published JWKS with the standard library only.
	jwksRes, _ := http.Get(p.URL + "/jwks")
	var jwks struct{ Keys []struct{ Kty, Crv, X, Y, Kid, Alg string } }
	json.NewDecoder(jwksRes.Body).Decode(&jwks)
	if len(jwks.Keys) != 1 || jwks.Keys[0].Alg != "ES256" || jwks.Keys[0].Crv != "P-256" {
		t.Fatalf("jwks: %+v", jwks)
	}
	b64 := base64.RawURLEncoding
	x, _ := b64.DecodeString(jwks.Keys[0].X)
	y, _ := b64.DecodeString(jwks.Keys[0].Y)
	pub := &ecdsa.PublicKey{Curve: elliptic.P256(), X: new(big.Int).SetBytes(x), Y: new(big.Int).SetBytes(y)}
	parts := strings.Split(token, ".")
	if len(parts) != 3 {
		t.Fatalf("not a JWS: %q", token)
	}
	sig, _ := b64.DecodeString(parts[2])
	digest := sha256.Sum256([]byte(parts[0] + "." + parts[1]))
	if len(sig) != 64 || !ecdsa.Verify(pub, digest[:], new(big.Int).SetBytes(sig[:32]), new(big.Int).SetBytes(sig[32:])) {
		t.Fatal("the signature does not verify against the JWKS")
	}
	var claims struct {
		Iss        string   `json:"iss"`
		Aud        []string `json:"aud"`
		CommonName string   `json:"common_name"`
		Exp, Iat   int64
	}
	payload, _ := b64.DecodeString(parts[1])
	json.Unmarshal(payload, &claims)
	if claims.Iss != p.Issuer || len(claims.Aud) != 1 || claims.Aud[0] != p.Audience || claims.CommonName != "ci-publisher" || claims.Exp <= claims.Iat {
		t.Fatalf("claims: %+v", claims)
	}
}
```

Run: `go test ./e2e/fakeaccess/` — Expected: FAIL, undefined `Start`.

`e2e/fakeaccess/fakeaccess.go`:

```go
// Package fakeaccess stands in for Cloudflare Access in tests. It turns service-token headers into a
// signed assertion header, the way the real edge does, so that the CLI's auth path and the Worker's
// verification path are both exercised unmodified. It refuses bad credentials with HTML, like Access.
package fakeaccess

import (
	"crypto/ecdsa"
	"crypto/elliptic"
	"crypto/rand"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"io"
	"net"
	"net/http"
	"net/http/httputil"
	"net/url"
	"time"
)

type Proxy struct{ URL, Issuer, Audience string }

const kid = "fake-access"

var b64 = base64.RawURLEncoding

// Start listens on a free local port. tokens maps a client id to its secret.
func Start(upstream string, tokens map[string]string) (*Proxy, func(), error) {
	target, err := url.Parse(upstream)
	if err != nil {
		return nil, nil, err
	}
	key, err := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
	if err != nil {
		return nil, nil, err
	}
	ln, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		return nil, nil, err
	}
	p := &Proxy{URL: "http://" + ln.Addr().String(), Audience: "hpm-e2e"}
	p.Issuer = p.URL
	forward := httputil.NewSingleHostReverseProxy(target)

	mux := http.NewServeMux()
	mux.HandleFunc("/jwks", func(w http.ResponseWriter, _ *http.Request) {
		w.Header().Set("content-type", "application/json")
		json.NewEncoder(w).Encode(map[string]any{"keys": []map[string]string{{
			"kty": "EC", "crv": "P-256", "kid": kid, "alg": "ES256", "use": "sig",
			"x": b64.EncodeToString(key.PublicKey.X.FillBytes(make([]byte, 32))),
			"y": b64.EncodeToString(key.PublicKey.Y.FillBytes(make([]byte, 32))),
		}}})
	})
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		id := r.Header.Get("CF-Access-Client-Id")
		secret, known := tokens[id]
		if !known || secret != r.Header.Get("CF-Access-Client-Secret") {
			w.Header().Set("content-type", "text/html; charset=utf-8")
			w.WriteHeader(http.StatusForbidden)
			io.WriteString(w, "<html><body><h1>Forbidden</h1><p>You do not have access to this application.</p></body></html>")
			return
		}
		token, err := sign(key, p, id)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		r.Header.Del("CF-Access-Client-Id")
		r.Header.Del("CF-Access-Client-Secret")
		r.Header.Set("Cf-Access-Jwt-Assertion", token)
		r.Host = target.Host
		forward.ServeHTTP(w, r)
	})

	srv := &http.Server{Handler: mux}
	go srv.Serve(ln)
	return p, func() { srv.Close() }, nil
}

func sign(key *ecdsa.PrivateKey, p *Proxy, commonName string) (string, error) {
	now := time.Now().Unix()
	header, _ := json.Marshal(map[string]string{"alg": "ES256", "typ": "JWT", "kid": kid})
	payload, _ := json.Marshal(map[string]any{
		"iss": p.Issuer, "aud": []string{p.Audience}, "iat": now, "exp": now + 300, "common_name": commonName,
	})
	signing := b64.EncodeToString(header) + "." + b64.EncodeToString(payload)
	digest := sha256.Sum256([]byte(signing))
	r, s, err := ecdsa.Sign(rand.Reader, key, digest[:])
	if err != nil {
		return "", err
	}
	// JWS ES256 is r || s, each 32 bytes big-endian. It is not ASN.1.
	sig := append(r.FillBytes(make([]byte, 32)), s.FillBytes(make([]byte, 32))...)
	return signing + "." + b64.EncodeToString(sig), nil
}
```

Run: `go test ./e2e/fakeaccess/` — Expected: PASS. Commit: "Fake Access proxy for end-to-end tests".

- [ ] **Step 2: The end-to-end test**

`e2e/e2e_test.go`. The build tag keeps `go test ./...` from ever starting wrangler.

```go
//go:build e2e

package e2e

import (
	"bytes"
	"fmt"
	"net"
	"net/http"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
	"syscall"
	"testing"
	"time"

	"github.com/StackCube/harness-package-manager-cli/e2e/fakeaccess"
	"github.com/StackCube/harness-package-manager-cli/internal/archive"
	"github.com/StackCube/harness-package-manager-cli/internal/manifest"
)

type rig struct {
	hpm, url string
	good     []string // environment for the member service token
}

func repoRoot(t *testing.T) string {
	t.Helper()
	wd, _ := os.Getwd()
	return filepath.Dir(wd)
}

func sh(t *testing.T, dir string, name string, args ...string) {
	t.Helper()
	cmd := exec.Command(name, args...)
	cmd.Dir = dir
	if out, err := cmd.CombinedOutput(); err != nil {
		t.Fatalf("%s %s: %v\n%s", name, strings.Join(args, " "), err, out)
	}
}

func start(t *testing.T) *rig {
	t.Helper()
	root := repoRoot(t)
	regDir := os.Getenv("HPM_REGISTRY_DIR")
	if regDir == "" {
		regDir = filepath.Join(root, "..", "registry")
	}
	if _, err := os.Stat(filepath.Join(regDir, "wrangler.jsonc")); err != nil {
		t.Skipf("no registry checkout at %s; set HPM_REGISTRY_DIR", regDir)
	}
	if _, err := os.Stat(filepath.Join(regDir, "node_modules")); err != nil {
		sh(t, regDir, "bun", "install", "--frozen-lockfile")
	}
	persist := t.TempDir()
	sh(t, regDir, "bun", "run", "dev:migrate", "--", "--persist-to", persist)
	sh(t, regDir, "scripts/dev-seed.sh", persist, "@smoke", "ci-publisher", "service")

	ln, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		t.Fatal(err)
	}
	port := ln.Addr().(*net.TCPAddr).Port
	ln.Close()

	proxy, stop, err := fakeaccess.Start(fmt.Sprintf("http://127.0.0.1:%d", port), map[string]string{"ci-publisher": "s3cret", "stranger": "other"})
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(stop)

	var logs bytes.Buffer
	worker := exec.Command("bunx", "wrangler", "dev", "--ip", "127.0.0.1", "--port", fmt.Sprint(port), "--persist-to", persist,
		"--var", "AUTH_ISSUER:"+proxy.Issuer, "--var", "AUTH_AUDIENCE:"+proxy.Audience,
		"--var", "AUTH_JWKS_URL:"+proxy.URL+"/jwks", "--var", "AUTH_TOKEN_HEADER:Cf-Access-Jwt-Assertion")
	worker.Dir = regDir
	worker.Env = append(os.Environ(), "CI=1")
	worker.Stdout, worker.Stderr = &logs, &logs
	worker.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}
	if err := worker.Start(); err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() {
		syscall.Kill(-worker.Process.Pid, syscall.SIGTERM)
		worker.Wait()
		if t.Failed() {
			t.Logf("wrangler dev output:\n%s", logs.String())
		}
	})

	deadline := time.Now().Add(60 * time.Second)
	for {
		req, _ := http.NewRequest("GET", proxy.URL+"/v1/index", nil)
		req.Header.Set("CF-Access-Client-Id", "ci-publisher")
		req.Header.Set("CF-Access-Client-Secret", "s3cret")
		if res, err := http.DefaultClient.Do(req); err == nil {
			res.Body.Close()
			if res.StatusCode == 200 {
				break
			}
		}
		if time.Now().After(deadline) {
			t.Fatalf("the Worker did not become ready\n%s", logs.String())
		}
		time.Sleep(500 * time.Millisecond)
	}

	bin := filepath.Join(t.TempDir(), "hpm")
	sh(t, root, "go", "build", "-o", bin, "./cmd/hpm")
	return &rig{hpm: bin, url: proxy.URL, good: []string{"HPM_ACCESS_CLIENT_ID=ci-publisher", "HPM_ACCESS_CLIENT_SECRET=s3cret"}}
}

// run executes hpm with exactly the given environment and returns its combined output and exit code.
func (r *rig) run(env []string, args ...string) (string, int) {
	cmd := exec.Command(r.hpm, args...)
	cmd.Env = append([]string{"PATH=" + os.Getenv("PATH"), "HOME=" + os.Getenv("HOME")}, env...)
	out, err := cmd.CombinedOutput()
	if exit, ok := err.(*exec.ExitError); ok {
		return string(out), exit.ExitCode()
	}
	if err != nil {
		return string(out) + err.Error(), -1
	}
	return string(out), 0
}

func copyDir(t *testing.T, from, to string) {
	t.Helper()
	files, err := archive.ReadDir(from)
	if err != nil {
		t.Fatal(err)
	}
	for _, f := range files {
		full := filepath.Join(to, filepath.FromSlash(f.Path))
		os.MkdirAll(filepath.Dir(full), 0o755)
		if err := os.WriteFile(full, f.Data, os.FileMode(f.Mode)); err != nil {
			t.Fatal(err)
		}
	}
}

func TestWalkingSkeleton(t *testing.T) {
	r := start(t)
	fixture := filepath.Join(repoRoot(t), "contract/vectors/archives/valid/skill-only/src")
	pkg := t.TempDir()
	copyDir(t, fixture, pkg)
	skill, _ := os.ReadFile(filepath.Join(fixture, "claude/skills/ping/SKILL.md"))
	var integrity string

	t.Run("1 publish", func(t *testing.T) {
		out, code := r.run(r.good, "publish", pkg, "--registry", r.url)
		if code != 0 || !strings.Contains(out, "published @smoke/ping@0.1.0") {
			t.Fatalf("exit %d\n%s", code, out)
		}
		for _, field := range strings.Fields(out) {
			if strings.HasPrefix(field, "sha256-") {
				integrity = field
			}
		}
		if out, code := r.run(r.good, "publish", pkg, "--registry", r.url); code == 0 || !strings.Contains(out, "version_exists") {
			t.Fatalf("republish: exit %d\n%s", code, out)
		}
		stranger := []string{"HPM_ACCESS_CLIENT_ID=stranger", "HPM_ACCESS_CLIENT_SECRET=other"}
		if out, code := r.run(stranger, "publish", pkg, "--registry", r.url); code == 0 || !strings.Contains(out, "scope_forbidden") {
			t.Fatalf("non-member: exit %d\n%s", code, out)
		}
		if out, code := r.run(nil, "publish", pkg, "--registry", r.url); code == 0 || !strings.Contains(out, "HPM_ACCESS_CLIENT_ID") {
			t.Fatalf("no credentials: exit %d\n%s", code, out)
		}
		wrong := []string{"HPM_ACCESS_CLIENT_ID=ci-publisher", "HPM_ACCESS_CLIENT_SECRET=wrong"}
		if out, code := r.run(wrong, "publish", pkg, "--registry", r.url); code == 0 || !strings.Contains(out, "HTTP 403") {
			t.Fatalf("an HTML 403 from the identity layer must be survived: exit %d\n%s", code, out)
		}
	})

	proj := t.TempDir()
	os.MkdirAll(filepath.Join(proj, ".claude"), 0o755)
	installed := filepath.Join(proj, ".claude/skills/ping/SKILL.md")

	t.Run("2 add, status, install into a second clone", func(t *testing.T) {
		if out, code := r.run(nil, "-C", proj, "init", "--registry", "@smoke="+r.url); code != 0 {
			t.Fatalf("init: exit %d\n%s", code, out)
		}
		if out, code := r.run(r.good, "-C", proj, "add", "@smoke/ping"); code != 0 {
			t.Fatalf("add: exit %d\n%s", code, out)
		}
		got, _ := os.ReadFile(installed)
		if !bytes.Equal(got, skill) {
			t.Fatal("the installed skill differs from what was published")
		}
		lock, err := manifest.LoadLock(proj)
		if err != nil {
			t.Fatal(err)
		}
		l := lock.Packages["@smoke/ping"]
		if l == nil || l.Integrity != integrity || l.Files["claude-code"][".claude/skills/ping/SKILL.md"] != archive.Hash(skill) {
			t.Fatalf("lock: %+v, published integrity %s", l, integrity)
		}
		if out, code := r.run(nil, "-C", proj, "status", "--ci"); code != 0 {
			t.Fatalf("a clean project must pass --ci, offline: exit %d\n%s", code, out)
		}
		clone := t.TempDir()
		for _, f := range []string{"hpm.json", "hpm.lock"} {
			b, _ := os.ReadFile(filepath.Join(proj, f))
			os.WriteFile(filepath.Join(clone, f), b, 0o644)
		}
		if out, code := r.run(r.good, "-C", clone, "install"); code != 0 {
			t.Fatalf("install: exit %d\n%s", code, out)
		}
		again, _ := os.ReadFile(filepath.Join(clone, ".claude/skills/ping/SKILL.md"))
		if !bytes.Equal(again, skill) {
			t.Fatal("two clones differ")
		}
	})

	t.Run("3 a hand edit is detected and never overwritten", func(t *testing.T) {
		edited := append(append([]byte{}, skill...), []byte("\nA local tweak.\n")...)
		os.WriteFile(installed, edited, 0o644)
		if out, code := r.run(nil, "-C", proj, "status"); code != 0 || !strings.Contains(out, "modified") {
			t.Fatalf("status: exit %d\n%s", code, out)
		}
		if out, code := r.run(nil, "-C", proj, "status", "--ci"); code != 1 {
			t.Fatalf("status --ci: exit %d\n%s", code, out)
		}
		for _, args := range [][]string{{"-C", proj, "install"}, {"-C", proj, "add", "@smoke/ping"}} {
			if out, code := r.run(r.good, args...); code == 0 {
				t.Fatalf("%v must refuse\n%s", args, out)
			}
			kept, _ := os.ReadFile(installed)
			if !bytes.Equal(kept, edited) {
				t.Fatalf("%v overwrote the edit", args)
			}
		}
	})
}
```

`status` runs with no credentials on purpose: it must need no network. `syscall.Setpgid` and `syscall.Kill` are POSIX; the end-to-end suite runs on macOS and Linux only, which is where CI and the team are. Say so in the README.

- [ ] **Step 3: Makefile, CI, README**

`Makefile`:

```make
.PHONY: e2e
e2e:
	go test -tags e2e -count=1 -timeout 5m ./e2e/...
```

Plain `go test ./...` must not start wrangler; the build tag guarantees that. `e2e/fakeaccess` has no tag, so its unit test runs in `make check`.

`e2e/registry.ref`: the full SHA of the registry commit M1-B merged as. CI checks the registry out at that commit.

CI gains a second job. The registry repo is private, so the job needs a read token, and it must not fail when the token has not been set up yet:

```yaml
  e2e:
    runs-on: ubuntu-latest
    env:
      HAS_TOKEN: ${{ secrets.REGISTRY_READ_TOKEN != '' }}
    steps:
      - uses: actions/checkout@v4
      - name: Skip when no registry token is configured
        if: env.HAS_TOKEN != 'true'
        run: echo "::warning::REGISTRY_READ_TOKEN is not set, so the end-to-end job did not run."
      - name: Read the pinned registry commit
        if: env.HAS_TOKEN == 'true'
        id: ref
        run: echo "sha=$(cat e2e/registry.ref)" >> "$GITHUB_OUTPUT"
      - uses: actions/checkout@v4
        if: env.HAS_TOKEN == 'true'
        with:
          repository: StackCube/harness-package-manager-registry
          ref: ${{ steps.ref.outputs.sha }}
          token: ${{ secrets.REGISTRY_READ_TOKEN }}
          path: registry
      - uses: actions/setup-go@v5
        if: env.HAS_TOKEN == 'true'
        with: { go-version-file: go.mod }
      - uses: oven-sh/setup-bun@v2
        if: env.HAS_TOKEN == 'true'
      - name: End to end
        if: env.HAS_TOKEN == 'true'
        run: HPM_REGISTRY_DIR="$PWD/registry" make e2e
```

⚠ Creating `REGISTRY_READ_TOKEN` (a fine-grained token with read access to the registry repo's contents, stored as an Actions secret on the CLI repo) is Rick's to do. Say so in your report and in the PR body. Until then the job warns and passes, and `make e2e` run locally is the evidence.

README: a "Commands" section for the five commands with one example each; "Identity" (the two environment variables, and that `hpm login` is M2); "What M1 does not do yet" (dependencies M3, merge files M4, publish from a project tree M5, adopt M6, aliases iteration 2); "End-to-end tests" (`make e2e`, what it needs, `HPM_REGISTRY_DIR`).

- [ ] **Step 4: ⚠ Run it, push, open the PR**

```bash
make check && make build && make e2e
git push -u origin m1c
gh pr create --title "M1-C: CLI walking skeleton and end to end" --body "init, publish, add, install and status on plan-then-apply, with the Claude Code adapter's forward mapping and an end-to-end test against the real Worker behind a fake Access proxy.

The e2e CI job needs a secret, REGISTRY_READ_TOKEN, with read access to the registry repo. Until it exists the job warns and passes.

Plan: meta repo docs/superpowers/plans/2026-09-19-m1c-cli-skeleton-and-e2e.md.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
gh pr checks --watch
```

Put the full `make e2e` output in your report.

---

### Task 7: Write M1's decisions back

**Repo:** workspace root, branch `m1-docs`

**Files:**
- Modify: `docs/superpowers/specs/2026-09-19-iteration-1-technical-design.md`, `docs/superpowers/plans/2026-09-19-m0-carryover.md`, `docs/prd.md`, `README.md`

- [ ] **Step 1: Design spec**

- §2: archives carry no directory entries; the gzip header is fixed; ustar only; vectors pin the uncompressed tar hash; `archive_too_large` also covers 64 MB uncompressed; `vectors/archives/` layout (`cases.json`, `valid/<name>/src` and `archive.tgz`, `invalid/*.tgz`).
- §3: the R2 key is `archives/@<scope>/<name>/<version>/<sha256 hex>.tgz`, and why (a lost publish race must not leave a row pointing at other bytes). `AUTH_JWKS_JSON` as an alternative to `AUTH_JWKS_URL`. The M1 subset of the publish pipeline and where the M3 checks slot in. Reads are per instance, so `scope_forbidden` is publish-only.
- §4: `registries` lookup for `install` goes by the locked registry URL. `-C/--dir`. The `Extra` passthrough in the lockfile, so an older binary never drops fields a newer one wrote. M1's single auth provider.
- §7.4: the fake Access proxy answers 403 with HTML for bad credentials, which is how the CLI's non-JSON path is tested. CI's end-to-end job depends on `REGISTRY_READ_TOKEN`.
- §8: the M1 row notes it was delivered as three plans, M1-A, M1-B and M1-C.
- §9: add amendment rows for the R2 key, the archive format tightening and the JWKS alternative.

- [ ] **Step 2: Carry-over and PRD**

Tick every carry-over item M1 closed. Add any new deferred items the M1 reviews produce under a new heading "From M1". Fix the PRD §9 routes table: `GET /index` returns "harness ids", not "harness folders present". Add decision-log rows: "Archive storage key | Includes the archive hash | §9" and "Archive vectors | Pin the uncompressed tar, not the gzip bytes | §6".

- [ ] **Step 3: README status, commit**

Status line: M0 and M1 done, what M1 proves (publish, install, drift detection, refusal to overwrite, end to end against the real Worker), next is M2 (deploy and real identity) and what Rick must supply for it: the Cloudflare account, the zone and the identity provider.

One commit, local. Offer Rick the usual three options for the branch.

---

## Exit check for M1-C, and for M1

- [ ] `make check` and `make e2e` pass locally; the PR's `check` job is green; the `e2e` job is green or has warned about the missing token.
- [ ] End-to-end steps 1 to 3 of design §7.4 pass against the real Worker: publish, republish refused with `version_exists`, a non-member refused with `scope_forbidden`, add, clean status, install into a second clone with identical hashes, an edit reported as `modified`, `--ci` exits 1, and neither `install` nor `add` overwrites the edit.
- [ ] A conflict of every kind (`owned`, `unmanaged`, `modified`, `unsupported`, `harness`) has a planner test that asserts no plan is returned, and `apply` has a test that a failure leaves `hpm.lock` untouched and a rerun converges.
- [ ] `grep -rn "os\.\(WriteFile\|Create\|Remove\|Rename\|Mkdir\)" internal | grep -v _test.go | grep -v "internal/apply/"` prints only `internal/cli/init.go`, which writes the first `hpm.json`. Everything else writes through `apply`.
- [ ] `hpm` contains no call to git.
- [ ] A bad credential (HTML 403 from the identity layer) produces a short, legible error and a non-zero exit.
- [ ] The design spec, the PRD and the carry-over list say what was built.
