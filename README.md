# regiontype.com

A typing drill that teaches place names. v0.3.9 — per-country administrative
divisions, UI in Korean, English, and Japanese.

## What changed in 0.3.9

- Settings now lets you pick a country and a language. Leave them unset and
  the site falls back to the browser's language and the relay's country code
  (`GET /where`)
- 46 countries' worth of administrative-division courses. Place names come
  from Natural Earth's local-language names; the one-line blurbs are still
  empty
- Korea keeps its province course next to the Seoul course. Japan gets its
  prefectures, the US gets its 50 states plus Washington DC
- A name containing a space is no longer confirmed by the space key —
  Enter confirms it instead

## What changed in 0.3.8

- Icons no longer come from a Google font — they're drawn as `assets/*.svg`
  masks instead. With zero outbound requests left, icons never go blank on a
  slow or blocked network
- An about page (`about.html`) now exists, linked from the title menu. It
  explains what a "gu" and a "dong" are and why you'd memorize them by
  typing, in a place search engines can reach
- Added link-preview metadata (OG, Twitter Card) and structured data
- The result screen now shows a global leaderboard. Only runs with the same
  course and time limit compete against each other
- Region cards got a tab. Next to the map where you pick a course there's
  now a **rank** panel, so you can see where everyone else is clustered —
  and where you'd land among them — before you even start typing

## What changed in 0.3.7

- In the light theme, correctly typed characters turn dark orange and typos
  turn red, so the two are distinguishable
- The back-arrow icon is now a local SVG so it still shows up without the
  icon font
- Shortened the mobile guidance copy

## What changed in 0.3.6

- Typing doesn't stop on a typo anymore. The field grows with whatever
  you've typed, so the screen mirrors your input exactly
- The middle dot in place names is swapped for a comma —
  `종로1·2·3·4가` can't be typed on a keyboard as-is
- Swapped the colors for correct characters and typos. They used to be close
  enough that you couldn't tell them apart at a glance

Carried over from earlier patches: the feedback dialog behind the dot next
to the logo, the grid toggle in settings, a device-theme favicon, the pixel
map and pointer proximity field shared by the title and settings screens,
the build-number readout, the prev/now/next queue and blur at the bottom of
play, Seoul's 25-district *dong*-level courses, and the `/mimi/` dev
preview.

## Versioning

In `0.3.7`, the leading `0.3` is the version and the trailing `7` is the
patch. The patch number comes straight from `VER` in `app.js` — `VER` exists
to bust the cache, so it goes up by 1 on every change (`0.69 → 0.70`), and
its **first decimal digit** is the patch number. `VER=0.70` means `v0.3.7`.

The changelog only lists **what changed in that patch**. Piling up every
past patch's entries on top makes it impossible to tell what's actually new.

The `?v=` query string is never shortened. Shortening it would collide with
an older number and defeat the cache-busting.

## Running it

    python3 -m http.server 3000

`http://localhost:3000` — it's static files only, so it deploys as-is to any
static host. Append `?rt=1` to run the answer-checking engine's self-test in
the console.

## Files

    index.html   screens (title / regions / options / play / result)
    about.html   about page — inherits only style.css tokens, never loads app.js
    style.css
    app.js       answer-checking engine + game loop + result card
    design/      design for the play/queue screens (`design.pen`)
    relay/       relay (Cloudflare Worker) — feedback → GitHub issue, leaderboard → D1
    mimi/        Mini Motorways-style board (dev only)
    data/*.course.json   entries, aliases, one-line blurbs (mode: sequence)
    data/*.geom.json     bitmap dot grids
    data/world.json         country list (build_world.py)
    data/i18n.json          UI strings (ko / en / ja). Hand-written
    data/*-pixels.json      title-screen pixel maps
    tools/build_map.py      GeoJSON → geom.json
    tools/build_dong.py     builds all 25 districts' dong-level courses in one pass
    tools/build_pixels.py   GeoJSON → pixel grid
    tools/build_world.py    Natural Earth admin-1 → country courses
    tools/build_size.py     course capacity → relay/size.mjs

## Updating map data

    curl -sL -o gu.json https://raw.githubusercontent.com/southkorea/seoul-maps/master/kostat/2013/json/seoul_municipalities_geo_simple.json
    python3 tools/build_map.py gu.json data/seoul-gu.geom.json

The 25 districts' dong-level courses are all generated in one pass. The grid
size is picked per district based on how many dots are filled in — a fixed
width would make a tall, narrow district explode with dots.

    curl -sL -o dong.json https://raw.githubusercontent.com/southkorea/seoul-maps/master/kostat/2013/json/seoul_submunicipalities_geo_simple.json
    python3 tools/build_dong.py dong.json gu.json data/

Gangseo-gu's course has a hand-written one-line blurb, so it's never
overwritten (see `KEEP` in build_dong.py). Every other course's blurb is
still empty — there was no way to fabricate copy for 400 places, so it's
left as a spot for a human to fill in.

## Title-screen pixel map

The whole country's provincial borders are plotted as a grid, with Seoul
picked out in a different color, so the roadmap — "v1 starts in Seoul, then
grows nationwide" — reads at a glance from a single image.

    curl -sL -o kr.json https://raw.githubusercontent.com/southkorea/southkorea-maps/master/kostat/2018/json/skorea-provinces-2018-geo.json
    python3 tools/build_pixels.py kr.json data/korea-pixels.json 34

The last argument is the column count (margins get trimmed, so the result
ends up narrower than that). Single-dot islands with no neighbors, like
Ulleungdo and Dokdo, are dropped as decorative noise.

## Country courses

Besides Seoul, every country's level-1 administrative divisions are plotted
from Natural Earth 10m. One-line blurbs are not fabricated for these.

    curl -sL -o /tmp/ne-admin1.geojson https://raw.githubusercontent.com/nvkelso/natural-earth-vector/master/geojson/ne_10m_admin_1_states_provinces.geojson
    curl -sL -o /tmp/ne-admin0.geojson https://raw.githubusercontent.com/nvkelso/natural-earth-vector/master/geojson/ne_50m_admin_0_countries.geojson
    python3 tools/build_world.py /tmp/ne-admin1.geojson /tmp/ne-admin0.geojson data/
    python3 tools/build_size.py

`relay/size.mjs` is the capacity table. Since the relay reads this file too
whenever a course is added, run `node relay/test.mjs` afterward to catch any
drift. `GET /where` lives on the deployed relay.

## Answer checking

Every keystroke is checked (including mid-composition Hangul input). An
abbreviation (e.g. "강남") is only accepted **when it's the only unclaimed
entry that abbreviation could still resolve to**.

- `강남` → confirms 강남구 immediately
- `중` → doesn't confirm while both 중구 and 중랑구 are still unclaimed
  (you have to type all of `중구`)
- A single-character stem is never accepted as an abbreviation
- A typo prefix is tolerated via a suffix check (`ㅋㅋ강남` → 강남구)

A wrong answer is auto-confirmed **the moment composition ends and the
result is not a prefix of any unclaimed entry's suffix** (`강난` → instant
wrong answer). The space key is never intercepted on `keydown`, since doing
so would break native IME composition confirmation itself. For a course
whose names contain spaces — where the space is a literal character — Enter
confirms the entry instead. There's no score penalty, only a broken combo.

## Feedback

The yellow dot next to the title logo is the feedback button. Hover or tap
it and it grows and reveals a flag icon; press it and a `<dialog>` opens as
a modal. There's no dedicated screen for this — the whole point is that "you
can reach us here" fits in one dot next to the logo.

Where feedback goes is controlled by a single constant near the top of
`app.js`.

    const FEEDBACK_URL = ''       // fill in to POST JSON here instead
    const FEEDBACK_REPO = 'pistolinkr/regiontype.com'

The payload sent is

    { kind, body, v, href, ua }

`kind` is bug / suggestion / place-name-or-info error. Text alone isn't
reproducible, so version, URL, and browser ride along with it. No reply
address is collected — issues are public, so writing one in would expose
someone's email outright; replies happen on the issue itself.

When `FEEDBACK_URL` is empty, it instead opens a pre-filled GitHub issue
draft for `FEEDBACK_REPO` in a new tab. That's the default because it needs
no backend at all, but it requires the reporter to have a GitHub account.
If that's unwanted, stand up the relay below instead.

### Relay (`relay/`)

You can't create a GitHub issue without a token, and putting a token on a
static site gets it stolen immediately. So a single Cloudflare Worker that
holds the token sits in between. That's its entire job — take the incoming
JSON, turn it into one issue, and hand it to the GitHub API.

    cd relay
    wrangler login
    wrangler secret put GH_TOKEN     # a fine-grained token, Issues:write on that repo only
    wrangler deploy

Paste the printed URL into `FEEDBACK_URL` in `app.js` and "Send" becomes an
actual issue filing. The token lives only inside the Worker and never ships
to the browser.

    node relay/test.mjs      checks that one issue and one score row are filtered correctly

Since it's an open endpoint that files issues, spam eventually arrives.
There's a per-IP window, and metadata is fenced inside a code block so a
submitted value can't tamper with the issue's formatting. If it leaks
anyway, Turnstile goes in front — worst case, since the token can only touch
that one repo's issues, emptying a repo that holds nothing but a README
ends it.

The window is counted with a `ratelimit` binding. **It used to be counted
with the Cache API, but that's silently ignored on a `workers.dev`
deployment** — `put` gets dropped and `match` always comes back empty, so
while the code claimed a window existed, it had never actually been active.
A wall that's quietly wide open is worse than no wall at all. So without the
binding, the endpoint now closes with a 503 instead of a silent 200.

No reply address is accepted in any form. Simply not showing an input field
for it isn't enough — as long as the relay reads that field, anyone can
stuff someone else's email address into a public issue.

### Leaderboard (`relay/` + D1)

Two more routes on the same relay. One run is defined by **course + time
limit**, and one person occupies one row within that — putting a 5-minute
run and a 1-minute run on the same board would make the scores meaningless.

    POST /score    {c, t, who, name, score, hits, tries} → {rank, top}
    GET  /top      ?c=course&t=seconds → {top}
    POST /dist     {c, t} → {bins, bucket, cap, total, score, over}
    POST /forget   {} → {gone}      removes all of your rows

    POST /auth/new  start passkey registration → challenge
    POST /auth/reg  finish passkey registration → session token
    POST /auth/go   start login       → challenge
    POST /auth/log  finish login      → session token
    GET  /auth/me   who am I right now

    cd relay
    wrangler d1 create rt-board                              # paste the printed id into wrangler.toml
    wrangler d1 execute rt-board --remote --file schema.sql
    wrangler deploy

If `database_id` is empty or nothing's deployed yet, the relay closes with a
503 and the site simply hides the leaderboard section entirely. The result
screen still works as normal.

A row's owner is `who`, not the display name — a random value the browser
generates once, never shown on screen and never included in a response.
Using the name as the owner key would let anyone plant a high score under
someone else's name and permanently lock them out of ever posting their own.
The "this is you" marker is likewise attached server-side as `me` — since
the server is the one that sanitizes and stores the name, matching it up by
name on the client would drift out of sync.

Scoring happens in the browser. All the relay checks is internal
consistency, and the ceiling is set jointly by **the course's capacity and
the time limit** — you can't visit more places than the course actually
has, and no single entry can take less than 0.5 seconds no matter how fast
you type. Without a capacity table (`SIZE`), a run claiming to have hit 500
places on a 25-place course would sail through with 250,000 points and sit
at #1 forever. A course name that isn't in the table is rejected outright.

That table is derived from `data/*.course.json`, so it goes stale over time.
`relay/test.mjs` diffs it against the source on every run and flags drift.

Even so, an honest score and a well-crafted lie can't be told apart here.
**The leaderboard is a hall of fame, not a verified record.** If abuse ever
gets bad enough to matter, the only real fix is moving scoring server-side.

### Login

**Only needed to post to the leaderboard.** The game itself runs fully
without logging in, and the distribution is visible either way — the only
thing missing is your own marker on it.

**No passwords, ever.** A static site handling passwords means owning
hashing, resets, and breach response — that's not a layer this repo is
equipped to carry. A passkey's secret never leaves the device, so **there's
nothing here to steal in the first place** — only a public key ever reaches
the server.

    wrangler secret put SESSION_KEY   # any long random value. Rotating it logs everyone out

Email magic links don't exist yet. Cloudflare Email Routing can only
receive mail, not send it, so this needs a sending provider (Resend,
Postmark, etc.) and an API key settled first.

Passkey verification is done **with zero dependencies**. At registration
the browser's `getPublicKey()` already hands back SPKI, so the server never
needs a CBOR parser. Signatures are verified with WebCrypto, after unpacking
the authenticator's DER signature into raw 64 bytes (`derToRaw`).

`attestation` is set to `none`. Instead of a device manufacturer's
certificate, the one thing this actually guarantees is **"this device
answered this challenge, from this origin."** That's all a leaderboard
needs.

Sessions are signed, stateless tokens carried via `Authorization: Bearer`.
No cookies, because the relay lives on a different origin (`workers.dev`)
than the site — and browsers keep clamping down harder on cross-site
cookies. Carrying it in a header also means no CSRF surface. Moving the
relay to `api.regiontype.com` would allow switching to an HttpOnly cookie.

**A row's owner is read from the token**, never from the request body — the
older "recovery code" scheme meant anyone who knew the code could post under
that name.

Being stateless means individual sessions can't be revoked one at a time.
In an emergency, rotating `SESSION_KEY` logs everyone out at once.

### Rank panel

Opened from the tab on a region card. It's a screen you open *before*
playing, so its purpose differs from the leaderboard on the result screen —
that one shows the run you just finished; this one shows **who you're about
to compete against**. Arrows page through courses, and the time limit
follows whatever's set in options.

The distribution is drawn out to the highest score that run could possibly
produce. Drawing it only out to where people are actually clustered would
give every run a different x-axis, making them impossible to compare. Rank
and top-percentile are only ever written from server-computed values — eyeballing
a bar's position and writing a number next to it would let what's shown drift
from what's real.

`/dist` is a read, yet it's a POST. `who` is a credential, and putting it in
the URL would leave it sitting in relay and access logs.

The display name is the only piece of another person's input that gets
shown publicly. It's truncated to 12 characters, stripped of invisible
characters (control, formatting, bidi-override), and rendered only via
`textContent`. The browser sanitizes it with **the exact same rules as the
relay** — saving a name here that would only pass client-side means every
later run from that browser gets a 400, and if the leaderboard panel then
collapses, the very form you'd use to re-enter a name disappears with it,
with no way back.

The fact that the name is public is disclosed right where you'd enter it
("this name appears on a leaderboard anyone can see"), and settings has a
**remove me from the leaderboard** option. The name updates even without
beating your score, so fixing a typo never requires setting a new personal
best. IP addresses are used only for the rate-limit window and are never
stored. What actually lives in D1 is just course, time, `who`, name, and
score.

## Accessibility

- Every feature is keyboard-operable. Toggle/step buttons carry an
  `aria-label`
- Claimed state is distinguished by more than color alone — also by
  **border and place-name label** (PRD 8.4)
- Time remaining is shown as both a gauge and a **number**, so it never
  depends on color contrast alone
- `prefers-reduced-motion` is respected, and animation can also be disabled
  individually in settings
- The mobile guidance screen switches on `pointer:coarse`, not viewport
  width — so a desktop user zoomed to 200% never gets bounced to it

## What's left before v1

Handling legal-dong / same-name-different-place cases, a location-based
mode, server-side ranking, a course editor. Only Gangseo-gu's dong-level
courses have their one-line blurbs filled in.
