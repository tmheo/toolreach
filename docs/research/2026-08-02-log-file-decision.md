# What toolreach's log file is and whether it is bounded

## What this answers

This is the decision record for GitHub issue #18, a grilling ticket.
It was run as a self-grilling: each question posed alone, answered with the strongest available recommendation, and the reasoning shown.

Source read at `~/workspace/github/workpulse`, `BINARY_VERSION` **0.9.0**, commit `1108c9f8e295865d49cdfd0b10594b52c75ab055` (2026-08-01).
The log file itself was measured locally on 2026-08-02; no external service was contacted and nothing from the file's contents is reproduced here.

**Verdict in one line: 0.1.0 writes no file log at all by default, `LogConfig` is deleted whole - both keys, not just `log.file` - and a debug log is reached with `--log-file` at a path the user names.**

---

## 0. The ticket's premise is wrong, and correcting it changes the answer

The ticket closes with:

> Note that `log.level` is genuinely wired and is not in question.

**It is not wired.** At 0.9.0, `cfg.Log.Level` is read at exactly two sites, both in `cmd/config.go`:

- `cmd/config.go:37` - `config show` prints it
- `cmd/config.go:80` - `config set log.level` writes it

Nothing constructs a handler from it. `initLogger` hard-codes both levels:

- `cmd/root.go:34` - stderr handler at `slog.LevelInfo`
- `cmd/logger.go:26` - file handler at `slog.LevelDebug`

A repo-wide grep for `slog.Level*` outside test files returns exactly those two lines plus the `multiHandler.Enabled` signature.

So **both keys in `LogConfig` are dead**, and the `log:` block is entirely ornamental: two keys, both settable, both printed by `config show`, neither read by anything that logs.

This is not a symmetric discovery. It is worse in one specific way: **`config show` reports `level: info` while the file is written at debug.** The one diagnostic surface a user has actively misreports the setting that causes the growth the ticket is about.

Correcting the premise changes the question from "wire `log.file` or drop it" - a coin flip - into "the whole block is dead and the hard-coded asymmetry between the two levels is the defect".

---

## Q1. What actually produced 766 MiB?

**Answer: one Debug line in one loop, not diffuse logging. Measured, not inferred.**

The live file, 2026-08-02 00:06:

| | |
| --- | --- |
| Size | **801,843,042 bytes** (766 MiB) |
| Lines | **6,428,161** |
| Mean line length | ~125 bytes |

This confirms and slightly exceeds the 799 MB [issue 13](https://github.com/tmheo/toolreach/issues/13) reported; the file is still growing.

Composition, from the first 200,000 lines:

| level | lines | share |
| --- | --- | --- |
| DEBUG | 168,108 | **84.1%** |
| INFO | 31,359 | 15.7% |
| WARN | 528 | 0.3% |
| ERROR | 5 | 0.002% |

And by message:

| message | lines | share |
| --- | --- | --- |
| `skipping page (no changes)` | 165,849 | **82.9%** |
| `page sync complete` | 25,592 | 12.8% |
| everything else (13 distinct messages) | 8,559 | 4.3% |

**83% of the file is a single message.**

`skipping page (no changes)` is emitted at `internal/sync/sync.go:509`, at **Debug**, **once per unchanged page**, on **every sync**.
[Issue 13](https://github.com/tmheo/toolreach/issues/13) counted 3,245 pages in `history/`, and on a steady-state re-sync almost every page is unchanged.

The whole production tree contains only **30 `slog.*` call sites** (4 Debug, 12 Info, 10 Warn, 4 Error). This is not a tool that logs a lot. It logs one thing per page, in a loop, at a level the file records and the terminal does not.

**That last clause is the mechanism.** The file handler is `LevelDebug` and stderr is `LevelInfo`, so the single highest-volume event in the tool is invisible to the user and unconditionally written to disk. Roughly 665 MB of the 766 MiB is one line nobody has ever read.

The file is not "logs grow". It is one hard-coded level asymmetry, made unfixable by the dead `log.level` key.

*(Side observation, not this ticket's to decide: the same sample contains 528 `attachment download failed` warnings, a real error rate the log was the only place recording. Noted; not ticketed, since it is a predecessor operational fact rather than a port decision.)*

---

## Q2. Does 0.1.0 rotate, cap, or not write a file log by default?

**Answer: it does not write one. No rotation, no cap, no file.**

Five reasons, strongest first.

**1. Nothing reads it.** A repo-wide grep for `workpulse.log` and `Log.File` in production returns four lines: the default value, the `config show` print, the `config set` write, and the `filepath.Join` that creates the path. **No command reads the file, no support workflow references it, no issue template asks for it.** A file nobody reads is not a log; it is exhaust.

**2. The user-visible log already exists.** stderr carries Info and above on every run. The file adds exactly one thing over stderr: Debug. And Debug is 84% one line.

**3. Rotation is the wrong fix for the wrong problem.** Rotation bounds the disk cost of a stream that is 83% noise. Fixing the level removes ~665 MB of the 766 MiB first; the ~100 MB remaining over the same multi-month period would not have needed rotation either. Adding a rotation dependency - or hand-rolling one - to bound a stream you should not be writing is paying twice for a problem you could have deleted.

**4. Rotation is a concurrency question this data directory does not need.** Two `toolreach` processes appending to one file with `O_APPEND` is safe for small writes. Two processes *rotating* one file is not. [Issue 13](https://github.com/tmheo/toolreach/issues/13) established that this data directory has a hard-won mutual-exclusion story built on a flock over a named file, and a naive rotation would sit entirely outside it - renaming a file that another process holds open, in a directory whose whole locking discipline exists because that class of mistake corrupts things silently.

**5. A public tool should not do this to strangers.** 766 MB accumulating in a home directory, recording page titles and space keys, from a binary installed off a Homebrew tap, with no notice and no command to inspect or clear it. The ticket's own framing - "tolerable for one maintainer who knows to delete it and not tolerable in a tool strangers install" - is correct and is on its own sufficient.

**But a debug log must stay reachable**, because "run it again with more detail and send me the output" is the entire remote-diagnosis story for a tool whose maintainer cannot see the user's tenant.

**So: `--log-file <path>` and `--log-level <level>`, both persistent flags, both off by default.**
`--log-file` adds a file sink alongside stderr; `--log-level` sets the threshold for both. Neither has a default path and neither has a config key.

---

## Q3. Does `log.file` survive, as a wired key or deleted?

**Answer: deleted - and so is `log.level`. `LogConfig` goes whole.**

The three candidates were: wire both keys; delete both and use flags; delete `log.file` and wire `log.level`.

**Logging is a per-invocation choice, and the config file is for durable ones.**
Every other key in the schema describes the connection or the corpus - `atlassian.*`, `storage.data_dir`, `sync.*`. Those are true until you change them. Logging verbosity is true for as long as you are chasing one bug. Putting it in a config file means the user who turned on debug to diagnose a sync leaves it on forever, **which is precisely how a 766 MB file happens.** A flag is self-limiting by construction: it applies to the run you typed it on.

**Deleting both keys is also cheaper than deleting one.** With `LogConfig` gone the `log:` block leaves the schema entirely, and yaml.v3 ignores unknown keys - so a copied predecessor config carrying a `log:` block still parses. [Issue 13](https://github.com/tmheo/toolreach/issues/13) already depends on exactly that behaviour for the `confluence:` rename hint, so this costs nothing and adds no migration surface.

**The counter-argument I take seriously**, and reject: an organization might want debug permanently on for everyone. That is served by a shell alias or a wrapper script, and if it becomes a genuine ask, adding a key later is additive and safe. Removing a published key is not. A public tool should ship the smaller schema.

**Concretely, `internal/config/config.go` loses:**

- the `Log LogConfig` field (line 23)
- the `LogConfig` struct (lines 51-55)
- the `Log:` block in `DefaultConfig` (lines 67-70)

and `cmd/config.go` loses `log.level` and `log.file` from both `show` and `set`.

This is the **second** config-struct deletion in the port. [Issue 4](https://github.com/tmheo/toolreach/issues/4) already deletes `UpdateCheckConfig` from the same file. Both should land in the same commit as the `confluence:` to `atlassian:` rename, which [issue 2](https://github.com/tmheo/toolreach/issues/2) also puts there - three edits, one file, one commit.

### A consequence worth naming

`initLogger` (`cmd/root.go:32-58`) currently calls `config.LoadConfig(defaultConfigPath())` **on every single command invocation** - through `cobra.OnInitialize`, so `version`, `--help` and a mistyped subcommand all read and parse the config file - purely to find the data directory so it can compute a log path.

With no default file log, that read goes away. **Deleting the file log removes a config-file read and parse from the startup path of every command.**

It also removes an oddity: because `LoadConfig` errors on a missing file and `initLogger` falls back to stderr-only, a fresh install with no config file has **no file log at all**. The file log's existence today is silently conditional on having run `config set` at some point. Nobody documented that, and nobody would guess it.

---

## Q4. Does the file stay in the data directory at all?

**Answer: no - and this decision stands independently of Q2 and Q3, which is why it is worth separating.**

Even if the file log were kept on by default, it should not live in `~/.toolreach`.

**1. It is neither of the directory's two planes.** [Issue 13](https://github.com/tmheo/toolreach/issues/13) established the data directory as control files (config, tokens, site selection) plus corpus (`snapshots/`, `history/`, `data/`). The log is neither credentials nor corpus. It is the only thing in there that is neither - which is exactly how it became the single largest file in a directory whose other contents are 41 GB of cache and 665 MB of irreplaceable snapshots and history.

**2. Issue 13's migration rule excludes it by construction.** That ticket decided what crosses a migration by asking what cannot be re-derived. A log is re-derivable by repetition and worthless to carry - yet leaving it in the data directory means every `cp -r` of a directory users are explicitly told to copy carries it along.

**3. Measured permissions inconsistency.** Every other file in the live data directory is mode `0600` - `config.yaml`, `oauth_tokens.json`, `oauth_site.json`, both lock files. `workpulse.log` is **`0644`**, from `cmd/logger.go:20`, whose parent `MkdirAll` is `0755`.

Today the `0700` directory masks it. But `storage.data_dir` relocating the whole tree is a supported configuration, and the moment it points somewhere the directory mode does not protect, the one world-readable file is the one recording page titles and space keys.

**So: `--log-file` writes exactly where the user pointed, at `0600`, and toolreach never chooses a log location on its own.** No default path means no permissions question about a directory toolreach did not create, and nothing in the data directory that is not credentials or corpus.

---

## Q5. What else does this decision touch?

### It amends a row in issue 2's contract table

[Issue 2](https://github.com/tmheo/toolreach/issues/2) settled:

| Surface | Predecessor | toolreach |
|---|---|---|
| Log file | `~/.workpulse/workpulse.log` | `~/.toolreach/toolreach.log` |

**That row is removed.** There is no default log file, so there is no name to settle. If `--log-file` is given, the path is the user's.

This is the second amendment to that closed table this session - [issue 17](https://github.com/tmheo/toolreach/issues/17) added four run-time temp-path prefixes it did not name. Both are recorded here rather than edited into the closed ticket.

### It settles two items issue 14's rename manifest left contingent

[Issue 14](https://github.com/tmheo/toolreach/issues/14) flagged these as blocked on this ticket:

- `internal/config/config_test.go:21` - `assert.Equal(t, "workpulse.log", cfg.Log.File)`. **Deleted, not renamed**, along with the key it asserts.
- `cmd/logger_test.go:17,32` - two `filepath.Join(tmpDir, "workpulse.log")` fixtures. These test `newFileLogHandler` with an explicit path, which is exactly what `--log-file` does, so they **survive with their filename changed to anything at all** - the name stops being meaningful.

Issue 14's count of hand-edit sites is unchanged; two of its contingent entries now have dispositions.

### CONTEXT.md needs nothing

The domain glossary carries no logging term at 0.9.0 - no Log File, no Log Level, nothing in `_Avoid_` lists pointing at one. [Issue 8](https://github.com/tmheo/toolreach/issues/8)'s 54 terms and [issue 17](https://github.com/tmheo/toolreach/issues/17)'s 55th stand unchanged.

That absence is itself evidence for the decision: the log was never part of how this tool describes itself.

### One constraint the port inherits

`internal/sync/sync.go:509` emits `skipping page (no changes)` once per unchanged page. With no default file log this costs nothing on disk, but the moment someone runs `--log-level debug` to diagnose a sync, they get one line per page - 3,245 of them in the measured corpus - burying the line they need.

`page list fetch complete` and `page sync complete` already carry counts, so the aggregate is available. **The port should emit one skip summary per space rather than one line per page.** Recorded as a constraint rather than resolved here, since it is a logging-content question and this map plans rather than ports.

---

## The decision, assembled

1. **0.1.0 writes no file log by default.** stderr only, at Info.
2. **No rotation and no size cap**, because there is no default file to bound.
3. **`--log-file <path>` and `--log-level <level>`**, persistent flags, both off by default. `--log-file` adds a file sink alongside stderr; the file is opened `0600`.
4. **`LogConfig` is deleted whole** - `log.file` and `log.level` both, the struct, the default block, and both `cmd/config.go` cases. Second config-struct deletion in the port, landing with issue 4's `UpdateCheckConfig` and issue 2's `atlassian:` rename in one commit.
5. **Nothing log-shaped lives in the data directory.** toolreach never chooses a log location.
6. **Issue 2's log-file contract row is removed.**
7. **The per-page skip line becomes a per-space summary**, so `--log-level debug` stays readable.

## Defects found while measuring

At 0.9.0, all inherited unless fixed:

1. **`log.level` is dead.** Read only by `config show` and `config set`; both handler levels are hard-coded. The ticket's premise that it is wired is false.
2. **`config show` misreports it**, printing `level: info` for a file written at `debug`.
3. **The file handler is `LevelDebug` while stderr is `LevelInfo`**, so the highest-volume event in the tool is written to disk and never shown - the direct cause of 83% of a 766 MiB file.
4. **`workpulse.log` is mode `0644`** while every other file in the data directory is `0600`.
5. **`initLogger` discards the closer** returned by `newFileLogHandler` (`cmd/root.go:49`), so the handle is never explicitly closed. Harmless for a CLI, but it is the return value the function exists to provide.
6. **The file log's existence is silently conditional on a config file existing**, because `LoadConfig` errors on a missing file and `initLogger` falls back to stderr-only. Undocumented and unguessable.
7. **Every command invocation reads and parses the config file** through `cobra.OnInitialize(initLogger)`, including `version` and `--help`, solely to compute a log path.

## Not established here

- Whether the 6.4 million lines contain anything a public tool should not write even to a user-chosen path. The sample read for level and message distribution showed page ids, space keys and page titles, which is tenant content but not credential material; no systematic scan for secrets was performed.
- The exact age of the file, and therefore the true accumulation rate. The measurement is a single point, and the message mix shows Korean strings from releases predating [issue 12](https://github.com/tmheo/toolreach/issues/12)'s language state, so it spans an unknown number of versions.
- Whether any user of the predecessor has ever opened this file. Nothing in the tree reads it, but that is evidence about the code, not about people.
