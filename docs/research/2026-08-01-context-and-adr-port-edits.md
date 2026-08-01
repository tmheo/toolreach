# CONTEXT.md and docs/adr/ port edits

Resolves [What changes in CONTEXT.md when the domain stops assuming one tenant and one owner](https://github.com/tmheo/toolreach/issues/8).

Counted against predecessor `BINARY_VERSION` **0.9.0**, commit `d27c583`, 2026-08-01.
`CONTEXT.md` is 388 lines and defines **70 terms**.
`docs/adr/` holds **7 ADRs**.
Every count below should be re-run against the commit the port actually copies.

The headline: the domain language survives the port almost intact.
Of 70 terms, 18 leave with the removal manifest, 50 need no edit at all, 2 need a sentence changed, and 1 new term is added.
The two edited terms are **Grant** and **Atlassian Access Session**, which is exactly where the ticket expected the strain.

---

## 1. Grant holds, but it acquires a second failure cause it cannot see

The term itself is sound and needs no redefinition.
`Grant` is deliberately defined as *what the current user consented to*, in toolreach's own vocabulary rather than Atlassian's scope vocabulary.
Per-organization sharing changes who registered the app; it does not change what a user consented to.
So the term, its `_Avoid_` list, and all thirteen derived Grant terms carry over untouched.

What breaks is the **last sentence** of the definition, line 70:

> Every Grant is distinct from the maximum permissions configured for the Atlassian app, which bound what the user could consent to rather than what they did.

The sentence is still true.
It was written, however, when the person configuring the app and the person hitting the failure were the same person, so the bound it names was never a live failure cause.
Under a per-organization shared app the registrant is a third party to the user, and the bound becomes a real and separate way to be missing a Grant.

The two causes are **indistinguishable to toolreach**.
[ADR 0007](#7-per-adr-verdicts) establishes that `accessible-resources` is positive-only, so nothing before the request separates them.
[What an organization's first user must do in the Atlassian developer console](https://github.com/tmheo/toolreach/issues/3) establishes that every scope failure arrives as the same opaque `401 {"code":401,"message":"Unauthorized; scope does not match"}`, so nothing in the failure separates them either.
There is no third evidence source.

### Edit

Replace line 70 with:

> Every Grant is distinct from the scopes configured on the Atlassian App Registration, which bound what the user could consent to rather than what they did.
> Because that registration belongs to the user's organization rather than to toolreach, an absent Grant has two causes toolreach cannot tell apart: the user did not consent to it, or the registration never offered it.

### New term

The registration needs a name, because two edited definitions and the new ADR all refer to it.
Insert after **Grant**:

> **Atlassian App Registration**:
> The 3LO app one organization registered in its own Atlassian developer console, whose configured scope list bounds what any of that organization's users can consent to.
> toolreach neither owns it nor can inspect it: its `client_id`, `client_secret` and `base_url` reach toolreach by environment variable, and every colleague of the registrant shares one registration.
> Enabling its Distribution is what lets anyone but the registrant consent at all, the same organization included.
> _Avoid_: the Atlassian app, toolreach's app, OAuth client, Grant

### What this does not decide

Whether the recovery guidance now names both actions, and in what words, belongs to [Whether toolreach can name the permission a 401 is missing](https://github.com/tmheo/toolreach/issues/16).
This ticket fixes only that the vocabulary must be able to express both causes.

---

## 2. Atlassian Access Session keeps its mechanism and changes its reason

Line 110:

> Login is the one exception that still turns a user away in advance, and only over identity: it enumerates the sites the new consent reaches and refuses one that does not include the configured site.

Correct under the new model, and the mechanism is unchanged.
The reason widens.
[Issue 3](https://github.com/tmheo/toolreach/issues/3) found that cross-organization consent works and is gated entirely on the consenting side, so one registration's credentials can be carried to a consent in any Atlassian organization at all.
The refusal was written to catch the wrong site inside one company; it now also catches a consent given against an entirely different tenant.

### Edit

Append to line 110:

> Because an Atlassian App Registration's credentials can be carried to a consent in any Atlassian organization, this refusal now also catches a consent given against an entirely different tenant, not only the wrong site inside one.

Nothing else in the term changes.
The single-use lifecycle, the bounded token lock, the refresh ordering and the "asks no question about what the token may do" sentence are all independent of who owns the app.

---

## 3. The terms that leave with the removal manifest

Lines 283 to 388 are a clean terminal cut, as [Removal manifest for the self-upgrade and update-check chain](https://github.com/tmheo/toolreach/issues/4) recorded.
That is **18 terms** and **10 occurrences** of the predecessor name:

Binary Release Availability, Binary Release Channel, Binary Build Identity, Binary Distribution Contract, Verified Binary Release, Binary Release Manifest, Staged Binary Artifact, Binary Upgrade Lock, Binary Replacement Transaction, Binary Replacement Commit Point, Binary Upgrade Outcome, Binary Upgrade Failure, Binary Release Credential, Binary Upgrade, Binary Installation Ownership, Binary Installation, Update Availability Check, Release Availability Observation.

### One dangling reference, and only one

**Data Directory Lock**, line 166:

> _Avoid_: sync mutex, lock-file existence check, Binary Upgrade Lock

That `_Avoid_` entry exists to stop a reader collapsing two distinct locks.
After the cut there is only one lock left and the entry names nothing.
Replace with:

> _Avoid_: sync mutex, lock-file existence check

A scan of lines 1 to 282 finds no other reference into the cut region.

### No replacement term for version identity

[Distribution and release pipeline for a public repo](https://github.com/tmheo/toolreach/issues/7) deletes `BINARY_VERSION`, makes the git tag the single version source, and adds a `debug.ReadBuildInfo()` fallback for `go install` builds.
That is three possible origins for the version string, which invites a term along the lines of **Binary Build Identity**.

**Rejected.**
The predecessor's term existed because the upgrade chain *decided* on the classification: it refused to replace a development build, refused to coerce invalid metadata, and skipped the update check on both.
Nothing in toolreach decides anything from the version string; it is printed.
Naming it would be inventing vocabulary for machinery that does not exist, which is the failure mode the `_Avoid_` lists exist to prevent.

---

## 4. Renaming the predecessor

**40 occurrences** in `CONTEXT.md`, of which 10 sit inside the cut, leaving **30 surviving occurrences on 30 lines**.
**16 occurrences** across the ADRs: 0001 (3), 0002 (1), 0003 (4), 0004 (3), 0005 (0), 0006 (1), 0007 (4).

Almost all are plain prose substitution to `toolreach` under [Name toolreach's public surface](https://github.com/tmheo/toolreach/issues/2).
Four are not:

| Site | Note |
|---|---|
| `CONTEXT.md:1` | The `# workpulse` heading becomes `# toolreach`. |
| `CONTEXT.md:3` | See below. The whole sentence is rewritten, not substituted. |
| `CONTEXT.md:8, 96, 163` | "workpulse data directory" becomes "toolreach data directory", whose path issue 2 settled as `~/.toolreach`. |
| `CONTEXT.md:68` | `` `workpulse auth status` `` becomes `` `toolreach auth status` ``, a command string rather than prose. |
| ADR 0001, 0003 | `workpulse auth login` in the Consequences sections, also command strings, and both sit in sentences that need a separate fix. See section 7. |

### Line 3 is rewritten, not renamed

Current:

> workpulse bridges Atlassian Cloud content into local agent workflows.
> Its language keeps read-side sync, write-side page publication, and generated local wiki artifacts separate.

The second sentence names three surfaces and omits Jira entirely, even though Jira owns 8 of the 70 terms and its own ADR.
That was already wrong in the predecessor.
It matters more in a public repo, because this is the first sentence a stranger reads.

Replace with:

> toolreach bridges Atlassian Cloud content into local agent workflows.
> Its language keeps read-side Confluence sync, write-side Confluence page publication, generated local wiki artifacts, and the Jira read and write surfaces separate.

### One redaction the audit already caught

`CONTEXT.md:311` carries the real predecessor repository slug in the trusted-identity sentence, flagged by [Public-exposure audit of the predecessor source](https://github.com/tmheo/toolreach/issues/5).
It is inside the cut region, so the removal manifest disposes of it and no separate redaction step is needed.
`WORKPULSE_GITHUB_TOKEN` at line 356 is the only uppercase occurrence in either document and is likewise inside the cut.

---

## 5. The "company Mermaid app" phrasing is wrong in public, and hides a real assumption

**Mermaid Diagram Block**, line 208:

> ... that workpulse renders into the company Mermaid app's Confluence storage extension plus a Source Fallback.

"the company Mermaid app" was written for readers who all worked at one company that had one particular Marketplace app installed.
Read by a stranger it says "your company's Mermaid app", which is not what the code does.

What the code actually does is pin a specific third-party Forge app by hard-coded ARI in `internal/confluence/pagepublish/renderer_mermaid.go:13`:

```
ari:cloud:ecosystem::extension/23392b90-.../63d4d207-.../static/mermaid-diagram
```

The two UUIDs are the app id and its environment id.
Neither is tenant-scoped, so this is **not** an exposure finding, and the same ARI works in any tenant that has that same Marketplace app installed.
It is a hard dependency on an app most tenants will not have.

`renderer_drawio.go:155` has the same shape through `ac:name="drawio"`, which is a Marketplace app macro rather than a native Confluence one.
**Draw.io Diagram Asset** (line 212) and ADR 0002 both call it "the Confluence draw.io macro", which reads as native and is not.

### Edit

Rephrase both terms to name the dependency rather than assume it.
For **Mermaid Diagram Block**, "the company Mermaid app" becomes "a specific third-party Confluence Marketplace Mermaid app, pinned by extension identity".
For **Draw.io Diagram Asset**, "the primary Confluence draw.io macro" becomes "the primary draw.io Marketplace macro".
ADR 0002 takes the same wording change.

### What this does not decide

What toolreach *does* when the tenant lacks either app is a new question, filed as its own ticket.
This ticket fixes only that the language must not assert an app is present.

---

## 6. Which ADRs toolreach owns, and which it inherits

All seven come across.
None needs a superseding ADR.

| ADR | Relationship | Edits |
|---|---|---|
| 0001 Confluence Page Writes Use REST v2 Storage Bodies | toolreach **makes** this decision; the code embodies it | rename ×3, plus the false migration sentence |
| 0002 Draw.io Publish Uses Attachment-Backed Macros | **makes** | rename ×1, plus the Marketplace wording from section 5 |
| 0003 Jira Issue Writes Use Explicit Fields | **makes** | rename ×4, plus the false migration sentence |
| 0004 Page Publish Accepts Markdown Page Sources | **makes** | rename ×3 |
| 0005 Atlassian Access Uses Single-Use Current-Evidence Sessions | **inherits**, and is partly a record of a decision that was overturned | **none** |
| 0006 Live Page Read Uses Shared Storage Body Interpretation | **makes** | rename ×1 |
| 0007 Authorization Is Established By The Request That Needs It | **makes**, and is the load-bearing one | rename ×4, plus one line issue 11 owns |

### 0005 is kept, unchanged, and that is the deliberate call

0005 is the one ADR whose own header says its central decision no longer describes the system.
The obvious move is to drop it as the private history of a mistake a public reader cannot verify.

**Rejected.**
0007 is unreadable without it.
0007 opens with "ADR 0005 made every Atlassian command evaluate semantic Grants before opening its product gate", and its entire force is *here is why the obvious design is wrong and cannot be repaired*.
Deleting 0005 leaves 0007 arguing against a ghost, and the pair is the most valuable thing this port carries into a public repo: it is what stops a contributor rebuilding the preflight capability gate.

It also happens to contain **zero** occurrences of the predecessor name, so keeping it costs nothing.
The parts of it that still hold, named in its own header, are the single-use Session, the scope-free SiteSelection, the bounded token lock and the refresh ordering.

### The rename reaches inside the ADRs

Standard practice treats an accepted ADR as immutable, which argues for leaving the predecessor's name in place as a matter of historical record.

**Rejected.**
These are not toolreach's history.
Under the standing Code-transfer decision toolreach starts from one initial commit with no history, so these documents arrive as current design documentation rather than as a record of anything toolreach did.
A reader who meets "workpulse" in `docs/adr/0001` of a public repo has no repository to resolve the name against, and a name that resolves to nothing is worse than a rename.

The provenance is not lost: it is stated once, in the new ADR below, where a reader can actually find it.

### Two sentences that are false on day one

ADR 0001:

> Existing users must run `workpulse auth login` again if their stored token lacks `write:page:confluence`.

ADR 0003 carries the identical sentence for `write:jira-work`.

Both are wrong twice over in toolreach v0.1.
There are no existing users, and under a per-organization shared app re-authenticating cannot help when the shortfall is in the Atlassian App Registration rather than in the consent.

Rewrite each to state the requirement without the migration framing, keeping the useful fact, which is the scope name:

> Page Write requires `write:page:confluence` both on the Atlassian App Registration and in the user's consent.

and correspondingly for `write:jira-work` in 0003.

### ADR 0007 line 45 is issue 11's to fix

> The cost lands on first use after a release that adds scopes: the run fails on the request that needed the new grant rather than before it, and the release's Upgrade Action carries the instruction instead.

The Upgrade Action is a field of the Binary Release Manifest, which leaves with the removal manifest.
0007's Consequences therefore names a delivery mechanism toolreach will not have, and this is the only place in either document where the removal of the upgrade chain breaks a surviving argument rather than just deleting text.

This ticket does not rewrite it.
[How a scope-adding release tells existing users to re-authorize](https://github.com/tmheo/toolreach/issues/11) is exactly the question of what replaces the Upgrade Action, and its resolution should carry this edit.

Two further sentences in 0007's Consequences need re-verifying rather than rewriting:

> Only five Grants have recovery guidance naming them ... The other eight reach the user as the generic re-authentication guidance.

Five plus eight is thirteen Grants, which matches the thirteen Grant terms in `CONTEXT.md` at 0.9.0.
Per the map's predecessor-baseline note, the port should recount both against the commit it copies.

---

## 7. One new ADR, and only one

### 0008 toolreach Uses A Per-Organization Shared Atlassian App

The auth model is the least obvious decision on the map and the only standing decision with no ADR behind it.
It also changes how **Grant** reads, which is a domain-language consequence, and that is what an ADR is for.

Draft:

> # toolreach Uses A Per-Organization Shared Atlassian App
>
> toolreach owns no Atlassian OAuth app.
> An organization's first user registers a 3LO app in that organization's own Atlassian developer console, enables Distribution, and shares its `client_id`, `client_secret` and `base_url` with colleagues, who pass them by environment variable.
> The ldflags build-defaults path stays in the code, and public releases ship it empty.
>
> **Status**: accepted
>
> **Considered Options**
>
> - toolreach registers one public Atlassian app and every user consents to it.
> - Every user registers their own app.
> - One user per organization registers an app and shares its credentials.
>
> **Consequences**
>
> A toolreach-owned app would make the maintainer a party to every user's tenant access and would put a Marketplace review between the project and its users; neither is a cost a CLI of this kind should carry.
> A per-user app makes every colleague repeat the developer console setup, which [issue 3](https://github.com/tmheo/toolreach/issues/3) measured at 18 scopes across two products.
>
> The chosen option makes the app's registrant a third party to the user hitting a failure.
> An absent Grant therefore has two causes toolreach cannot distinguish: the user did not consent, or the registration never offered the scope.
> Nothing in `accessible-resources` (ADR 0007) or in Atlassian's `401` body separates them.
>
> Distribution must be enabled on the registration before anyone but the registrant can consent, the same organization included, and the consent screen carries an unavoidable "not reviewed by Atlassian" warning with no same-organization exception.
> Adding a scope to a release requires both a change on the registration by whoever holds it and a fresh consent by every user.
>
> One callback URL per registration, exact-match including path.
>
> This repository's `CONTEXT.md` and ADRs 0001 to 0007 were carried over from a private predecessor project and renamed; toolreach starts from one initial commit and does not share history with it.

### No ADR for the release pipeline

[Issue 7](https://github.com/tmheo/toolreach/issues/7) produced a substantial decision with recorded trade-offs, which argues for an ADR.

**Rejected.**
Every existing ADR is about how the product behaves against Atlassian or how modules are cut, and the predecessor gave no ADR even to its far more elaborate signed-manifest release chain.
Build and CI sit outside this repository's ADR boundary, and the issue 7 resolution plus the pipeline files are the record.

### No ADR for the config key rename

`confluence:` to `atlassian:` is a rename with no rejected alternative worth recording.
One line in the new ADR or in `CONTEXT.md` carries the reason, which is that one registration already serves both products.

### No superseding ADR for the removed upgrade chain

There is nothing to supersede.
The chain never had an ADR, and [issue 4](https://github.com/tmheo/toolreach/issues/4) is its record.

---

## Summary of edits

**CONTEXT.md**

1. Line 1: heading to `# toolreach`.
2. Line 3: rewrite to name the Jira surfaces.
3. Line 70: replace the app-permissions sentence, naming the two indistinguishable causes.
4. After **Grant**: insert **Atlassian App Registration**.
5. Line 110: append the cross-organization clause.
6. Line 166: drop `Binary Upgrade Lock` from the `_Avoid_` list.
7. Line 208 and line 212: name the Marketplace apps rather than assume them.
8. Lines 283 to 388: delete, 18 terms.
9. 30 remaining prose occurrences of the predecessor name to `toolreach`.

Result: 53 terms, of which 52 come from the predecessor and 1 is new.

**docs/adr/**

1. 16 name occurrences to `toolreach` across 0001, 0002, 0003, 0004, 0006, 0007.
2. 0001 and 0003: replace the "existing users must run auth login again" sentence.
3. 0002: Marketplace wording for draw.io.
4. 0005: no edit.
5. 0007 line 45: deferred to issue 11.
6. New `0008-toolreach-uses-a-per-organization-shared-atlassian-app.md`.
