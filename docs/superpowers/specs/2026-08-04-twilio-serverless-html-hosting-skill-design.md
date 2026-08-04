# Publishing HTML to Twilio Serverless — Skill Design

**Date:** 2026-08-04
**Status:** Approved, ready for implementation planning

## Goal

A Claude Code skill that uploads standalone HTML files to a Twilio account and serves them from
Twilio Serverless (Functions & Assets), producing shareable `https://<service>-<hash>.twil.io/<slug>`
URLs. Pages are public by default; individual pages can be put behind a password.

Hard constraints:

- **No Twilio CLI.** No `twilio` binary, no Serverless Toolkit, no `npm install`.
- **No Twilio SDK.** The Asset upload endpoint lives on `serverless-upload.twilio.com` and is
  explicitly unsupported by the SDKs, so raw HTTP is required regardless.
- **Credentials from the environment.** API Key + Secret preferred, Auth Token as fallback.
- Runs on Node 24 (native `fetch`, `FormData`, `Blob`, `node:test`) — zero dependencies.

## Non-goals

- Custom domains. Serverless serves `*.twil.io` only.
- Staging/multiple environments. One environment, `production`.
- Per-page passwords. One shared password across all gated pages.
- Sidecar assets (separate CSS/JS/image files). Input pages are self-contained single files.
- Mirroring the repo. Publishing never removes a live page; see [Sync semantics](#sync-semantics).
- Analytics, access logs, view counts.

## Decisions

| Decision | Choice | Rationale |
| --- | --- | --- |
| Access model | Public by default, per-page opt-in gating | Most pages are shareable; a few are internal |
| Gate UX | Password-only HTML form + signed cookie | No username field; no secret in the URL, history, or referrer |
| Landing page | Auto-generated index at path `/` | Never drifts; a hand-written `index.html` in the folder wins if present |
| Sync semantics | Upsert, never remove | Safe from a partial checkout or single-file edit |
| Scope selection | Interactive: all `*.html` or one file | No implicit default; the script requires an explicit flag |
| Chrome theme | Dark (`#0e0e13` / `#f22f46`) | Chosen for index + gate as one "site chrome" identity |
| Implementation | Skill + small Node CLI (4 subcommands) | Risky logic lives in tested code, not re-derived per run |
| State | Twilio account is source of truth; local manifest is a cache | Survives a fresh clone; degrades gracefully |

### Note on theming

The repo's memory note records that *page content* uses a light editorial house style
(`#ECF1F3` / `#14283A`, Avenir Next). That still holds for authoring pages. This spec's dark chrome
applies only to the two surfaces the skill itself generates — the index and the gate — which were
deliberately chosen to read as site chrome distinct from the documents they wrap. Individual
documents keep their own palettes as content.

## Credentials

Read from the environment, never written to disk or logs:

| Variable | Role |
| --- | --- |
| `TWILIO_ACCOUNT_SID` | Required. Account context. |
| `TWILIO_API_KEY` + `TWILIO_API_SECRET` | Preferred. Basic auth: key SID as username, secret as password. |
| `TWILIO_AUTH_TOKEN` | Fallback when no API key is set. Basic auth: account SID as username. Emits a one-line recommendation to switch to an API key. |
| `TWILIO_PAGES_PASSWORD` | Optional. Consumed only by `set-password`. |

Missing credentials fail before any network call, naming the specific missing variable.

## Resource model

```
Service  "pages"          UniqueName=pages, FriendlyName="Shared HTML Pages",
                          IncludeCredentials=false, UiEditable=true
└── Environment "production"   UniqueName=production, no DomainSuffix (shortest domain)
    │                          → https://pages-<hash>.twil.io
    ├── Asset      /                       generated index            (public)
    ├── Asset      /<slug>                 public page                (public)
    ├── Asset      /gated/<slug>           gated page body            (private)
    └── Function   /<slug>                 gate for that page         (public)
```

`IncludeCredentials=false` because the gate function needs no account credentials.
`UiEditable=true` so the service remains inspectable in the Twilio Console.

### Paths

Paths are extensionless (`/conversation-design`, not `/conversation-design.html`). Content-Type comes
from the upload, not the path extension, so clean URLs cost nothing.

A gated page's private asset sits at `/gated/<slug>` rather than `/<slug>` so it can never collide
with the Function that serves it.

### Slugs

`<basename without .html>` → lowercased → non-alphanumeric runs collapsed to `-` → trimmed of
leading/trailing `-`. `Agent-Connect-Implementation-Guide.html` → `agent-connect-implementation-guide`.

Two input files that produce the same slug is a hard error naming both files. Never resolved
silently.

### Titles

First `<title>…</title>`, with `&amp; &lt; &gt; &quot; &#39; &#x27;` decoded, whitespace collapsed,
trimmed. Absent or empty falls back to the slug in Title Case.

### `index.html` is special

An `index.html` in the input folder is uploaded at path `/` only. It is never published at `/index`,
never assigned a slug, never gated, and never appears as a row in a listing. Its presence suppresses
index generation entirely.

### Resource matching, and why nothing is ever deleted

Assets are matched by `friendly_name`, which equals the served path without its leading slash:
`conversation-design`, `gated/conversation-design`, or `index`. Functions are matched by
`friendly_name` equal to the slug.

A gating transition therefore creates a *new* Asset rather than repointing an existing one, since the
path changes. The superseded Asset stays in the account, excluded from the build.

The skill never issues a `DELETE`. `unpublish` excludes versions from the snapshot; it does not
destroy Asset, Function, or Version resources. An unpublished page can be restored by re-deploying
its file, and a stray failed upload is inert rather than corrupting.

## Endpoints used

| Purpose | Method + URL |
| --- | --- |
| List/create Service | `GET`/`POST` `https://serverless.twilio.com/v1/Services` |
| List/create Environment | `GET`/`POST` `.../v1/Services/{sid}/Environments` |
| Read live deployment | `GET` `.../v1/Services/{sid}/Environments/{envSid}/Deployments` |
| Read a build's snapshot | `GET` `.../v1/Services/{sid}/Builds/{buildSid}` |
| List/create Asset | `GET`/`POST` `.../v1/Services/{sid}/Assets` |
| List/create Function | `GET`/`POST` `.../v1/Services/{sid}/Functions` |
| **Upload AssetVersion** | `POST` `https://serverless-upload.twilio.com/v1/Services/{sid}/Assets/{assetSid}/Versions` |
| **Upload FunctionVersion** | `POST` `https://serverless-upload.twilio.com/v1/Services/{sid}/Functions/{fnSid}/Versions` |
| Create Build | `POST` `.../v1/Services/{sid}/Builds` |
| Create Deployment | `POST` `.../v1/Services/{sid}/Environments/{envSid}/Deployments` |
| Set env variable | `POST` `.../v1/Services/{sid}/Environments/{envSid}/Variables` |

Upload endpoints take `multipart/form-data` with fields `Content`, `Path`, `Visibility`. All others
take `application/x-www-form-urlencoded`.

Builds are created with `Runtime=node22` and no `Dependencies` — the gate function uses only
built-in `Runtime`, `Twilio.Response`, and `crypto`.

## Deploy flow

1. Resolve credentials. Fail fast.
2. Determine scope. The skill asks the operator "all `*.html`, or one file?" before invoking the
   script. The script itself requires `--all` or `--file <name>`; there is no default.
3. Find-or-create Service by `UniqueName`, then Environment by `UniqueName`.
4. **Read the live snapshot.** Active Deployment → its `build_sid` → `GET` that Build → capture
   `asset_versions[]` and `function_versions[]` (each carries `sid` and `path`). This is the
   carry-forward base. No live deployment yet → empty base.
5. Read `.twilio-pages.json`. Missing, partial, or stale is fine.
6. Per selected file: derive slug, extract title, compute SHA-256. Skip it and report `unchanged` only
   if **all three** hold: the hash matches the manifest, the expected path is present in the live
   snapshot, and no gating transition is requested for that slug. `--force` disables skipping.

   The third condition is load-bearing. A `--gate` on an unedited file has an identical hash, but the
   transition needs the content re-uploaded as a *private* asset at a new path — a version's
   visibility cannot be changed in place. Skipping on hash alone would report success while leaving
   the page ungated.
7. For each changed file: find-or-create the Asset, then upload an AssetVersion — `Content` as a
   `Blob` typed `text/html; charset=utf-8`, `Path` per the gating decision, `Visibility` `public`
   or `private`.
8. For each gated file: find-or-create the Function and upload a FunctionVersion containing the gate
   handler with the slug and title interpolated. `Visibility=public`.
9. Generate the index from (files in this run ∪ pages carried forward) and upload it as a public
   Asset at path `/`. A hand-written `index.html` present in the input folder is uploaded at `/`
   instead, and no index is generated.
10. Assemble the complete snapshot (see below), create the Build, poll `GET` on it every 1s until
    `completed` or `failed`, capped at 120s.
11. **Only on `completed`:** create the Deployment. Print the URLs. Write `.twilio-pages.json`.

### Atomicity

Steps 1–10 are inert. Assets, versions, and builds are invisible to visitors until a Deployment
points at them. Any failure before step 11 — crash, 401, failed build, poll timeout — leaves the
live site serving exactly what it served before. There is exactly one mutating call and it is last.

## Snapshot merge

The Twilio docs are explicit: *"You must specify all Function or Asset Versions when creating a
Build. Builds only use the provided Versions, and do not reference the Versions used by previous
Builds."* Omission is deletion. This is the highest-risk logic in the skill and the merge rules below
are normative.

Let `live` be the version lists from step 4, `fresh` the versions created this run, and `path(v)` a
version's path.

```
resultAssets    = fresh.assets    ∪ { v ∈ live.assets    : path(v) ∉ paths(fresh.assets)    ∪ dropped }
resultFunctions = fresh.functions ∪ { v ∈ live.functions : path(v) ∉ paths(fresh.functions) ∪ dropped }
```

`dropped` is populated by the transition rules:

| Transition | Fresh versions | Paths added to `dropped` |
| --- | --- | --- |
| New public page | public asset `/<slug>` | — |
| New gated page | private asset `/gated/<slug>`, function `/<slug>` | — |
| Update in place | same shape as current | — |
| **Public → gated** (`--gate`) | private asset `/gated/<slug>`, function `/<slug>` | asset `/<slug>` |
| **Gated → public** (`--ungate`) | public asset `/<slug>` | asset `/gated/<slug>`, function `/<slug>` |
| **Unpublish** (`unpublish <slug>`) | — | asset `/<slug>`, asset `/gated/<slug>`, function `/<slug>` |

The public → gated row is security-relevant. If the old public asset at `/<slug>` were carried
forward alongside the new Function at the same path, the resolution order between a public asset and
a Function sharing a path is unspecified — the asset could keep serving and silently bypass the gate.
The old public asset version must be excluded from the snapshot, not merely shadowed.

The index asset at `/` is regenerated and superseded on every deploy.

Before creating the Build, assert that no two versions in either result list share a path. A
duplicate is a bug in the merge; abort rather than deploy.

## Manifest

`.twilio-pages.json`, written to the input folder and committed. Contains no secrets — SIDs are
identifiers, not credentials.

```json
{
  "version": 1,
  "serviceSid": "ZSxxxxxxxx",
  "serviceUniqueName": "pages",
  "environmentSid": "ZExxxxxxxx",
  "domain": "pages-4821.twil.io",
  "pages": {
    "conversation-design": {
      "file": "conversation-design.html",
      "title": "Conversation Design",
      "sha256": "e3b0c442...",
      "gated": false,
      "updatedAt": "2026-08-04T17:22:10.000Z"
    }
  }
}
```

The manifest is a **cache, not state**. It supplies titles for carried-forward pages, the gated flag,
and hashes for skip-unchanged. Deleting it costs slug-derived titles and one redundant upload cycle;
it never changes what stays live, because the live build is the carry-forward source of truth.

## Index page

Dark chrome. `#0e0e13` plane, `#fff` text, `rgba(255,255,255,0.10)` hairlines, `#f22f46` accent,
`system-ui` and `ui-monospace` only — no external font requests.

Uppercase letter-spaced mono eyebrow, page count, then one row per page: title, `/<slug>` in mono,
last-updated date, and a `PROTECTED` badge for gated pages. Sorted by title. Gated pages are listed
by title — the index reveals that a protected page exists, which is acceptable since the URL alone
grants nothing.

## Gate

One Function per gated page, generated from a template with slug and title interpolated.

| Request | Response |
| --- | --- |
| `GET`, no or invalid cookie | `200` + password form |
| `POST`, correct password | `302` to self + `Set-Cookie` |
| `POST`, wrong password | `200` + form with an error line |
| `GET`, valid cookie | `200` + the private asset's HTML |

Body is read via `Runtime.getAssets()['/gated/<slug>'].open()`, returned through `Twilio.Response`
with `Content-Type: text/html; charset=utf-8`.

### Cookie scheme

- Value: `HMAC-SHA256(PAGES_COOKIE_SECRET, SHA256(PAGES_PASSWORD))`, hex.
- Attributes: `HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800` (7 days).
- Compared with `crypto.timingSafeEqual` on equal-length buffers; the submitted password is compared
  the same way.
- Rotating `PAGES_PASSWORD` changes the expected HMAC, so **every outstanding cookie is invalidated
  on rotation** — revocation with no redeploy.

Cookies are read by parsing the raw `event.request.headers.cookie` string rather than relying on a
pre-parsed helper.

### Form

Centered card, max-width ~420px, on the dark chrome. Mono `PROTECTED` eyebrow above a `#f22f46`
rule, the page title in `system-ui`, one line of muted explanatory text, a single autofocused
`<input type="password" name="password">` with a hairline border, and a `#f22f46` "View page"
submit button. No username field.

## CLI surface

```
pages.mjs deploy   (--all | --file <name>) [--gate a,b] [--ungate c] [--force]
                   [--dry-run] [--dir .] [--service pages] [--env production]
pages.mjs status
pages.mjs unpublish <slug>
pages.mjs set-password [--password <value>]      # else reads TWILIO_PAGES_PASSWORD
```

- `deploy --dry-run` prints the planned actions and the assembled snapshot without creating a Build.
- `status` prints the service, domain, live pages with visibility and last-deployed date, and flags
  local files that are not yet published as well as manifest/live drift.
- `unpublish` is the only removal path, by design.
- `set-password` generates `PAGES_COOKIE_SECRET` (32 random bytes, hex) if absent, then sets
  `PAGES_PASSWORD`. Never echoes either value. Requires no redeploy.

## Refusals and error handling

Fail before uploading anything:

| Condition | Behavior |
| --- | --- |
| Missing credentials | Name the missing variable; no network call |
| Gated file requested but `PAGES_PASSWORD` unset | Refuse — a gate with no password is either unopenable or open |
| Slug collision between two input files | Refuse, naming both files |
| File > 25 MB (public) or > 10 MB (private) | Refuse, naming the file and the limit |
| Would exceed 1,000 public or 50 private assets in one build | Refuse with current counts |
| Input not valid UTF-8 | Refuse, naming the file |
| `--gate`/`--ungate` naming a slug whose file is not in the current selection | Refuse — a transition must re-upload the content, so the file must be present |
| `unpublish <slug>` for an unknown slug | Refuse, listing live slugs |
| Duplicate path in assembled snapshot | Refuse — internal invariant violation |

Fail during execution, with the live site untouched:

| Condition | Behavior |
| --- | --- |
| `401` | "Credentials rejected — verify the API key belongs to this account" |
| `403`/`404` on the service | Report the account SID in use and the service looked for |
| Build `failed` | Print the build's error output verbatim |
| Poll exceeds 120s | Report the build SID so it can be inspected or deployed manually |
| Network error mid-upload | Report which files uploaded; the run is a no-op for visitors |

## Assumptions to verify

Three behaviors are assumed from documentation but not confirmed. Each is verified by the
integration deploy, and each has a defined fallback. They are not open questions blocking design.

1. **`Set-Cookie` survives `Twilio.Response.appendHeader`.** Fallback if not: after a correct
   password, redirect to `/<slug>?t=<hmac>` — the derived token, never the password itself — and
   accept the token from the query string.
2. **Function response body ceiling.** The largest current page is 73 KB and private assets may be
   10 MB, but the Function response limit is unconfirmed. Measure it; then refuse to gate any page
   above the measured limit with an error suggesting the page stay public.
3. **Environment domain shape with no `DomainSuffix`.** Expected `pages-<hash>.twil.io`. Whatever it
   resolves to is read back from the Environment resource and recorded in the manifest rather than
   constructed by string concatenation.

## Testing

`node:test`, zero dependencies. `fetch` is injected into the API layer, so every request sequence is
asserted against fixtures with no network access.

Unit:

- Slug derivation, including collision detection.
- Title extraction: missing, empty, entity-escaped, multiline, multiple `<title>` tags.
- **Snapshot merge** against a recorded live-build fixture — every row of the transition table,
  carry-forward, supersede-by-path, dedupe assertion.
- Cookie HMAC generation and verification, including wrong password and tampered cookie.
- Multipart body assembly for the upload endpoints.
- Index HTML generation, including a page present only in carry-forward.
- `index.html` special-casing: published at `/` only, absent from `/index` and from any listing.
- Skip-unchanged logic across all three conditions — in particular that an unedited file with a
  requested `--gate` is *not* skipped.
- Manifest read with missing, partial, and malformed files.
- Every refusal in the table above.

Integration — one manual deploy to a throwaway `pages-test` service, asserting:

- Public page returns `200` with `Content-Type: text/html`.
- Index at `/` returns `200`.
- Gated page returns the form; wrong password re-prompts; correct password serves the HTML.
- Re-running with no edits reports `unchanged` and uploads nothing.
- **Deploying one file leaves the other pages live** — the upsert invariant.
- `--gate` on a previously public page makes `/<slug>` no longer serve the raw asset.
- Rotating the password invalidates an existing cookie.

## File layout

```
~/.claude/skills/publishing-html-to-twilio-serverless/
├── SKILL.md
├── scripts/
│   ├── pages.mjs                     # CLI entry: deploy | status | unpublish | set-password
│   ├── lib/
│   │   ├── twilio-api.mjs            # injected fetch, Basic auth, form/multipart encoding
│   │   ├── snapshot.mjs              # merge rules — the normative logic
│   │   ├── page-files.mjs            # slug, title, hash, file selection
│   │   ├── index-page.mjs            # index generation
│   │   ├── gate-template.mjs         # gate function source template
│   │   └── manifest.mjs              # read/write .twilio-pages.json
│   └── test/*.test.mjs
└── references/
    └── serverless-api-notes.md       # endpoints, limits, the snapshot constraint, gotchas
```

Installed in the personal skills directory so it works on any folder of HTML, not only this repo.

`SKILL.md` instructs the operator-facing flow: ask scope (all vs one file) before running, surface
the printed URLs, and never pass a password on a command line where a shell history would capture
it.

## Reference limits

| Limit | Value |
| --- | --- |
| Public asset size | 25 MB |
| Private asset size | 10 MB |
| Public assets per build | 1,000 |
| Private assets per build | 50 |
| Build statuses | `building` → `completed` \| `failed` |
| Build poll interval | 1s, capped at 120s |
| Runtime | `node22` |
