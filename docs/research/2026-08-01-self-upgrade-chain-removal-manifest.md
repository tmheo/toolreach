# Removal manifest for the self-upgrade and update-check chain

## Question

The map rules the self-upgrade and release-verification chain out of scope for toolreach v0.1.
"Remove it" is not yet a set of instructions.
This manifest names every file, workflow job, document, and glossary term that leaves, with the import graph checked rather than assumed, and names the surviving files the removal edits.

Source of record: the predecessor repository `workpulse` at `~/workspace/github/workpulse`, module `github.com/LGU-CTO/workpulse`, `BINARY_VERSION` 0.8.3.

## Method

The package graph came from `go list -f '{{.ImportPath}}|{{join .Imports ","}}' ./...`, which reports what the compiler actually resolves rather than what a grep of import blocks suggests.
File-level attribution came from `grep -rln 'workpulse/internal/<pkg>"' --include='*.go'` per package, so a package's importers are enumerated file by file, test files included.
Nothing below rests on a package name reading as if it belonged to the chain.

## The two charting claims, checked

**`internal/publication` stays. Confirmed, and the picture is wider than charting recorded.**
Its importers are `cmd/sync.go`, five files in `internal/storage` (`filesystem.go`, `publish.go`, `reclaim.go`, `synced_pages.go`, and their tests), and `tools/measure/main.go`.
Nothing in the chain imports it.
The name collides with "release publication" but the package is the atomic-file-publication primitive behind the data directory, and `tools/measure` is a third consumer charting did not name.

**`internal/github` leaves. Confirmed, and it is narrower than the name suggests.**
Its only importers are `cmd/upgrade.go`, `cmd/update_check_lifecycle.go`, and `cmd/update_check_lifecycle_test.go`.
It is 333 production lines implementing one thing, `NewReleaseSource`, a GitHub Releases API adapter satisfying `binaryrelease.ReleaseSource`.
It is not a general GitHub client and leaves nothing behind.

## Packages that leave

Every one of these is imported only by another member of this set, by `cmd/upgrade.go` / `cmd/update_check_lifecycle.go`, or by `tools/releasemanifest`.
The set is closed: no surviving package imports any of them.

| Package | Go files | Production LOC | Test LOC | Only reached from |
|---|---|---|---|---|
| `internal/binaryupgrade` | 18 | 1,525 | 2,499 | `cmd/upgrade.go`, `tools/releasemanifest` |
| `internal/releasemanifest` | 6 + `testdata/` | 658 | 867 | `internal/binaryupgrade`, `tools/releasemanifest` |
| `internal/updatecheck` | 16 | 605 | 1,373 | `cmd/update_check_lifecycle.go` |
| `internal/binaryrelease` | 5 | 348 | 330 | the four above, `cmd/upgrade.go`, `cmd/update_check_lifecycle.go` |
| `internal/github` | 2 | 333 | 439 | `cmd/upgrade.go`, `cmd/update_check_lifecycle.go` |
| `internal/binaryinstallation` | 2 | 115 | 98 | the five above, `cmd/upgrade.go`, `cmd/update_check_lifecycle.go` |

Two non-Go assets leave with `internal/releasemanifest`: the embedded `trusted-root-public-good.json` (8 KB, pulled in with `//go:embed`) and `testdata/` (5 fixture files, 40 KB, including a captured `workpulse-v0.8.0-release.json` and its Sigstore bundle).

`tools/releasemanifest/main.go` (90 lines) leaves entirely.
It is the `generate` / `verify` / `identity` / `verify-signature` CLI that CI and the release workflow drive, and it imports `internal/binaryupgrade` and `internal/releasemanifest`.
The two sibling tools stay: `tools/measure` and `tools/scopeprobe` touch none of this.

## Command files that leave

| File | LOC | Note |
|---|---|---|
| `cmd/upgrade.go` | 226 | the `upgrade` command and all of its Korean-language outcome and failure rendering |
| `cmd/update_check_lifecycle.go` | 182 | `applicationExecution`, `executeCommand`, `beginUpdateAvailabilityCheck`, `inspectBinaryInstallation`, `invocationInstallation`, the notice renderer |
| `cmd/upgrade_adapter_test.go` | 302 | |
| `cmd/update_check_lifecycle_test.go` | 501 | |
| `cmd/binary_release_architecture_test.go` | 233 | six architecture tests policing the chain's own module boundaries |
| `cmd/upgrade_command_test.go` | 15 | |

Totals across packages, tools, and commands: **4,082 production lines and 6,657 test lines, 10,739 in all.**
The map's "~3.0K LOC" plus "~605 LOC" counted production only, and both figures reconcile exactly (2,979 and 605).
The test half is the part worth planning for: roughly 6.7K of the predecessor's ~38K test lines go with the chain, and none of it needs porting or renaming.

## Surviving files the removal edits

The coupling is astonishingly small.
`grep` for every identifier `cmd/update_check_lifecycle.go` exports into the rest of `cmd` finds exactly two live lines outside the leaving files, both in `cmd/root.go`:

- `cmd/root.go:17` — `PersistentPreRunE: beginUpdateAvailabilityCheck` on `rootCmd`. Delete the field.
- `cmd/root.go:23` — `return executeCommand(ctx, rootCmd, newApplicationExecution())` inside `Execute`. Becomes `rootCmd.ExecuteContext(ctx)`.

That second line is the one to think about rather than delete blindly.
`executeCommand` is also what threads the signal-cancellable context from `main.go` into every command through `ExecuteContextC`, so the replacement must keep passing `ctx`; only the update-check bookkeeping around it goes.

`internal/config/config.go` sheds the `update_check` key: the `UpdateCheckConfig` struct (lines 13-16), the `UpdateCheck` field on `Config` (line 24), and its default in `DefaultConfig` (line 71).
Its only reader is `cmd/update_check_lifecycle.go:47`.
Ticket [Name toolreach's public surface](https://github.com/tmheo/toolreach/issues/2) already reshapes this file for the `confluence:` to `atlassian:` rename, so the two edits should land together.

Nothing else in `cmd`, `internal`, or `main.go` refers to the chain.

## What `cmd/version.go` retains

`cmd/version.go` is 25 lines: three package-level vars (`version`, `commit`, `date`) injected by GoReleaser `ldflags`, and a `versionCmd` that prints them.
The file itself is untouched by the removal.

What changes is that `version` loses its second reader.
Today it feeds `binaryinstallation.Inspect(executablePath, version)` in `inspectBinaryInstallation()`, which classifies the build as a release version, the exact `dev` marker, or invalid metadata, and that classification gates both upgrade eligibility and the update notice.
Once the chain leaves, `version` is consumed only by its own `Printf`.
So `version.go` retains everything and loses nothing, but the `ldflags` trio drops from build identity to display string, and no code path any longer cares whether the string parses as semver.

One latent inconsistency, noted rather than fixed here: `rootCmd.Version` is never set, so `workpulse --version` does not exist, only `workpulse version`.
The Homebrew formula's `test` block in `.goreleaser.yaml` runs `#{bin}/workpulse --version`.
That test does not run in CI, which is why it has survived.
A public tap makes `brew test` a thing strangers run, so toolreach should either set `rootCmd.Version` or fix the formula test.

## CI jobs

**`.github/workflows/trusted-root.yml` leaves entirely** (one job, weekly cron).
It exists solely to diff `internal/releasemanifest/trusted-root-public-good.json` against Sigstore TUF and fails pointing at `docs/release-integrity.md`.
Both the file it guards and the doc it cites leave.

**`.github/workflows/ci.yml` loses two jobs of its five.**

- `release-contract` leaves whole. Every step serves the chain: it reads `BINARY_VERSION`, runs a GoReleaser snapshot, then runs `tools/releasemanifest generate` and `verify` against `dist/artifacts.json`. Note before deleting it that its GoReleaser dry run is currently the only thing proving `.goreleaser.yaml` parses and builds on a pull request. If toolreach wants that assurance without the manifest, it needs a bare `goreleaser release --snapshot` step, which is ticket [Distribution and release pipeline for a public repo](https://github.com/tmheo/toolreach/issues/7)'s call, not this manifest's.
- `test-binary-release-platforms` leaves whole. It is a Windows-only job whose two substantive steps are `go test -race ./internal/binaryupgrade ./internal/binaryrelease ./internal/releasemanifest ./internal/updatecheck` and a cross-compile of the `binaryupgrade` tests for `windows/arm64`. Its own header comment records that its macOS leg was already folded into `test`. When the four packages leave, only `go vet ./...` on Windows remains, which is a judgement call rather than a removal: keeping a Windows leg is cheap and the predecessor supports Windows, but the job as written has no other reason to exist.
- `test` and `deploy-coverage-badge` are untouched by the removal. The coverage threshold is 60% and the chain's packages are heavily tested, so deleting them moves the repository coverage number by an unknown amount in an unknown direction. Measure after the port rather than assuming the threshold still holds.

**`.github/workflows/release.yml` loses more than half its steps.**
Of eleven steps, six leave outright: "Verify release identity and main history" (`tools/releasemanifest identity` plus the main-ancestry check), "Prepare release-specific Upgrade Action", "Generate and verify Binary Release Manifest", "Install Cosign", "Sign and verify Binary Release Manifest", and "Verify Binary Release Manifest with the shipped verifier".
"Upload signed distribution contract" leaves.
"Verify draft assets and publish release" survives in spirit but must be rewritten: it currently derives the expected asset list from `dist/workpulse-release.json`.
The GoReleaser build step, the draft-publish, the Homebrew formula push, and the failure cleanup survive as the shape ticket [Distribution and release pipeline for a public repo](https://github.com/tmheo/toolreach/issues/7) starts from.
`id-token: write` at workflow level exists only for Sigstore keyless signing and should drop with it.

**`.goreleaser.yaml`** keeps its build, archive, checksum, and changelog blocks but sheds two constraints imposed by the upgrade extractor:

- `archives[].files: - none*`, whose inline comment states plainly that it is there because an installed `workpulse` refuses to extract an archive holding more than one executable. With the extractor gone, a public release can and should ship LICENSE and README inside the archives.
- `snapshot.version_template: "{{ .Env.WORKPULSE_RELEASE_CONTRACT_VERSION }}"`, which exists for the `release-contract` CI job alone.

`release.prerelease: auto` and `release.header` also lose their stated reasons (the header renders `RELEASE_UPGRADE_ACTION`, which the deleted step produced), though `prerelease: auto` is worth keeping on its own merits.

## `BINARY_VERSION`, the integrity doc, and the release-evidence practice

**`BINARY_VERSION`** (a single line, `0.8.3`) has four readers.
Two leave with the chain: `ci.yml`'s `release-contract` job and `release.yml`'s identity check, which is what makes the file authoritative today, since `tools/releasemanifest identity` refuses to release when the tag and the file disagree.
Two survive and are the reason the file cannot simply be deleted:

- `docs/releases/2026-06-28-post-v0.4.5-release-train.md` names it in a release checklist.
- **`hooks/session-start.sh`** reads it as the pinned version the Claude Code plugin installs. This is the finding least visible from the map. The plugin hook is a *second, independent* self-updating path: on session start it compares `workpulse version` output against `BINARY_VERSION` and, if they differ, downloads `workpulse_${VERSION}_${OS}_${ARCH}.tar.gz` from the GitHub release via `gh release download` or an authenticated `curl`, extracts it, and moves it over the installed binary. It shares none of the chain's code, none of its verification, and none of its Homebrew-ownership guard, and it will keep working after every file in this manifest is deleted. Whether toolreach ships a plugin hook that silently replaces the user's binary with no signature check is a decision the map does not yet hold. See "What this surfaces" below.

**`docs/release-integrity.md`** (174 lines) leaves entirely.
It documents only the manifest contract, the pinned workflow signing identity, the transparency-log disclosure accepted for a private repository, and the independent `cosign verify-blob` command.
Every sentence describes machinery in this manifest.
Its replacement in a public repository is a much shorter note about GitHub Releases checksums and the tap, which belongs to ticket [Distribution and release pipeline for a public repo](https://github.com/tmheo/toolreach/issues/7).

**`docs/releases/`** (9 dated files) is a different matter, and the manifest's recommendation is: **do not port it, and do not treat that as a loss.**
The directory is a per-release evidence record, one file per release train, mixing smoke-test results with checklists.
Three observations bear on the decision.
It is an undocumented practice: `AGENTS.md` and `CLAUDE.md` describe the issue tracker, triage labels, and domain docs, and say nothing about it.
Its files are historical records of releases of a private binary that toolreach will never have made.
And two of the nine (`2026-07-29-release-verification-dependency-release.md`, `2026-07-29-upgrade-path-repair-release.md`) are about the chain itself.
Since the map's Code-transfer decision is a fresh start with no history, these are history.
The practice can be restarted in toolreach on its first real release if it earns its place; carrying the predecessor's nine files across would import evidence for events that never happened in this repository.

**`.github/upgrade-actions/`** leaves.
It holds one file, `hierarchy-read-grant.md`, consumed only by the deleted "Prepare release-specific Upgrade Action" step.
It is not inert (see below).

## What the chain leaves in `CONTEXT.md`

`CONTEXT.md` is 388 lines and its `## Language` section runs to the end of the file.
The chain's terms occupy a contiguous, terminal block: **lines 283 to 388, eighteen terms**, from `Binary Release Availability` through `Release Availability Observation`.

`Binary Release Availability`, `Binary Release Channel`, `Binary Build Identity`, `Binary Distribution Contract`, `Verified Binary Release`, `Binary Release Manifest`, `Staged Binary Artifact`, `Binary Upgrade Lock`, `Binary Replacement Transaction`, `Binary Replacement Commit Point`, `Binary Upgrade Outcome`, `Binary Upgrade Failure`, `Binary Release Credential`, `Binary Upgrade`, `Binary Installation Ownership`, `Binary Installation`, `Update Availability Check`, `Release Availability Observation`.

Three facts make this a clean cut.
The block is contiguous and terminal, so removal is `sed '283,388d'` plus dropping the blank line, with no interleaving to unpick.
The term immediately before it, `Page Version` at line 279, belongs to the Confluence domain and no term in the block is referenced by any term outside it.
And two of the definitions are actively wrong for a public tool, not merely obsolete: `Verified Binary Release` names the trusted identity as "the protected `LGU-CTO/workpulse` release workflow", and `Binary Release Credential` defines a GitHub token "for private workpulse release access".
Leaving them in a public repository's glossary would publish the predecessor's private infrastructure as toolreach's vocabulary.

No ADR describes the chain.
`docs/adr/0001` through `0007` are all Confluence, Jira, or authorization decisions.
`docs/adr/0007` mentions the chain once, at line 45, and that mention matters (see below).

## Does any surviving path read a release manifest for a non-upgrade reason?

**No.**
`internal/releasemanifest` has exactly nine importing files: its own four tests, `internal/binaryupgrade` (three files), and `tools/releasemanifest/main.go`.
Every one of them is in this manifest.
There is no configuration reader, no diagnostics command, and no telemetry path that opens `workpulse-release.json`.

The nearest thing to an exception is `internal/releasemanifest/workflow_policy_test.go`, which reads `.github/workflows/release.yml` and asserts the release workflow's shape.
It is a test of the chain's own trust root and leaves with it, but it is worth knowing that deleting the package silently deletes the only automated check on the release workflow's structure.

## What this surfaces

Four things the removal exposes that this ticket cannot settle.

**ADR-0007 loses its delivery vehicle.**
`docs/adr/0007-authorization-is-established-by-the-request-that-needs-it.md:45` states that when a release adds OAuth scopes, the run fails on the request that needed the grant "and the release's Upgrade Action carries the instruction instead".
The Upgrade Action is manifest-borne and printed by `workpulse upgrade` after a successful install.
Delete the chain and that sentence describes a mechanism that no longer exists, while the ADR's decision, that authorization is established lazily by the request needing it, remains correct and carries over.
`.github/upgrade-actions/hierarchy-read-grant.md` is a live instance of the pattern, and three separate skill documents (`skills/workpulse/SKILL.md:124`, `skills/workpulse/references/cli-reference.md:299`, `skills/llm-wiki/SKILL.md:74`) instruct agents to recognize the post-upgrade 401 and tell the user to re-run `auth login`.
Those skill instructions survive the removal unharmed; only the automated delivery of the notice goes.
A public tool distributing through Homebrew and `go install` needs some answer here, most likely release notes, and the map's Onboarding decision already commits to a scope-shortfall failure that names the missing permission, which covers much of the same ground.

**The plugin's own self-update path is untouched by this removal.**
`hooks/session-start.sh` is described above.
It is out of this manifest's scope because it is not part of the chain, but toolreach cannot ship it unexamined: it currently pulls from `LGU-CTO/workpulse`, assumes a private repository needing `gh` or a `GITHUB_TOKEN`, and replaces the user's binary with no verification at all.

**Three private-release environment variables must not survive.**
`WORKPULSE_GITHUB_TOKEN`, `GH_TOKEN`, and `GITHUB_TOKEN` are read by `binaryrelease.ResolveCredential` / `ResolveEnvCredential`, documented in README, and defined in `CONTEXT.md`.
They exist because the predecessor's releases are private.
They leave with the chain, which is correct, and the check that they are gone from README and the glossary is part of the public-exposure sweep in ticket [Public-exposure audit of the predecessor source](https://github.com/tmheo/toolreach/issues/5).

**A fourth CI job's fate is a real decision.**
Removing `test-binary-release-platforms` removes Windows from CI apart from `go vet`.
The predecessor builds Windows amd64 and arm64 archives, so dropping Windows testing while continuing to publish Windows binaries is a choice, not a consequence.

## Removal order

Ordered so the tree compiles at each step and nothing is deleted while still referenced.

1. `cmd/root.go` — drop `PersistentPreRunE`, rewrite `Execute` to call `rootCmd.ExecuteContext(ctx)` while still passing `ctx`.
2. Delete the six `cmd` files listed above.
3. Delete `tools/releasemanifest/`.
4. Delete the six `internal` packages with their `testdata/` and embedded trusted root.
5. `internal/config/config.go` — remove `UpdateCheckConfig`, the `Config.UpdateCheck` field, and its default.
6. `go mod tidy`. Expect `github.com/sigstore/sigstore-go` and `github.com/Masterminds/semver/v3` to drop out; `github.com/gofrs/flock` stays because `internal/publication` and `internal/atlassianaccess` use it.
7. Delete `.github/workflows/trusted-root.yml` and `.github/upgrade-actions/`.
8. `.github/workflows/ci.yml` — remove `release-contract` and `test-binary-release-platforms`.
9. `.github/workflows/release.yml` — remove the six steps, rewrite the asset verification, drop `id-token: write`.
10. `.goreleaser.yaml` — remove `archives[].files: none*` and `snapshot.version_template`, review `release.header`.
11. `README.md` — delete lines 675 to 799 (`## Binary Upgrade` through `### macOS Quarantine`) and amend the Features bullet at line 19, which ends "and private-release upgrade support".
12. `CONTEXT.md` — delete lines 283 to 388.
13. `docs/release-integrity.md` — delete. `docs/releases/` — do not port.
14. `BINARY_VERSION` and `hooks/session-start.sh` — hold pending the plugin-distribution decision; the file has no remaining Go or CI reader once step 9 lands.

## Primary sources

All paths relative to `~/workspace/github/workpulse` at `BINARY_VERSION` 0.8.3.

- `go list -deps=false -f '{{.ImportPath}}|{{join .Imports ","}}' ./...` — the package graph.
- `cmd/root.go`, `cmd/version.go`, `cmd/upgrade.go`, `cmd/update_check_lifecycle.go`.
- `internal/config/config.go` lines 13-24 and 60-75.
- `.github/workflows/ci.yml`, `.github/workflows/release.yml`, `.github/workflows/trusted-root.yml`.
- `.goreleaser.yaml`.
- `CONTEXT.md` lines 283-388.
- `docs/adr/0007-authorization-is-established-by-the-request-that-needs-it.md` line 45.
- `docs/release-integrity.md`.
- `hooks/session-start.sh`.
- `README.md` lines 19 and 675-799.
- `.github/upgrade-actions/hierarchy-read-grant.md`.
