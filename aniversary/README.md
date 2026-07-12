# Anniversary Timeline (TRMNL private plugin)

Shows today's date in large type on the top half of the screen, and a
horizontal timeline of upcoming/recent special dates (birthdays,
anniversaries, one-time events) on the bottom half.

## Files

| File                    | Purpose                                                        |
|--------------------------|----------------------------------------------------------------|
| `shared.liquid`          | Paste into the **Shared** tab. All logic + CSS lives here.     |
| `full.liquid`             | Paste into the **Full** markup tab.                            |
| `half_horizontal.liquid`  | Paste into the **Half horizontal** markup tab.                 |
| `half_vertical.liquid`    | Paste into the **Half vertical** markup tab.                   |
| `quadrant.liquid`         | Paste into the **Quadrant** markup tab.                        |
| `settings.yml`            | Reference spec for the Custom Fields to create in the dashboard (not an upload — TRMNL's field builder is a UI, this file just documents what to enter). |

## Setup

1. In the TRMNL dashboard: **Plugins → Private Plugin → + New**.
2. Name it "Anniversary Timeline", strategy **Static / No Code** (no polling
   URL or webhook needed — everything is driven by custom fields).
3. Under **Custom Fields**, add the four fields described in `settings.yml`:
   - `special_dates` — long text area
   - `months_back` — number, default `1`
   - `months_forward` — number, default `4`
   - `author_bio` — a way for users to reach you (required by TRMNL's
     marketplace checker for public plugins; not read by any template)
4. Open the **Markup Editor** and paste each `.liquid` file into its matching
   tab (Shared / Full / Half horizontal / Half vertical / Quadrant).
5. Use the live previewer to sanity-check rendering, then add an instance and
   fill in your dates.

## Entering dates

`special_dates` is one event per line:

```
DATE|Label|RECURRENCE
```

- `RECURRENCE` is `once` or `yearly`.
- For a one-time event, `DATE` must be a full `YYYY-MM-DD`.
- For a yearly recurring event, `DATE` can be:
  - `YYYY-MM-DD` — a real start date (birth year, wedding year). The
    timeline shows an "N yrs" badge each year computed from that year.
  - `MM-DD` — just month and day, if you don't want to track/expose a start
    year. No age badge is shown.

Example:

```
2026-08-20|Wedding Anniversary|yearly
1990-03-02|Mom's Birthday|yearly
03-15|Unknown-year Birthday|yearly
2026-12-25|Office Holiday Party|once
```

Blank lines are ignored. The timeline window defaults to 1 month back / 4
months forward from today (in the device owner's timezone); both are
user-configurable per instance via `months_back` / `months_forward`. Note
that `months_back`/`months_forward` only affect the graphical timeline
(full/half_horizontal/half_vertical) — the quadrant layout and the "Next: ...
in N days" caption always look at the closest upcoming occurrence(s)
regardless of that window, since there usually isn't room to show much on a
quadrant tile anyway.

## What's new

- **Dot shape**: one-time events render as a square dot, yearly-recurring
  events as a circle, so you can tell them apart on the timeline at a
  glance without reading labels.
- **"Next: ... in N days"**: the full layout shows a one-line summary under
  the year, naming the single closest upcoming date and a day count,
  independent of what's visible on the timeline below it.
- **Empty state**: if nothing falls inside the graphical window, the
  timeline shows "No dates in range" instead of a bare line.
- **Quadrant redesign**: quadrant no longer tries to cram a timeline into
  ~400×240px. It shows the closest 1–2 upcoming dates as a compact text
  list (date, label, days-until, age badge), which is far more legible at
  that size. If nothing is upcoming at all, it shows "No upcoming dates".

## How it works (confirmed on-device)

- **Custom field values are nested**, not top-level merge variables. A
  field with keyname `special_dates` shows up at
  `trmnl.plugin_settings.custom_fields_values.special_dates`, not as a bare
  `{{ special_dates }}`. All four layout templates read it from there and
  pass it explicitly into `{% render "anniversary_screen", ... %}` — if you
  add more custom fields later, remember to both read them from
  `trmnl.plugin_settings.custom_fields_values.<keyname>` in the layout file
  and add them to the `render` call's parameter list, since `{% render %}`
  uses an isolated variable scope (nothing from the caller is visible
  inside `shared.liquid` unless it's passed in explicitly — that's also why
  `trmnl` itself is passed through on every render call, so `shared.liquid`
  can read `trmnl.user.utc_offset`).
- **Date math** uses Liquid's `date: "%s"` filter to turn date strings into
  Unix timestamps for comparison/positioning, confirmed working on-device.
  "Today" comes from `"now" | date: "%s"` shifted by `trmnl.user.utc_offset`
  for local time (`trmnl.system.timestamp_utc` is also populated and would
  work equally well as the UTC source).
- **Window sizing** approximates a "month" as 30 days for placing the
  start/end of the timeline and the month-tick ruler. Tick labels can drift
  a day or two near month boundaries — cosmetic only.
- **Dot positioning**: each dot (today or event) is its own absolutely
  positioned, auto-sized wrapper centered exactly on the timeline
  (`transform: translate(-50%, -50%)`), and its label is a *separate*
  absolutely positioned child anchored above/below the dot. Coupling the
  label into the same centered block (an earlier version did this) throws
  the dot itself off-center by roughly half the label's height — worth
  remembering if you adjust the timeline CSS later.
- **Collision handling** for events close together is just alternating
  above/below the line (`stagger` on `dot_count`) — with several events in
  a short window, labels can still overlap. If that becomes an issue, the
  fix is either shortening the default window or thinning labels (e.g.
  dot-only for events within N days of another).
- No native "timeline" component exists in the TRMNL framework, so it's
  hand-built with absolutely-positioned divs over a flex container.
- **"Closest upcoming" tracking** (powers the "Next: ... in N days" caption
  and the quadrant list) is done with two plain scalar variables
  (`next1_*` / `next2_*`) updated during the same line-parsing loop that
  builds the timeline dots — a manual "smallest and second-smallest"
  comparison, not a sorted array. This was a deliberate choice over
  building a JSON array and using `sort`/`parse_json`/`limit`: those
  filters are documented as available, but only the primitives already
  proven working on a real device (`assign`, `if`, `for`, `date`, `split`)
  were used here, to avoid another multi-round debugging cycle like the
  `custom_fields_values` and dot-centering ones above. **Not yet verified
  on-device** — if "Next: ..." or the quadrant list look wrong, that's the
  first place to check.

## Known limitations

- The title bar shows both the plugin title and the instance name
  (`trmnl.plugin_settings.instance_name`). If you haven't renamed your
  instance, both sides show "Anniversary Timeline" — rename the instance in
  the dashboard for a cleaner-looking bar.
- `{% for yoffset in (yoffset_start..1) %}` deliberately uses a variable for
  the negative lower bound rather than a literal `(-1..1)` — done
  defensively while debugging and left in place since it's harmless, even
  though it may not have been the actual cause of any bug encountered.

## Pre-submission checklist

Based on TRMNL's [plugin publishing best-practices post](https://trmnl.com/blog/plugin-recipe-publishing-tips)
and CHEF (their automated pre-review checker):

- [x] All four markup layouts provided (full, half_horizontal, half_vertical,
      quadrant) — required for public listing.
- [x] No `opacity` in CSS anywhere. First pass replaced it with the
      Framework's `text--gray-NN` classes on de-emphasized text, per CHEF's
      suggestion — but confirmed on-device, the gray dither pattern breaks
      small text (under ~16px) into illegible dots rather than a clean
      gray. Reverted those specific elements to plain solid black text,
      relying on size/weight alone for the visual hierarchy instead.
      `text--gray-NN` may still be worth using for *larger* text if you
      want a genuinely gray look somewhere later, just not below ~16-18px.
- [x] Static/no-code strategy — no external API calls, so the "renderer
      won't wait more than 5 seconds" async-timeout pitfall doesn't apply.
- [x] Custom title bar icon, inlined as a data URI rather than an external
      image link.
- [x] Shared markup used to keep the four layouts DRY.
- [x] User-entered labels are truncated (`| truncate: 20` / `24` / `30`)
      so a long label can't overflow into neighboring dots or wrap
      indefinitely.
- [x] Zero inline `style=` attributes anywhere. Structural flex/height rules
      use CSS classes (`aniv-view`, `aniv-layout`, `aniv-half--r*`, plus
      real Framework classes like `gap--small`). Dot/tick/today
      *positioning* (`left: N%`) initially seemed unavoidably inline since
      it's a per-render computed value — but CHEF flagged it anyway, so it
      now uses a generated `.pos-0`..`.pos-100` class per whole percentage
      point (`{% for i in (0..100) %}` in the stylesheet) instead of a
      `style=` attribute, with `pct`/`today_pct` rounded to match.
- [x] `author_bio` custom field added in the dashboard.
- [ ] **Category** — CHEF flags this as separate from `author_bio`; set it
      in the plugin settings view (not part of `custom_fields`) — drives
      marketplace browse/search visibility. See note in `settings.yml`.
- [ ] **Icon** — TRMNL's plugin settings view has a dedicated icon upload
      separate from the title-bar image in markup. Upload `icon.svg` there.
- [ ] **Featured image** — generate/upload one in the plugin settings view
      for the marketplace preview card.
- [ ] **Use demo data for anything public** — if you submit screenshots or
      demo credentials for review, swap in placeholder dates/names rather
      than real family information; the live examples used while building
      this (e.g. real birthdays) shouldn't end up in a public listing.
- [ ] **Test every layout on-device**, not just full — half_horizontal,
      half_vertical, and quadrant have only been reviewed in code so far,
      not seen live. Test on both TRMNL OG (1-bit) and TRMNL X if you have
      access to both, and TRMNL X portrait orientation, per their
      "test every view across OG/X, landscape and portrait" guidance.

## Publishing to the marketplace

Per TRMNL's process, this plugin already ships all four required markup
layouts (full, half_horizontal, half_vertical, quadrant), which is a
prerequisite for public listing. To submit for marketplace review:

1. Get it working well as a private plugin first (test with real data for a
   few days).
2. Email **team@trmnl.com** with subject "Public plugin submission -
   Anniversary Timeline", including:
   - Plugin ID (from the plugin settings URL)
   - Owner email
   - A short note on why this is useful beyond your own use (e.g. "no
     existing TRMNL plugin shows a visual timeline of upcoming birthdays/
     anniversaries")
   - A short screen recording of a fresh install/configuration
   - Demo account credentials for their review
   - Your plan for promoting it (if any)
3. TRMNL's team tests it, may request revisions, then soft-launches it.
