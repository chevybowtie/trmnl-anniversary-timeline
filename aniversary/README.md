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
3. Under **Custom Fields**, add the three fields described in `settings.yml`:
   - `special_dates` — long text area
   - `months_back` — number, default `1`
   - `months_forward` — number, default `4`
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
user-configurable per instance via `months_back` / `months_forward`.

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

## Known limitations

- The title bar shows both the plugin title and the instance name
  (`trmnl.plugin_settings.instance_name`). If you haven't renamed your
  instance, both sides show "Anniversary Timeline" — rename the instance in
  the dashboard for a cleaner-looking bar.
- `{% for yoffset in (yoffset_start..1) %}` deliberately uses a variable for
  the negative lower bound rather than a literal `(-1..1)` — done
  defensively while debugging and left in place since it's harmless, even
  though it may not have been the actual cause of any bug encountered.

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
