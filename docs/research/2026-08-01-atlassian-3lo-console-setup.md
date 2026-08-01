# Atlassian OAuth 2.0 (3LO) developer console setup

## What this answers

This is primary-source research for GitHub issue #3, "What an organization's first user must do in the Atlassian developer console".
It records, from Atlassian's own documentation, what one person per organization has to configure so that the 18 scopes toolreach requests can be consented to by colleagues, and what breaks when the configuration is partly wrong.
It contains facts and sourced quotations only, with no recommendations about what toolreach should do.

Every factual claim carries an inline source URL.
Claims that come from the Atlassian developer community (Atlassian-hosted, but not official documentation) are marked as such, with the responder's Atlassian Staff status noted where it applies.
Claims that could not be established from documentation are collected in the final section.

Research date: 2026-08-01.
Documentation pages carried a "Last updated Jul 31, 2026" stamp at the time of reading.

### The scope set under study

The 18 scopes are taken verbatim from `~/workspace/github/workpulse/internal/oauth/oauth.go` lines 44-69.
Thirteen are Confluence scopes, four are Jira scopes, and one is `offline_access`.
The authorize request also uses `audience=api.atlassian.com`, `prompt=consent`, PKCE S256, and the fixed loopback callback `http://localhost:19876/callback`.

---

## 1. Scope-to-console mapping

### How the console models permissions

The console does not present a flat list of scopes.
It presents a list of *APIs*, and scopes live inside an API.
"Note, if you haven't already added an API to your app, you should do this now: 1. Select **Permissions** in the left menu. 2. Next to the API you want to add, select **Add**." (https://developer.atlassian.com/cloud/oauth/getting-started/enabling-oauth-3lo/)

"The **Permissions** page lists the APIs included in your app.
To add or remove individual scopes for an API, select **Configure**, and in the list of scopes, select **Add** or **Remove**." (https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/)

So the first user must add two APIs to the app, one for Confluence and one for Jira, and then configure scopes inside each.
The exact names of those API entries in the console UI are not given in the documentation and are flagged as an unknown below.

### What the "console label" column below means

The scope reference pages carry a column titled "Summary" for classic scopes and "Title" for granular scopes.
Both pages state: "The title and description are displayed to the user on the consent screen during the authorization flow." (https://developer.atlassian.com/cloud/confluence/scopes-for-oauth-2-3LO-and-forge-apps/, https://developer.atlassian.com/cloud/jira/platform/scopes-for-oauth-2-3LO-and-forge-apps/)

The documentation therefore establishes these strings as the **consent screen** labels.
It does not state that the console's own checkboxes use the same strings, although they are the only human-readable labels Atlassian publishes for these scopes.

### Classic versus granular

Both scope reference pages split their tables into a "Classic scopes" section and a "Granular scopes" section.
Classic carries the note "Where available, the recommendation is to use classic scopes" (Confluence) and "Where available, the recommendation is to use these scopes" (Jira).
Granular carries the note "Use these scopes only when you can't use classic scopes."
(https://developer.atlassian.com/cloud/confluence/scopes-for-oauth-2-3LO-and-forge-apps/, https://developer.atlassian.com/cloud/jira/platform/scopes-for-oauth-2-3LO-and-forge-apps/)

Neither page describes a "view more granular scopes" toggle, an expander, or any other console gesture needed to reach granular scopes.
Neither page says that classic and granular scopes cannot be combined in one app.

The strongest documented evidence that they *can* be combined is the API reference itself.
Jira Cloud platform REST v3 endpoints print both families side by side as alternatives for the same operation, for example on `GET /rest/api/3/project/{projectIdOrKey}`: "OAuth 2.0 scopes required: Classic RECOMMENDED : `read:jira-work` Granular : `read:issue-type:jira`, `read:project:jira`, `read:project.property:jira`, `read:user:jira`, `read:application-role:jira`". (https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-projects/)
Confluence v1 endpoints do the same, for example "Get URI to download attachment": "Classic RECOMMENDED : `readonly:content.attachment:confluence` Granular : `read:attachment:confluence`". (https://developer.atlassian.com/cloud/confluence/rest/v1/api-group-content---attachments/)

Confluence REST API **v2** endpoints, by contrast, print a single granular scope with no classic alternative.
`GET /pages`, `GET /pages/{id}`, `GET /spaces/{id}/pages`, `GET /labels/{id}/pages` and `GET /pages/{id}/children` all state "OAuth 2.0 scopes required: `read:page:confluence`" with no classic option. (https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-page/)
This is why toolreach's Confluence set is granular while its Jira set is classic: the v2 Confluence API only documents granular scopes.

There is a documented scope-count ceiling to be aware of.
"It's recommended that you use less than 50 scopes in an application.
When adding scopes in the developer console, a count of the scopes added to your app is displayed." (https://developer.atlassian.com/cloud/confluence/scopes-for-oauth-2-3LO-and-forge-apps/)
An app with 18 scopes is well inside that.

Both pages also note: "Some scopes automatically imply that the app is granted other scopes."
The pages do not publish the implication graph.

### Scope mapping table

| code scope | console / consent label | permission group (API to add) | family | notes and deprecation |
| --- | --- | --- | --- | --- |
| `read:page:confluence` | View pages | Confluence | granular | Required by Confluence v2 `GET /pages`, `GET /pages/{id}`, `GET /spaces/{id}/pages`, `GET /labels/{id}/pages`, `GET /pages/{id}/children`. No classic alternative documented for v2. (https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-page/) |
| `read:space:confluence` | View spaces | Confluence | granular | Required by Confluence v2 space endpoints. (https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-space/) Not deprecated. |
| `read:content:confluence` | View content | Confluence | granular | Listed in the granular table with description "View content, including pages, blogposts, whiteboards, databases, Smart Links in the content tree, folders, custom content, attachments, comments, and content templates." **Not observed as the required scope on any endpoint reference page scanned** (Confluence v2 groups page, blog-post, attachment, space, folder, database, smart-link, whiteboard, children, descendants, ancestors, content, content-properties, comment, custom-content, label, task, user, version, like, operation, classification-level, redactions, space-roles, admin-key, data-policies, app-properties; Confluence v1 groups content, content-body, search, content---attachments). Carries no deprecation marker on the scope page. Its actual effect is an open question, see the final section. |
| `read:content-details:confluence` | View content details | Confluence | granular | Documented as the granular alternative to classic `search:confluence` on Confluence v1 `/content` and `/search` endpoints, and as part of the granular set for v1 create/update attachment. (https://developer.atlassian.com/cloud/confluence/rest/v1/api-group-content/, https://developer.atlassian.com/cloud/confluence/rest/v1/api-group-search/, https://developer.atlassian.com/cloud/confluence/rest/v1/api-group-content---attachments/) |
| `read:folder:confluence` | View folders | Confluence | granular | Required by Confluence v2 folder endpoints. (https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-folder/) |
| `read:hierarchical-content:confluence` | Read hierarchical content | Confluence | granular | Required by Confluence v2 children and descendants endpoints. (https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-children/, https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-descendants/) |
| `read:database:confluence` | View databases | Confluence | granular | Required by Confluence v2 database endpoints. (https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-database/) |
| `read:embed:confluence` | View Smart Links in the content tree | Confluence | granular | Required by Confluence v2 smart-link endpoints. (https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-smart-link/) |
| `read:whiteboard:confluence` | View whiteboards | Confluence | granular | Required by Confluence v2 whiteboard endpoints. (https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-whiteboard/) |
| `read:attachment:confluence` | View and download content attachments | Confluence | granular | Required by Confluence v2 attachment endpoints, and documented as the granular alternative on v1 "Get URI to download attachment". (https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-attachment/, https://developer.atlassian.com/cloud/confluence/rest/v1/api-group-content---attachments/) |
| `readonly:content.attachment:confluence` | Download content attachments | Confluence | **classic** | This is the one Confluence scope in toolreach's set that sits in the **Classic scopes** table, not the granular one. (https://developer.atlassian.com/cloud/confluence/scopes-for-oauth-2-3LO-and-forge-apps/) It is the documented classic scope for v1 "Get URI to download attachment". It was created specifically for 3LO apps: "We have created a new OAuth 2.0 (3LO) scope `readonly:content.attachment:confluence` that gives 3LO apps permission to download attachments." (Atlassian Staff announcement, https://community.developer.atlassian.com/t/new-download-attachment-rest-api-endpoint/50398) **Not deprecated** on the scope page. Historical note: a 2021 Forge CLI bug reported "Some scopes defined are not supported: readonly:content.attachment:confluence", resolved by upgrading the CLI; this affected Forge manifests, not OAuth apps. (https://community.developer.atlassian.com/t/some-scopes-defined-are-not-supported-readonly-content-attachment-confluence/52850) |
| `write:page:confluence` | Create and update pages | Confluence | granular | Required by Confluence v2 `POST /pages`, `PUT /pages/{id}`, `PUT /pages/{id}/title`. (https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-page/) |
| `write:attachment:confluence` | Create and update content attachments | Confluence | granular | Documented as part of the granular alternative for Confluence v1 create/update attachment, alongside `read:content-details:confluence`; the classic alternative there is `write:confluence-file`. (https://developer.atlassian.com/cloud/confluence/rest/v1/api-group-content---attachments/) |
| `read:jira-work` | View Jira issue data | Jira | classic | "Read Jira project and issue data, search for issues and objects associated with issues like attachments and worklogs." (https://developer.atlassian.com/cloud/jira/platform/scopes-for-oauth-2-3LO-and-forge-apps/) |
| `read:jira-user` | View user profiles | Jira | classic | "View user information in Jira that the user has access to, including usernames, email addresses, and avatars." (same page) |
| `read:project:jira` | View projects | Jira | granular | On the project endpoints this scope is **only one member of a required granular set**, never sufficient alone. `GET /rest/api/3/project`, `GET /rest/api/3/project/search` and `GET /rest/api/3/project/{projectIdOrKey}` all document "Granular : `read:issue-type:jira`, `read:project:jira`, `read:project.property:jira`, `read:user:jira`, `read:application-role:jira`" against the single classic scope `read:jira-work`. (https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-projects/) |
| `write:jira-work` | Create and manage issues | Jira | classic | "Create and edit issues in Jira, post comments as the user, create worklogs, and delete issues." (https://developer.atlassian.com/cloud/jira/platform/scopes-for-oauth-2-3LO-and-forge-apps/) |
| `offline_access` | not published | not published | neither classic nor granular | `offline_access` does not appear in either the Confluence or the Jira scope reference table. It is documented only as an authorization-URL parameter: "To get a refresh token in your initial authorization flow, add `offline_access` to the `scope` parameter of the authorization URL." (https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/) Whether it also has to be ticked anywhere in the console is not stated. |

### Deprecation check summary

None of the 18 scopes carries a deprecation notice on its scope reference page as of the 2026-07-31 revision of those pages.
The two scopes the ticket singled out both survive: `readonly:content.attachment:confluence` is present and current in the Classic table, and `read:content:confluence` is present and current in the Granular table.
The open question about `read:content:confluence` is not deprecation, it is that no endpoint reference page scanned names it as a requirement.

---

## 2. Distribution / sharing toggle

### What the control does

An app is private until sharing is turned on.
"When you create an OAuth 2.0 (3LO) app, it's private by default.
This means that only you can install and use it.
If you want to distribute your app to other users, you must enable sharing." (https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/)

The steps are: developer console, select the app, "Select **Distribution** in the left menu", "Enable sharing using the toggle switch in the **Enable sharing** section". (same page)

The app list shows the resulting state as "Distribution status: Sharing: your app can be shared via link" or "Not sharing: your app can't be shared via link". (same page)

### Is Distribution required for colleagues in the same organization?

Yes.
The documentation states the app "is private by default" and that "only you can install and use it", without carving out same-organization users. (https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/)

An Atlassian Staff member states it explicitly and covers the same-site case directly.
"If you have just created your OAuth 2.0 app in the developer console, then, by default, it is in developer mode.
And it will only work for you, as the creator of that app.
When you use the authorization URL in your browser, then you will see a warning about being in developer mode.
**The same authorization URL will not work for anyone else, even if they are in the same site as you.**
So, to share with other people, regardless of same site or different, you will have to turn on 'distribution' in the developer console." (ibuchanan, Atlassian Staff, https://community.developer.atlassian.com/t/69937)

So Distribution is not an "outside the organization" control.
It is the on/off switch for anyone other than the app's creator, inside or outside.

There is also a documented pre-distribution state visible to the creator: a "developer mode" warning shown on the consent screen when the creator authorizes their own private app.
That warning is described in the same community post; it is not described on the official documentation page.

### What the Distribution form asks for

The official documentation lists no form fields at all.
It describes only the toggle. (https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/, https://developer.atlassian.com/cloud/oauth/getting-started/managing-oauth-apps/)

Two community reports name required fields.
"Vendor name and Privacy Policy fields are mandatory, but I don't really know what info I have to put there?" (https://community.developer.atlassian.com/t/69937)
The Atlassian Staff reply does not dispute that they are mandatory: "I don't think there are any consequences of these values, so almost anything will do just to satisfy the required field.
That said [...] maybe some semi-reasonable values like your own company name and URL to your company's privacy policy (or maybe just a link to your corporate web site)." (ibuchanan, Atlassian Staff, same thread)
A separate thread also reports the reporter "enabled the Sharing toggle under Distribution settings and added a Privacy Policy URL". (https://community.developer.atlassian.com/t/oauth-3lo-failed-to-retrieve-client-for-non-contributors-despite-sharing-mode-enabled/100834)

No source found mentions a terms-of-use URL, a support contact field, or a security self-assessment as part of the Distribution form.
The security self-assessment and questionnaires that Atlassian documents belong to the Marketplace partner and app-approval track, not to the Distribution toggle. (https://developer.atlassian.com/platform/marketplace/security-requirements/)

### Does the "not reviewed by Atlassian" warning still appear for same-organization colleagues?

The documented behavior is that it does appear, and the documentation makes no exception for same-organization users.
"Users trying to install an unapproved OAuth 2.0 integration are warned that the app has not yet been reviewed by Atlassian.
To get your integration reviewed and approved, follow the steps on Listing a third party integration on the Atlassian Marketplace.
Note, you don't need an informative Atlassian Marketplace listing to submit your integration for approval." (https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/)

The observed behavior has diverged from the documentation at least once.
A developer reported sharing with users outside their organization and seeing *no* warning; Atlassian Staff confirmed the missing warning was a bug and raised FRGE-743, and confirmed that if you do not want users to see the warning, the app has to be approved and listed. (https://community.developer.atlassian.com/t/sharing-of-oauth-2-0-3lo-app-does-not-show-warning-that-the-app-has-not-yet-been-reviewed-by-atlassian/58967)
Whether that bug is fixed today, and what the warning currently looks like, is not established.

Approval is not required in order to share.
"You can share the app without going through the approval procedure.
For 3LO apps, the listing is informational only with limited Marketplace features." (CaterinaCurti, Atlassian Staff, same thread)
Getting approved means going through the Marketplace submission flow for a non-installable integration, which requires an App Key, a version and build number, an app file or information URL, a name, compatible applications, and giving the Marketplace review team access to a test instance of the software. (https://developer.atlassian.com/platform/marketplace/knowledge-base/listing-a-third-party-integration-on-the-atlassian-marketplace/)

Enabling sharing is orthogonal to the Marketplace.
"Enabling sharing doesn't make your app available on the Atlassian Marketplace.
Although OAuth 2.0 (3LO) apps can be listed on the Atlassian Marketplace, they will appear as informational listings only, with limited Marketplace features." (https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/)

Distribution is also per-user, not per-organization.
"OAuth 2.0 (3LO) apps are installed on a per-user basis, so you'll have to send the link to all the users you want to grant access to." (same page)

---

## 3. Callback URL registration

### What the documentation says

Almost nothing.
The complete official text is: "Enter the **Callback URL**.
Set this to any URL that is accessible by the app.
When you implement OAuth 2.0 (3LO) in your app (see next section), the `redirect_uri` must match this URL." (https://developer.atlassian.com/cloud/oauth/getting-started/enabling-oauth-3lo/)

The documentation does not state scheme rules, loopback rules, port rules, wildcard support, or how many callback URLs may be registered.
Every worked example in the OAuth guides uses `https://YOUR_APP_CALLBACK_URL`. (https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/)

### What is established from Atlassian Staff answers in the community

**Plain `http` on loopback is accepted.**
Asked directly "would localhost work?", an Atlassian Staff member answered "Yes - localhost will work." (dmorrow, Atlassian Staff, https://community.developer.atlassian.com/t/41372)
A separate community report states that the console's "Save changes" button is greyed out for non-HTTPS callback URLs, with a reply that "callback URLs in Atlassian Authorization webpage only allows https for any URL you write... except if it's 'localhost'". (https://community.developer.atlassian.com/t/oauth-2-0-callback-url-using-http-not-https/58722)
The second half of that pair is a community member's observation, not staff-confirmed.

**Only one callback URL per app.**
"An OAuth 2.0 app can only have a single callback URL.
You'll need to create a separate app per website, then design your codebase to accept multiple sets of OAuth secrets." (mventnor, Atlassian Staff, https://community.developer.atlassian.com/t/multiple-callback-urls-for-a-single-outh-2-0-3lo-app/32413)
The same thread has an Atlassian Staff member opening a feature request for multiple callback URLs, later tracked as https://jira.atlassian.com/browse/ECO-716, with no updates reported.
Changing the single URL takes effect immediately and needs no approval, per a Marketplace Partner in that thread who tested it.

**Matching is exact, including path.**
A developer whose registered callback was `https://app.ourdomain.com` while sending `redirect_uri=https://app.ourdomain.com/auth/atlassian_oauth2/callback` started failing after a backend change.
Atlassian Staff response: "We are rolling out a back-end change, so some of the compute and storage behind the OAuth 2.0 implementation have changed. [...] it seems the old implementation was not as 'picky' about all the specified URLs and the new implementation is." (ibuchanan, Atlassian Staff, https://community.developer.atlassian.com/t/oauth2-redirect-uri-not-registered-for-client-but-nothing-changed/69113)
Registering the full path fixed it.
So `http://localhost:19876/callback` must be registered with that exact path and port, not just `http://localhost:19876`.

**Mismatch produces a specific user-visible error.**
A mismatched `redirect_uri` sends the user to `https://id.atlassian.com/error` with query parameters including `error=unauthorized_client` and `error_description=Callback URL mismatch. <url> is not in the list of allowed callback URLs`, plus a `tracking` id. (https://community.developer.atlassian.com/t/41372)
Note the wording "the list of allowed callback URLs" even though only one can be registered.

**Not established:** whether `127.0.0.1` is treated the same as `localhost`, whether the port is part of the exactness rule (near-certain given the path result, but not directly evidenced), and whether wildcards are accepted at all.

---

## 4. Failure modes

### 4a. A scope the binary requests was never added to the app

**Not established from documentation.**
No Atlassian documentation page states what `https://auth.atlassian.com/authorize` does when the `scope` parameter names a scope that is not configured on the app.
The only related instruction is a constraint on the caller, not a description of the failure: "`scope` (required) Set this to the desired scopes: Separate multiple scopes with a space.
**Only choose from the scopes that you have already added to the APIs for your app in the developer console.**
You may specify scopes from multiple products." (https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/)

No documented error shape for `invalid_scope` at the authorize endpoint was found on developer.atlassian.com, and a community search for that error string returned no 3LO OAuth thread describing it.

One thing *is* established about the ordering of the flow, by direct unauthenticated probe of the public endpoint rather than by documentation.
A GET to `https://auth.atlassian.com/authorize` with a nonexistent `client_id` returns `302` to `https://id.atlassian.com/login?continue=<the original authorize URL>`, not to an error page.
Client and scope validation therefore happens only after the user has signed in.
Practical consequence: any misconfiguration surfaces after login, never before it.
This is an observed behavior of a public endpoint, not a documented guarantee.

### 4b. Consent succeeds and the API call fails later

This is the well-evidenced path, and the error is uniform and unhelpful.

The API returns HTTP 401 with the body `{"code":401,"message":"Unauthorized; scope does not match"}`.
That exact body is reproduced in multiple community threads for both products.
Jira example: a call to `/rest/agile/1.0/board` logged `401` and `{"code":401,"message":"Unauthorized; scope does not match"}`; Atlassian Staff identified the two missing scopes by hand from the API reference. (https://community.developer.atlassian.com/t/how-to-solve-unauthorized-scope-does-not-match/81389)
Confluence example: "I now get the error 'Unauthorized; scope does not match' but I have no idea which scope is a problem." (https://community.developer.atlassian.com/t/unauthorized-scope-does-not-match-confluence/73795)

The error body **does not name the missing scope**.
That is the single most important fact for onboarding design: the failure is a 401 with an opaque message, and diagnosing it means cross-referencing the failing endpoint against its documented "OAuth scopes required" field by hand.

The documented way to find the requirement is per-endpoint, not per-scope.
"To find out which scopes an operation requires, check the **OAuth scopes required** field in the relevant API documentation." (https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/)

Separately, scopes are not the only thing that can produce a denial.
"Note, the permissions held by the user an app is acting for always constrain the app, regardless of the app's scopes." (same page)
So a correctly-scoped app still fails on content the user cannot see.

### 4c. Does the family (classic versus granular) change which failure you get?

**Not established from documentation.**
No source found states that classic scopes fail at authorize time while granular scopes fail at API time, or the reverse.
Both families produce the same `Unauthorized; scope does not match` body in the community reports above.

### 4d. What the user actually sees

- Callback URL wrong: an `https://id.atlassian.com/error` page, reached after login, with `error=unauthorized_client` and `error_description=Callback URL mismatch. ... is not in the list of allowed callback URLs`. (https://community.developer.atlassian.com/t/41372)
- Sharing not enabled and the user is not the app creator: at least one report of the authorization page showing "failed to retrieve client". (https://community.developer.atlassian.com/t/oauth-3lo-failed-to-retrieve-client-for-non-contributors-despite-sharing-mode-enabled/100834) That thread has no confirmed resolution and no Atlassian Staff reply, so treat the exact string as unconfirmed.
- App is the creator's own private app: a "developer mode" warning box on the consent screen. (ibuchanan, Atlassian Staff, https://community.developer.atlassian.com/t/69937)
- App is shared but unapproved: per documentation, a warning "that the app has not yet been reviewed by Atlassian". (https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/) Observed behavior has diverged, see section 2.
- Scope missing at API time: nothing on screen from Atlassian, because the failure is a 401 JSON body returned to the client.
- Too many scopes: an app carrying roughly 80 granular scopes produced a Tomcat HTML error page, `HTTP Status 400 - Bad Request`, message "Request header is too large". Atlassian Staff confirmed the cause was the number of scopes, and reducing the scope count fixed it. (https://community.developer.atlassian.com/t/56311) At 18 scopes this is not a live risk, but it is the documented reason for the "less than 50 scopes" guidance.

### 4e. Existing tokens that predate a scope change

Re-consent is required, and this is documented twice.

On the console page: "To add or remove individual scopes for an API, select **Configure**, and in the list of scopes, select **Add** or **Remove**.
**Note that users who previously consented to the scopes will need to re-consent to the new scopes.**" (https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/)

On the grant model: "An authorization grant is when a user consents to your app.
For OAuth 2.0 (3LO) apps, the consent is valid for all sites the app is installed in, as long as the scopes used by your app's APIs don't change.
A user's grant can change when either of the following occur:
The user revokes the grant. [...]
The user consents to a new grant of the app.
**The scopes in the new grant override the scopes in the existing grant.**
Therefore, since a grant can change over time, it's important that you check that the user has granted the app the scopes it requires." (same page)

An existing refresh token is not automatically invalidated by a scope change, and the documentation does not say it is.
The documented refresh-token failure is a different one: "403 Forbidden with `{"error": "invalid_grant", "error_description": "Unknown or invalid refresh token."}`", caused by password change or by an expired rotating refresh token, and "If your refresh token expires, your user will need to complete the entire authorization flow from the beginning again." (same page)
Rotating refresh tokens are valid for 90 days, with a 10 minute reuse leeway window. (same page)

**Not established:** whether an access token minted before a scope was added silently keeps working with the old scope set until it expires, or whether the platform re-evaluates against the app's current scope configuration.
The wording "the scopes in the new grant override the scopes in the existing grant" describes what happens after re-consent, not what happens to an old token in the meantime.

### 4f. What `accessible-resources` returns in `scopes`

`GET https://api.atlassian.com/oauth/token/accessible-resources` with the access token as a bearer token returns an array of containers, each with `id` (the cloudid), `name`, `url`, `scopes`, and `avatarUrl`. (https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/)

The documented example for a Confluence site is:

```json
[
  {
    "id": "1324a887-45db-1bf4-1e99-ef0ff456d421",
    "name": "Site name",
    "url": "https://your-domain.atlassian.net",
    "scopes": [
      "write:confluence-content",
      "read:confluence-content.all",
      "manage:confluence-configuration"
    ],
    "avatarUrl": "https://site-admin-avatar-cdn.prod.public.atl-paas.net/avatars/240/flag.png"
  }
]
```

The documentation describes the field as "the scopes associated with that access", scoped to that container.
It adds two caveats that matter for a health-check design.
"It's important to understand that this endpoint won't tell you anything about the user's permissions, which may limit the resources that your app can access via the site's APIs."
"Note, the `id` is not unique across containers (that is, two entries in the results can have the same `id`), so you may need to infer the type of container from its scopes."
(all from https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/)

The response also differs by grant type.
Account-level grants "may return multiple sites"; resource-level grants return "only the specific site(s) the user selected on the consent screen".
Atlassian documents that "The consent screen currently requires the user to select the site where they want to install the app" as a known limitation. (same page)

**Not established:** whether the `scopes` array lists the scopes the *token* carries, the scopes the *app* is configured with, or the intersection.
The documented examples show only three scopes each and are clearly illustrative rather than exhaustive, so they do not settle the question.
This matters directly if toolreach wants to use this endpoint to detect a partially-configured app before the first API call.

---

## 5. Cross-organization consent

### Can a user in a different Atlassian organization consent?

Yes, and Distribution is the only gate the app owner controls.
"So, to share with other people, **regardless of same site or different**, you will have to turn on 'distribution' in the developer console." (ibuchanan, Atlassian Staff, https://community.developer.atlassian.com/t/69937)

A concrete case exists.
A developer reported "I have shared my oauth2 integration with other users (outside my organization), by sharing my app through the developer console", and the Atlassian Staff reply confirmed sharing outside the organization works without Marketplace approval. (https://community.developer.atlassian.com/t/sharing-of-oauth-2-0-3lo-app-does-not-show-warning-that-the-app-has-not-yet-been-reviewed-by-atlassian/58967)

The official documentation neither confirms nor denies cross-organization consent.
Its description of distribution is organization-agnostic: it speaks only about "other users" and per-user installation. (https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/)

### What the consenting user's own organization can do to stop it

This is the important half, because the gate is on the consenting side, not on the app-registering side.

**Block user apps.**
"Organization and site admins can also allow the installation or uninstallation of user apps to the admin level by selecting **Block user apps** in the Settings tab.
By default, user installed apps are allowed." (https://support.atlassian.com/organization-administration/docs/installing-and-managing-app-access/)
The same control is described from the security side: "Organization and Site admins can also choose to restrict app installation to admin level by selecting **Block user apps** in the Connected apps settings." (https://support.atlassian.com/security-and-access-policies/docs/manage-your-users-third-party-apps/)
The default is permissive, so an ordinary colleague can normally consent without an admin.

**Admin revocation.**
"For OAuth 2.0 (3LO) apps such as 'Atlassian for VS Code', the Uninstall button revokes all users.
Uninstalling an app requires all connected users to be removed first." (https://support.atlassian.com/security-and-access-policies/docs/manage-your-users-third-party-apps/)
So one admin action can revoke every colleague's grant at once.

**User revocation.**
"Note that a user can revoke their app grants at any time using their own connected apps screen." (same page)

**Data security policy app access rules.**
"By default, installed apps can access data in any Confluence spaces and Jira projects.
Data security policies with app access rules allow organization admins to block app access in specific spaces and projects." (https://developer.atlassian.com/cloud/confluence/data-security-policy-developer-guide/)
This is space-level and project-level, so a user can hold a valid token with correct scopes and still get nothing back for particular spaces.
For Confluence, a blocked read is signalled by a response header rather than an error: "we return a custom response header.
The custom response header will be in the format `Atlassian-DataSecurityPolicy: app_access_blocked`", applied to `GET /wiki/api/v2/pages/{id}`, `/blogposts/{id}`, `/databases/{id}`, `/embeds/{id}`, `/folders/{id}` and `/whiteboards/{id}`. (same page)
Confluence `GET spaces` and Jira Data Policies REST APIs expose a `dataPolicy.anyContentBlocked` boolean so an app can detect the situation. (same page)
Fine-grained app access rules require an Atlassian Guard Standard subscription; all organization admins can apply a blanket override that blocks all eligible Marketplace and custom apps. (https://support.atlassian.com/security-and-access-policies/docs/block-app-access/)

Note that most Confluence v2 endpoints toolreach reads from are marked "Not exempt from app access rules", so they are blockable, while Jira project endpoints are marked "Exempt from app access rules". (https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-page/, https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-projects/)

### Worth warning users about

- A colleague in another organization consenting to your app means your `client_secret` is being used to mint tokens against their organization's data, and their admin can revoke all of it at once via Uninstall.
- The consenting user's admin, not the app registrant, controls whether user-level app installs are allowed at all.
- Data security policies can silently narrow what a fully-consented token can read, with the signal delivered as a response header rather than an error status.

---

## Not established from documentation

Each item below needs a human with access to the live developer console, or a live end-to-end test, to confirm.

1. **The exact API entry names on the Permissions page.** Documentation says "Next to the API you want to add, select **Add**" but never names the entries. Which entry covers Confluence v2 granular scopes, and which covers Jira classic scopes, has to be read off the console.
2. **Whether granular scopes are behind a toggle, expander, or "view more granular scopes" control in the console.** No documentation page describes the scope-selection UI at all.
3. **Whether the console's scope checkboxes use the same label strings as the consent screen.** The docs establish those strings only as consent-screen labels.
4. **Whether classic and granular scopes can be enabled together on one app in the console UI.** The API reference presents them as per-endpoint alternatives and never forbids mixing, and toolreach's set mixes them, but no documentation states the console permits it.
5. **Where `offline_access` is enabled.** It appears in no scope table and no console instruction; docs only say to add it to the authorization URL's `scope` parameter.
6. **What `read:content:confluence` actually grants.** It is current and non-deprecated in the granular table, but was not found as a required scope on any endpoint reference page scanned. It may be an implied/umbrella scope, since both scope pages note "Some scopes automatically imply that the app is granted other scopes" without publishing the graph.
7. **The complete list of fields the Distribution form requires.** Documentation lists none. Community reports name "Vendor name" and "Privacy Policy" as mandatory. Terms of use, support contact, and a security self-assessment were not found in any source as part of Distribution.
8. **Whether the "not reviewed by Atlassian" warning currently appears, and its exact wording and appearance.** Documented as appearing; reported by a developer as absent and confirmed as a bug (FRGE-743) by Atlassian Staff; current state unknown.
9. **Whether that warning differs for same-organization versus cross-organization consenters.** No source distinguishes them.
10. **What `auth.atlassian.com/authorize` does when `scope` names a scope not configured on the app.** No documented error shape, no `error=invalid_scope` example, no community thread found. Needs a deliberate test: register an app with a subset of the 18 scopes and request all 18.
11. **Whether that behavior differs between classic and granular scopes.** Untested and undocumented.
12. **The exact string shown when a non-creator opens the authorization URL of an unshared app.** One report says "failed to retrieve client", unconfirmed by Atlassian.
13. **Callback URL rules beyond loopback-http.** Specifically: whether `127.0.0.1` is accepted and treated identically to `localhost`, whether any wildcard form is accepted, and whether the console blocks a plain-http non-loopback URL by disabling Save (community-reported, not staff-confirmed).
14. **What the `scopes` array in `accessible-resources` reflects** - the token's granted scopes, the app's configured scopes, or their intersection. This determines whether it can be used as a pre-flight check for a partially configured app.
15. **Whether an access token issued before a scope was added keeps working unchanged until expiry.** Documentation covers re-consent and grant override, but not the in-between state of an already-issued token.
16. **Whether Atlassian's authorize endpoint supports or requires PKCE.** No developer.atlassian.com page found mentions `code_challenge`, `code_challenge_method`, or PKCE for 3LO, and a community search returned nothing. toolreach sends PKCE S256; whether Atlassian validates it, ignores it, or would reject it is undocumented.
17. **How many callback URLs can be registered today.** Atlassian Staff stated one, and the feature request (ECO-716 / FRGE-627) was reported as having no updates; whether that is still true needs a console look.
