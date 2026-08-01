# The rename surface in the test suite and testdata

## What this answers

This is the findings asset for GitHub issue #14, "The rename surface in the test suite and testdata".

The question is whether porting the predecessor's ~38K-line test suite is a sed run or a week, and where the predecessor's name sits inside an assumption rather than inside a string.

Everything below was counted against `~/workspace/github/workpulse`, `BINARY_VERSION` **0.9.0**, commit `1108c9f8e295865d49cdfd0b10594b52c75ab055` (2026-08-01), branch `main`.
The port must re-run these counts against the commit it actually copies.

**Verdict: a sed run plus roughly 30 hand edits across 9 files - call it one afternoon, not a week.**
The hand edits are not where the ticket expected them, and the two places the ticket predicted trouble turn out to be clean.

---

## 1. The totals

| | files | lines |
| --- | --- | --- |
| All `*_test.go` | 119 | 38,000 |
| Leaving with the removal manifest | 32 | **6,657** |
| **Surviving** | **87** | **31,343** |
| All production `*.go` | 125 | 26,758 |

The 6,657 figure is a clean cross-check.
[Issue 4](https://github.com/tmheo/toolreach/issues/4)'s removal manifest derived **6,657 test lines** by summing per-package counts; subtracting the surviving files from the total here reaches the same number by a different route.
The boundary is confirmed, not re-derived.

Name occurrences (case-insensitive `workpulse`) in `*_test.go`:

| | occurrences |
| --- | --- |
| Total | 537 |
| In leaving files | 220 |
| **In surviving files** | **317** |

---

## 2. testdata: zero rename work

This is the ticket's central worry, and it does not exist.

The predecessor has **33 `testdata/` files across 5 directories**:

| directory | files | files containing the name |
| --- | --- | --- |
| `internal/jira/adf/testdata` | 23 | 0 |
| `internal/releasemanifest/testdata` | 5 | 1 (**leaves**) |
| `internal/confluence/storagebody/testdata` | 2 | 0 |
| `testdata` (repo root) | 2 | 0 |
| `internal/jira/testdata` | 1 | 0 |

**28 surviving fixture files, zero occurrences of the name in their content, zero in their filenames.**

The only fixtures carrying it are `workpulse-v0.8.0-release.json` and `workpulse-v0.8.0-release.sigstore.json` in `internal/releasemanifest/testdata`, which leave whole with the chain - the removal manifest already lists them.

The reason is structural rather than lucky.
Every surviving fixture is *input to* toolreach rather than *output from* it: Confluence storage XHTML, Atlassian Document Format JSON pairs, a Jira issue sample, and one Markdown golden (`testdata/representative.golden.md`) which is the rendering of a Confluence body and mentions no tool.
A fixture only carries a tool's name when it is a snapshot of that tool's own artifacts, and the predecessor snapshots none.

The `~/.workpulse` paths the ticket expected in fixtures live in Go code instead, built at run time with `filepath.Join`, which is why the sed reaches them.

**There are no golden files to re-record.**

---

## 3. The 317 surviving occurrences, classified

| class | occurrences | disposition |
| --- | --- | --- |
| Import path `github.com/LGU-CTO/workpulse/...` | 81 | mechanical |
| `WORKPULSE_*` environment variables | 96 | 84 mechanical, 12 not - see 3b |
| Local identifier `workpulseDir` | 53 | mechanical |
| Data-directory literal `".workpulse"` / `"~/.workpulse"` | 43 | mechanical |
| Log file name `workpulse.log` | 4 | 3 mechanical, 1 contingent - see 4 |
| Everything else (prose comments, assertions, fixture strings) | 40 | mixed - see 4 and 5 |

**261 of 317 (82%) are pure substitution.**

### 3a. Mechanical, by file

The 177 imports plus identifiers plus data-directory literals are spread thinly.
The heaviest single file is `cmd/sync_hierarchy_test.go` at 28 `workpulseDir` occurrences, all of the form:

```go
require.NoError(t, os.MkdirAll(workpulseDir, 0700))
config.SaveConfig(cfg, filepath.Join(workpulseDir, config.ConfigFileName))
```

The file names `config.ConfigFileName`, `config.OAuthTokenFile` and `config.OAuthSiteFile` rather than string literals, so the "token / site files unchanged" decision in [issue 2](https://github.com/tmheo/toolreach/issues/2) costs the test suite nothing.

Eleven files carry the `".workpulse"` literal: `cmd/auth_test.go` (7), `cmd/jira_test.go` (7), `cmd/confluence_test.go` (7), `cmd/sync_hierarchy_test.go` (7), `cmd/helpers_test.go` (4), `internal/config/config_test.go` (4), and one each in `cmd/atlassian_access_test.go`, `cmd/sync_data_directory_test.go`, `cmd/sync_space_failure_test.go`, `cmd/snapshot_test.go`, `cmd/sync_authorization_test.go`.

### 3b. Environment variables, by name

| variable | occurrences | disposition |
| --- | --- | --- |
| `WORKPULSE_CONFLUENCE_BASE_URL_OVERRIDE` | 36 | prefix only - **must not take the config rename**, see 5b |
| `WORKPULSE_ACCESSIBLE_RESOURCES_URL_OVERRIDE` | 25 | prefix only |
| `WORKPULSE_JIRA_BASE_URL_OVERRIDE` | 14 | prefix only |
| `WORKPULSE_OAUTH_CLIENT_ID` | 6 | **shape change**, see 5a |
| `WORKPULSE_TOKEN_URL_OVERRIDE` | 2 | prefix only |
| `WORKPULSE_SYNC_CONCURRENCY_HELPER` | 2 | prefix only |
| `WORKPULSE_EMAIL` | 2 | **deleted** (issue 2) |
| `WORKPULSE_API_TOKEN` | 2 | **deleted** (issue 2) |
| `WORKPULSE_CRASH_*` / `WORKPULSE_FALLBACK_CRASH_DIR` (5 distinct) | 5 | prefix only |
| `WORKPULSE_BASE_URL` | 1 | **shape change**, see 5a |
| `WORKPULSE_NO_UPDATE_CHECK` | 1 | **deleted** - belongs to the leaving chain, see 6 |

84 take a plain prefix substitution; 12 do not.

---

## 4. Load-bearing: tests that assert on a name

Thirteen sites. This is the whole hand-edit surface that a grep for the name can find.

| # | site | what it asserts | note |
| --- | --- | --- | --- |
| 1 | `cmd/root_test.go:17` | `rootCmd.Use == "workpulse"` | plain rename |
| 2 | `cmd/root_test.go:29` | the exact `rootCmd.Long` sentence | **also a content change** - see 4a |
| 3 | `cmd/root_test.go:33-43` | `"workpulse version %s (commit: %s, built: %s)\n"` | **tautological** - see 5c |
| 4 | `cmd/sync_data_directory_test.go:132` | error contains `"another workpulse is using the data directory"` | contingent on [issue 13](https://github.com/tmheo/toolreach/issues/13) - see 4b |
| 5 | `cmd/confluence_publish_contract_test.go:247` | `"  Recovery: workpulse confluence page publish 'page-123' --body-file 'page.xml'\n"` | exact command string in user-facing output |
| 6 | `cmd/confluence_test.go:813` | output contains `"workpulse-page-12345-"` | a **temp-directory prefix** issue 2 does not name - see 4c |
| 7 | `build_requirement_docs_test.go:30-31` | `skills/workpulse/SKILL.md` and `plugins/workpulse/skills/workpulse/SKILL.md` exist and name the go.mod Go version | asserts the plugin directory layout issue 2 renames |
| 8 | `internal/config/config_test.go:21` | `cfg.Log.File == "workpulse.log"` | contingent on [issue 18](https://github.com/tmheo/toolreach/issues/18) - the key may not survive at all |
| 9 | `internal/config/config_test.go:143,148` | a validation error contains `"workpulse config set"` | tracks two production strings in `internal/config/config.go:179,197` |
| 10 | `internal/config/config_test.go:98` | `WORKPULSE_BASE_URL` overrides `Confluence.BaseURL` | shape change - see 5a |
| 11 | `internal/config/config_test.go:80,183` | `WORKPULSE_OAUTH_CLIENT_ID` overrides `Confluence.ClientID` | shape change - see 5a |
| 12 | `internal/oauth/edge_test.go:339-344` | nothing - see 5c | shape change **and** tautological |
| 13 | `internal/oauth/edge_test.go:349-354` | deprecated `WORKPULSE_EMAIL` / `WORKPULSE_API_TOKEN` have no effect | issue 2 already rules: restated without them, or dropped |

### 4a. `rootCmd.Long` is a content change wearing a rename's clothes

`cmd/root_test.go:29` asserts:

> `workpulse is a CLI tool that syncs Confluence Cloud pages to the local filesystem and converts them to Markdown.`

[Issue 8](https://github.com/tmheo/toolreach/issues/8) already found this sentence **omits Jira entirely** and must be rewritten, not renamed.
So the test is edited twice for two different reasons, and only one of them is a rename.

### 4b. The data-directory collision message is contingent on issue 13

`"another workpulse is using the data directory"` reads naturally as "another **toolreach**".
But issue 13 decided that sharing one data directory must be **refused** and that the guard is a **name-free** `*-data.lock`, precisely because renaming the lock file lets two differently-named tools sweep each other's staging entries.
The message the guard prints therefore has to say "another tool", not "another toolreach", or it contradicts the guard it reports.

This is a rename that the port must **not** perform mechanically.

### 4c. Four temp-path prefixes issue 2's contract table does not name

Production builds four run-time paths from the project name, none of which appears in issue 2's surface table:

| production site | pattern |
| --- | --- |
| `internal/confluence/pageread/artifacts.go:68` | `workpulse-page-%s-*` |
| `internal/confluence/pagepublish/prepare.go:49` | `workpulse-page-publish-` |
| `internal/confluence/client.go:178` | `workpulse-hierarchy-*.jsonl` |
| `internal/sync/hierarchy_queue.go:24` | `workpulse-hierarchy-queue-*.bin` |

All four survive the port.
Only the first is asserted by a test (site 6 above); the other three rename silently.
They are process-local temp files under `os.MkdirTemp("")`, so they carry no compatibility obligation - but they are user-visible in `pageread` output, which is why one of them ended up in an assertion.
Issue 2's table should gain a row, or the port should record that temp prefixes follow the binary name by rule.

---

## 5. Traps

The ticket named four suspected trap classes. Two are empty, two are real, and there is a fifth it did not name.

### 5a. Real: three variables change **shape**, not just prefix

Issue 2's rule is `TOOLREACH_` + the config path, and the predecessor's variables do **not** mirror their config paths:

| predecessor | naive prefix sed | issue 2's actual name |
| --- | --- | --- |
| `WORKPULSE_BASE_URL` | `TOOLREACH_BASE_URL` | **`TOOLREACH_ATLASSIAN_BASE_URL`** |
| `WORKPULSE_OAUTH_CLIENT_ID` | `TOOLREACH_OAUTH_CLIENT_ID` | **`TOOLREACH_ATLASSIAN_CLIENT_ID`** |
| `WORKPULSE_OAUTH_CLIENT_SECRET` | `TOOLREACH_OAUTH_CLIENT_SECRET` | **`TOOLREACH_ATLASSIAN_CLIENT_SECRET`** |

A `WORKPULSE_` → `TOOLREACH_` sweep produces three names the binary never reads.
The first two have **7 test occurrences** between them; the third has none in tests but exists at `internal/config/config.go:153`.

Two of those seven fail loudly (`config_test.go:98` and `:80,183` assert the override took effect).
The third does not - see 5c.

### 5b. Real: 50 occurrences must take the rename **rename** but not the **config** rename

`WORKPULSE_CONFLUENCE_BASE_URL_OVERRIDE` (36) and `WORKPULSE_JIRA_BASE_URL_OVERRIDE` (14) are the two largest name clusters in the whole surviving suite, and they sit at the intersection of two renames while needing exactly one.

They name the **product** whose base URL is being overridden, not a config key.
The pair exists precisely because Confluence and Jira have different base URLs (`internal/oauth/resources.go:38-47` builds `.../ex/confluence/{cloudID}/wiki` and `.../ex/jira/{cloudID}`).

If the `confluence:` → `atlassian:` config rename is applied by sweeping the word, the Confluence one becomes `TOOLREACH_ATLASSIAN_BASE_URL_OVERRIDE` - which is both wrong (it overrides only the Confluence base, not both products) and one word away from the real `TOOLREACH_ATLASSIAN_BASE_URL` from 5a.

**Correct: `TOOLREACH_CONFLUENCE_BASE_URL_OVERRIDE` and `TOOLREACH_JIRA_BASE_URL_OVERRIDE`.**
Prefix only.

### 5c. Real, and the ticket's closing question answered: **No, the `WORKPULSE_EMAIL` test is not the only one of its class**

The ticket asks whether the vestigial-variable test is the only test whose behaviour does not actually depend on the name it uses.
It is not. There are **three**, and the other two are worse because they guard live contracts.

**`internal/oauth/edge_test.go:337-345`, `TestEdge_EnvClientIDOverride`:**

```go
t.Setenv("WORKPULSE_OAUTH_CLIENT_ID", "env-client-id-override")
// The OAuthFlow itself doesn't read env vars - the config layer does.
// This test documents the expected contract: env var takes precedence.
// Verify by checking the env is set and the value is correct.
envVal := os.Getenv("WORKPULSE_OAUTH_CLIENT_ID")
assert.Equal(t, "env-client-id-override", envVal)
```

It sets a variable and asserts `os.Getenv` returns what it just set.
It passes under **any** name, including a misspelled one, and its own comment admits it never reaches the code it claims to test.
This is the one place in the suite where getting the 5a shape change wrong is **silent**.

**`cmd/root_test.go:36-45`, `TestVersionCmdOutputFormat`:**

```go
buf := &bytes.Buffer{}
versionCmd.SetOut(buf)
// Simulate what versionCmd.Run does
output := fmt.Sprintf("workpulse version %s (commit: %s, built: %s)\n", version, commit, date)
assert.Contains(t, output, "workpulse version ")
```

It formats the expected string itself, asserts the string it just formatted contains its own prefix, and never invokes `versionCmd.Run` or reads `buf`.
It is the only test guarding the version output format - the format [issue 10](https://github.com/tmheo/toolreach/issues/10) builds the plugin preflight on top of, via `toolreach version --short`.
A port that renames the literal and leaves the tautology intact ships a preflight guarded by nothing.

Both should be rewritten to invoke the real path during the port, not merely renamed.

### 5d. Empty: no test constructs a lock or journal filename by hand

The ticket suspected a concurrency test building `.workpulse-data.lock` or `.workpulse-publication-journal.json` as a literal.

It does not happen.
Both names are unexported constants inside `internal/publication`:

- `internal/publication/lock.go:11` - `dataDirectoryLockFileName = ".workpulse-data.lock"`
- `internal/publication/journal.go:14-15` - `journalFileName`, `journalTemporaryPattern`

Grepping the 87 surviving test files for either literal returns **nothing**.
`internal/publication`'s own tests (`lock_test.go`, `journal_test.go`, `crash_unix_test.go`, `carryforward_test.go`, `durability_test.go` and four more) go through the package.

The one dotfile literal in a surviving test is `internal/config/config_test.go:313,325`, `"~/.workpulse-char"` - and that is a `Storage.DataDir` round-trip fixture testing that a hyphen survives, not a lock name.

### 5e. Empty: no fixture directory layout encodes the name

Covered by section 2. No `testdata/` tree contains a `.workpulse` path segment.
Every test that needs a data directory builds it at run time with `filepath.Join(home, ".workpulse")`, which the sed reaches.

### 5f. Not named by the ticket: occurrences that must **not** be renamed

Four occurrences in `internal/confluence/storagebody/interpretation_test.go:115-138` use `WORKPULSE` as arbitrary sentinel text inside a Confluence storage body:

```go
body := `<p><a href="WORKPULSE&#77;ARKERTARGET1">ordinary link</a></p>` + ...
assert.Contains(t, markdown, `[ordinary link](WORKPULSEMARKERTARGET1)`)
```

The point of the fixture is that `&#77;` (the letter `M`) sits at a split point, so the test proves entity decoding and inline-comment-marker stripping reassemble a token.
The word is a nonsense sentinel; it names nothing.

Renaming it is harmless (both sides move together) but it is not rename work, and a reviewer counting occurrences will mistake it for some.
The better port action is to replace the sentinel with a name-free token so nobody has to think about it again.

Two more of this shape, both arbitrary fixture values rather than names:

- `cmd/jira_test.go:2323,2355` - the Jira label `"workpulse-smoke"`.
- `internal/config/config_test.go:210-211` - `"/tmp/workpulse-data"`, a `ResolveDataDir` passthrough fixture.

---

## 6. One correction to the removal manifest's boundary

`cmd/root_test.go:53` calls `t.Setenv("WORKPULSE_NO_UPDATE_CHECK", "1")`.

`cmd/root_test.go` is **not** in the removal manifest's leaving set - it survives, and it is the file that also holds the `Use` / `Short` / `Long` characterization tests.
But `WORKPULSE_NO_UPDATE_CHECK` is read only by `cmd/update_check_lifecycle.go:102`, which leaves.

So the removal manifest's file-level boundary has exactly one leak: **one surviving test file carries one line that must be deleted rather than renamed.**
The manifest's line counts are unaffected; only its "nothing else refers to the chain" claim needs this footnote.

Once `PersistentPreRunE` goes from `cmd/root.go:17`, the `t.Setenv` is inert rather than wrong, so this is a tidiness fix, not a breakage.

---

## 7. The surface a grep for the name cannot find

This is the manifest's most important structural point: **the rename surface and the name-occurrence surface are different sets.**

The 317 occurrences overstate the work in one direction and understate it in another.

### 7a. The `confluence:` to `atlassian:` rename carries **zero** occurrences of the name

The standing Config-schema decision renames the top-level key.
In the surviving test suite that is:

| surface | occurrences | files |
| --- | --- | --- |
| Go field selector `.Confluence.` | 56 | 8 |
| YAML fixture text `confluence:\n  ...` | 5 | 3 (`cmd/jira_test.go`, `cmd/atlassian_access_test.go`, `cmd/helpers_test.go`) |
| Config-path string `confluence.base_url` | 2 | 1 (`cmd/helpers_test.go`) |
| **Total** | **63** | **11** |

Not one of those 63 contains the word `workpulse`.
A port that measures its progress by the name-occurrence count will declare the test suite done with 63 edits outstanding.

The 5 YAML fixtures are the ones to watch: `cmd/atlassian_access_test.go:54` writes `"confluence: [invalid"` to test a malformed-YAML path, so the rename must keep it malformed.

### 7b. Name-free tests the rename still breaks

`cmd/root_test.go:23`, `TestRootCommandShort`, asserts:

> `CLI tool for syncing Confluence pages to the local filesystem`

No name in it, and it is false for toolreach for the same reason issue 8 gave about `Long` - it omits Jira.
A characterization test pins prose, and prose the port rewrites breaks a test that no grep for the name will surface.

`cmd/atlassian_access_migration_test.go` is a whole file in this class.
It is an SSA call-graph architecture test policing that no command reaches a list of legacy authorization identifiers.
It contains no occurrence of the name, and issue 13 deletes the deprecated `Email` / `APIToken` migration-detection pair it was written around, so its identifier list moves.

---

## 8. The verdict, priced

| bucket | occurrences | effort |
| --- | --- | --- |
| Pure substitution (imports, `workpulseDir`, `.workpulse` literals, 84 env vars) | 261 | one sed, minutes |
| Load-bearing sites needing a human | 13 sites, ~24 occurrences | ~30 lines |
| Must not be renamed (sentinels, arbitrary fixtures) | 8 | read and leave |
| Config-key rename, name-free | 63 | second sed plus 5 YAML fixtures |
| Rewrite rather than rename (two tautologies) | 2 tests | the only real work |
| Delete | 5 (`NO_UPDATE_CHECK`, `EMAIL`, `API_TOKEN`) | minutes |
| `testdata/` | **0** | none |

**A sed run, not a week.**
The suite is roughly 82% mechanical by occurrence and near-100% mechanical by file - only 9 of 87 surviving test files need a human to open them.

The residual risk is concentrated, not diffuse: **two tautological tests** (5c) that pass under a wrong name and guard contracts other tickets depend on, and **50 occurrences** (5b) that must take one rename and refuse the other.

Order the port should run it in, since three of the edits are contingent on other decisions:

1. Module path and identifier sed - unblocked.
2. Env-var rename with the three shape changes hand-applied, and the two `*_OVERRIDE` names deliberately left product-scoped.
3. Config-key rename - the 63 name-free occurrences.
4. Rewrite the two tautologies to invoke the real code path.
5. **Wait for issue 18** before touching `internal/config/config_test.go:21` - `log.file` may not survive.
6. **Wait for issue 13's** name-free lock guard before rewording `cmd/sync_data_directory_test.go:132`.
7. **Apply issue 8's** `Short` / `Long` rewrite in the same commit as the `root_test.go` rename.

## 9. Not established here

- Whether the two tautological tests have siblings in the **production** code's own characterization suite; this pass looked only at whether a test's behaviour depends on the name.
- Whether `cmd/atlassian_access_migration_test.go`'s SSA identifier list contains anything else issue 13's deletion invalidates; only the `Email` / `APIToken` pair was traced.
- Runtime cost of the suite before and after, which decides nothing here but would tell the port whether the removed chain's 6,657 lines were also its slowest.
