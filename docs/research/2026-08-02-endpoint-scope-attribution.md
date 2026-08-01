# Naming the permission a scope 401 is missing

## What this answers

This is the findings asset for GitHub issue #16, "Whether toolreach can name the permission a 401 is missing".

The standing Onboarding decision promises "a scope-shortfall failure that names the permission that is missing".
Atlassian returns the same opaque body for every such failure, so keeping that promise means toolreach carries the knowledge itself.
This asset measures what carrying it costs, and reaches a verdict.

Everything counted here was counted against the predecessor at `~/workspace/github/workpulse`, `BINARY_VERSION` **0.9.0**, commit `1108c9f8e295865d49cdfd0b10594b52c75ab055` (2026-08-01), on branch `main`.
The port must re-run these counts against the commit it actually copies.

No Atlassian documentation was fetched while writing this.
Every scope requirement quoted below is sourced from `docs/research/2026-08-01-atlassian-3lo-console-setup.md`, which did the primary-source reading for issue #3 and carries the URLs.
Rows that asset does not cover are graded and named as ungraded, not guessed at silently.

---

## 1. The call surface, counted

### 1a. What actually talks to Atlassian

At 0.9.0 exactly four production files construct an outbound HTTP request:

| file | requests | note |
| --- | --- | --- |
| `internal/confluence/client.go` | 11 request-issuing methods | Confluence v2 and two v1 attachment endpoints |
| `internal/jira/client.go` | 14 request-issuing methods | Jira platform v3 |
| `internal/atlassianaccess/session.go` | 1 | `GET /oauth/token/accessible-resources` |
| `internal/github/releases.go` | 2 | GitHub, not Atlassian, and leaves with the removal manifest (issue #4) |

Token exchange and refresh go through `golang.org/x/oauth2` against `https://auth.atlassian.com/oauth/token`, so they issue requests without appearing in that grep.

**27 Atlassian-facing request-issuing call sites**: 11 Confluence + 14 Jira + accessible-resources + the token endpoint.

`DownloadAttachment` delegates to `DownloadAttachmentTo` and is not counted separately; `DownloadAttachmentTo` issues **two** requests (a v2 lookup and a v1 download) and is counted once as a call site but twice as a URL template.

### 1b. Distinct URL templates

**34 concrete URL templates**: 18 Confluence, 14 Jira, 2 platform.

The 18 Confluence templates come from 11 methods because two methods are polymorphic over the five Hierarchy Content Types (`pages`, `folders`, `databases`, `embeds`, `whiteboards`), from `HierarchyContentType.DirectChildrenPath()` at `internal/confluence/types.go:145-160`:

- `WalkDirectChildren` → `/api/v2/{collection}/{id}/direct-children`, five collections.
- `GetHierarchyContent` → `/api/v2/{collection}/{id}`, the same five.

`GetPageDetails` and `GetFolder` are separate call sites that hit two of those same five templates, which is why 11 methods produce 18 templates rather than 11 + 8.

---

## 2. Endpoint-to-scope mapping

### Evidence grades

- **A** - the issue #3 asset quotes the scope requirement for *this exact endpoint*, with a source URL.
- **B** - the asset quotes the requirement for the *API group* this endpoint belongs to, but not the endpoint itself.
- **C** - neither. The scope named is inferred from the product's scope semantics and toolreach's requested set. **Not evidence.**

### 2a. Confluence - 18 templates

| # | method | template | scope the reference names | grade | ambiguous? |
| --- | --- | --- | --- | --- | --- |
| 1 | `GetSpaces` | `GET /api/v2/spaces` | `read:space:confluence` | B | no |
| 2 | `GetSpace` | `GET /api/v2/spaces/{id}` | `read:space:confluence` | B | no |
| 3 | `GetSpacePages` | `GET /api/v2/spaces/{id}/pages` | `read:page:confluence` | A | no |
| 4 | `WalkDirectChildren` | `GET /api/v2/pages/{id}/direct-children` | `read:hierarchical-content:confluence` | B | **yes** |
| 5 | `WalkDirectChildren` | `GET /api/v2/folders/{id}/direct-children` | `read:hierarchical-content:confluence` | B | **yes** |
| 6 | `WalkDirectChildren` | `GET /api/v2/databases/{id}/direct-children` | `read:hierarchical-content:confluence` | B | **yes** |
| 7 | `WalkDirectChildren` | `GET /api/v2/embeds/{id}/direct-children` | `read:hierarchical-content:confluence` | B | **yes** |
| 8 | `WalkDirectChildren` | `GET /api/v2/whiteboards/{id}/direct-children` | `read:hierarchical-content:confluence` | B | **yes** |
| 9 | `GetPageDetails`, `GetHierarchyContent(page)` | `GET /api/v2/pages/{id}` | `read:page:confluence` | A | no |
| 10 | `GetFolder`, `GetHierarchyContent(folder)` | `GET /api/v2/folders/{id}` | `read:folder:confluence` | B | no |
| 11 | `GetHierarchyContent(database)` | `GET /api/v2/databases/{id}` | `read:database:confluence` | B | no |
| 12 | `GetHierarchyContent(embed)` | `GET /api/v2/embeds/{id}` | `read:embed:confluence` | B | no |
| 13 | `GetHierarchyContent(whiteboard)` | `GET /api/v2/whiteboards/{id}` | `read:whiteboard:confluence` | B | no |
| 14 | `CreatePage` | `POST /api/v2/pages` | `write:page:confluence` | A | no |
| 15 | `UpdatePage` | `PUT /api/v2/pages/{id}` | `write:page:confluence` | A | no |
| 16 | `DownloadAttachmentTo` | `GET /api/v2/pages/{id}/attachments` | `read:attachment:confluence` | B | no |
| 17 | `DownloadAttachmentTo` | `GET /rest/api/content/{id}/child/attachment/{aid}/download` | classic `readonly:content.attachment:confluence` **or** granular `read:attachment:confluence` | A | **yes** |
| 18 | `UploadAttachment` | `PUT /rest/api/content/{id}/child/attachment` | granular `read:content-details:confluence` **and** `write:attachment:confluence` (classic alternative `write:confluence-file` is **not requested**) | A | **yes** |

Rows 4-8 are marked ambiguous for a specific reason.
The issue #3 asset records `read:hierarchical-content:confluence` for the v2 *children* and *descendants* groups, and separately records `read:page:confluence` for `GET /pages/{id}/children` in the *page* group.
The same URL therefore appears in two API groups with two different documented requirements, and whether a `direct-children` call on a database also needs `read:database:confluence` is not established.
This is exactly the failure mode the ticket warns about: a table that confidently prints one scope here has a real chance of printing the wrong one.

Row 18 is ambiguous in the other direction - two scopes are required *together*, so a 401 has two candidates and the body distinguishes neither.

### 2b. Jira - 14 templates

toolreach's Jira set is classic (`read:jira-work`, `read:jira-user`, `write:jira-work`) plus one granular scope.
Every Jira v3 endpoint documents a classic scope and a granular alternative set; since toolreach holds the classic scopes, the classic column is the one a table would print.

| # | method | template | classic scope | grade |
| --- | --- | --- | --- | --- |
| 1 | `GetIssue` | `GET /rest/api/3/issue/{key}` | `read:jira-work` | C |
| 2 | `SearchIssues` | `GET /rest/api/3/search/jql` | `read:jira-work` | C |
| 3 | `GetProjects` | `GET /rest/api/3/project` | `read:jira-work` | **A** |
| 4 | `CreateIssue` | `POST /rest/api/3/issue` | `write:jira-work` | C |
| 5 | `UpdateIssue` | `PUT /rest/api/3/issue/{key}` | `write:jira-work` | C |
| 6 | `CreateIssueLink` | `POST /rest/api/3/issueLink` | `write:jira-work` | C |
| 7 | `AddIssueComment` | `POST /rest/api/3/issue/{key}/comment` | `write:jira-work` | C |
| 8 | `GetIssueTransitions` | `GET /rest/api/3/issue/{key}/transitions` | `read:jira-work` | C |
| 9 | `TransitionIssue` | `POST /rest/api/3/issue/{key}/transitions` | `write:jira-work` | C |
| 10 | `GetIssueTypes` | `GET /rest/api/3/issue/createmeta/{key}/issuetypes` | `read:jira-work` | C |
| 11 | `GetIssueFields` | `GET /rest/api/3/issue/createmeta/{key}/issuetypes/{id}` | `read:jira-work` | C |
| 12 | `GetIssueEditFields` | `GET /rest/api/3/issue/{key}/editmeta` | `read:jira-work` | C |
| 13 | `GetCurrentUser` | `GET /rest/api/3/myself` | `read:jira-user` | C |
| 14 | `SearchAssignableUsers` | `GET /rest/api/3/user/assignable/search` | `read:jira-user` | C |

**13 of 14 Jira rows are grade C.**
That is the honest state of the mapping in hand: for almost the whole Jira surface, nobody has read the reference page.
Producing a grade-A Jira table means opening 13 endpoint reference pages, and re-opening them whenever Atlassian revises a requirement.

### 2c. Platform - 2 templates

| method | template | scope |
| --- | --- | --- |
| `atlassianaccess.Session` | `GET https://api.atlassian.com/oauth/token/accessible-resources` | none; a valid bearer token is the whole requirement |
| `golang.org/x/oauth2` | `POST https://auth.atlassian.com/oauth/token` | none; `offline_access` governs whether a refresh token exists at all |

### 2d. Counts the ticket asked for

- **34** distinct URL templates, of which **32** can produce a scope 401 (the two platform endpoints cannot).
- **7 of 32 map to more than one scope** (5 direct-children variants, the v1 attachment download's classic-or-granular pair, the v1 attachment upload's required granular pair) - **22%**.
- **13 of 32 are grade C** - the scope named is an inference, not a sourced fact.
- Distinct scopes a table would ever print: **12** (`read:space`, `read:page`, `read:folder`, `read:database`, `read:embed`, `read:whiteboard`, `read:hierarchical-content`, `read:attachment`, `readonly:content.attachment`, `read:content-details`, `write:page`, `write:attachment` on the Confluence side; `read:jira-work`, `read:jira-user`, `write:jira-work` on the Jira side - 15 in total).

### 2e. Two of the 18 requested scopes are printed by nothing

`read:content:confluence` - issue #3 already established it is required by no endpoint reference page scanned.
This mapping confirms it from the other side: no call site toolreach makes would ever cause a table to print it.

`read:project:jira` - the ticket's own premise, and it holds.
The project endpoints document a five-scope granular set (`read:issue-type`, `read:project`, `read:project.property`, `read:user`, `read:application-role`) as the alternative to classic `read:jira-work`.
toolreach requests **one** of the five, so the granular path can never be satisfied and every project call succeeds, if it succeeds, on `read:jira-work`.
`read:project:jira` cannot be the reason any call works.
A table that printed it for a project 401 would name a permission whose presence or absence changes nothing.

So **2 of the 18 requested scopes are non-load-bearing**, and both are exactly the kind of thing a naming table gets wrong.

---

## 3. What the predecessor actually does - and it is not what the domain says

This is the decisive evidence, and it is all inside the predecessor rather than inside Atlassian's docs.

### 3a. The claim

`CONTEXT.md` defines **13 concrete Grants** plus the base term **Grant**.
Of the 13, exactly **4** say "A missing one has recovery guidance naming it":

- Hierarchy Read Grant
- Non-page Metadata Read Grant ("naming the exact content types that were refused")
- Issue Write Grant
- Jira Issue Read Grant

The other **9** say "Nothing names it, so a missing one reaches the user as the generic re-authentication guidance": Space Read, Page Read, Page Write, Published Attachment Write, Jira Project Metadata Read, Jira User Discovery, Jira Comment Write, Jira Issue Link Write, Jira Transition Write.

So the domain language already narrows the naming promise to **4 of 13**, or 31%.

### 3b. What the binary delivers

`internal/oauth/resources.go` holds six guidance strings. Tracing each to its production call site:

| symbol | production call site | reachable? |
| --- | --- | --- |
| `MissingHierarchyReadGrantMessage` | `internal/sync/hierarchy.go:328` | **yes**, and correct |
| `InvalidNonPageMetadataAuthenticationMessage` | `internal/sync/parent_resolution.go:398` | **yes**, and wrong - see 3c |
| `MissingNonPageMetadataReadGrantMessage(missing []string)` | none | **dead code**. Referenced by no production file and by no test file at 0.9.0 |
| `MissingJiraIssueWriteGrantMessage` | `cmd/jira.go:556` | **unreachable** - see 3d |
| `MissingJiraReadGrantMessage` | `cmd/jira.go:558` | **unreachable** - see 3d |
| `ReauthenticateMessage` | `hierarchy.go:330`, `parent_resolution.go:402` | yes; this is the generic fallback |

**One** of the four claimed Grants reaches a user with an accurate name.

### 3c. The precise namer was built, then not used - and the stand-in names the wrong thing

`MissingNonPageMetadataReadGrantMessage(missing []string)` exists precisely to satisfy the Non-page Metadata Read Grant's promise to name "the exact content types that were refused".
It is called by nothing.

What is wired instead, at `parent_resolution.go:395-403`, is a constant whose text is:

> `Confluence 상위 항목 정보 읽기 인증이 거부되었습니다. 확인 대상: database, embed, whiteboard.`

It names a **fixed** three-type list.
The routes actually probed come from `parentProbeOrder` at `parent_resolution.go:19-25`, which is **five** types in this order:

```
folder, page, database, embed, whiteboard
```

`parentRoutes` walks them in order and returns on the first 401.
So on the common path the refusal comes from **folder**, the first probe - and the message names a set that does not contain it.

This is the ticket's own test case, live in the predecessor at 0.9.0: *a table that names the wrong permission is worse than one that names none.*
The predecessor has one, and it is the only place in the tree that tries to be precise about which permission failed.

Two further defects in the same six lines:

- The doc comment on `InvalidNonPageMetadataAuthenticationMessage` says it is emitted "when Atlassian rejects the authentication on a typed metadata request **rather than** reporting a scope mismatch".
  The call site is the `if scopeMismatch` branch.
  The comment states the opposite of the code.
- The symbol is named `Invalid...AuthenticationMessage` while occupying the scope-mismatch slot, which is how the comment came to be wrong.

### 3d. Both Jira namers are unreachable in production

`cmd/jira.go:552-560` selects a named message when the error is `*issuewrite.GrantError`.

`GrantError` is constructed in exactly two places at 0.9.0, `internal/jira/issuewrite/create_test.go:275` and `update_test.go:218`.
**No production file constructs one.**

It can only arrive through the `Authorizer` port (`internal/jira/issuewrite/authorization.go:13`), whose production implementation is `jiraIssueWriteAccess` in `cmd/jira.go` wrapping the `atlassianaccess` session.
That session performs the accessible-resources call and site resolution; it has no way to know which Jira Grant a later call will need, and it does not import `issuewrite`.

So `MissingJiraIssueWriteGrantMessage` and `MissingJiraReadGrantMessage` are strings a user cannot see.
The port would carry a named-permission branch that no production path enters.

### 3e. Outside `sync`, the scope-mismatch signal is discarded entirely

`AuthError.ScopeMismatch()` is consumed at exactly **three** production sites: `hierarchy.go:327`, `parent_resolution.go:343`, `parent_resolution.go:369`.
All three are in `internal/sync`.

That leaves **24 of the 27** Atlassian-facing call sites that never ask the question.
A scope 401 from `GetSpaces`, `CreatePage`, `UploadAttachment`, `GetIssue`, `SearchIssues` or any Jira write reaches the user as `AuthError.Error()`:

> `authentication failed: please run 'workpulse auth login' to authenticate with Atlassian`

which is the *revoked token* message.
Not merely unnamed - actively misdirecting, since re-authenticating cannot fix a registration that never offered the scope.

### 3f. Score

| | claimed | delivered |
| --- | --- | --- |
| Grants with a naming promise | 4 of 13 (31%) | - |
| Grants whose name reaches a user correctly | - | **1 of 13 (8%)** |
| Grants whose namer prints a set excluding the likely culprit | - | 1 |
| Grants whose namer is unreachable or dead | - | 2 dead-or-unreachable symbols covering 2 Grants |
| Call sites that even ask whether it was a scope failure | - | 3 of 27 (11%) |

---

## 4. Verdict

**No endpoint-to-scope table, at any size. Name the Grant at the call site, and carry a 13-row Grant-to-scope map beside `scopes.txt`.**

### 4a. Why no table

The ticket's framing inherits Atlassian's framing, and Atlassian's framing does not apply to toolreach.

Atlassian's documented diagnosis method - "check the OAuth scopes required field in the relevant API documentation" - is written for someone holding a 401 and *not* holding the code that produced it.
They have to work backwards from a URL to a requirement, so they need a lookup table.

toolreach is never in that position.
At the moment the 401 arrives, control is inside `GetFolder`, or inside `UploadAttachment`, or inside `TransitionIssue`.
The endpoint is not an unknown to be recovered - it is a compile-time constant three lines up.
A table keyed on endpoints would be toolreach looking up a fact it already has.

Everything the ticket worries about is a property of the table, not of the problem:

- **Rot.** A table rots because Atlassian revises requirements. A per-call-site constant naming a *Grant* does not, because a Grant is toolreach's word for a capability and survives Atlassian re-partitioning its scopes.
- **The missing entry.** Cannot occur. There is no lookup, so there is nothing to miss. A new call site that names no Grant gets the generic message, which is a correct degradation rather than a gap.
- **Ambiguity.** The seven ambiguous templates in section 2 stop being ambiguous, because the Grant is what gets named. `UploadAttachment` needs two scopes together, but it needs exactly one **Published Attachment Write Grant**, and that is the sentence the user needs.
- **Maintenance.** 34 rows of someone else's documentation becomes 13 rows of toolreach's own vocabulary, which a contributor edits in the same commit as the code.

### 4b. What replaces it

Three pieces, none of them a table of endpoints.

**1. A Grant constant at every call site.**
One `oauth.Grant` value per Confluence and Jira client method - 25 assignments, from a closed set of 13.
The 401 handling collapses to one helper:

```
func ScopeShortfall(grant Grant, err error) error
```

The zero Grant yields the existing generic guidance, so a call site added later and left unannotated degrades correctly rather than lying.
This also fixes 3e: all 27 call sites route through one seam instead of 3 of 27.

**2. A 13-row Grant-to-scope map, authored beside `scopes.txt`.**
Issue #11 already lifts the 18-scope literal into `internal/oauth/scopes.txt` behind `//go:embed`.
The map lives next to it and says which scope literal each Grant corresponds to.
This is the only place a scope string appears in prose, and it is 13 rows a human wrote about toolreach's own capabilities, not 34 rows transcribed from a vendor.

Its rot surface is guarded for free: issue #11's release-time scope diff is already reading `scopes.txt`, so it can also assert **every scope in `scopes.txt` is claimed by at least one Grant**.
That check, run today, fails on `read:content:confluence` and `read:project:jira` - the two non-load-bearing scopes in section 2e.
The port should either drop them or record why they are requested.

**3. The three-state message, which is what the user actually needed.**
The Grant name alone does not resolve the two-cause problem issue #8 raised: a named permission still leaves the reader unsure whether to re-consent or go find the registrant.
Issue #11's **Consent Scope Record** closes it, and the Grant-to-scope map is exactly the index that makes the record answerable per failure:

| state | evidence | message |
| --- | --- | --- |
| the Grant's scope is absent from the Consent Scope Record | local, exact | your consent predates this capability - run `toolreach auth login` |
| the Grant's scope is present in the record and the 401 still fired | local, exact | your consent is current, so the **Atlassian App Registration** is missing this scope - the person who registered the app must add it |
| no Grant assigned at this call site | - | the existing generic re-authentication guidance |

Without a per-Grant scope map, the Consent Scope Record can only answer "which scopes did this release add", the upgrade case.
With it, the record answers "was *this* failure's scope ever consented to", which is the whole question.
That makes 13 rows load-bearing for issue #11 rather than optional decoration for issue #6, and is the strongest argument for keeping the map at 13 rows and not at 0.

### 4c. What this costs the standing Onboarding decision

The promise is kept, narrowed once, and made honest.

Kept: every scope 401 names a capability in toolreach's own words and says who has to act.
Narrowed: it names a **Grant**, not an Atlassian scope string, in the sentence a human reads; the scope literal appears as supporting detail from the map, so a registrant can search the console for it.
Honest: it no longer promises precision the evidence cannot support, which is where the predecessor's non-page-metadata message went wrong.

`toolreach auth setup` - which issue #11 already made the durable source of truth for the scope checklist - is where the full 18-scope list belongs.
It should print the Grant each scope serves, generated from the same map, so the registrant sees what they are turning on rather than a list of vendor strings.

---

## 5. Follow-ups this hands to other tickets

- **Issue #6** (onboarding flow) owns the exact wording of the three-state message and where it appears.
- **Issue #11** owns `scopes.txt` and the release-time diff; this asset adds one assertion to that gate and one file beside it.
- **Issue #12** already established `internal/oauth/resources.go` is rewritten in English by whoever owns its content; this asset is that content's shape.

## 6. Defects found while measuring

All at 0.9.0, all inherited by the port unless fixed:

1. `MissingNonPageMetadataReadGrantMessage` is dead code - no production or test reference.
2. `InvalidNonPageMetadataAuthenticationMessage` names `database, embed, whiteboard` while `parentProbeOrder` probes `folder, page, database, embed, whiteboard`, so it omits the two types probed first.
3. That symbol's doc comment says it fires when Atlassian is **not** reporting a scope mismatch; the call site is the scope-mismatch branch.
4. `issuewrite.GrantError` is constructed only in tests, making both Jira named messages unreachable in production.
5. 24 of 27 Atlassian call sites present a scope 401 as the revoked-token message.
6. `read:content:confluence` and `read:project:jira` are requested but cannot be the reason any call succeeds.

## 7. Not established here

- Grade-C rows: 13 of 14 Jira endpoint scope requirements were not read from Atlassian's reference for this ticket. They are inferences from the classic scope descriptions issue #3 quoted.
- Whether `direct-children` on a non-page container requires the container's type scope in addition to `read:hierarchical-content:confluence`.
- Whether Atlassian accepts a partially-satisfied granular set alongside a classic scope, which is what would decide whether `read:project:jira` is merely useless or actively confusing.
