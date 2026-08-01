# What toolreach assumes about the Mermaid and draw.io Marketplace apps

## What this answers

This is the decision record for GitHub issue #17, a grilling ticket.
It was run as a self-grilling: each question posed alone, answered with the strongest recommendation available, and the reasoning shown.

Everything counted or quoted below is from `~/workspace/github/workpulse` at `BINARY_VERSION` **0.9.0**, commit `1108c9f8e295865d49cdfd0b10594b52c75ab055` (2026-08-01).

**Evidence grades.** Claims about what the predecessor *emits* are read from source and marked **[code]**.
Claims about what Confluence *does* with that markup in a tenant lacking the app are marked **[inferred]** - no Atlassian documentation was fetched and no live tenant was touched, per the standing constraint on outward-facing actions.
The decision below is built so that it does not depend on any **[inferred]** claim being right.

---

## The shape of the surface

Page Publish registers **four** placeholder renderers (`internal/confluence/pagepublish/renderer.go:64-71`): `wp:mermaid`, `wp:drawio`, `wp:attachment`, `wp:image`.

Two of the four emit a Marketplace-app macro. The other two emit `<ac:image>` and attachment links, which are native Confluence storage elements present in every tenant.

So the blast radius is exactly two placeholders, and the question is what those two do in a tenant that has neither app - which, under the standing destination, is most tenants.

---

## Q1. What does a publish into a tenant without the app actually produce?

**Answer: two completely different outcomes, and the difference is decided by what the renderer emits, not by what Confluence does with it.**

### Mermaid **[code]**

`renderMermaidExtension` (`renderer_mermaid.go:160-164`) emits **two siblings**:

```
<ac:adf-extension>
  <ac:adf-node type="extension"> ... extension-key = <app-uuid>/<env-uuid>/static/mermaid-diagram ... </ac:adf-node>
  <ac:adf-fallback> ...the same node again... </ac:adf-fallback>
</ac:adf-extension>
<ac:structured-macro ac:name="expand">
  <ac:structured-macro ac:name="code"><ac:parameter ac:name="language">mermaid</ac:parameter>
    <ac:plain-text-body><![CDATA[ the full diagram source ]]></ac:plain-text-body>
  </ac:structured-macro>
</ac:structured-macro>
```

`expand` and `code` are **native** Confluence macros.
They are emitted **unconditionally** - `renderMermaidFallback(block)` is appended after `renderMermaidExtension` on every render, not only when something fails.

So a tenant without the Mermaid app still receives the whole diagram source, in a labelled collapsible block, rendered by macros every Confluence has.
**The content survives. Only the picture is lost.**

One defect noticed in passing: `<ac:adf-fallback>` is handed *the same node it is a fallback for* (`renderer_mermaid.go:162` passes `node` twice).
The ADF fallback slot therefore contributes no degradation at all.
It does not matter today, because the real fallback is the sibling `expand`, but it means the extension's own fallback mechanism is unused. **[code]**

### Draw.io **[code]**

`renderDrawioMacro` (`renderer_drawio.go:143-180`) emits **one** element:

```
<ac:structured-macro ac:name="drawio"> ...16 parameters... </ac:structured-macro>
```

No sibling. No native element. Nothing else in the body.

Two files *are* uploaded to the page as attachments (`renderer_drawio.go:51-70`):

- the `.drawio` source, as `drawioAttachment`
- **a PNG preview, as `imageAttachment`** - whenever `preview-source-file` is set

And in the Markdown Page Source path the PNG is **always** set: `renderMarkdownImage` (`renderer_markdown.go:276-284`) only produces a `wp:drawio` placeholder *because* it found a PNG image link with a same-basename `.drawio` beside it, and it writes all three attributes:

```go
fmt.Fprintf(b, `<wp:drawio source-file="%s" title="%s" preview-source-file="%s" />`, ...)
```

**So the fallback image already exists, is already validated, is already uploaded to the page - and is never referenced in the body.**

The element needed to reference it already exists too: `renderer_file.go:150` is a one-line `<ac:image><ri:attachment ri:filename="..."/></ac:image>` emitter used by the `wp:image` renderer.

The two are simply not connected.

### What Confluence shows **[inferred]**

A tenant without the Mermaid app renders an unresolvable `com.atlassian.ecosystem` extension - most likely a blank or "app not available" block - followed by the working source expand.

A tenant without a draw.io app renders Confluence's standard unknown-macro error box, and the reader sees an error where a diagram should be, with the picture that would have shown it sitting unlinked in the page's attachment list.

**These inferences are not load-bearing.**
The asymmetry is already fully established from source: one renderer emits a native sibling and one does not.
Whatever Confluence does with an unknown macro, "readable source in a native macro" is strictly better than "nothing", and that comparison needs no external evidence.

---

## Q2. Is this one question or two?

**Answer: two, and conflating them is exactly what produced the asymmetry.**

The ticket suspected this and it holds up.

**Mermaid is a vendor-identity question.**
`mermaidExtensionKey` (`renderer_mermaid.go:13`) pins one app by app id and environment id.
Another vendor's Mermaid app has different UUIDs, so the pin either matches or it does not - there is no partial match. The question is *which app*.

**Draw.io is a degradation question.**
`ac:name="drawio"` pins no vendor at all. It is a macro *name*, and any Marketplace app that registers that name satisfies it.
Draw.io's problem is not that it names the wrong app; it is that when no app answers the name, nothing happens. The question is *what happens when there is no app*.

Treating them as one question is what let draw.io ship without a fallback: Mermaid's pin was obviously fragile so it got a fallback, draw.io's macro name looked robust so it did not.
But robustness of *identity* and robustness of *outcome* are unrelated. Draw.io's macro name is more portable than Mermaid's ARI **and** its failure is worse.

The two need different fixes, and only one of them is about configuration.

---

## Q3. Does v0.1 detect the absence, and can it?

**Answer: no, and it should not try - because detection is the wrong axis.**

Four reasons, in descending order of how much they settle the matter.

**1. Detection buys nothing once degradation is unconditional.**
Detection is only worth paying for if the tool would *behave differently* knowing the answer. If both renderers always emit a native sibling, a tenant without the app already gets a correct, readable page. There is no branch to take. The strongest form of this answer is that the question dissolves rather than resolves.

**2. There is no error to catch.**
A publish that renders an unknown macro **succeeds at the API level**. Confluence stores storage-format XHTML without validating macro names - which is precisely why the predecessor can emit `ac:name="drawio"` into a tenant that has never heard of it. There is no non-2xx to observe, so no existing failure path grows a new branch. **[inferred, but strongly implied by the code working at all]**

**3. The API that would answer it is out of reach, and reaching for it is disproportionate.**
Listing installed apps means the UPM / plugins administration API, which requires Confluence *admin*, and toolreach's 18 scopes contain nothing near it.
Adding an admin scope so the tool can name a diagram vendor is grotesque on its own terms, and [issue 11](https://github.com/tmheo/toolreach/issues/11) makes it worse: every added scope is an organization-level migration, gating a re-consent by every user, plus a registrant console change. Paying that to improve a diagram's error message is not a trade anyone would take.

**4. The read-back alternative rots.**
Fetching the published page with a rendered body format and grepping for an error marker is a second round trip per publish, depends on undocumented rendered-HTML strings, and would break silently when Atlassian restyles its error box. It is the shape of check that looks clever and is unmaintainable - the same reason [issue 16](https://github.com/tmheo/toolreach/issues/16) refused an endpoint-to-scope table.

**What replaces detection.** Documentation, in one sentence: `toolreach auth setup` and the publish command's help say which Marketplace apps upgrade the rendering, and that their absence degrades rather than fails. That is the whole of the user-facing treatment.

---

## Q4. Is the pinned identity configuration, or a deliberate constant?

**Answer: a deliberate constant, unconfigurable in v0.1 - and the fallback is what makes that acceptable.**

The obvious generous answer is a config key, `publish.mermaid_extension_ari` or an environment variable. I considered it and it is a trap.

**The value is undiscoverable.**
It is an opaque `ari:cloud:ecosystem::extension/<app-uuid>/<env-uuid>/static/mermaid-diagram`. Nobody can fill that in from a Marketplace listing, an admin screen, or any API in toolreach's scope set. The only way to obtain it is to insert the macro by hand in the Confluence editor and read the page's storage format back.
**A config key nobody can fill is worse than a constant**, because it advertises a supported path and delivers a dead end - the same defect [issue 18](https://github.com/tmheo/toolreach/issues/18) is opened about for `log.file`. Shipping a second one knowingly, in the same release, would be indefensible.

**The counter-argument, and why it loses.**
"An organization with a different Mermaid app is stuck." True - and the fallback unsticks them. They get the diagram source in a code block, which is exactly what a Mermaid-less tenant would get anyway. Nothing is lost relative to not having the feature.

**This is the reframe that decides it.**
With an unconditional native fallback, the pinned ARI stops being a *requirement* and becomes an *optimization*: a bonus that fires for tenants which happen to have that app, and costs nothing in tenants which do not.
A constant is a perfectly reasonable way to express an optimization. It would not be a reasonable way to express a requirement.

**So the constant is conditional on the fallback existing.** Keeping the ARI hard-coded *without* the fallback - which is 0.9.0's state - is not defensible for a public tool. Keeping it hard-coded *with* the fallback is.

**Draw.io needs no equivalent decision**, because `ac:name="drawio"` pins no vendor. There is nothing to configure. The macro name is the interface.

If a real user turns up with a different Mermaid app and a tenant to test against, the config key is a v0.2 conversation with evidence behind it. Inventing it now would be designing for an imagined user against an unfillable field.

---

## Q5. What changes in the domain language and ADR 0002, beyond issue 8's wording fix?

[Issue 8](https://github.com/tmheo/toolreach/issues/8) already specified wording-only edits: "the company Mermaid app" becomes "a specific third-party Confluence Marketplace Mermaid app, pinned by extension identity"; "the primary Confluence draw.io macro" becomes "the primary draw.io Marketplace macro"; ADR 0002 takes the same change.

This ticket adds four things, and one of them is a correction rather than an addition.

### 5a. **Draw.io Diagram Asset**'s definition is false today

`CONTEXT.md:212` currently ends:

> ... the PNG remains a preview or **fallback** asset, not the primary Confluence rendering.

There is no fallback. The renderer uploads the PNG and never references it, so the PNG is an orphaned attachment, not a fallback.

The domain language already promised the behaviour this ticket decides to build.
That is a third instance of the pattern [issue 16](https://github.com/tmheo/toolreach/issues/16) and [issue 14](https://github.com/tmheo/toolreach/issues/14) each found: **the predecessor's glossary describes an intent the code did not finish.**

Consequence: implementing the fallback makes the existing sentence true and needs no edit beyond issue 8's.
*Not* implementing it would force a new sentence saying the PNG is uploaded and unused - an embarrassing thing to publish in a public glossary, and a good independent reason to implement it.

### 5b. **Source Fallback**'s rationale changes, and it stops being general enough

`CONTEXT.md:275-277`:

> An expand/code macro copy of generated diagram source kept near the rendered diagram so agents can inspect or repair it without cluttering the page.

The mechanism is unchanged. The *reason* is not.
"So agents can inspect or repair it" is an internal-tooling convenience. In a public tool the same construct is the **reader's only content** in any tenant without the app. It moves from convenience to correctness, and a rationale that reads as convenience is exactly what invites a future contributor to delete it.

It is also now too narrow: draw.io gets a fallback too, and draw.io's is an image, not source. Calling an `<ac:image>` a "Source Fallback" would be wrong.

**Recommendation: one new term, `Native Fallback`, taking the general rule; `Source Fallback` is redefined as its Mermaid-specific instance.**

> **Native Fallback**:
> A rendering built only from Confluence macros present in every tenant, emitted beside any Publish Placeholder rendering that depends on a Marketplace app.
> It is emitted unconditionally rather than on failure, because nothing in a publish observes whether the app is installed.
> It is what makes a pinned app identity an optimization rather than a requirement.
> _Avoid_: error fallback, degraded rendering, duplicate diagram

Term count moves **54 to 55** (issue 8 set 53 to 54 with Consent Scope Record).

### 5c. **Mermaid Diagram Block** gains one clause

Beyond issue 8's rename of "the company Mermaid app", it should say the extension is best-effort and the Native Fallback is the guaranteed rendering. Otherwise the term still reads as though the app is required.

### 5d. ADR 0002 is **amended, not superseded** - and one new ADR

ADR 0002's Considered Options already contains:

> - Render draw.io as a static PNG image attachment only.

rejected because the chosen option "preserves an editable draw.io source instead of flattening diagrams to static images".

**That rejection stands and is not reversed here.** The PNG does not become the primary rendering; the `.drawio`-backed macro remains primary. What this decision adds is a *sibling*, which is an option ADR 0002 never weighed because it framed the choice as either-or - PNG **or** macro. Mermaid's renderer, written later, demonstrates the third shape: primary plus native sibling.

So 0002 keeps its decision, keeps its rejection, and gains a Consequences paragraph naming the third option and why it does not contradict the second.
[Issue 8](https://github.com/tmheo/toolreach/issues/8)'s finding that all seven ADRs come across and none needs superseding **survives intact**.

**One new ADR: 0009, "Marketplace macros always ship a Native Fallback."**

It earns its place on the deletion test, which is the strongest argument for any ADR.
A contributor reading a published page's storage body sees a diagram macro immediately followed by an expand containing the same diagram's source, concludes it is duplicated output, and deletes it.
Nothing in the code explains why it is there, because in the tenant that contributor is testing against, the app *is* installed and the fallback looks redundant.
That is precisely the failure ADR 0007's Deletion Test exists to prevent, and it is the same reason [issue 8](https://github.com/tmheo/toolreach/issues/8) kept ADR 0005 deliberately.

0009 also carries the Q4 reasoning, which nothing else records: the hard-coded ARI is only defensible *because* the fallback exists, so deleting the fallback silently converts a constant-as-optimization into a constant-as-requirement.

That takes toolreach to **9 ADRs**: seven ported, 0008 for the per-organization shared app, 0009 for this.

---

## The decision, assembled

1. **Both Marketplace-backed placeholders emit an unconditional Native Fallback.** Mermaid already does (expand + code with the source). Draw.io gains one: an `<ac:image>` referencing the PNG preview it already uploads, using the emitter that already exists at `renderer_file.go:150`.
2. **When no PNG preview is available** - the hand-written Storage Body path, where `preview-source-file` is optional - draw.io's Native Fallback is a native link to the uploaded `.drawio` attachment plus its title, so the reader gets a downloadable diagram rather than an error box. The Markdown path, which is the agent-facing one, always has a PNG.
3. **No detection.** No app-inventory API, no read-back check, no new scope. Documentation in `auth setup` and command help instead.
4. **The Mermaid ARI stays a hard-coded constant** in v0.1, recorded as a deliberate constraint, defensible only because of point 1.
5. **`ac:name="drawio"` stays as-is.** It pins no vendor and needs no decision.
6. **CONTEXT.md**: one new term (`Native Fallback`, 54 to 55), `Source Fallback` redefined as its Mermaid instance with the rationale corrected, `Mermaid Diagram Block` gains a best-effort clause, `Draw.io Diagram Asset` needs nothing beyond issue 8's wording once the fallback exists.
7. **ADR 0002** amended in Consequences, not superseded. **ADR 0009** added for the general rule.

## Defects found while measuring

At 0.9.0, both inherited by the port unless fixed:

1. `renderer_mermaid.go:162` passes the same node into `<ac:adf-fallback>` that it wraps, so the ADF fallback slot provides no degradation.
2. `renderer_drawio.go:61-70` uploads a PNG preview attachment that no code path ever references in the page body - dead output rather than dead code, and the reason `CONTEXT.md:212` is currently false.

## Not established here

- What Confluence actually renders for an unresolvable `com.atlassian.ecosystem` extension and for an unknown structured macro. Graded **[inferred]** throughout; the decision is built not to depend on it. A live tenant would settle both in one publish, and [issue 15](https://github.com/tmheo/toolreach/issues/15) is where that belongs if it is ever worth doing.
- Whether more than one Marketplace app registers `ac:name="drawio"`, and whether their parameter sets agree. The 16 parameters `renderDrawioMacro` writes were presumably read off one vendor's output; a second vendor might ignore some. This does not change the decision - the Native Fallback covers a mis-parameterized macro exactly as it covers an absent one - but it would change how confident the primary rendering deserves to sound in the docs.
