# Public-exposure audit of the predecessor source

Resolves [Public-exposure audit of the predecessor source](https://github.com/tmheo/toolreach/issues/5), a ticket on [Map: port workpulse to a publicly distributable toolreach v0.1](https://github.com/tmheo/toolreach/issues/1).

Audited tree: the predecessor checkout at `~/workspace/github/workpulse`, `BINARY_VERSION` 0.8.3, 353 tracked files.
Build output (`dist/`, `bin/`, the root binary) and local agent tooling (`.claude/`, `.codex/`) are gitignored and untracked, so nothing there can travel.

## Re-verification at 0.9.0

The predecessor moved to `BINARY_VERSION` 0.9.0 after this audit was written, adding `jira search --fields`, correcting a `--limit` help text, and updating both skill copies to match.
The tree is now 354 tracked files.

Findings A through L and N survive the change unaltered, and two of the quantities were recounted rather than assumed: the `@MX:*` split is still exactly 27 `SPEC`/`ANCHOR` and 30 `NOTE`/`REASON`/`WARN` lines, and the module path is still imported by exactly 114 Go files.
`internal/oauth/` was untouched, so nothing here bears on the scope inventory either.

**Finding M is the exception and has been corrected below.**
0.9.0 put a real Jira project key into a test file, which is the first time a live work identifier has appeared anywhere outside `docs/releases/`.
The lesson generalizes past this one release: an audit of a moving predecessor describes a moment, and the port should re-run the identifier greps against whatever commit it actually copies rather than trusting a snapshot.

## Redaction key

This document lives in a public repository, so it does not reproduce the identifiers it is asking someone to delete.
A concentrated, indexed list of an organization's internal names is a worse disclosure than the scattered originals.
Placeholders below resolve against the predecessor checkout:

| Placeholder | What it is | Recover with |
|---|---|---|
| `<ORG>` | predecessor GitHub organization slug | `head -1 go.mod` |
| `<COMPANY>` | company name in README prose | `sed -n '10p;884p' README.md` |
| `<CORP-EMAIL>` | maintainer's corporate address | `grep -h email .claude-plugin/plugin.json` |
| `<TENANT-A>`, `<TENANT-B>` | the two real Atlassian site hostnames | see finding B |
| `<JIRA-KEY>` | Jira project key used in release smoke tests | see finding C |
| `<JIRA-KEY-2>` | a second, real Jira project key, this one a live delivery project rather than a smoke-test one | `grep -rn 'project = ' internal/jira/client_test.go` |

Whoever does the port has the private checkout and can expand these in seconds.
This is a judgement call about the audit artifact, not about the port: if you would rather the doc name things outright, say so and it can be rewritten.

## Verdict summary

| # | Finding | Verdict |
|---|---|---|
| A | Repo slug `<ORG>/workpulse` outside import paths, 17 sites | must-remove |
| B | Two real Atlassian tenant hostnames, 4 occurrences | must-remove |
| C | Real Jira and Confluence work identifiers in release evidence, and in `internal/jira/client_test.go` as of 0.9.0 | must-remove |
| D | `docs/releases/` and `docs/release-integrity.md` | must-remove (do not port) |
| E | `<CORP-EMAIL>` in three plugin manifests | must-remove |
| F | Homebrew private-download-strategy and its README section | must-remove |
| G | `.github/` secrets and the trusted-root workflow | must-remove |
| H | `PRIVACY.md` | should-rewrite (from scratch) |
| I | Licence line, README prose, missing `LICENSE` file | should-rewrite |
| J | `@MX:*` annotations, 57 lines in 14 files | should-rewrite (split, see below) |
| K | Korean-only user-facing strings, 37 files | should-rewrite (needs a decision first) |
| L | Go module path and imports, 114 files | rename surface, not exposure |
| M | Jira and ADF test fixtures | harmless |
| N | 23 generic `*.atlassian.net` placeholders | harmless |

## must-remove

### A. Repo slug outside import paths

`<ORG>/workpulse` appears in 137 files.
114 of those are Go import paths and are covered by the rename (finding L).
The remaining 17 sites hard-code the private repository as an identity rather than as a package prefix:

- `README.md:3,4,5,6` - CI, Release, pkg.go.dev, and coverage badges. The coverage badge additionally points at an `<ORG>.github.io` Pages endpoint that will not exist.
- `README.md:33,39,43,58,825,840` - `brew tap`, Releases link, `gh release download -R`, `go install`, and both plugin marketplace `add` commands.
- `.goreleaser.yaml:19-24` - ldflags `-X` paths, and `:63` `owner:`.
- `AGENTS.md:9` - names the issue tracker repo.
- `hooks/session-start.sh:6` - `REPO="<ORG>/workpulse"`.
- `CONTEXT.md:311` - the trusted-identity sentence. `CONTEXT.md` comes across as a file under the standing Code-transfer decision, so this one line travels unless it is edited.
- `plugins/workpulse/.codex-plugin/plugin.json:6,27,35` - `name`, `developerName`, and a `privacyPolicyURL` pointing into the private repo.
- `plugins/workpulse/skills/workpulse/SKILL.md:43,47,52` - install instructions.
- `internal/github/releases.go:26` and `internal/releasemanifest/signature.go:23` - hard-coded repository constants. Both packages are already ruled out of scope with the self-upgrade chain, so these two leave for free.

### B. Real Atlassian tenant hostnames

Two real site hostnames, 4 occurrences in 3 files:

- `cmd/spaces_test.go` - 1 occurrence of `<TENANT-A>`. This is the only one in production or test code, and it is a plain fixture value with no behavioural meaning. Replace with `example.atlassian.net`.
- `docs/releases/2026-06-28-post-release-smoke.md` - 1 occurrence.
- `docs/releases/2026-06-30-markdown-page-source-post-release-smoke.md` - 2 occurrences.

The two release documents are covered wholesale by finding D.

### C. Real Jira and Confluence work identifiers

The release-evidence documents record live operations against the internal tenant.
`docs/releases/2026-06-28-post-release-smoke.md` alone contains a real Jira project key `<JIRA-KEY>`, a created issue key in that project, a Jira comment id, two Confluence page ids (a created child and its homepage parent), a phrase describing an internally approved personal space, GitHub Actions run ids, and internal issue numbers.
The other release documents follow the same template.
None of this is dangerous on its own, but together it maps a slice of the organization's Confluence and Jira layout.

**As of 0.9.0 this finding is no longer confined to release evidence.**
`internal/jira/client_test.go` hard-codes `<JIRA-KEY-2>`, a real delivery project, in three places: an issue key in a fixture payload, a second issue key that the fixture's `issuelinks` points at, and a `project = <JIRA-KEY-2>` JQL string in the call under test.
`docs/releases/2026-08-01-jira-search-field-selection-release.md` names the same project in prose, and adds a GitHub Actions run id, a release URL, and a Homebrew tap commit, but that document is covered by finding D and never ports.

The distinction that matters is where the identifier sits.
Finding D neutralizes everything in `docs/releases/` by not porting it.
The test suite has no such protection: it comes across, which is what [The rename surface in the test suite and testdata](https://github.com/tmheo/toolreach/issues/14) is scoped to measure.
So this is the first work identifier that would reach a public repository unless someone deletes it, and the fix is to rewrite the three sites to a synthetic key in the same style the rest of the fixtures already use.

### D. `docs/releases/` and `docs/release-integrity.md`

This answers the ticket's last open bullet directly: **yes, the release-evidence documents do contain internal material**, per finding C.

The recommendation is stronger than redaction: do not port either path.

- All 9 files in `docs/releases/` attest to the self-upgrade and release-verification chain, which the map has already ruled out of scope. They document a history that toolreach does not have and a mechanism it will not ship.
- `docs/release-integrity.md` is the specification of that same chain. Line 35 explicitly records maintainers accepting that the public transparency-log record discloses the private repository name - a sentence that only makes sense inside the private repo.
- `.github/upgrade-actions/hierarchy-read-grant.md` is an Upgrade Action payload for the release manifest and leaves with it.

Deleting rather than redacting also removes the temptation to keep half of a mechanism whose trust root is gone.

### E. Corporate email in plugin manifests

`<CORP-EMAIL>` appears in three manifests:

- `.claude-plugin/plugin.json:7`
- `.claude-plugin/marketplace.json:5`
- `plugins/workpulse/.codex-plugin/plugin.json:7`

This is the only person-identifying material found anywhere in the tree.
It is a real corporate address that will be published verbatim in a plugin marketplace listing, and it ties the public tool to the employer.
Replace with a public contact, or drop the field where the manifest schema allows it.

Otherwise the source is clean of people: no author, maintainer, or reviewer names in comments, no `Co-authored-by` trailers, no reviewer handles.
Git history is moot under the standing Code-transfer decision, which starts from one fresh commit.

### F. Homebrew private-download-strategy

`.goreleaser.yaml:63-71` configures the tap for a private repository:

- `owner: <ORG>` and `name: homebrew-workpulse`
- `token: "{{ .Env.HOMEBREW_TAP_GITHUB_TOKEN }}"`
- `custom_require: lib/custom_download_strategy`
- `download_strategy: GitHubPrivateRepositoryReleaseDownloadStrategy`

The last two exist only because release assets sat behind private-repo auth.
A public tap needs neither, and shipping them would produce a formula that fails for every user.

`README.md:717-739` is the matching user-facing section, "Private Repository Authentication".
It instructs the reader to mint a GitHub Personal Access Token with **`repo` scope** to download releases.
Published as-is that is both false and actively bad advice - it asks strangers to create a broadly-scoped token for a public download.
Remove the section.

`README.md:19` also advertises "private-release upgrade support" as a feature.

### G. `.github/` secrets and the trusted-root workflow

Referenced secrets, none of which will exist:

- `ATLASSIAN_CLIENT_ID`, `ATLASSIAN_CLIENT_SECRET`, `ATLASSIAN_BASE_URL` in `release.yml`. These feed the `.goreleaser.yaml` ldflags build defaults. Under the standing Auth-model decision public releases ship those defaults empty, so both the secrets and the three `-X` lines that consume them go. `ci.yml:42` already sets `ATLASSIAN_CLIENT_SECRET: ""`, which is the shape the release path should adopt.
- `WORKPULSE_RELEASE_APP_ID` and `WORKPULSE_RELEASE_APP_PRIVATE_KEY` in `release.yml`. GitHub App credentials for the release chain; they leave with it.
- `GITHUB_TOKEN`, 4 uses. Standard and harmless.

`trusted-root.yml` (53 lines) is entirely the out-of-scope trust root. Delete it.

`ci.yml:255-280` deploys the coverage badge to GitHub Pages. Not internal, but it is wired to the `<ORG>` Pages endpoint the README badge reads, so the two move together or both go.

## should-rewrite

### H. `PRIVACY.md`

1.1 KB, last updated 2026-04-04, and wrong on four counts under the current design:

- Line 3 calls the tool "an internal command-line tool".
- It describes Confluence sync only. Jira read and write, which the tool has had for months, are absent - so the document understates what data the tool touches.
- Line 15 names "Confluence base URL, client ID" as stored configuration, which is the old single-app shape. Under the per-organization auth model the stored credentials belong to the user's own registered app.
- Line 28, "contact the repository maintainers", is not a contact route for a stranger.

Rewrite from scratch rather than editing.
Note the dependency: `plugins/workpulse/.codex-plugin/plugin.json` links it as `privacyPolicyURL`, so if the file is dropped the manifest field must be dropped with it.

### I. Licence and README prose

- `README.md:884` reads "Proprietary. Internal use only at <COMPANY>." This directly contradicts the standing Apache-2.0 decision.
- `README.md:10` frames the tool as "Built for teams at <COMPANY> who use coding agents...". The whole opening positioning statement needs replacing, not just the company name.
- There is **no `LICENSE` file in the tree at all**. Apache-2.0 needs the file added, not carried over.

### J. `@MX:*` annotations

57 annotation lines across 14 files: 23 `SPEC`, 16 `NOTE`, 9 `REASON`, 5 `WARN`, 4 `ANCHOR`.
Most carry an `[AUTO]` marker, so they were machine-generated.
Files: `cmd/auth.go`, `cmd/jira.go`, `internal/confluence/{client,pageref,ratelimit}.go`, `internal/jira/{client,client_test,types}.go`, `internal/jira/adf/converter.go`, `internal/oauth/{oauth,server,token}.go`, and both copies of `skills/llm-wiki/scripts/wiki_assets.py`.

Confirmed: **the identifiers resolve to nothing, even inside the predecessor repo.**
Grepping `SPEC-JIRA-001` returns only the four files that cite it - no spec document, no requirements directory, no manifest defines it.
They are already dangling pointers into an external system today; publishing them only makes the dangling public.

They disclose the existence and naming scheme of an internal traceability tool, not its content.
That is a low-grade leak, so the real question is editorial.
The three options in the ticket do not have one answer, because the annotations are two different things wearing one prefix:

- **`SPEC` and `ANCHOR`, 27 lines - strip.** These are pure cross-references (`// @MX:SPEC: SPEC-AUTH-001 REQ-AUTH-002`, `// @MX:ANCHOR: [AUTO] Sole constructor...`). A reference to an unreachable document is noise to every public reader. Retargeting would mean inventing a public spec corpus toolreach does not have and, per the map, is not going to grow before v0.1.
- **`NOTE`, `REASON`, and `WARN`, 30 lines - keep the prose, drop the prefix.** These are self-contained explanations that survive the move intact, for example `// @MX:WARN: Token file access requires strict permission validation (0600 only).` followed by `// @MX:REASON: Token files with group/world-readable permissions expose OAuth credentials.` Converting them to plain comments keeps the value and leaves no dangling scheme. Watch for stragglers: a few `NOTE` lines cite a `REQ-` id inline in their prose (`cmd/jira.go:1040`, `:1190`), and those citations need trimming too.

This is a decision the port can execute mechanically once someone confirms the split. If you disagree with it, it deserves its own ticket rather than a footnote here.

### K. Korean-only user-facing strings

37 tracked files contain Hangul.
This is not an exposure finding - nothing internal is disclosed - but it is a public-distribution finding, and it was not on the ticket's list.

The strings are user-facing, not comments.
`internal/oauth/resources.go` holds the grant-shortfall messages that the standing Onboarding decision relies on to name the missing permission, and they are Korean.
`internal/sync/parent_resolution.go`, `cmd/auth.go`, `cmd/jira.go`, `cmd/sync.go` and others wrap errors in Korean.
`.github/pull_request_template.md` is Korean.

A tool that "anyone can point at their own Atlassian tenant" will fail in a language most of its users cannot read, and the failure path is precisely the one the onboarding design leans on.
This needs a decision before the port, not a judgement call during it.
Filed as a ticket on the map.

## Not exposure

### L. Go module path and 114 import files

`module github.com/<ORG>/workpulse` in `go.mod`, plus 114 files importing under that prefix.
Purely mechanical and fully covered by the rename to the public module path.
Listed here only so it is not mistaken for 114 separate leaks - it is one edit applied 114 times.

The same applies to the `WORKPULSE_` environment-variable prefix, the `~/.workpulse` data directory, and the `confluence:` config key that the standing Config-schema decision already renames.

### M. Test fixtures

`internal/jira/testdata/issue-sample.json` and the ADF fixtures use obviously synthetic identities: display names paired with `@example.com` addresses and `accountId` values like `user-001`.
No real Atlassian account ids anywhere in the tree.
`testdata/` at the repo root holds 2 files and contains no organization tokens.

**Correction, 0.9.0.**
As originally written this finding said the fixtures were synthetic without qualification, and at 0.8.3 that was true: `<JIRA-KEY-2>` appeared in no Go file in the tree.
0.9.0's inline fixture in `internal/jira/client_test.go` breaks it, and the detail is under finding C.
Account ids, display names and email addresses remain synthetic throughout, so the correction is confined to project and issue keys.
Whoever ports the test suite should re-run the check rather than read this paragraph, because it describes 0.9.0 and the predecessor keeps moving.

### N. Generic hostname placeholders

25 distinct `*.atlassian.net` hostnames appear in the tree.
Two are real (finding B).
The other 23 are test placeholders - `example`, `acme`, `mycompany`, `your-domain`, and state-machine names like `old`, `new`, `current`, `saved`, `someone-elses`.
They are fine as they are.

## Sequencing note

Findings D, F, and G delete material that the map has already ruled out of scope for independent reasons.
Doing the out-of-scope removal first shrinks the exposure surface before anyone starts redacting, and leaves findings A, B, C, and E as the only deliberate scrubbing work.
