---
name: travel-itinerary-timeline
description: Build a mobile-first, single-file HTML web app that shows a trip itinerary as a day-by-day vertical timeline, auto-selects today's day and highlights the current activity in real time, and lets the user tap a stop to expand its details (and optionally a photo + short description). Use when the user asks to turn a trip schedule / itinerary / education-trip schedule into a web app, a "일정표 앱", "일정 타임라인", "여행 일정 웹앱", a schedule page they'll mainly check on their phone, or says to reuse "the method used before" for a trip timeline. Covers the requirement-gathering questions to ask first, the reusable HTML/CSS/JS skeleton (template.html in this skill folder), picking a subject-specific visual identity instead of recycling one look, place-info enrichment via web research with the sandbox's network limits, testing with Playwright, and publishing to both git and a Claude Artifact.
---

# Travel itinerary timeline app

A single `index.html` (no build step, no framework) that renders a trip's
schedule as day tabs + a vertical timeline, highlights whichever stop is
happening right now based on the viewer's own clock, and expands on tap.
This was built once for a Jeju school trip; this skill generalizes that
method so it can be re-run for any trip without re-deriving the design.

`template.html` in this same folder is the proven skeleton (CSS tokens,
tabs, timeline, highlight engine, tap-to-expand, photo-with-fallback,
a manual refresh button for forcing a reload on a phone that's holding a
stale cached copy). Copy it to the project as a starting point — do not
rewrite the engine from scratch each time, only the content and the
visual identity. Note past items are marked only by a dimmed timeline
dot/line, not a dimmed card — an earlier version faded the whole past
card to 55% opacity and that was deliberately reverted (it read as
"broken" more than "done" on a glance-and-go phone screen).

## 1. Ask before building

Don't start writing code from a one-line request. Ask (AskUserQuestion is
fine, 2-3 questions, then proceed — don't keep asking after this):

- **Data source**: will the user paste the actual schedule text (most
  common — then you parse it yourself, no in-app editor needed), do they
  want to add/edit stops inside the app (needs localStorage, more work),
  or does it need to be shared/edited across multiple people's phones
  (needs a backend — Artifact DB capability or similar; confirm this is
  really wanted before building it, it's a different app).
- **Layout**: day tabs + vertical timeline (most common for multi-day
  trips, what template.html implements), a single long scroll with no
  tabs (fine for a 1-day itinerary), or a horizontal time-axis calendar
  view (rarer, more complex — confirm before attempting).
- **Interactivity**: tap-to-expand detail (default, already in the
  template), a "done" checkbox, a map link, auto-scroll-to-current-item.
  Keep it to what's asked; don't pile on features nobody requested.
- **Dates**: get the actual trip dates before writing the DAYS array —
  the auto-highlight logic is useless without real calendar dates to
  compare the viewer's clock against.

Then ask the user to paste the itinerary text. Parse it into a per-day
list of `{ start, end?, title, detail? }` items yourself — don't ask them
to pre-format it as JSON.

## 2. Design a visual identity for THIS trip, not a reused default

Before touching `template.html`, spend one minute on a tiny design plan
(see the `artifact-design` skill if publishing to an Artifact too — load
it then, since it governs Artifact output specifically). Pick:

- **Accent color + neutral tone** grounded in the actual place/subject —
  not a generic AI teal/green or warm-cream-terracotta default. For the
  Jeju trip this was tangerine orange (the region's citrus) + basalt-grey
  neutrals. A different trip needs a different pair — derive it from that
  place's own material, not from what was used last time.
- **One typographic touch** tied to the subject if there's a natural one
  (the Jeju build used a monospace face for clock/time digits — a
  boarding-pass/departure-board feel, since the trip starts and ends at
  an airport). Skip it if nothing fits; don't force it.
- Keep everything else (layout, card structure, timeline mechanics) as
  template.html already has it — that part isn't where the creative
  decision belongs.

Implement both light and dark themes properly: a bare `:root` light
palette, `@media (prefers-color-scheme: dark)` guarded with
`:root:not([data-theme="light"])`, and a `:root[data-theme="dark"]`
override block. template.html already has this structure — just swap the
hex values.

## 3. Fill in the data

In `DAYS`, one object per day: `date` (`YYYY-MM-DD`, real calendar date —
required for the auto-today-tab and highlight logic to work), optional
`theme`/`subtitle` banner text, and `items[]` of
`{ start: "HH:MM", end?: "HH:MM", title, detail? }`. Leaving `end` off is
fine — the renderer fills it from the next item's `start` automatically
(see `withEffectiveEnd` in the script), so back-to-back items don't need
redundant end times.

## 4. Optional: place-info cards (photo + description on tap)

If the user wants real attraction info (what was asked for as "비짓제주
같은 곳에서 여행 정보를 찾아서 넣어줘"), add a `PLACES` map and reference
it from the relevant items as `place: PLACES.slug`. Only do this for
actual destinations, not transit/meal/logistics items.

**Description text**: use WebSearch (not WebFetch — see network note
below) to pull 2-4 factual sentences per place from an encyclopedic
source (Wikipedia, local tourism-board sites, Namuwiki-equivalent).
Paraphrase, don't copy verbatim, and keep a source link + label so the
card can credit it ("위키백과에서 더 보기 ↗").

**Photos — read this before attempting anything**: this sandbox's
outbound network is an allowlist (package registries, github raw, a
couple of Anthropic hosts, Google Fonts) — not a blocklist of specific
sites. General web hosts including Wikimedia, Wikipedia, tourism-board
sites, and every common stock-photo/CDN host (Unsplash, Pixabay, Imgur,
jsDelivr, unpkg, Google Maps' photo-serving domains) are all blocked
identically. Confirm this is really what's happening — don't assume —
with:

```
curl -sS http://127.0.0.1:35745/__agentproxy/status
```

and a couple of direct `curl -o /dev/null -w "%{http_code}\n" https://<host>`
probes if still unsure. If they come back `403`/`connect_rejected`, this
is an organization policy, not a bug — per `/root/.ccr/README.md`, report
it, don't route around it, and don't keep trying different providers
hoping one is unblocked (it won't be — it's not a per-site block).

So: **do not try to download photo bytes in this session yourself.** Two
paths that actually work, and prefer the first — the Jeju build started
with plain `https://` hotlinks and later migrated every photo to
base64 specifically because hotlinks kept breaking (hotlink protection,
URL expiry, and they never render inside a Claude Artifact's CSP at all):

1. **Base64 data URI, embedded directly in `PLACES[...].img`** —
   `"data:image/jpeg;base64,<...>"`. This is the one that survived:
   no hotlink/CORS/expiry issues, and — because a `data:` URI is inline
   page content, not an external fetch — it's the *only* way a real
   photo renders inside a Claude Artifact preview too, whose CSP blocks
   essentially every external image host. You still can't fetch the
   source bytes yourself in this sandbox, so get them the same way as
   path 2 below (the user supplies the URL or the file), then base64
   this session's end: `base64 -i photo.jpg` (or `-w0` on Linux to
   avoid line-wrapping) if the user uploaded a file, or ask them to
   paste the image URL and only reach for path 2 if fetching it
   yourself is still blocked.
2. **Plain external `<img src="https://...">` hotlink** — ask the user
   to open the source page themselves, copy the real image URL
   (right-click → "copy image address" on desktop, long-press on
   mobile), and send it back; use it as-is. This needs no fetch on your
   end, but only renders in the real deployed page, never inside an
   Artifact preview, and can break later if the source site changes its
   URLs. Treat it as a fallback when base64 isn't practical (e.g. a
   large batch the user won't want to re-visit one by one), not the
   default.

For either path, when asking a search tool (WebSearch) to produce a raw
image URL directly rather than a page URL, treat the answer as
unverified unless the exact string also appears in the tool's own
returned links/snippets — it will sometimes fabricate a plausible-
looking CDN URL that was never actually in the results. Never ship a URL
(or encode it) that you can't trace back to real search output.

**Always degrade gracefully when a photo is missing or fails to load** —
never a broken-image icon. `template.html`'s `.place-photo` block does
this: an `onerror` handler on the `<img>` sets `this.hidden = true` and
reveals a sibling `.photo-fallback` (icon + caption). Two CSS traps that
will silently defeat this if you rebuild it from memory instead of
copying the template:
- The `hidden` attribute only hides an element via the UA stylesheet's
  `[hidden] { display: none }`, which loses to *any* later same-specificity
  rule that sets `display` unconditionally (`.place-photo img { display:
  block }`, `.photo-fallback { display: flex }`). Add explicit
  `.place-photo img[hidden] { display: none }` /
  `.photo-fallback[hidden] { display: none }` overrides, or the fallback
  never visually appears even though `hidden` is set correctly in the DOM.
- Don't add `loading="lazy"` to a photo that lives inside a collapsed
  accordion (`max-height: 0; overflow: hidden`) — lazy-loading's
  viewport heuristic can permanently skip an element that's never had
  real layout height, so the image never loads even after the user
  expands the card.

If the user wants locally-hosted image *files* in the repo instead of
either of the above (rare now that base64 covers the Artifact case too
— mainly useful if they'd rather manage photos as files than paste
base64 blobs), set `img: "images/<slug>.jpg"` and write an
`images/README.txt` listing the exact filenames needed and the source
page for each, so they can fill the folder themselves later via GitHub's
web upload with zero further code changes on your end.

## 5. Test before calling it done

- `node --check` the extracted `<script>` body (syntax only).
- Serve the file locally (`python3 -m http.server`) and screenshot it
  with Playwright at a phone viewport (~390×844). Check if `playwright`
  is importable first; `pip install --quiet playwright` if not — the
  Chromium binary itself is already preinstalled
  (`/opt/pw-browsers/chromium`, `PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1`), so
  only the Python/Node bindings need installing.
- To verify the time-based highlight logic without waiting for a real
  clock to line up, inject a fake `Date` via `page.add_init_script(...)`
  before `goto`, pinned to a timestamp that falls inside one of the trip
  days, and confirm the right item gets `.current` + the "진행중" badge
  and earlier items get `.past`.
- Check both themes (`color_scheme="dark"` on `new_page`).
- One screenshot pass is enough — fix what it shows, don't loop
  screenshotting the same thing repeatedly.

## 6. Publish

**To git**: commit the real `index.html` (plus an `images/` folder and
its README if used) to the project repo and push. This is the durable,
deployable copy — works as-is on GitHub Pages or any static host, no
build step.

**To a Claude Artifact** (for an instant mobile-shareable link without
waiting on a deploy): load the `artifact-design` skill first, then
publish a trimmed copy — same `<title>`/`<style>`/body/`<script>`, but
with the `<!doctype>`/`<html>`/`<head>`/`<body>` wrapper tags stripped
(the Artifact tool supplies its own skeleton). Tell the user explicitly
that the Artifact preview's CSP blocks external image hosts (and most
external stylesheet/script hosts beyond Google Fonts + a few script
CDNs), so hotlinked photos that work fine in the real deployed page may
not render inside the Artifact preview itself — that's an Artifact
platform limit, not a bug in the page.

Keep both copies in sync on later edits — regenerate the Artifact
fragment from the git `index.html` rather than hand-editing it
separately, so they don't drift.

## 7. Deployment questions (Vercel, etc.)

If asked to deploy to a third-party host directly from this session,
check reachability first (`curl -o /dev/null -w "%{http_code}\n"
https://<the host's API/dashboard domain>`) rather than assuming it'll
work. Most third-party deploy targets (Vercel's API included) sit
outside this sandbox's allowlist, and even when reachable, deploying to
the user's own account needs their own login — never ask for or accept a
pasted auth token to do this on their behalf. Point them at the GUI path
instead: connect the GitHub repo from the host's own dashboard (their
browser, their login) — this is usually the officially-recommended path
anyway, not a fallback — and it auto-redeploys on every future push to
the branch. If they ask whether using a different Claude Code client
(desktop app vs web) changes this: it doesn't — the network restriction
belongs to this remote container, not to whichever app is just
displaying the session. Only a Claude Code instance actually running
locally on their own machine has the user's normal, unrestricted network
access.
