# TF-Message — Trello Power-Up

Client-communication Power-Up for the UC Davis Tech Foundry (Davis/GBSF + Aggie Square, Sacramento). Staff generate standardized reply emails (pickup notices, drop-off and directions guides, quotes, holds, appointments, status updates) from a Trello project card, with the client's details pre-filled from the Zapier-generated card description. Internal tool, used daily by TF staff. One of a suite of sibling Power-Ups (TF-Recharge, TF-Manufacture, TF-Plan, TF-QMS) that share a common chassis and visual language.

**Current version: 1.26** — the whole Power-Up is the single file `tf-message.html`, served via GitHub Pages from this repo at `https://tf-staff.github.io/tf-message/tf-message.html`. The Trello Power-Up's iframe connector URL points at that file. There is no build step: edit the file, bump `VERSION`, push.

## Non-negotiable conventions

1. **Bump `var VERSION` on every functional or visual change.** The version is shown in the Trello card-back section title (`TF-Message · v1.26`) and the composer header; it is how Steven confirms GitHub Pages' ~10-minute cache has caught up. Patch-level (`1.17.1`) for cosmetic fixes, minor for features.
2. **No em dashes in any client-facing message text.** All generated prose uses periods, colons, semicolons, or commas instead. Em dashes are fine in UI chrome (token descriptions, placeholders, warnings) that clients never see. A sweep of 1,944 renders (all templates × sites × tiers × variants × platforms) is the bar; keep it at zero.
3. **No "Good news" openers.** Pickup messages open with "Your project is complete and ready for pickup" / "Part of your project is ready for pickup."
4. **Palette: Trello dark-card grays plus blue only**, matching TF-Recharge/TF-Manufacture. Green only for the transient "copied" flash, amber only for missing-value markers and warnings, red only for errors. Light mode is supported via `[data-color-mode="light"]`.
5. **The card-back (condensed) panel does one-tap actions only.** Nothing on it reveals input fields. Messages that need typed details hand off to the composer, opened already at that message. Nothing is reachable two ways.
6. **Messages are data, not code.** Every template body is plain text with `{tokens}`; staff edit bodies, button names, and hyperlinks board-wide from the ✎ Edit panel without touching the file. Keep new templates in this form.
7. **Verify with a headless harness before shipping.** See "Verification" below. Steven prefers simulation-first iteration; every change in the build history shipped with a passing jsdom regression run.
8. **Work only on the current file.** Never regenerate from an older version. If a session's container resets, the last shipped build lives on `main` here.
9. **Merge to `main` immediately after development and testing pass.** Develop on the session branch, push it, then fast-forward/merge into `main` and push `main` so GitHub Pages deploys. No PR step.

## Architecture: the four-mode chassis

One HTML file plays four roles, chosen in `boot()` by `?view=` and whether it is framed:

| Mode | How it's loaded | What it does |
|---|---|---|
| **connector** | Trello loads the bare URL in a hidden iframe | `TrelloPowerUp.initialize({'card-back-section'})`. Reads card name/desc, `msgTpl`, `msgLinks`, and `t.member('fullName')`; packs them into a `d=` URL param; returns the summary iframe + an "Open" action that launches the editor modal. |
| **summary** | `?view=summary&d=…` | Condensed card-back panel. |
| **editor** | `?view=editor&d=…[&tpl=&site=&scope=]` | Full composer in a Trello modal. |
| **standalone** | File opened directly in a browser | Empty composer (handy for previewing wording). No simulator bar anymore; hydrate via the console if needed. |

**Data routing: URL params are primary, the iframe channel is fallback.** Card data, wording overrides, link overrides, and the member's first name travel in the `d` param (JSON, URL-encoded). This was the lesson from the recharge simulator's flaky iframe channel. Both summary and editor also fall back to `t.card()` / `t.get()` if `d` is absent.

**Critical: the summary URL must be deterministic.** Trello re-invokes card-back sections constantly during its normal render cycle. Any volatile value in the iframe URL (a timestamp cache-buster, the sign-off preference) causes Trello to reload the iframe every cycle. Symptoms seen in production: white flashing, ghost text, and user selections (location slider) resetting. Rules that follow:
- No `cb=Date.now()` in URLs. The data itself is the cache key; the URL changes only when the description, overrides, or links change.
- The **sign-off preference is never in the URL**. Summary and editor fetch `t.get('board','private','signoff')` themselves.
- A pre-paint guard in `<head>` (`html.pre-hide-app #app{display:none}`) hides the composer markup in framed non-editor contexts, and `html{background:transparent}` prevents white frames.

**Saving uses a close-proof dual path**, same as the sibling Power-Ups: write via the iframe's `t.set`, and on failure relay through `BroadcastChannel('tf-msg')` to the connector, which still holds a live Trello handle (`saveTpl`, `saveLinks`, `saveSignoff` message types).

## Data model (Trello pluginData)

| Key | Scope | Content |
|---|---|---|
| `msgTpl` | board / shared | `{tplId: {t: text, n: name}}`. `t` is a string for single-text templates, or `{variantKey: text}` for `texts` templates (pickup handoff variants). Legacy string-valued entries from early versions still load. Only *changed* templates are stored; untouched templates track the file defaults. |
| `msgLinks` | board / shared | `{key: {label, url}}` — edits to the seven built-in links plus any staff-added links (key slugged from the label). |
| `signoff` | board / private | `'personal'` or `'team'` — per member, per board. |

**4 KB cap policy: refuse, never truncate.** `OVR_LIMIT = 3800` chars for `msgTpl`; the Edit panel shows a live budget, turns amber at 85%, and rolls back a save that would overflow with a warning. Trello silently truncates/rejects past ~4096.

Locker codes and site facts live in `CONFIG` at the top of the file. `CONFIG.LINKS` holds Steven's published Box share links (all eight set as of v1.19) but `msgLinks` overrides take precedence, so staff should edit links in the Edit panel, not the file.

## The description parser (`parseDesc`)

Card descriptions are generated by Zapier from Jotform intake, in two dialects:

**Internal** (recharge clients): `Collection/Delivery Preference: Collection at Davis Location | Collection at Aggie Square Location (Sacramento)`, `Client Phone:`, `Client Email: [x](mailto:x)`, `Secondary Contacts:`, `Department: … PI: …`, `Project Billing: Aggie Enterprise: <acct>`, `Authorization  Complete my request so long as it's within my budget | Send me a quote before proceeding`, `Max Budget: $ $900`, `Target Completion Date MM/DD/YYYY`, and a trailing `Purchasing Snippet:` block of the form `Supplies for internal recharge project: <Name> - <Project> - , Recharge information:  Aggie Enterprise Acct:   <acct>`.

**External**: header line `EXTERNAL SERVICE REQUEST`, `Ideal Date of Completion:` instead of Target, no account string, and the snippet form `Supplies for external recharge project:  <Name>-  - <Project>` (note the odd spacing/dashes). Tier is detected from the header.

Parser rules and hard-won pitfalls:
- Account string = the longest hyphen-joined alphanumeric run with ≥6 segments anywhere in the text.
- Name/project come from the snippet: everything after `recharge project:`, cut at ` - ,` if present (internal), then split at the first `-`. Handles both dialects.
- **Strip markdown horizontal rules** (`---`, `___`, `***`) that Zapier sometimes emits under the "Purchasing Snippet:" heading; without this the copied snippet started with `------`.
- The snippet is treated as **verbatim bytes**: rendered and copied through a `pre` block so Zapier's exact spacing survives (purchasing reconciles on it).
- Location is `davis` / `as` / `unknown` (no preference found). `unknown` is a real state with its own message family, not an error.
- "Usable data" = any of email, snippet, account, client name, site, or any email address anywhere in the text. With none, the card-back panel collapses to a bare "No message data found" (no background, no italics, matching TF-Manufacture's empty state). Card title alone does not count.

## Template engine

Each template: `{ id, group, site: 'as'|'davis'|'unknown'|'any', tier?: 'external', quick?: true, title, fields:[…], tokens?(v,c), text | texts:{box,inperson,delivery} }`.

- **Rendering:** `expandText(t)` substitutes `{tokens}` from `baseTokens()` (greeting, signoff, ready/partialNote, locker/code, addresses, project refs, siteFacility…) merged with the template's own `tokens()`. `textToBlocks` then turns `1. ` lines into `<ol>`, `- ` lines into `<ul>`, blank lines into paragraphs; `{quoteBlock}` and `{snippetBlock}` lines become verbatim `pre` blocks.
- **Overrides:** `effectiveText`/`effectiveTitle` consult `OVR` first. For `texts` templates the override is keyed by variant (`state.vals.handoff`).
- **Field types:** `text`, `textarea`, `select`, `seg` (segmented, with `def` default and `options:[[value,label]]`), `check` (with optional `disables:` to grey out another field), `lockers` (numbered tiles 2–8; tile 1 is greyed and reserved for RadOnc), `quote` (textarea + clipboard-paste button; auto-extracts `TOTAL: $x` into the amount field). Fields support `showIf(vals)` for conditional reveal (used for the in-person Reason dropdown). `buildFields(hostSel)` restores values on rebuild.
- **`quick:true`** marks templates with no required input. These get a one-tap "Copy message" on the card-back panel; all others get "Open in composer…".
- **Pickup closing:** the three pickup templates (all handoff variants) end with `{completionNote}\n\n{closing}` instead of `{signoff}`. On Entire scope `completionNote` is the outcome/feedback/photos paragraph with the "any questions" line blended in; on Partial it is just "Please let us know if you have any questions." `{closing}` is the sign-off minus the questions line.
- **Sign-off token:** Team → "Best regards, / The UC Davis Tech Foundry Team"; Personal → "Best regards, / <member first name> / UC Davis Tech Foundry" (degrades to org line if Trello withholds the name).
- Tier-aware tokens (`authLine`, `acctPrompt`, `acctBlock`, `billingLine`, `startLine`) switch wording for external clients (DocuSign estimate flow, no Aggie Enterprise language).
- **Addresses already contain room numbers.** Never prepend "Room 1730" to `asAddress` (this duplicated the room in v1.14; fixed).

### Template catalog (24)

| Group | id | site/tier | quick | notes |
|---|---|---|---|---|
| Pick-up/Drop-off | pickup_as | as | | Scope seg; Handoff seg (Collection box / Scheduled in-person / Delivery-install) with per-variant texts; locker tiles (box only); Reason dropdown (in-person only) |
| | pickup_davis | davis | | same minus lockers (TEAM box, code 1125) |
| | pickup_radonc | as | | locker 1, "Radiation Oncology Collection Point" |
| | dropoff_as / dropoff_davis | as / davis | ✓ | |
| | ask_location | unknown | | asks client which site; is the card-back default under Unknown |
| Directions | dir_davis / dir_as | davis / as | ✓ | "Visiting us" |
| | dir_both | unknown | ✓ | "Our locations" |
| Appointment | appt_propose | any | | Exact/Range time seg; Platform seg (In person / Zoom / Phone / Client choice); in-person address follows location slider, Unknown offers both |
| | appt_request | any | ✓ | Platform seg |
| Job status | received | any | ✓ | authLine branches on quote-first vs proceed-within-budget vs external |
| | quote | any | | quote-paste field, amount, valid-for |
| | docusign | external | ✓ | "Estimate sent (DocuSign)" |
| | hold | any | | reason dropdown incl. Supplies |
| | production | any | ✓ | optional projected date/time |
| | supplies | any | ✓ | expected arrival + TBD checkbox |
| | delay | any | | revised date + TBD checkbox |
| | shipped | any | | carrier + tracking |
| | overdue, done_fb | any | ✓ | |
| Admin | welcome | any | ✓ | tier-aware billing/intake lines |
| | psnip | any | | verbatim purchasing snippet |
| | closure | any | | dates + reopening |

## UI layout

**Composer (editor modal):** title + version + ✎ Edit → client strip (click-to-copy chips: name, email, phone, PI, Dept, Acct truncated-display/full-copy) → labeled control bar (Location: Davis/Aggie Square/Unknown · Client type: Internal/External · Sign-off: Personal/Team, each label grouped with its slider so they wrap as a unit) → Message type seg (Pick-up/Drop-off · Directions · Appointment · Job status · Admin) → template chips filtered by site + tier → fields → preview (contenteditable; manual edits suppress auto-rerender until Reset/re-pick) → Copy message / Copy plain text / Reset preview. Switching message type deselects and clears the preview.

**✎ Edit panel ("Edit message"):** template dropdown (grouped, shows "• modified"), Button name input, Variant seg (only for `texts` templates), textarea, clickable token legend, Hyperlinks manager (label + URL rows for all links, amber border when unset, Insert-at-cursor, + Add link), Save for this board / Revert to default, storage budget.

**Card-back (condensed) panel, rows in order:** Copy (Project tag = card title, Purchasing snippet, Client email, Email 1..N for every other address) → Location → Sign-off → Message (grouped, alphabetically sorted dropdown + one context button) → Scope (Entire/Partial, pickup only) → Pickup (RadOnc tile, locker tiles 2–8 / TEAM box button / "which location?" note; pickup only). Purchasing snippet and RadOnc are excluded from the dropdown because they have their own chip/tile. Flipping Location resets the dropdown to that site's pickup default.

## Clipboard

`copyRich(html, text)` tries the async Clipboard API with `text/html` + `text/plain`, then falls back to **`copyRichFallback`**: a hidden contenteditable, selected, `execCommand('copy')`. This fallback is what actually runs inside Trello iframes (Clipboard API is permission-blocked there) and is what keeps hyperlinks, lists, and bold intact in Outlook/Gmail. Before it existed, copies silently degraded to plain text and links vanished. Rich payload is wrapped in Calibri 11pt. Missing-value `<span class="miss">` markers are stripped to text in copied HTML.

## Verification (harness)

No test files are committed; the pattern used throughout was an ad-hoc jsdom script. Reproduce with:

```bash
npm install jsdom
```

```js
// composer: load with the Trello script tag removed → boots standalone
const html = fs.readFileSync('tf-message.html','utf8').replace('<script src="https://p.trellocdn.com/power-up.min.js"></script>','');
const dom = new JSDOM(html, { runScripts:'dangerously', url:'https://x.com/f.html' });
// then after ~300ms: dom.window.eval(`hydrate('Card title', 'description text')`), click .tpl chips, read #pv

// summary: stub TrelloPowerUp and pass data via ?view=summary&d=
const stub = `<script>window.TrelloPowerUp={iframe:function(){return{set:function(){},get:function(){return Promise.resolve('')},sizeTo:function(){},getContext:function(){return{theme:'dark'}},modal:function(o){window.__modal=o;}}},initialize:function(){}};<\/script>`;
const d = encodeURIComponent(JSON.stringify({n:'Card', d:DESC, o:{}, l:{}, m:'Steven'}));
// url: 'https://x.com/f.html?view=summary&d='+d ; monkeypatch dom.window.copyRich / copyPlain to capture payloads
```

Also run `node --check` on the extracted `<script>` body. Useful invariants to assert: zero em dashes across all renders; exactly one room-number mention per facility line; Project tag chip never triggers the non-empty state alone; identical `d` → identical summary URL from the connector.

Three real Zapier descriptions (Hoehne/Davis/internal, Pivetti/no-pref/internal, Johnson/AS/internal) plus the Koury external sample were the parser fixtures; they appear in earlier test scripts in the chat history but not in the repo. Worth committing as `test/fixtures/` next time one is touched.

## Version history

1.0 composer + 16 templates · 1.1 removed occupancy tracking, numbered locker buttons, scope seg, subject lines removed · 1.2 Settings wording editor (board-wide) · 1.3 category seg, renamed TF-Message · 1.4–1.5 snippet moved into Admin; Pickup/Drop-off merged · 1.6 Recharge-quote paste with TOTAL extraction · 1.7 TBD checkbox, click-to-copy client strip · 1.8 simulator stripped, deploy · 1.9 card-back quick-action panel · 1.10 Appointment category, labeled panel · 1.11 ✎ Edit with renaming + hyperlink manager, exact/range time · 1.12 Personal/Team sign-off via `t.member`, Unknown location family · 1.13 Internal/External tier, external parser dialect, DocuSign flow · 1.14 Handoff variants (in-person reason, delivery/install), Supplies on order · 1.15 label cleanup, room-dup fix · 1.16 version in section title · 1.17 sign-off own row, empty-data collapse · 1.18 Scope on card-back · 1.19 **deterministic URLs + pre-paint guard (flash fix)** · 1.20 sign-off out of URL (location-reset fix), rich clipboard fallback, snippet rule-stripping, RadOnc tile, mailto removed · 1.21–1.23 card-back reworked: one-tap only, dropdown picker with context button, Copy row first · 1.24 Project tag chip · 1.25 In production template, "Good news" removed, Davis link relabeled · 1.26 completion note on pickup messages (Entire scope only).

## Open items / backlog

- **Next deploy (queued by Steven):** `CONFIG.FACTS.davisAddress` is `451 Health Sciences Drive, Room 1703 (GBSF), Davis, CA 95616`, which renders as brackets-in-brackets in the Davis pickup box text ("rear of GBSF (… Room 1703 (GBSF) …)"). Fix: drop the brackets around GBSF in the address, i.e. `451 Health Sciences Drive, Room 1703 GBSF, Davis, CA 95616`. Related: `dir_both` and `ask_location` lines (`- Davis: Room 1703 (GBSF), {davisAddress}`) repeat the room number already in the address; check while there.
- Commit the jsdom harness and parser fixtures to the repo (`test/`), so verification is one `node test/run.js`.
- `CONFIG.FACTS.asHours` has a comment flagging that the three source PDFs disagree on Aggie Square opening time (6:00/6:30/7:00). Steven has since set it to "7:00 AM–6:00 PM, weekdays and Saturday"; confirm against the published Box PDFs and remove the comment.
- Cross-Power-Up quote insertion: TF-Recharge could write its quote text into card pluginData for TF-Message to pick up without the clipboard hop (needs a shared key convention; Power-Ups can't read each other's pluginData directly).
- Staff may want the RadOnc / Purchasing-snippet exclusions from the card-back dropdown revisited once the panel has had real use.
- Wording overrides saved before v1.22 may still contain em dashes; Revert-to-default in the Edit panel picks up the clean text.
