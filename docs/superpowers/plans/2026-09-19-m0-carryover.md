# M0 carry-over

Small items found by the M0 reviews and deliberately not fixed in M0. None blocks M1. Fold them into the first contract release M1 makes (v0.3.0) and the first consumer commits, and tick them off here.

## Contract (next release)

- [ ] Harness id enums need `"type": "string"`. Without it oapi-codegen types `harnesses` as `[]interface{}`. (Found while planning M1; M1-A Task 1 does it.)

- [ ] `openapi.yaml`: declare `400` on `getPackage` and `getArchive` (they carry the patterned path parameters `spec/errors.md` says produce `bad_request`), and `403` on the read operations if `scope_forbidden` can apply to reads; otherwise narrow the wording in `spec/errors.md`. The two documents must agree on where each code can arise.
- [ ] `openapi.yaml`: `PackageInfo.versions` items need the version pattern, like `latest`.
- [ ] Add a consistency test for the four-id harness enum, which is inlined in five places across the schemas.
- [ ] `CHANGELOG.md` 0.2.0 says nine new invalid manifest vectors; there are ten. Correct it in the next section's notes rather than rewriting history.
- [ ] `spec/changelog.md`: the duplicate-section and sub-heading rules have no vector. Add them before M3, which is the first milestone to parse changelogs.
- [ ] `test/schemas.test.ts`: `test.each` titles are not interpolated by Bun, so a failing manifest vector is unnamed in CI. Use the tuple and `%s` form already used in `test/vectors.test.ts`.
- [ ] Canonical JSON vectors: add UTF-16 key ordering with supplementary-plane characters, and non-integer and exponent numbers (`0.1`, `1e21`). Needed by M4.
- [ ] `spec/archive.md`: state whether directory entries are required, gzip header determinism (mtime 0, no file name, OS byte), and what happens to a path longer than the ustar limit. Do this with the archive vectors in M1.
- [ ] `scripts/bundle.ts`: the `wrappers` map is hard-coded. It fails loudly if a third external schema is added without an entry, so this is a reminder, not a defect.
- [ ] `vectors/manifests/cases.json`: `reason` repeats the file name. Drop it or replace it with the expected error code.

## Both consumers

- [ ] `scripts/contract-pull.sh`: remove the hard-coded `Co-Authored-By: Claude` trailer from the commit it makes. People will run this script.
- [ ] `scripts/contract-pull.sh`: if `git read-tree` itself fails, print the recovery hint (`git reset --hard HEAD`) like the two explicit checks do.
- [ ] `scripts/check-contract.sh`: friendly messages when `contract/` is missing from `HEAD` and when `contract.lock` lacks `tag` or `tree`.
- [ ] Consider shipping `check-contract.sh` inside the contract repo so there is one copy.
- [ ] Decide on `CODEOWNERS` for `contract/` and `contract.lock`. The offline check cannot catch a commit that edits `contract/` and rewrites the lock to match; review is the guard.

## CLI

- [ ] `internal/manifest/validate.go`: wrap the `ReadFile` and `AddResource` errors with the kind, like the other two paths. Do this when M1 adds the real load path.
- [ ] Masterminds/semver disagrees with the contract's prerelease rule (`^1.0.0-next.1` picks `1.1.0-next.1`). The resolver in M3 needs a prerelease filter on top of it; the contract vectors already cover the case.
- [ ] The registry client must handle non-JSON error bodies from Access and the platform (M1).

## Registry

- [ ] `app.onError` answering `internal_error` with the contract shape, with the first route that can throw (M1).
- [ ] `wrangler.jsonc` `compatibility_date` is pinned to what the workerd bundled with `@cloudflare/vitest-pool-workers` 0.22 accepts. The M2 deploy script should render the production date.

## Docs

- [ ] `docs/prd.md` §9 routes table still says `GET /index` returns "harness folders present". It returns harness ids.
- [x] Meta `README.md` status line: update once the two M0 pull requests are merged.

## Known and accepted

- Contract commits `cecc3da` and `06e28ad` carry the co-author text on the subject line instead of as a trailer. They sit under the published tags, so they stay.
- The URL pattern `^https?://[!-~]+$` is deliberately loose. It is the price of three validators agreeing byte for byte.
- `go.mod` says `go 1.26.2`, which `go mod init` wrote. Compatible with the 1.26 floor.
