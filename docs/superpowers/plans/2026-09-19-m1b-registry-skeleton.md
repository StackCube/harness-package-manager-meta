# M1-B: Registry Walking Skeleton Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A Worker that serves the four `/v1` routes for real: identity verified on every request, publish with the core checks, archives in R2, metadata in D1, all proven inside `workerd` and against the contract's archive vectors.

**Architecture:** Hono app with one auth middleware that turns a configured JWT into a neutral `Identity`. Routes are thin: they parse parameters, call one function, and map a `Refusal` to the contract error shape. `src/store/` is the only code that touches the D1 and R2 bindings. The archive reader is a small pure module (gunzip, ustar parse, path rules) tested against `contract/vectors/archives`. Archives are stored under a key that includes their hash, so a lost publish race can never leave a row pointing at someone else's bytes.

**Tech Stack:** TypeScript, Hono, `jose`, `semver`, `@cfworker/json-schema`, Cloudflare D1 and R2, Vitest on `@cloudflare/vitest-pool-workers` 0.22.

**Spec:** `docs/superpowers/specs/2026-09-19-iteration-1-technical-design.md` §3 and §7.3. Contract: `repos/registry/contract/` at v0.4.0 (`openapi.yaml`, `spec/errors.md`, `spec/archive.md`, `spec/harnesses.md`). Requires M1-A to be merged.

## Global Constraints

- Repo: `repos/registry`. Branch `m1b`, one pull request against `main`. Push, PR and merge are ⚠ steps.
- Never deploy. `wrangler deploy` only with `--dry-run`. No `wrangler login`. No secrets in CI.
- One error shape, via `errorResponse` only. Codes and statuses exactly as `contract/spec/errors.md`.
- Publish check order is normative: `contract/spec/archive.md`, first failure wins. M1 implements these checks, in this order: identity → scope membership → compressed size → gzip and tar validity → entry types and path rules → env files → unknown fragments → manifest schema → name and version match the URL → README present → version equals a published version → version lower than the highest published. `changelog_missing`, `trigger_line_missing` and `scaffold_marker_present` arrive in M3; leave a comment at the point in the pipeline where each will go.
- There is no auth bypass of any kind. Tests use the same middleware with a test key set supplied through configuration.
- Only `src/store/*` may reference `env.DB` or `env.ARCHIVES`.
- Runtime validation never uses `eval` or `new Function`.
- `src/generated/api.ts` changes only through `bun run gen`.
- Reads are per instance: any authenticated identity may read every package. Only publishing checks scope membership.
- Scopes are stored and compared with their leading `@`.
- Commit messages end with a real trailer: blank line, then `Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>`.
- The installed `@cloudflare/vitest-pool-workers` is 0.22: `cloudflareTest()` plugin from the package root, `readD1Migrations` from the package root, `applyD1Migrations` and `env` from `cloudflare:test`. `SELF` is deprecated there in favour of `import { exports } from "cloudflare:workers"` and `exports.default.fetch()`; use whichever the installed types accept without a deprecation warning, consistently, and say which in your report.

## File map

```
repos/registry/
  wrangler.jsonc                 + d1_databases, r2_buckets, vars
  vitest.config.ts               + test key pair, migrations, archive vectors as bindings
  migrations/0001_init.sql       packages, versions, scope_members
  src/
    env.ts                       Env type, config reader
    index.ts                     app wiring, onError, notFound
    errors.ts                    + Refusal
    params.ts                    scope/name/version parsing → bad_request
    auth/identity.ts             Identity type, claims → Identity
    auth/middleware.ts           verify JWT, set c.var.identity
    publish/archive.ts           gunzip, ustar reader, rules
    publish/harnesses.ts         folder → harness id
    publish/pipeline.ts          publish(): the ordered checks, then store
    store/db.ts                  every D1 statement
    store/archives.ts            every R2 call
    routes/read.ts               index, pkg, archive
    routes/publish.ts            PUT
  test/
    env.d.ts                     ProvidedEnv typing
    setup.ts                     apply migrations
    helpers.ts                   signToken, request helpers, seedMember
    tar.ts                       build ustar + gzip in tests
    auth.test.ts  archive.test.ts  publish.test.ts  read.test.ts  app.test.ts  validate.test.ts
```

---

### Task 1: Configuration, identity and the error boundary

**Files:**
- Create: `src/env.ts`, `src/auth/identity.ts`, `src/auth/middleware.ts`, `test/env.d.ts`, `test/helpers.ts`, `test/auth.test.ts`
- Modify: `src/errors.ts`, `src/index.ts`, `wrangler.jsonc`, `vitest.config.ts`, `package.json`, `test/app.test.ts`

**Interfaces:**
- Produces:

```ts
// src/env.ts
export type Env = {
  DB: D1Database;
  ARCHIVES: R2Bucket;
  AUTH_ISSUER: string;
  AUTH_AUDIENCE: string;
  AUTH_TOKEN_HEADER: string;      // "Cf-Access-Jwt-Assertion" or "Authorization"
  AUTH_JWKS_URL?: string;         // exactly one of AUTH_JWKS_URL / AUTH_JWKS_JSON
  AUTH_JWKS_JSON?: string;
  AUTH_EMAIL_CLAIM?: string;      // default "email"
  AUTH_SERVICE_CLAIM?: string;    // default "common_name"
  AUTH_GROUPS_CLAIM?: string;     // default "groups"
};
export type AppEnv = { Bindings: Env; Variables: { identity: Identity } };

// src/auth/identity.ts
export type Identity = { kind: "person" | "service"; id: string; groups: string[] };
export function identityFromClaims(claims: Record<string, unknown>, env: Env): Identity | null;

// src/errors.ts
export class Refusal extends Error {
  constructor(public status: ContentfulStatusCode, public code: ErrorCode, message: string,
              public details: Record<string, unknown> = {});
}

// test/helpers.ts
export function signToken(claims: Record<string, unknown>, opts?: { issuer?: string; audience?: string; expiresIn?: string }): Promise<string>;
export function authed(token: string, init?: RequestInit): RequestInit;   // sets the configured header
export function call(path: string, init?: RequestInit): Promise<Response>; // fetches the Worker under test
```

- [ ] **Step 1: Dependencies and bindings**

```bash
git checkout -b m1b
bun add jose semver && bun add -d @types/semver
```

`wrangler.jsonc` gains, after `compatibility_flags`:

```jsonc
  "d1_databases": [
    { "binding": "DB", "database_name": "hpm-registry", "database_id": "00000000-0000-0000-0000-000000000000", "migrations_dir": "migrations" }
  ],
  "r2_buckets": [{ "binding": "ARCHIVES", "bucket_name": "hpm-archives" }],
  // Local development defaults. The M2 deploy script renders real values per client.
  "vars": {
    "AUTH_ISSUER": "http://127.0.0.1:8788",
    "AUTH_AUDIENCE": "hpm-local",
    "AUTH_TOKEN_HEADER": "Cf-Access-Jwt-Assertion",
    "AUTH_JWKS_URL": "http://127.0.0.1:8788/jwks"
  }
```

Port 8788 is where the CLI's end-to-end harness (M1-C) runs its fake Access proxy.

`vitest.config.ts` becomes async. It makes an ES256 key pair in Node, hands the public half to the Worker as ordinary configuration and the private half to the tests:

```ts
import { readdirSync, readFileSync } from "node:fs";
import { join, relative } from "node:path";
import { cloudflareTest, readD1Migrations } from "@cloudflare/vitest-pool-workers";
import { exportJWK, generateKeyPair } from "jose";
import { defineConfig } from "vitest/config";

const vectorRoot = join(__dirname, "contract/vectors/archives");
function archiveVectors(): Record<string, string> {
  const out: Record<string, string> = {};
  const walk = (dir: string): void => {
    for (const e of readdirSync(dir, { withFileTypes: true })) {
      const p = join(dir, e.name);
      if (e.isDirectory()) walk(p);
      else if (e.name.endsWith(".tgz")) out[relative(vectorRoot, p)] = readFileSync(p).toString("base64");
    }
  };
  walk(vectorRoot);
  return out;
}

export default defineConfig(async () => {
  const { publicKey, privateKey } = await generateKeyPair("ES256", { extractable: true });
  const pub = { ...(await exportJWK(publicKey)), kid: "test-key", alg: "ES256", use: "sig" };
  const priv = { ...(await exportJWK(privateKey)), kid: "test-key", alg: "ES256" };
  return {
    test: { include: ["test/**/*.test.ts"], setupFiles: ["./test/setup.ts"] },
    plugins: [
      cloudflareTest({
        wrangler: { configPath: "./wrangler.jsonc" },
        miniflare: {
          bindings: {
            AUTH_ISSUER: "https://issuer.test",
            AUTH_AUDIENCE: "hpm-test",
            AUTH_TOKEN_HEADER: "Cf-Access-Jwt-Assertion",
            AUTH_JWKS_URL: "",
            AUTH_JWKS_JSON: JSON.stringify({ keys: [pub] }),
            TEST_PRIVATE_JWK: JSON.stringify(priv),
            TEST_MIGRATIONS: await readD1Migrations(join(__dirname, "migrations")),
            TEST_ARCHIVE_VECTORS: JSON.stringify(archiveVectors()),
          },
        },
      }),
    ],
  };
});
```

Keep the two explanatory comments the file already has. `migrations/` does not exist until Task 2; create the empty directory with a `.gitkeep` now so the config loads, and create `test/setup.ts` as an empty module (`export {};`). If `__dirname` is unavailable in this ESM config, use `import.meta.dirname`.

`test/env.d.ts`:

```ts
import type { D1Migration } from "@cloudflare/vitest-pool-workers";
import type { Env } from "../src/env";

declare module "cloudflare:test" {
  interface ProvidedEnv extends Env {
    TEST_PRIVATE_JWK: string;
    TEST_MIGRATIONS: D1Migration[];
    TEST_ARCHIVE_VECTORS: string;
  }
}
```

- [ ] **Step 2: Test helpers**

`test/helpers.ts`:

```ts
import { env, SELF } from "cloudflare:test";
import { importJWK, SignJWT } from "jose";

export async function signToken(
  claims: Record<string, unknown>,
  opts: { issuer?: string; audience?: string; expiresIn?: string } = {},
): Promise<string> {
  const jwk = JSON.parse(env.TEST_PRIVATE_JWK);
  const key = await importJWK(jwk, "ES256");
  return new SignJWT(claims)
    .setProtectedHeader({ alg: "ES256", kid: jwk.kid })
    .setIssuer(opts.issuer ?? env.AUTH_ISSUER)
    .setAudience(opts.audience ?? env.AUTH_AUDIENCE)
    .setIssuedAt()
    .setExpirationTime(opts.expiresIn ?? "5m")
    .sign(key);
}

export function authed(token: string, init: RequestInit = {}): RequestInit {
  const headers = new Headers(init.headers);
  headers.set(env.AUTH_TOKEN_HEADER, token);
  return { ...init, headers };
}

export function call(path: string, init?: RequestInit): Promise<Response> {
  return SELF.fetch(`https://registry.test${path}`, init);
}

export const person = (email: string, groups: string[] = []) => signToken({ email, groups });
export const service = (commonName: string) => signToken({ common_name: commonName });
```

If you chose `exports.default.fetch` over `SELF`, change only `call`.

- [ ] **Step 3: Failing tests**

`test/auth.test.ts`:

```ts
import { expect, test } from "vitest";
import { authed, call, person, service, signToken } from "./helpers";

const whoami = "/v1/index"; // any route behind the middleware

async function code(res: Response) {
  return ((await res.json()) as { code: string }).code;
}

test("no token is unauthenticated", async () => {
  const res = await call(whoami);
  expect(res.status).toBe(401);
  expect(await code(res)).toBe("unauthenticated");
});

test("a garbage token is unauthenticated", async () => {
  const res = await call(whoami, authed("not.a.jwt"));
  expect(res.status).toBe(401);
  expect(await code(res)).toBe("unauthenticated");
});

test("wrong issuer, wrong audience and expired tokens are all refused", async () => {
  for (const token of [
    await signToken({ email: "a@x.test" }, { issuer: "https://evil.test" }),
    await signToken({ email: "a@x.test" }, { audience: "someone-else" }),
    await signToken({ email: "a@x.test" }, { expiresIn: "-1m" }),
  ]) {
    const res = await call(whoami, authed(token));
    expect(res.status).toBe(401);
  }
});

test("a token with neither an email nor a service name is refused", async () => {
  const res = await call(whoami, authed(await signToken({ sub: "x" })));
  expect(res.status).toBe(401);
});

test("a person and a service token are both admitted", async () => {
  for (const token of [await person("rick@ourorg.example"), await service("ci-publisher")]) {
    const res = await call(whoami, authed(token));
    expect(res.status).not.toBe(401);
  }
});

test("the 404 for an unknown route still requires identity", async () => {
  expect((await call("/v1/nope")).status).toBe(401);
  const res = await call("/v1/nope", authed(await person("rick@ourorg.example")));
  expect(res.status).toBe(404);
  expect(await code(res)).toBe("not_found");
});
```

Unit tests for the claim mapping, in the same file:

```ts
import { env } from "cloudflare:test";
import { identityFromClaims } from "../src/auth/identity";

test("claims map to a neutral identity", () => {
  expect(identityFromClaims({ email: "Rick@OurOrg.example", groups: ["platform", 7] }, env)).toEqual({
    kind: "person", id: "rick@ourorg.example", groups: ["platform"],
  });
  expect(identityFromClaims({ common_name: "ci-publisher" }, env)).toEqual({
    kind: "service", id: "ci-publisher", groups: [],
  });
  expect(identityFromClaims({ email: "", common_name: "" }, env)).toBeNull();
});
```

Update `test/app.test.ts`: its one test must now send a token, because nothing is reachable without identity. Assert the same 404 body as before.

Run: `bun run test` — Expected: FAIL. The auth tests get 404 where they expect 401.

- [ ] **Step 4: Implement**

`src/env.ts`: the `Env` and `AppEnv` types from the Interfaces block, with `import type { Identity } from "./auth/identity";`.

`src/errors.ts`, appended:

```ts
/** Thrown anywhere below a route to refuse a request with a contract error. */
export class Refusal extends Error {
  constructor(
    public status: ContentfulStatusCode,
    public code: ErrorCode,
    message: string,
    public details: Record<string, unknown> = {},
  ) {
    super(message);
  }
}
```

`src/auth/identity.ts`:

```ts
import type { Env } from "../env";

export type Identity = { kind: "person" | "service"; id: string; groups: string[] };

/** Maps verified JWT claims to a neutral identity. Returns null when the token names nobody. */
export function identityFromClaims(claims: Record<string, unknown>, env: Env): Identity | null {
  const str = (v: unknown) => (typeof v === "string" && v.trim() !== "" ? v.trim() : null);
  const rawGroups = claims[env.AUTH_GROUPS_CLAIM || "groups"];
  const groups = Array.isArray(rawGroups) ? rawGroups.filter((g): g is string => typeof g === "string") : [];
  const email = str(claims[env.AUTH_EMAIL_CLAIM || "email"]);
  if (email) return { kind: "person", id: email.toLowerCase(), groups };
  const svc = str(claims[env.AUTH_SERVICE_CLAIM || "common_name"]);
  if (svc) return { kind: "service", id: svc, groups: [] };
  return null;
}
```

`src/auth/middleware.ts`:

```ts
import type { MiddlewareHandler } from "hono";
import { createLocalJWKSet, createRemoteJWKSet, jwtVerify, type JWTVerifyGetKey } from "jose";
import type { AppEnv, Env } from "../env";
import { Refusal } from "../errors";
import { identityFromClaims } from "./identity";

// One key set per isolate. Remote sets cache and rate-limit their own fetches.
let cached: { source: string; keys: JWTVerifyGetKey } | undefined;

function keySet(env: Env): JWTVerifyGetKey {
  const source = env.AUTH_JWKS_JSON || env.AUTH_JWKS_URL || "";
  if (!source) throw new Error("Set AUTH_JWKS_URL or AUTH_JWKS_JSON");
  if (cached?.source !== source) {
    cached = {
      source,
      keys: env.AUTH_JWKS_JSON
        ? createLocalJWKSet(JSON.parse(env.AUTH_JWKS_JSON))
        : createRemoteJWKSet(new URL(env.AUTH_JWKS_URL!)),
    };
  }
  return cached.keys;
}

export const requireIdentity: MiddlewareHandler<AppEnv> = async (c, next) => {
  const header = c.req.header(c.env.AUTH_TOKEN_HEADER) ?? "";
  const token = header.replace(/^Bearer\s+/i, "").trim();
  if (!token) throw new Refusal(401, "unauthenticated", "No identity token was presented.");
  let claims: Record<string, unknown>;
  try {
    const { payload } = await jwtVerify(token, keySet(c.env), {
      issuer: c.env.AUTH_ISSUER,
      audience: c.env.AUTH_AUDIENCE,
    });
    claims = payload;
  } catch {
    throw new Refusal(401, "unauthenticated", "The identity token could not be verified.");
  }
  const identity = identityFromClaims(claims, c.env);
  if (!identity) throw new Refusal(401, "unauthenticated", "The identity token names neither a person nor a service.");
  c.set("identity", identity);
  await next();
};
```

A misconfigured key set throws a plain `Error`, which `onError` turns into `internal_error`. That is deliberate: a broken deployment must not look like a bad token.

`src/index.ts`:

```ts
import { Hono } from "hono";
import { requireIdentity } from "./auth/middleware";
import type { AppEnv } from "./env";
import { errorResponse, Refusal } from "./errors";

const app = new Hono<AppEnv>();

app.use("*", requireIdentity);

// Routes are mounted here in Tasks 4 and 5.

app.notFound((c) => errorResponse(c, 404, "not_found", `No such route: ${c.req.method} ${new URL(c.req.url).pathname}`));

app.onError((err, c) => {
  if (err instanceof Refusal) return errorResponse(c, err.status, err.code, err.message, err.details);
  console.error(err);
  return errorResponse(c, 500, "internal_error", "The registry failed to handle the request.");
});

export default app;
```

Hono runs `notFound` after the middleware chain, so an unknown route is 401 without a token and 404 with one, as the test demands. If the installed Hono version answers 404 before the `*` middleware runs, mount a final `app.all("*", …)` that throws the `not_found` refusal instead, and say so in your report.

Add one test to `test/app.test.ts` that proves the error boundary without adding a production route: build a tiny Hono app in the test that reuses the same `onError` handler. To make that possible, export the handler from `src/index.ts` as a named export `handleError` and register it with `app.onError(handleError)`.

```ts
import { Hono } from "hono";
import { handleError } from "../src/index";

test("an unexpected throw becomes internal_error and leaks nothing", async () => {
  const boom = new Hono();
  boom.get("/", () => { throw new Error("secret detail"); });
  boom.onError(handleError);
  const res = await boom.request("/");
  expect(res.status).toBe(500);
  const body = (await res.json()) as { code: string; message: string };
  expect(body.code).toBe("internal_error");
  expect(JSON.stringify(body)).not.toContain("secret detail");
});
```

Run: `bun run test` — Expected: PASS, with `console.error` output from the boom test only. If that output counts as noise in your run, stub `console.error` with `vi.spyOn(console, "error").mockImplementation(() => {})` inside that test and assert it was called once.

Run: `bun run typecheck && bunx wrangler deploy --dry-run --outdir dist`.
Commit: "Identity middleware, Refusal and the error boundary".

---

### Task 2: D1 schema and the store

**Files:**
- Create: `migrations/0001_init.sql`, `src/store/db.ts`, `src/store/archives.ts`, `test/store.test.ts`
- Modify: `test/setup.ts`, `test/helpers.ts`

**Interfaces:**
- Produces:

```ts
// src/store/db.ts
export type VersionRow = {
  scope: string; name: string; version: string; integrity: string; size: number;
  manifest: PackageManifest; harnesses: HarnessId[]; readme: string;
  publishedBy: string; publishedAt: string;
};
export type HarnessId = "claude-code" | "copilot" | "kiro" | "codex";
export async function isMember(db: D1Database, scope: string, identity: Identity): Promise<boolean>;
export async function listVersions(db: D1Database, scope?: string): Promise<VersionRow[]>;          // unordered
export async function packageVersions(db: D1Database, scope: string, name: string): Promise<VersionRow[]>;
export async function getVersion(db: D1Database, scope: string, name: string, version: string): Promise<VersionRow | null>;
export async function insertVersion(db: D1Database, row: VersionRow): Promise<"inserted" | "exists">;

// src/store/archives.ts
export function archiveKey(scope: string, name: string, version: string, integrity: string): string;
export async function putArchive(bucket: R2Bucket, key: string, bytes: Uint8Array): Promise<void>;
export async function getArchive(bucket: R2Bucket, key: string): Promise<R2ObjectBody | null>;

// test/helpers.ts
export async function seedMember(scope: string, principal: string, kind: "email" | "group" | "service"): Promise<void>;
```

`PackageManifest` is `components["schemas"]["PackageManifest"]` from `src/generated/api.ts`.

- [ ] **Step 1: Migration**

`migrations/0001_init.sql`:

```sql
CREATE TABLE packages (
  scope      TEXT NOT NULL,
  name       TEXT NOT NULL,
  created_at TEXT NOT NULL,
  PRIMARY KEY (scope, name)
);

CREATE TABLE versions (
  scope           TEXT NOT NULL,
  name            TEXT NOT NULL,
  version         TEXT NOT NULL,
  integrity       TEXT NOT NULL,
  size            INTEGER NOT NULL,
  manifest_json   TEXT NOT NULL,
  harnesses_json  TEXT NOT NULL,
  readme          TEXT NOT NULL,
  -- Filled from M3, when publish checks the changelog and the trigger rule.
  changelog_entry TEXT NOT NULL DEFAULT '',
  triggers_json   TEXT NOT NULL DEFAULT '{}',
  published_by    TEXT NOT NULL,
  published_at    TEXT NOT NULL,
  PRIMARY KEY (scope, name, version),
  FOREIGN KEY (scope, name) REFERENCES packages (scope, name)
);

CREATE TABLE scope_members (
  scope     TEXT NOT NULL,
  principal TEXT NOT NULL,
  kind      TEXT NOT NULL CHECK (kind IN ('email', 'group', 'service')),
  PRIMARY KEY (scope, principal, kind)
);
```

Remove `migrations/.gitkeep`.

`test/setup.ts`:

```ts
import { applyD1Migrations, env } from "cloudflare:test";

await applyD1Migrations(env.DB, env.TEST_MIGRATIONS);
```

- [ ] **Step 2: Failing tests**

`test/store.test.ts`:

```ts
import { env } from "cloudflare:test";
import { expect, test } from "vitest";
import { archiveKey, getArchive, putArchive } from "../src/store/archives";
import { getVersion, insertVersion, isMember, listVersions, packageVersions, type VersionRow } from "../src/store/db";
import { seedMember } from "./helpers";

const row = (over: Partial<VersionRow> = {}): VersionRow => ({
  scope: "@t", name: "pkg", version: "1.0.0",
  integrity: "sha256-" + "a".repeat(64), size: 10,
  manifest: { name: "@t/pkg", version: "1.0.0", description: "d", keywords: [], owners: ["o@x.test"] },
  harnesses: ["claude-code"], readme: "# pkg", publishedBy: "o@x.test", publishedAt: "2026-09-19T10:00:00.000Z",
  ...over,
});

test("membership by email, by group and by service name", async () => {
  await seedMember("@m", "rick@ourorg.example", "email");
  await seedMember("@m", "platform", "group");
  await seedMember("@m", "ci-publisher", "service");
  const p = (id: string, groups: string[] = []) => ({ kind: "person" as const, id, groups });
  expect(await isMember(env.DB, "@m", p("rick@ourorg.example"))).toBe(true);
  expect(await isMember(env.DB, "@m", p("other@ourorg.example", ["platform"]))).toBe(true);
  expect(await isMember(env.DB, "@m", p("other@ourorg.example", ["sales"]))).toBe(false);
  expect(await isMember(env.DB, "@m", { kind: "service", id: "ci-publisher", groups: [] })).toBe(true);
  // A person whose email equals a service principal gains nothing, and the reverse.
  expect(await isMember(env.DB, "@m", p("ci-publisher"))).toBe(false);
  expect(await isMember(env.DB, "@other", p("rick@ourorg.example"))).toBe(false);
});

test("insert, read back, and refuse a duplicate version", async () => {
  expect(await insertVersion(env.DB, row())).toBe("inserted");
  expect(await insertVersion(env.DB, row({ integrity: "sha256-" + "b".repeat(64) }))).toBe("exists");
  const got = await getVersion(env.DB, "@t", "pkg", "1.0.0");
  expect(got).toEqual(row());
  expect(await getVersion(env.DB, "@t", "pkg", "9.9.9")).toBeNull();
});

test("list by scope and by package", async () => {
  await insertVersion(env.DB, row({ scope: "@l", version: "1.0.0" }));
  await insertVersion(env.DB, row({ scope: "@l", version: "1.1.0" }));
  await insertVersion(env.DB, row({ scope: "@l2", version: "1.0.0" }));
  expect((await listVersions(env.DB, "@l")).length).toBe(2);
  expect((await listVersions(env.DB)).length).toBeGreaterThanOrEqual(3);
  expect((await packageVersions(env.DB, "@l", "pkg")).map((v) => v.version).sort()).toEqual(["1.0.0", "1.1.0"]);
});

test("the archive key carries the hash, and archives round-trip", async () => {
  const integrity = "sha256-" + "c".repeat(64);
  const key = archiveKey("@t", "pkg", "1.0.0", integrity);
  expect(key).toBe(`archives/@t/pkg/1.0.0/${"c".repeat(64)}.tgz`);
  await putArchive(env.ARCHIVES, key, new Uint8Array([1, 2, 3]));
  const obj = await getArchive(env.ARCHIVES, key);
  expect(new Uint8Array(await obj!.arrayBuffer())).toEqual(new Uint8Array([1, 2, 3]));
  expect(await getArchive(env.ARCHIVES, "archives/none")).toBeNull();
});
```

Add to `test/helpers.ts`:

```ts
export async function seedMember(scope: string, principal: string, kind: "email" | "group" | "service"): Promise<void> {
  await env.DB.prepare("INSERT OR IGNORE INTO scope_members (scope, principal, kind) VALUES (?, ?, ?)")
    .bind(scope, principal, kind).run();
}
```

Tests in one file share storage unless the pool isolates them. Every test above uses its own scope so that it does not depend on isolation.

Run: `bun run test test/store.test.ts` — Expected: FAIL, modules not found.

- [ ] **Step 3: Implement**

`src/store/db.ts`:

```ts
import type { Identity } from "../auth/identity";
import type { components } from "../generated/api";

type PackageManifest = components["schemas"]["PackageManifest"];
export type HarnessId = "claude-code" | "copilot" | "kiro" | "codex";

export type VersionRow = {
  scope: string; name: string; version: string; integrity: string; size: number;
  manifest: PackageManifest; harnesses: HarnessId[]; readme: string;
  publishedBy: string; publishedAt: string;
};

type Raw = {
  scope: string; name: string; version: string; integrity: string; size: number;
  manifest_json: string; harnesses_json: string; readme: string; published_by: string; published_at: string;
};

const COLUMNS = "scope, name, version, integrity, size, manifest_json, harnesses_json, readme, published_by, published_at";

const fromRaw = (r: Raw): VersionRow => ({
  scope: r.scope, name: r.name, version: r.version, integrity: r.integrity, size: r.size,
  manifest: JSON.parse(r.manifest_json), harnesses: JSON.parse(r.harnesses_json), readme: r.readme,
  publishedBy: r.published_by, publishedAt: r.published_at,
});

export async function isMember(db: D1Database, scope: string, identity: Identity): Promise<boolean> {
  const own = identity.kind === "person" ? "email" : "service";
  const direct = await db
    .prepare("SELECT 1 FROM scope_members WHERE scope = ? AND kind = ? AND principal = ? LIMIT 1")
    .bind(scope, own, identity.id).first();
  if (direct) return true;
  for (const group of identity.groups) {
    const hit = await db
      .prepare("SELECT 1 FROM scope_members WHERE scope = ? AND kind = 'group' AND principal = ? LIMIT 1")
      .bind(scope, group).first();
    if (hit) return true;
  }
  return false;
}

export async function listVersions(db: D1Database, scope?: string): Promise<VersionRow[]> {
  const stmt = scope
    ? db.prepare(`SELECT ${COLUMNS} FROM versions WHERE scope = ?`).bind(scope)
    : db.prepare(`SELECT ${COLUMNS} FROM versions`);
  return (await stmt.all<Raw>()).results.map(fromRaw);
}

export async function packageVersions(db: D1Database, scope: string, name: string): Promise<VersionRow[]> {
  const res = await db.prepare(`SELECT ${COLUMNS} FROM versions WHERE scope = ? AND name = ?`).bind(scope, name).all<Raw>();
  return res.results.map(fromRaw);
}

export async function getVersion(db: D1Database, scope: string, name: string, version: string): Promise<VersionRow | null> {
  const r = await db
    .prepare(`SELECT ${COLUMNS} FROM versions WHERE scope = ? AND name = ? AND version = ?`)
    .bind(scope, name, version).first<Raw>();
  return r ? fromRaw(r) : null;
}

/** The primary key is the immutability guard. A duplicate is reported, never overwritten. */
export async function insertVersion(db: D1Database, row: VersionRow): Promise<"inserted" | "exists"> {
  try {
    await db.batch([
      db.prepare("INSERT OR IGNORE INTO packages (scope, name, created_at) VALUES (?, ?, ?)")
        .bind(row.scope, row.name, row.publishedAt),
      db.prepare(`INSERT INTO versions (${COLUMNS}) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`)
        .bind(row.scope, row.name, row.version, row.integrity, row.size, JSON.stringify(row.manifest),
          JSON.stringify(row.harnesses), row.readme, row.publishedBy, row.publishedAt),
    ]);
    return "inserted";
  } catch (err) {
    if (String(err).includes("UNIQUE constraint failed")) return "exists";
    throw err;
  }
}
```

`src/store/archives.ts`:

```ts
/**
 * The key includes the archive's hash. Two racing publishes of the same version write different
 * objects; the one whose row wins points at its own bytes, and the loser's object is an orphan.
 */
export function archiveKey(scope: string, name: string, version: string, integrity: string): string {
  return `archives/${scope}/${name}/${version}/${integrity.replace(/^sha256-/, "")}.tgz`;
}

export async function putArchive(bucket: R2Bucket, key: string, bytes: Uint8Array): Promise<void> {
  await bucket.put(key, bytes, { httpMetadata: { contentType: "application/gzip" } });
}

export async function getArchive(bucket: R2Bucket, key: string): Promise<R2ObjectBody | null> {
  return bucket.get(key);
}
```

This key differs from design §3, which has `archives/@scope/name/version.tgz`. The hash in the key closes a race the design missed. Record the amendment in your report; the design spec is updated in M1-C's final task.

Run: `bun run test` — Expected: PASS. Commit: "D1 schema and the store".

---

### Task 3: The archive reader

**Files:**
- Create: `src/publish/archive.ts`, `src/publish/harnesses.ts`, `test/tar.ts`, `test/archive.test.ts`

**Interfaces:**
- Produces:

```ts
// src/publish/archive.ts
export const MAX_COMPRESSED = 10 * 1024 * 1024;
export const MAX_UNCOMPRESSED = 64 * 1024 * 1024;
export type ArchiveFile = { path: string; mode: number; data: Uint8Array };
export async function sha256(bytes: Uint8Array): Promise<string>;            // "sha256-<hex>"
export async function readArchive(gz: Uint8Array): Promise<{ files: ArchiveFile[]; tar: Uint8Array }>; // throws Refusal
export function checkPath(path: string): Refusal | null;
export function isEnvFile(path: string): boolean;
export function isUnknownFragment(path: string): boolean;

// src/publish/harnesses.ts
export function harnessesOf(paths: string[]): HarnessId[];   // in the fixed order of spec/harnesses.md

// test/tar.ts
export type Entry = { name: string; data?: string | Uint8Array; mode?: number; typeflag?: string; linkname?: string };
export function tarOf(entries: Entry[]): Uint8Array;
export async function gzip(bytes: Uint8Array): Promise<Uint8Array>;
export async function archiveOf(entries: Entry[]): Promise<Uint8Array>;      // gzip(tarOf(entries))
```

- [ ] **Step 1: The test tar builder**

`test/tar.ts`. A ustar header is 512 bytes; the checksum is the byte sum of the header with the checksum field read as eight spaces.

```ts
export type Entry = { name: string; data?: string | Uint8Array; mode?: number; typeflag?: string; linkname?: string };

const enc = new TextEncoder();

function header(e: Entry, size: number): Uint8Array {
  const h = new Uint8Array(512);
  const put = (s: string, at: number) => h.set(enc.encode(s), at);
  const octal = (n: number, width: number) => n.toString(8).padStart(width - 1, "0") + "\0";
  put(e.name, 0);
  put(octal(e.mode ?? 0o644, 8), 100);
  put(octal(0, 8), 108);
  put(octal(0, 8), 116);
  put(octal(size, 12), 124);
  put(octal(0, 12), 136);
  put("        ", 148);
  put(e.typeflag ?? "0", 156);
  if (e.linkname) put(e.linkname, 157);
  put("ustar\0", 257);
  put("00", 263);
  let sum = 0;
  for (const b of h) sum += b;
  put(sum.toString(8).padStart(6, "0") + "\0 ", 148);
  return h;
}

export function tarOf(entries: Entry[]): Uint8Array {
  const parts: Uint8Array[] = [];
  for (const e of entries) {
    const data = typeof e.data === "string" ? enc.encode(e.data) : (e.data ?? new Uint8Array());
    parts.push(header(e, data.length), data, new Uint8Array((512 - (data.length % 512)) % 512));
  }
  parts.push(new Uint8Array(1024));
  const out = new Uint8Array(parts.reduce((n, p) => n + p.length, 0));
  let at = 0;
  for (const p of parts) { out.set(p, at); at += p.length; }
  return out;
}

export async function gzip(bytes: Uint8Array): Promise<Uint8Array> {
  const stream = new Blob([bytes]).stream().pipeThrough(new CompressionStream("gzip"));
  return new Uint8Array(await new Response(stream).arrayBuffer());
}

export const archiveOf = async (entries: Entry[]) => gzip(tarOf(entries));
```

- [ ] **Step 2: Failing tests**

`test/archive.test.ts`:

```ts
import { env } from "cloudflare:test";
import { describe, expect, test } from "vitest";
import cases from "../contract/vectors/archives/cases.json";
import { Refusal } from "../src/errors";
import { checkPath, isEnvFile, isUnknownFragment, readArchive, sha256 } from "../src/publish/archive";
import { harnessesOf } from "../src/publish/harnesses";
import { archiveOf } from "./tar";

const vectors: Record<string, string> = JSON.parse(env.TEST_ARCHIVE_VECTORS);
const bytes = (rel: string) => Uint8Array.from(atob(vectors[rel]), (c) => c.charCodeAt(0));

async function refusal(p: Promise<unknown>): Promise<Refusal> {
  try { await p; } catch (e) { if (e instanceof Refusal) return e; throw e; }
  throw new Error("expected a Refusal");
}

describe("contract archive vectors", () => {
  test("the vectors were loaded", () => {
    expect(cases.valid.length).toBeGreaterThan(0);
    expect(Object.keys(vectors).length).toBe(cases.valid.length + cases.invalid.length);
  });

  test.each(cases.valid.map((c) => [c.name, c] as const))("valid: %s", async (_n, c) => {
    const { files, tar } = await readArchive(bytes(c.archive));
    expect(await sha256(tar)).toBe(c.tarSha256);
    expect(files.map((f) => f.path)).toEqual(c.files.map((f) => f.path));
    for (const [i, f] of files.entries()) {
      expect(await sha256(f.data), f.path).toBe(c.files[i].sha256);
      expect(f.mode.toString(8).padStart(4, "0"), f.path).toBe(c.files[i].mode);
    }
  });

  test.each(cases.invalid.map((c) => [c.name, c] as const))("hostile: %s", async (_n, c) => {
    expect((await refusal(readArchive(bytes(c.archive)))).code).toBe(c.code);
  });
});

describe("rules", () => {
  test("paths", () => {
    for (const p of ["hpm.json", "claude/skills/x/SKILL.md", "a/b..c/d", ".env.example"]) expect(checkPath(p)).toBeNull();
    for (const p of ["/etc/passwd", "../x", "a/../b", "a/..", "", "a//b", "a\\b", "./a"]) expect(checkPath(p)?.code).toBe("path_forbidden");
  });
  test("env files by base name at any depth", () => {
    expect([".env", "x/.env.local", "x/prod.env"].every(isEnvFile)).toBe(true);
    expect([".env.example", "x/.env.example", "environment.md", "x/env"].some(isEnvFile)).toBe(false);
  });
  test("fragments", () => {
    for (const p of ["claude/CLAUDE.append.md", "claude/settings.merge.json", "claude/mcp.merge.json"]) expect(isUnknownFragment(p)).toBe(false);
    for (const p of ["claude/other.merge.json", "claude/skills/x/notes.append.md", "kiro/settings.merge.json"]) expect(isUnknownFragment(p)).toBe(true);
    expect(isUnknownFragment("claude/skills/merge/SKILL.md")).toBe(false);
  });
  test("harness ids from top-level folders, in table order", () => {
    expect(harnessesOf(["kiro/steering/x.md", "claude/skills/x/SKILL.md", "hpm.json", "docs/x.md"])).toEqual(["claude-code", "kiro"]);
    expect(harnessesOf(["hpm.json", "README.md"])).toEqual([]);
  });
});

describe("limits and edge cases", () => {
  test("an archive with no files is invalid", async () => {
    expect((await refusal(readArchive(await archiveOf([])))).code).toBe("archive_invalid");
  });
  test("a duplicate entry is invalid", async () => {
    const gz = await archiveOf([{ name: "a", data: "1" }, { name: "a", data: "2" }]);
    expect((await refusal(readArchive(gz))).code).toBe("archive_invalid");
  });
  test("a bad header checksum is invalid", async () => {
    const { tarOf, gzip } = await import("./tar");
    const tar = tarOf([{ name: "a", data: "1" }]);
    tar[0] = "b".charCodeAt(0);
    expect((await refusal(readArchive(await gzip(tar)))).code).toBe("archive_invalid");
  });
  test("directory entries are accepted and ignored", async () => {
    const gz = await archiveOf([{ name: "claude/", typeflag: "5", mode: 0o755 }, { name: "claude/a.md", data: "x" }]);
    expect((await readArchive(gz)).files.map((f) => f.path)).toEqual(["claude/a.md"]);
  });
  test("an executable keeps 0755 and everything else is 0644", async () => {
    const gz = await archiveOf([{ name: "run.sh", data: "x", mode: 0o700 }, { name: "a.md", data: "x", mode: 0o600 }]);
    expect((await readArchive(gz)).files.map((f) => f.mode)).toEqual([0o644, 0o755]);
  });
});
```

The last test sorts by path: `a.md` then `run.sh`. JSON imports need `resolveJsonModule`, which is already on.

Run: `bun run test test/archive.test.ts` — Expected: FAIL, modules not found.

- [ ] **Step 3: Implement**

`src/publish/harnesses.ts`:

```ts
import type { HarnessId } from "../store/db";

// contract/spec/harnesses.md. Order here is the order in the index.
const FOLDERS: [string, HarnessId][] = [["claude", "claude-code"], ["copilot", "copilot"], ["kiro", "kiro"], ["codex", "codex"]];

export function harnessesOf(paths: string[]): HarnessId[] {
  const top = new Set(paths.filter((p) => p.includes("/")).map((p) => p.slice(0, p.indexOf("/"))));
  return FOLDERS.filter(([folder]) => top.has(folder)).map(([, id]) => id);
}
```

`src/publish/archive.ts`:

```ts
import { Refusal } from "../errors";

export const MAX_COMPRESSED = 10 * 1024 * 1024;
export const MAX_UNCOMPRESSED = 64 * 1024 * 1024;

export type ArchiveFile = { path: string; mode: number; data: Uint8Array };

const invalid = (msg: string) => new Refusal(400, "archive_invalid", msg);
const dec = new TextDecoder();

export async function sha256(bytes: Uint8Array): Promise<string> {
  const digest = new Uint8Array(await crypto.subtle.digest("SHA-256", bytes));
  return "sha256-" + Array.from(digest, (b) => b.toString(16).padStart(2, "0")).join("");
}

export function isEnvFile(path: string): boolean {
  const base = path.slice(path.lastIndexOf("/") + 1);
  if (base === ".env.example") return false;
  return base === ".env" || base.startsWith(".env.") || base.endsWith(".env");
}

export function checkPath(path: string): Refusal | null {
  const forbid = (msg: string) => new Refusal(400, "path_forbidden", `${msg}: ${JSON.stringify(path)}`, { path });
  if (path === "") return forbid("Empty path");
  if (path.startsWith("/")) return forbid("Absolute path");
  if (path.includes("\\")) return forbid("Backslash in path");
  for (const seg of path.split("/")) {
    if (seg === "") return forbid("Empty path segment");
    if (seg === ".." || seg === ".") return forbid("Relative path segment");
  }
  return null;
}

// contract/spec/archive.md: fragments may only be the known merge files of their harness folder.
const KNOWN_FRAGMENTS = new Set(["claude/CLAUDE.append.md", "claude/settings.merge.json", "claude/mcp.merge.json"]);

export function isUnknownFragment(path: string): boolean {
  const base = path.slice(path.lastIndexOf("/") + 1);
  return /\.(merge|append)\./.test(base) && !KNOWN_FRAGMENTS.has(path);
}

async function gunzip(gz: Uint8Array): Promise<Uint8Array> {
  const chunks: Uint8Array[] = [];
  let total = 0;
  try {
    const reader = new Blob([gz]).stream().pipeThrough(new DecompressionStream("gzip")).getReader();
    for (;;) {
      const { done, value } = await reader.read();
      if (done) break;
      total += value.length;
      if (total > MAX_UNCOMPRESSED) {
        await reader.cancel();
        throw new Refusal(413, "archive_too_large", "The uncompressed content is over 64 MB.");
      }
      chunks.push(value);
    }
  } catch (err) {
    if (err instanceof Refusal) throw err;
    throw invalid("The body is not valid gzip.");
  }
  const out = new Uint8Array(total);
  let at = 0;
  for (const c of chunks) { out.set(c, at); at += c.length; }
  return out;
}

const text = (block: Uint8Array, from: number, to: number) => {
  const slice = block.subarray(from, to);
  const end = slice.indexOf(0);
  return dec.decode(end === -1 ? slice : slice.subarray(0, end));
};

function octal(block: Uint8Array, from: number, to: number): number {
  const s = text(block, from, to).trim();
  if (s === "") return 0;
  if (!/^[0-7]+$/.test(s)) throw invalid("A tar header has a field that is not octal.");
  return parseInt(s, 8);
}

function checksumOk(block: Uint8Array): boolean {
  let sum = 0;
  for (let i = 0; i < 512; i++) sum += i >= 148 && i < 156 ? 32 : block[i];
  return sum === octal(block, 148, 156);
}

export async function readArchive(gz: Uint8Array): Promise<{ files: ArchiveFile[]; tar: Uint8Array }> {
  if (gz.length > MAX_COMPRESSED) throw new Refusal(413, "archive_too_large", "The compressed body is over 10 MB.");
  const tar = await gunzip(gz);
  if (tar.length === 0 || tar.length % 512 !== 0) throw invalid("The body is not a tar stream.");

  const files: ArchiveFile[] = [];
  const seen = new Set<string>();
  let at = 0;
  while (at + 512 <= tar.length) {
    const block = tar.subarray(at, at + 512);
    if (block.every((b) => b === 0)) break;
    if (!checksumOk(block)) throw invalid("A tar header checksum does not match.");
    if (text(block, 257, 262) !== "ustar") throw invalid("Only ustar archives are accepted.");

    const size = octal(block, 124, 136);
    const typeflag = String.fromCharCode(block[156] || 48);
    const prefix = text(block, 345, 500);
    const path = (prefix ? prefix + "/" : "") + text(block, 0, 100);
    const dataStart = at + 512;
    at = dataStart + Math.ceil(size / 512) * 512;
    if (dataStart + size > tar.length) throw invalid("A tar entry runs past the end of the archive.");

    if ("xgLK".includes(typeflag)) throw invalid("Extended tar headers are not accepted.");
    if (typeflag === "5") continue;
    if (typeflag !== "0") {
      throw new Refusal(400, "path_forbidden", `Only files and directories are accepted: ${JSON.stringify(path)}`, { path });
    }
    const bad = checkPath(path);
    if (bad) throw bad;
    if (seen.has(path)) throw invalid(`Duplicate entry ${JSON.stringify(path)}.`);
    seen.add(path);

    const mode = octal(block, 100, 108) & 0o111 ? 0o755 : 0o644;
    files.push({ path, mode, data: tar.slice(dataStart, dataStart + size) });
  }
  if (files.length === 0) throw invalid("The archive has no files.");
  files.sort((a, b) => (a.path < b.path ? -1 : a.path > b.path ? 1 : 0));
  return { files, tar };
}
```

`readArchive` enforces, in order: compressed size, gzip, tar structure, entry types, path rules. Environment files and fragments are checked by the pipeline in Task 4, after every entry has passed the structural rules, so that the order in `spec/archive.md` holds across the whole archive and not per entry. The `env-file` vectors therefore expect `env_file_present` from the pipeline, not from `readArchive`.

That makes two hostile vectors (`env-file`, `env-suffix`) fail the vector test above as written. Change that test to run hostile cases through a helper that applies the same two rules after `readArchive`:

```ts
async function readAndCheck(gz: Uint8Array) {
  const out = await readArchive(gz);
  const env = out.files.find((f) => isEnvFile(f.path));
  if (env) throw new Refusal(400, "env_file_present", "env", { path: env.path });
  return out;
}
```

and export the real version of that helper from `src/publish/archive.ts` as `checkContents(files: ArchiveFile[]): void`, which throws `env_file_present` for the first env file by path order and then `path_forbidden` for the first unknown fragment. Use `checkContents` in both the test and the pipeline. Do not duplicate the rule in the test.

Run: `bun run test` — Expected: PASS, including every contract archive vector. A vector that disagrees is a contract defect: stop and report.

Commit: "Archive reader verified against the contract vectors".

---

### Task 4: Publish

**Files:**
- Create: `src/params.ts`, `src/publish/pipeline.ts`, `src/routes/publish.ts`, `test/publish.test.ts`
- Modify: `src/index.ts`

**Interfaces:**
- Consumes: `readArchive`, `checkContents`, `sha256`, `harnessesOf`, `validate("package-manifest", …)`, `isMember`, `packageVersions`, `insertVersion`, `archiveKey`, `putArchive`, `Refusal`.
- Produces:

```ts
// src/params.ts
export type Target = { scope: string; name: string };
export function parseScope(raw: string): string;                       // throws bad_request
export function parseTarget(scope: string, name: string): Target;      // throws bad_request
export function parseVersion(raw: string): string;                     // throws bad_request

// src/publish/pipeline.ts
export type PublishResult = components["schemas"]["PublishResult"];
export type PublishRequest = { declaredLength: number; readBody: () => Promise<Uint8Array> };
export async function publish(env: Env, identity: Identity, target: Target & { version: string }, request: PublishRequest, now: Date): Promise<PublishResult>;
```

The pipeline receives a thunk for the body, not the bytes, because scope membership must be decided before the body is read (`contract/spec/archive.md`: "checked before the body is read").

- [ ] **Step 1: Failing tests**

`test/publish.test.ts`:

```ts
import { env } from "cloudflare:test";
import { beforeAll, expect, test } from "vitest";
import { getVersion } from "../src/store/db";
import { authed, call, person, seedMember, service } from "./helpers";
import { archiveOf, type Entry } from "./tar";

const SCOPE = "@pub";
let rick: string;

beforeAll(async () => {
  await seedMember(SCOPE, "rick@ourorg.example", "email");
  await seedMember(SCOPE, "ci-publisher", "service");
  rick = await person("rick@ourorg.example");
});

const manifest = (name: string, version: string, extra: object = {}) =>
  JSON.stringify({ name, version, description: "d", keywords: [], owners: ["rick@ourorg.example"], ...extra });

const pkg = (name: string, version: string, more: Entry[] = []): Entry[] => [
  { name: "hpm.json", data: manifest(`${SCOPE}/${name}`, version) },
  { name: "README.md", data: `# ${name}\n` },
  { name: "CHANGELOG.md", data: `# Changelog\n\n## [${version}] - 2026-09-19\n- Added: x\n` },
  { name: `claude/skills/${name}/SKILL.md`, data: "---\nname: x\ndescription: y\n---\n" },
  ...more,
];

const put = async (name: string, version: string, entries: Entry[], token = rick) =>
  call(`/v1/pkg/${SCOPE}/${name}/${version}`, authed(token, {
    method: "PUT", headers: { "content-type": "application/gzip" }, body: await archiveOf(entries),
  }));

const code = async (res: Response) => ((await res.json()) as { code: string }).code;

test("a member publishes, and the row and the archive exist", async () => {
  const res = await put("ok", "1.0.0", pkg("ok", "1.0.0"));
  expect(res.status).toBe(201);
  const body = (await res.json()) as any;
  expect(body).toMatchObject({ name: `${SCOPE}/ok`, version: "1.0.0", publishedBy: "rick@ourorg.example" });
  expect(body.integrity).toMatch(/^sha256-[0-9a-f]{64}$/);
  expect(body.size).toBeGreaterThan(0);
  const row = await getVersion(env.DB, SCOPE, "ok", "1.0.0");
  expect(row?.harnesses).toEqual(["claude-code"]);
  expect(row?.readme).toBe("# ok\n");
  const key = `archives/${SCOPE}/ok/1.0.0/${body.integrity.slice(7)}.tgz`;
  expect(await env.ARCHIVES.head(key)).not.toBeNull();
});

test("a service identity that is a member publishes under its own name", async () => {
  const res = await put("svc", "1.0.0", pkg("svc", "1.0.0"), await service("ci-publisher"));
  expect(res.status).toBe(201);
  expect(((await res.json()) as any).publishedBy).toBe("ci-publisher");
});

test("a non-member is refused before the body is read", async () => {
  const stranger = await person("stranger@ourorg.example");
  const res = await call(`/v1/pkg/${SCOPE}/x/1.0.0`, authed(stranger, { method: "PUT", body: "this is not even gzip" }));
  expect(res.status).toBe(403);
  expect(await code(res)).toBe("scope_forbidden");
});

test("refusals, one per code, with the contract's status", async () => {
  const cases: [string, number, string, () => Promise<Response>][] = [
    ["malformed version in the URL", 400, "bad_request", () => put("x", "1.0", pkg("x", "1.0"))],
    ["malformed name in the URL", 400, "bad_request", () => put("Bad_Name", "1.0.0", pkg("x", "1.0.0"))],
    ["not gzip", 400, "archive_invalid", () =>
      call(`/v1/pkg/${SCOPE}/x/1.0.0`, authed(rick, { method: "PUT", body: "plain text" }))],
    ["a symlink", 400, "path_forbidden", () => put("x", "1.0.0", pkg("x", "1.0.0", [{ name: "l", typeflag: "2", linkname: "hpm.json" }]))],
    ["a parent path", 400, "path_forbidden", () => put("x", "1.0.0", pkg("x", "1.0.0", [{ name: "claude/../../e.md", data: "x" }]))],
    ["an env file", 400, "env_file_present", () => put("x", "1.0.0", pkg("x", "1.0.0", [{ name: "claude/.env", data: "A=1" }]))],
    ["an unknown fragment", 400, "path_forbidden", () => put("x", "1.0.0", pkg("x", "1.0.0", [{ name: "claude/odd.merge.json", data: "{}" }]))],
    ["no manifest", 400, "manifest_invalid", () => put("x", "1.0.0", pkg("x", "1.0.0").slice(1))],
    ["a manifest that is not JSON", 400, "manifest_invalid", () => put("x", "1.0.0", [{ name: "hpm.json", data: "{" }, ...pkg("x", "1.0.0").slice(1)])],
    ["a manifest that fails the schema", 400, "manifest_invalid", () =>
      put("x", "1.0.0", [{ name: "hpm.json", data: JSON.stringify({ name: `${SCOPE}/x`, version: "1.0.0" }) }, ...pkg("x", "1.0.0").slice(1)])],
    ["a name that differs from the URL", 400, "name_mismatch", () => put("x", "1.0.0", pkg("other", "1.0.0"))],
    ["a version that differs from the URL", 400, "name_mismatch", () => put("x", "1.0.0", pkg("x", "1.0.1"))],
    ["no README", 400, "readme_missing", () => put("x", "1.0.0", pkg("x", "1.0.0").filter((e) => e.name !== "README.md"))],
  ];
  for (const [label, status, want, run] of cases) {
    const res = await run();
    expect(res.status, label).toBe(status);
    expect(await code(res), label).toBe(want);
  }
});

test("the first failure wins: an env file beats a bad manifest", async () => {
  const res = await put("x", "1.0.0", [{ name: "hpm.json", data: "{" }, { name: ".env", data: "A=1" }]);
  expect(await code(res)).toBe("env_file_present");
});

test("an equal version is version_exists and a lower one is version_not_higher", async () => {
  expect((await put("ver", "1.2.0", pkg("ver", "1.2.0"))).status).toBe(201);
  const same = await put("ver", "1.2.0", pkg("ver", "1.2.0", [{ name: "claude/extra.md", data: "different bytes" }]));
  expect(same.status).toBe(409);
  expect(await code(same)).toBe("version_exists");
  const lower = await put("ver", "1.1.9", pkg("ver", "1.1.9"));
  expect(lower.status).toBe(409);
  expect(await code(lower)).toBe("version_not_higher");
  expect((await put("ver", "1.2.1-next.1", pkg("ver", "1.2.1-next.1"))).status).toBe(201);
  const belowPrerelease = await put("ver", "1.2.1-next.0", pkg("ver", "1.2.1-next.0"));
  expect(await code(belowPrerelease)).toBe("version_not_higher");
});

test("two racing publishes of one version: exactly one wins, and its row points at its own bytes", async () => {
  const a = pkg("race", "1.0.0", [{ name: "claude/a.md", data: "from a" }]);
  const b = pkg("race", "1.0.0", [{ name: "claude/b.md", data: "from b" }]);
  const results = await Promise.all([put("race", "1.0.0", a), put("race", "1.0.0", b)]);
  expect(results.map((r) => r.status).sort()).toEqual([201, 409]);
  const winner = (await results.find((r) => r.status === 201)!.json()) as any;
  const row = await getVersion(env.DB, SCOPE, "race", "1.0.0");
  expect(row?.integrity).toBe(winner.integrity);
  const obj = await env.ARCHIVES.get(`archives/${SCOPE}/race/1.0.0/${row!.integrity.slice(7)}.tgz`);
  const { sha256 } = await import("../src/publish/archive");
  expect(await sha256(new Uint8Array(await obj!.arrayBuffer()))).toBe(row!.integrity);
});

test("a body over the limit is refused by its declared length", async () => {
  const res = await call(`/v1/pkg/${SCOPE}/big/1.0.0`, authed(rick, {
    method: "PUT", headers: { "content-length": String(10 * 1024 * 1024 + 1) }, body: "x",
  }));
  expect(res.status).toBe(413);
  expect(await code(res)).toBe("archive_too_large");
});
```

If the test runtime overrides a hand-set `content-length`, replace the last test's request with a real body of `MAX_COMPRESSED + 1` zero bytes and keep the same assertions.

Run: `bun run test test/publish.test.ts` — Expected: FAIL, 404 where 201 is expected.

- [ ] **Step 2: Implement parameters**

`src/params.ts`. The patterns are copied from the contract; Task 5's read routes use the same functions.

```ts
import { Refusal } from "./errors";

const SCOPE = /^@[a-z0-9][a-z0-9-]*$/;
const NAME = /^[a-z0-9][a-z0-9-]*$/;
const VERSION =
  /^(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)(-(0|[1-9]\d*|\d*[A-Za-z-][0-9A-Za-z-]*)(\.(0|[1-9]\d*|\d*[A-Za-z-][0-9A-Za-z-]*))*)?$/;

const bad = (what: string, value: string) =>
  new Refusal(400, "bad_request", `${what} is malformed: ${JSON.stringify(value)}`, { [what]: value });

export type Target = { scope: string; name: string };

export function parseScope(raw: string): string {
  if (!SCOPE.test(raw)) throw bad("scope", raw);
  return raw;
}
export function parseTarget(scope: string, name: string): Target {
  if (!NAME.test(name)) throw bad("name", name);
  return { scope: parseScope(scope), name };
}
export function parseVersion(raw: string): string {
  if (!VERSION.test(raw)) throw bad("version", raw);
  return raw;
}
```

Add a unit test in `test/publish.test.ts` that reads the version pattern out of `contract/schemas/package-manifest.json` (`$defs.version.pattern`) and asserts `VERSION.source` equals it. Export `VERSION` for that purpose. A copied pattern with no test is how drift starts.

- [ ] **Step 3: Implement the pipeline**

`src/publish/pipeline.ts`:

```ts
import semver from "semver";
import type { Identity } from "../auth/identity";
import type { Env } from "../env";
import { Refusal } from "../errors";
import type { components } from "../generated/api";
import type { Target } from "../params";
import { archiveKey, putArchive } from "../store/archives";
import { insertVersion, isMember, packageVersions } from "../store/db";
import { validate } from "../validate";
import { checkContents, MAX_COMPRESSED, readArchive, sha256 } from "./archive";
import { harnessesOf } from "./harnesses";

export type PublishResult = components["schemas"]["PublishResult"];
type PackageManifest = components["schemas"]["PackageManifest"];

const dec = new TextDecoder("utf-8", { fatal: true });

/** The ordered checks of contract/spec/archive.md, then the two writes. */
export type PublishRequest = { declaredLength: number; readBody: () => Promise<Uint8Array> };

export async function publish(
  env: Env, identity: Identity, target: Target & { version: string }, request: PublishRequest, now: Date,
): Promise<PublishResult> {
  const fullName = `${target.scope}/${target.name}`;

  // Identity was verified by the middleware. Scope membership comes before the body is looked at.
  if (!(await isMember(env.DB, target.scope, identity))) {
    throw new Refusal(403, "scope_forbidden", `${identity.id} may not publish to ${target.scope}.`, { scope: target.scope });
  }

  // archive_too_large by declared length, before buffering anything
  if (request.declaredLength > MAX_COMPRESSED) {
    throw new Refusal(413, "archive_too_large", "The compressed body is over 10 MB.");
  }
  const body = await request.readBody();

  // archive_too_large by real length, archive_invalid, path_forbidden (types and paths)
  const { files } = await readArchive(body);
  // env_file_present, path_forbidden (unknown fragment)
  checkContents(files);

  // manifest_invalid
  const manifestFile = files.find((f) => f.path === "hpm.json");
  if (!manifestFile) throw new Refusal(400, "manifest_invalid", "The archive has no hpm.json at its top level.");
  let manifest: PackageManifest;
  try {
    manifest = JSON.parse(dec.decode(manifestFile.data));
  } catch {
    throw new Refusal(400, "manifest_invalid", "hpm.json is not valid JSON.");
  }
  const checked = validate("package-manifest", manifest);
  if (!checked.valid) throw new Refusal(400, "manifest_invalid", "hpm.json fails the package manifest schema.", { errors: checked.errors });

  // name_mismatch
  if (manifest.name !== fullName || manifest.version !== target.version) {
    throw new Refusal(400, "name_mismatch", `hpm.json says ${manifest.name}@${manifest.version}, the URL says ${fullName}@${target.version}.`);
  }

  // readme_missing
  const readme = files.find((f) => f.path === "README.md");
  if (!readme) throw new Refusal(400, "readme_missing", "The archive has no README.md at its top level.");

  // M3: changelog_missing goes here.

  // version_exists, then version_not_higher
  const published = (await packageVersions(env.DB, target.scope, target.name)).map((v) => v.version);
  if (published.includes(target.version)) {
    throw new Refusal(409, "version_exists", `${fullName}@${target.version} is already published.`);
  }
  const highest = published.sort(semver.rcompare)[0];
  if (highest && semver.lt(target.version, highest)) {
    throw new Refusal(409, "version_not_higher", `${target.version} is lower than the highest published version, ${highest}.`, { highest });
  }

  // M3: trigger_line_missing and scaffold_marker_present go here.

  const integrity = await sha256(body);
  await putArchive(env.ARCHIVES, archiveKey(target.scope, target.name, target.version, integrity), body);
  const publishedAt = now.toISOString();
  const outcome = await insertVersion(env.DB, {
    scope: target.scope, name: target.name, version: target.version, integrity, size: body.length,
    manifest, harnesses: harnessesOf(files.map((f) => f.path)), readme: dec.decode(readme.data),
    publishedBy: identity.id, publishedAt,
  });
  if (outcome === "exists") {
    throw new Refusal(409, "version_exists", `${fullName}@${target.version} is already published.`);
  }
  return { name: fullName, version: target.version, integrity, size: body.length, publishedAt, publishedBy: identity.id };
}
```

A README that is not valid UTF-8 makes `dec.decode` throw a `TypeError`, which would surface as `internal_error`. Catch it and refuse with `readme_missing` and the message "README.md is not valid UTF-8." Add that case to the refusal table in the test.

- [ ] **Step 4: Route and wiring**

`src/routes/publish.ts`:

```ts
import { Hono } from "hono";
import type { AppEnv } from "../env";
import { parseTarget, parseVersion } from "../params";
import { publish } from "../publish/pipeline";

export const publishRoutes = new Hono<AppEnv>();

publishRoutes.put("/v1/pkg/:scope/:name/:version", async (c) => {
  const target = { ...parseTarget(c.req.param("scope"), c.req.param("name")), version: parseVersion(c.req.param("version")) };
  const request = {
    declaredLength: Number(c.req.header("content-length") ?? "0") || 0,
    readBody: async () => new Uint8Array(await c.req.arrayBuffer()),
  };
  return c.json(await publish(c.env, c.get("identity"), target, request, new Date()), 201);
});
```

The route holds no checks of its own beyond parsing the URL. The pipeline owns the order.

`src/index.ts`: `app.route("/", publishRoutes);` before `notFound`.

Run: `bun run test` — Expected: PASS. Then `bun run check` and the dry-run.
Commit: "Publish with the core checks, in the contract's order".

---

### Task 5: Read routes

**Files:**
- Create: `src/routes/read.ts`, `test/read.test.ts`
- Modify: `src/index.ts`

**Interfaces:**
- Consumes: `listVersions`, `packageVersions`, `getVersion`, `archiveKey`, `getArchive`, `parseScope`, `parseTarget`, `parseVersion`.
- Produces: `readRoutes`, a `Hono<AppEnv>` with `GET /v1/index`, `GET /v1/pkg/:scope/:name`, `GET /v1/archive/:scope/:name/:version`.

- [ ] **Step 1: Failing tests**

`test/read.test.ts`. It publishes through the real route so that the two halves are tested together.

```ts
import { beforeAll, expect, test } from "vitest";
import indexSchema from "../contract/schemas/index.json";
import { Validator, type Schema } from "@cfworker/json-schema";
import { sha256 } from "../src/publish/archive";
import { authed, call, person, seedMember } from "./helpers";
import { archiveOf, type Entry } from "./tar";

const SCOPE = "@read";
let reader: string;
let archives: Record<string, Uint8Array> = {};

const entries = (name: string, version: string, deps: object = {}, extra: Entry[] = []): Entry[] => [
  { name: "hpm.json", data: JSON.stringify({ name: `${SCOPE}/${name}`, version, description: `${name} ${version}`, keywords: ["k"], owners: ["o@x.test"], dependencies: deps }) },
  { name: "README.md", data: `# ${name} ${version}\n` },
  { name: `claude/skills/${name}/SKILL.md`, data: "x" },
  ...extra,
];

beforeAll(async () => {
  await seedMember(SCOPE, "pub@ourorg.example", "email");
  const pub = await person("pub@ourorg.example");
  reader = await person("anyone@ourorg.example"); // not a member of any scope
  // Published in ascending order, because the registry refuses a lower version.
  const ordered: [string, string, object, Entry[]][] = [
    ["alpha", "1.2.0-next.1", {}, []],
    ["alpha", "1.2.0", {}, []],
    ["alpha", "1.10.0", {}, []],
    ["beta", "0.1.0", { [`${SCOPE}/alpha`]: "^1.2" }, [{ name: "kiro/steering/beta.md", data: "x" }]],
  ];
  for (const [name, version, deps, extra] of ordered) {
    const body = await archiveOf(entries(name, version, deps, extra));
    archives[`${name}@${version}`] = body;
    const res = await call(`/v1/pkg/${SCOPE}/${name}/${version}`, authed(pub, { method: "PUT", body }));
    if (res.status !== 201) throw new Error(`seed publish failed: ${res.status} ${await res.text()}`);
  }
});

const get = (path: string, headers: Record<string, string> = {}) => call(path, authed(reader, { headers }));

test("the index validates against the contract schema and orders versions by semver", async () => {
  const res = await get(`/v1/index?scope=${encodeURIComponent(SCOPE)}`);
  expect(res.status).toBe(200);
  const body = (await res.json()) as any;
  const result = new Validator(indexSchema as Schema, "2020-12", false).validate(body);
  expect(result.errors).toEqual([]);
  expect(body.packages.map((p: any) => p.name)).toEqual([`${SCOPE}/alpha`, `${SCOPE}/beta`]);
  const alpha = body.packages[0];
  expect(alpha.versions.map((v: any) => v.version)).toEqual(["1.2.0-next.1", "1.2.0", "1.10.0"]);
  expect(alpha.description).toBe("alpha 1.10.0");
  const beta = body.packages[1].versions[0];
  expect(beta.dependencies).toEqual({ [`${SCOPE}/alpha`]: "^1.2" });
  expect(beta.harnesses).toEqual(["claude-code", "kiro"]);
  expect(beta.integrity).toBe(await sha256(archives["beta@0.1.0"]));
  expect(beta.publishedBy).toBe("pub@ourorg.example");
});

test("any authenticated identity may read; the scope filter is optional and validated", async () => {
  expect((await get("/v1/index")).status).toBe(200);
  const empty = (await (await get("/v1/index?scope=@nothing-here")).json()) as any;
  expect(empty.packages).toEqual([]);
  const bad = await get("/v1/index?scope=NoAtSign");
  expect(bad.status).toBe(400);
  expect(((await bad.json()) as any).code).toBe("bad_request");
});

test("the index carries an ETag and honours If-None-Match", async () => {
  const first = await get(`/v1/index?scope=${SCOPE}`);
  const etag = first.headers.get("etag")!;
  expect(etag).toMatch(/^"[0-9a-f]{64}"$/);
  const again = await get(`/v1/index?scope=${SCOPE}`, { "if-none-match": etag });
  expect(again.status).toBe(304);
  expect(await again.text()).toBe("");
  expect((await get(`/v1/index?scope=${SCOPE}`, { "if-none-match": '"other"' })).status).toBe(200);
});

test("package info is the latest version's manifest and README", async () => {
  const res = await get(`/v1/pkg/${SCOPE}/alpha`);
  expect(res.status).toBe(200);
  const body = (await res.json()) as any;
  expect(body).toMatchObject({ name: `${SCOPE}/alpha`, latest: "1.10.0", readme: "# alpha 1.10.0\n" });
  expect(body.manifest.version).toBe("1.10.0");
  expect(body.versions).toEqual(["1.2.0-next.1", "1.2.0", "1.10.0"]);
});

test("the archive route returns the exact bytes that were published", async () => {
  const res = await get(`/v1/archive/${SCOPE}/alpha/1.2.0`);
  expect(res.status).toBe(200);
  expect(res.headers.get("content-type")).toBe("application/gzip");
  expect(new Uint8Array(await res.arrayBuffer())).toEqual(archives["alpha@1.2.0"]);
});

test("not found and bad request", async () => {
  for (const path of [`/v1/pkg/${SCOPE}/nope`, `/v1/archive/${SCOPE}/alpha/9.9.9`, `/v1/archive/${SCOPE}/nope/1.0.0`]) {
    const res = await get(path);
    expect(res.status, path).toBe(404);
    expect(((await res.json()) as any).code).toBe("not_found");
  }
  for (const path of [`/v1/pkg/${SCOPE}/Bad_Name`, `/v1/archive/${SCOPE}/alpha/1.0`, "/v1/pkg/noat/alpha"]) {
    const res = await get(path);
    expect(res.status, path).toBe(400);
    expect(((await res.json()) as any).code).toBe("bad_request");
  }
});
```

"Latest" is the highest version by semver precedence, so a package whose only versions are prereleases reports the highest prerelease.

Run: `bun run test test/read.test.ts` — Expected: FAIL with 404s.

- [ ] **Step 2: Implement**

`src/routes/read.ts`:

```ts
import { Hono } from "hono";
import semver from "semver";
import type { AppEnv } from "../env";
import { Refusal } from "../errors";
import type { components } from "../generated/api";
import { parseScope, parseTarget, parseVersion } from "../params";
import { sha256 } from "../publish/archive";
import { archiveKey, getArchive } from "../store/archives";
import { getVersion, listVersions, packageVersions, type VersionRow } from "../store/db";

type Index = components["schemas"]["Index"];
type PackageInfo = components["schemas"]["PackageInfo"];

export const readRoutes = new Hono<AppEnv>();

const ascending = (a: VersionRow, b: VersionRow) => semver.compare(a.version, b.version);

export function buildIndex(rows: VersionRow[]): Index {
  const byName = new Map<string, VersionRow[]>();
  for (const r of rows) {
    const key = `${r.scope}/${r.name}`;
    byName.set(key, [...(byName.get(key) ?? []), r]);
  }
  return {
    packages: [...byName.keys()].sort().map((name) => {
      const versions = byName.get(name)!.sort(ascending);
      const latest = versions[versions.length - 1].manifest;
      return {
        name,
        description: latest.description,
        keywords: latest.keywords,
        versions: versions.map((v) => ({
          version: v.version, integrity: v.integrity, size: v.size,
          dependencies: v.manifest.dependencies ?? {}, harnesses: v.harnesses,
          publishedAt: v.publishedAt, publishedBy: v.publishedBy,
        })),
      };
    }),
  };
}

readRoutes.get("/v1/index", async (c) => {
  const rawScope = c.req.query("scope");
  const scope = rawScope === undefined ? undefined : parseScope(rawScope);
  const body = JSON.stringify(buildIndex(await listVersions(c.env.DB, scope)));
  const etag = `"${(await sha256(new TextEncoder().encode(body))).slice(7)}"`;
  if (c.req.header("if-none-match") === etag) return c.body(null, 304, { etag });
  return c.body(body, 200, { "content-type": "application/json; charset=utf-8", etag });
});

readRoutes.get("/v1/pkg/:scope/:name", async (c) => {
  const t = parseTarget(c.req.param("scope"), c.req.param("name"));
  const versions = (await packageVersions(c.env.DB, t.scope, t.name)).sort(ascending);
  if (versions.length === 0) throw new Refusal(404, "not_found", `No such package: ${t.scope}/${t.name}.`);
  const latest = versions[versions.length - 1];
  const info: PackageInfo = {
    name: `${t.scope}/${t.name}`, latest: latest.version, manifest: latest.manifest,
    readme: latest.readme, versions: versions.map((v) => v.version),
  };
  return c.json(info);
});

readRoutes.get("/v1/archive/:scope/:name/:version", async (c) => {
  const t = parseTarget(c.req.param("scope"), c.req.param("name"));
  const version = parseVersion(c.req.param("version"));
  const row = await getVersion(c.env.DB, t.scope, t.name, version);
  if (!row) throw new Refusal(404, "not_found", `No such version: ${t.scope}/${t.name}@${version}.`);
  const obj = await getArchive(c.env.ARCHIVES, archiveKey(t.scope, t.name, version, row.integrity));
  if (!obj) throw new Error(`Archive object missing for ${t.scope}/${t.name}@${version}`);
  return c.body(obj.body, 200, { "content-type": "application/gzip", "content-length": String(obj.size) });
});
```

`src/index.ts`: `app.route("/", readRoutes);` beside the publish routes.

If the generated `Index` type is stricter than what `buildIndex` returns (for example `harnesses` as a literal union array), fix the types at the source (`HarnessId` in `src/store/db.ts` should then be derived from the generated type) rather than casting.

Run: `bun run test` — Expected: PASS, all files.
Commit: "Read routes: index with ETag, package info, archive".

---

### Task 6: Local development, docs and the pull request

**Files:**
- Create: `scripts/dev-seed.sh`
- Modify: `README.md`, `package.json`, `.github/workflows/ci.yml`

**Interfaces:**
- Produces, for M1-C's end-to-end harness:
  - `bun run dev:migrate -- --persist-to <dir>` applies migrations to a local D1.
  - `scripts/dev-seed.sh <persist dir> <scope> <principal> <email|group|service>` adds one scope member to that local D1.
  - `bunx wrangler dev --port <p> --persist-to <dir> --var AUTH_JWKS_URL:<url> --var AUTH_ISSUER:<iss> --var AUTH_AUDIENCE:<aud>` starts the Worker against it.

- [ ] **Step 1: Scripts**

`package.json` scripts gain:

```json
"dev:migrate": "wrangler d1 migrations apply DB --local"
```

`scripts/dev-seed.sh`:

```bash
#!/usr/bin/env bash
# Add one scope member to a LOCAL D1. Never touches a remote database.
set -euo pipefail
DIR="${1:?usage: dev-seed.sh <persist dir> <scope> <principal> <email|group|service>}"
SCOPE="${2:?scope, with its @}"; PRINCIPAL="${3:?principal}"; KIND="${4:?email|group|service}"
case "$KIND" in email|group|service) ;; *) echo "kind must be email, group or service" >&2; exit 1;; esac
case "$SCOPE$PRINCIPAL" in *"'"*) echo "quotes are not allowed" >&2; exit 1;; esac
cd "$(dirname "$0")/.."
bunx wrangler d1 execute DB --local --persist-to "$DIR" \
  --command "INSERT OR IGNORE INTO scope_members (scope, principal, kind) VALUES ('$SCOPE', '$PRINCIPAL', '$KIND');"
```

`chmod +x`. Prove it end to end in a scratch directory and put the transcript in your report: migrate, seed `@smoke ci-publisher service`, then `wrangler d1 execute DB --local --persist-to <dir> --command "SELECT * FROM scope_members"` shows the row. `wrangler d1 migrations apply` may prompt for confirmation; pass whatever non-interactive flag the installed wrangler offers (check `--help`) and record it in the `dev:migrate` script.

- [ ] **Step 2: README**

Add sections: "Configuration" (the `AUTH_*` variables, that exactly one of `AUTH_JWKS_URL` and `AUTH_JWKS_JSON` is set, the Access values: issuer `https://<team>.cloudflareaccess.com`, JWKS `https://<team>.cloudflareaccess.com/cdn-cgi/access/certs`, audience = the Access application's AUD tag, header `Cf-Access-Jwt-Assertion`); "Running locally" (the three commands above); "What M1 checks on publish" (the ordered list from Global Constraints, and what arrives in M3); "Storage" (the three tables; the R2 key format and why it carries the hash).

- [ ] **Step 3: CI**

No new steps are needed: `bun run check` already runs every test. Confirm the workflow still has no secrets and no deploy.

- [ ] **Step 4: ⚠ Push and open the PR**

```bash
bun run check && bunx wrangler deploy --dry-run --outdir dist
git push -u origin m1b
gh pr create --title "M1-B: registry walking skeleton" --body "Four /v1 routes behind the identity middleware, publish with the core checks in the contract's order, D1 and R2 behind src/store, archive reader verified against the contract vectors. Plan: meta repo docs/superpowers/plans/2026-09-19-m1b-registry-skeleton.md.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
gh pr checks --watch
```

---

## Exit check for M1-B

- [ ] `bun run check` passes; the wrangler dry-run bundles; CI is green on the PR with no secrets.
- [ ] Every code in `contract/spec/errors.md` that M1 can produce has at least one test asserting both the code and the HTTP status: `unauthenticated`, `scope_forbidden`, `not_found`, `bad_request`, `archive_too_large`, `archive_invalid`, `path_forbidden`, `env_file_present`, `manifest_invalid`, `name_mismatch`, `readme_missing`, `version_exists`, `version_not_higher`, `internal_error`.
- [ ] Every contract archive vector passes inside `workerd`.
- [ ] The index response validates against `contract/schemas/index.json`.
- [ ] `grep -rn "env\.DB\|env\.ARCHIVES" src | grep -v "^src/store/"` prints only lines that pass the binding into a `src/store` function.
- [ ] No route is reachable without a verified token, including unknown routes.
- [ ] The local development commands in the README work from a clean checkout, and the transcript is in the report.
- [ ] Amendments for the design spec are listed in the report: the R2 key carries the hash; the uncompressed limit; `AUTH_JWKS_JSON` as an alternative to a JWKS URL.
