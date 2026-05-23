---
name: reunion-planner
description: >
  Plans a group reunion weekend by researching hotels and pubs within a defined
  geographic area, enriching a Google My Maps CSV export with ratings, costs,
  facilities, and narrative reviews, then producing enriched CSVs and an
  interactive HTML report the group can browse. Use this skill whenever a user
  wants to: plan a group weekend away, research venues for a reunion or event,
  enrich a My Maps CSV with hotel/pub details, compare hotels by facilities and
  room cost, or rate pubs for character and amenities. Trigger on phrases like
  "plan our reunion", "research these hotels and pubs", "enrich my My Maps CSV",
  "find venues for our weekend", "rate these pubs", or any time a CSV with WKT
  coordinates and venue names is provided alongside a group event brief.
---

# Reunion Weekend Planner

A skill for planning a group reunion weekend. Takes a Google My Maps CSV export
(hotels + pubs with WKT geometry), researches each venue via web search, adds
additional venue suggestions, and produces enriched CSVs plus an interactive
HTML report.

---

## Step 0 — Intake: Gather Parameters Before Doing Anything Else

Before parsing files or running any searches, conduct a structured intake
conversation. Extract answers from the conversation history first — do not
ask for information the user has already provided. Then ask only for what
is missing.

Work through these parameter groups in order, batching questions where
possible to avoid a lengthy back-and-forth:

---

### 0A — Existing Knowledge Check

Scan the conversation for any information already provided about:
- Group name / occasion
- Event dates
- Group size (number of people / rooms required)
- Geographic constraints (named towns, travel time limits, reference locations)
- Ambience preferences
- Hotel preferences (room type, facilities must-haves)
- Pub criteria
- Any venue names or opinions already mentioned by the user

Present a summary of what you have already gathered and confirm it before
proceeding. For example:

> "Before I start researching, here's what I've already noted from our
> conversation. Please correct anything that's wrong:
>
> - **Event**: 50th anniversary reunion, weekend of 19–20 September 2026
> - **Group**: ~20 men, aged 68–69, architects who started at Manchester
>   Polytechnic in 1976
> - **Geography**: within ~1 hour's drive of Manchester Airport, Chester,
>   and Preston
> - **Ambience**: [not yet confirmed — see below]
> - **Hotels**: single rooms preferred; block-booking needed; restaurant
>   and bar essential
> - **Pubs**: continental/craft beer important; dartboard and pool table
>   desirable; traditional character preferred; Nobby's Nuts aspirational
>
> I still need a few things from you before I start..."

---

### 0B — Missing Parameters to Prompt

For each item below, check whether it was already provided. Only ask for
items that are genuinely missing.

**Geography — distance constraints**

If the user has named reference locations but not specified a travel time
or distance limit, ask:

> "You've mentioned [location A], [location B], and [location C] as
> reference points. What's the maximum travel time you'd want from each —
> is 1 hour driving a fair limit, or stricter/looser? And should all three
> be satisfied simultaneously, or is it more 'within reach of at least one'?"

If no reference locations have been given at all, ask:

> "What's the geographic constraint for the venue? For example: 'within
> 1 hour of Manchester Airport', or 'somewhere in the Cheshire/Lancashire
> border area'. Named towns, postcodes, or a rough radius all work."

**Group size and room count**

> "Roughly how many people are coming? I need an approximate room count
> to assess whether hotels can accommodate the group — even a ballpark
> (e.g. '15–20') is enough."

**Ambience**

> "What feel are you going for with the hotel? For example:
> - Country house / manor house (formal, character, grounds)
> - Rural inn (informal, village pub attached)  
> - Modern spa hotel (leisure facilities priority)
> - Any of the above as long as it has [X]
>
> And is there anything you definitely want to avoid — e.g. chain hotels,
> conference centres, places that feel corporate?"

**Dates** (if not provided)

> "What are the arrival and departure dates? I'll use these to check room
> pricing if September rates are published."

**Pub criteria** (if not provided)

> "Any specific pub criteria beyond the usual? The group has mentioned
> continental beer, dartboard, pool table, and Nobby's Nuts — should I
> take those at face value or are some more serious than others?"

**Report audience**

> "Is the HTML report for your eyes as organiser, or something you'd share
> with the whole group to browse and vote? This affects whether I add
> shortlisting / voting features."

**CSV files**

If hotel and/or pub CSVs have not been uploaded yet:

> "Please upload your Google My Maps CSV exports — one for hotels and one
> for pubs. The format should be WKT,name,description with POINT geometry."

---

### 0C — Build the Session Brief

Once all parameters are confirmed, assemble and display a **Session Brief**
that will govern all subsequent steps. Store this in working memory for the
rest of the session. Example structure:

```
SESSION BRIEF
─────────────────────────────────────────────────
Event:        [name / occasion]
Dates:        [arrival] to [departure]
Group size:   [N] people / approx [N] rooms needed
─────────────────────────────────────────────────
GEOGRAPHY
Reference points:
  - [Location A] — max [X] mins drive
  - [Location B] — max [X] mins drive
  - [Location C] — max [X] mins drive
Rule: [all three / at least one / centroid approach]
─────────────────────────────────────────────────
HOTELS
Ambience:     [country house / rural inn / spa / no preference]
Room type:    [single / double single occupancy / twin]
Must-haves:   [restaurant, bar, parking, ...]
Nice to have: [pool, spa, games room, EV charging, ...]
Avoid:        [chains, conference centres, ...]
─────────────────────────────────────────────────
PUBS
Priority criteria:  [continental beer, real ale, food, ...]
Desirable:          [dartboard, pool table, outdoor space, ...]
Tongue-in-cheek:    [Nobby's Nuts]
Character:          [traditional inn preferred / any]
─────────────────────────────────────────────────
OUTPUT
Report audience:    [planner only / shareable with group]
Shortlist/voting:   [yes / no]
─────────────────────────────────────────────────
CSV FILES
Hotels CSV:   [filename] — [N] venues
Pubs CSV:     [filename] — [N] venues
─────────────────────────────────────────────────
```

Ask the user to confirm the brief before proceeding to Step 1.

---

## Step 1 — Parse the CSVs

Read both files using pandas with `on_bad_lines='skip'` to handle any
malformed rows (hotel names containing commas can break naive parsing):

```python
import pandas as pd, re

def parse_wkt_csv(path):
    df = pd.read_csv(path, on_bad_lines='skip')
    def parse_wkt(wkt):
        m = re.match(r'POINT \(([^ ]+) ([^ )]+)\)', str(wkt))
        if m:
            return float(m.group(1)), float(m.group(2))
        return None, None
    df[['lon','lat']] = df['WKT'].apply(lambda x: pd.Series(parse_wkt(x)))
    return df

hotels = parse_wkt_csv('Hotels.csv')
pubs   = parse_wkt_csv('Pubs.csv')
```

Deduplicate by name (Statham Lodge appears twice in the reference dataset).
Print a summary: number of hotels, number of pubs, geographic bounding box.

---

## Step 2 — Validate Geography

Using the reference points and travel time limits from the Session Brief,
estimate straight-line distance from each venue to each reference point.
Use the haversine formula:

```python
from math import radians, sin, cos, sqrt, atan2

def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1))*cos(radians(lat2))*sin(dlon/2)**2
    return R * 2 * atan2(sqrt(a), sqrt(1-a))
```

As a rule of thumb, 1 hour driving in rural England ≈ 50–60km straight line.
Flag any venue where the nearest reference point exceeds the limit.
Do not remove flagged venues — note them with `[OUTSIDE RANGE - verify]`
in the source_note column. The user may still want to consider them.

Add a `distance_notes` column to each row summarising the km to each
reference point, e.g.: `MAN: 18km | Chester: 34km | Preston: 61km`.

---

## Step 3 — Research Hotels

For each hotel, run a web search:
`"{hotel name}" hotel {nearest_town} review facilities rooms`

Extract and record per the Session Brief's must-haves and nice-to-haves.
Always capture:
- Star rating (AA or equivalent)
- Guest rating + review count (TripAdvisor / Google / Booking.com)
- Room types — flag singles explicitly; note twin = 2 single beds as acceptable
- Room price range for the event dates if published; otherwise current rates
- Facilities matching the Session Brief criteria
- Group booking capability (block-book [N] rooms?)
- Overall character matching the Session Brief ambience preference
- URL (hotel's own website preferred over aggregator)

Score priority 1–5 based on Session Brief fit:
- +2 if ambience matches preference exactly
- +1 if restaurant and bar both present
- +1 if single rooms or twin confirmed available
- +1 if guest rating ≥ 4.0 / 8.0
- -1 if chain/budget/corporate when rural character requested
- -1 if group capacity likely insufficient for the room count required

### Additional hotel suggestions

After the CSV list, search for up to 5 hotels not in the CSV using
location terms derived from the Session Brief's geographic area.
Mark these `[NEW]` in source_note.

---

## Step 4 — Research Pubs

For each pub, search CAMRA's WhatPub first (most reliable for UK pub data),
then TripAdvisor or Google for food and ambience reviews:
`"{pub name}" {town} CAMRA whatpub`
`"{pub name}" {town} pub review`

Score against Session Brief pub criteria. Always note:
- Pub type and character
- Food availability
- Real ale (cask)
- Continental / craft beer (use Session Brief to weight importance)
- Dartboard (check reviews — often mentioned; call direct if unclear)
- Pool table
- Awards (CAMRA, Good Beer Guide, Good Pub Guide, local Pub of the Year)
- Ambience score 1–5

Apply the Nobby's Nuts heuristic: award a ★ if any reviewer or listing
mentions pork scratchings, bar snacks on the bar, or "proper pub snacks".
Handle this with appropriate affection — it is a nostalgic criterion, not
a hard requirement.

### Additional pub suggestions

Search for up to 5 pubs not in the CSV matching the Session Brief area and
pub criteria. Prioritise CAMRA Good Beer Guide entries.
Mark these `[NEW]` in source_note.

---

## Step 5 — Produce Enriched CSVs

**MANDATORY: Use Python's `csv` module with `quoting=csv.QUOTE_ALL`.**
Never write CSV rows by f-string or string concatenation. The description
field contains commas, newlines, and special characters — without QUOTE_ALL
the file breaks when opened in Excel, Google My Maps, or pandas.

```python
import csv

with open('hotels_enriched.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=hotel_fields, quoting=csv.QUOTE_ALL)
    writer.writeheader()
    writer.writerows(hotel_rows)
```

**Always verify before delivering:**
```python
import pandas as pd
df = pd.read_csv('hotels_enriched.csv')
assert df.shape[1] == len(hotel_fields), f"Column mismatch: got {df.shape[1]}"
```

### Hotels CSV schema

```
WKT, name, description, star_rating, guest_rating, guest_rating_count,
price_range_pp, single_rooms, restaurant, bar, pool_spa, games_room,
parking, group_booking, character, priority_score, distance_notes, url, source_note
```

`description` field template (rendered as tooltip in My Maps):
```
[★ rating] [Character type] | Guest: [X]/5 ([N] reviews)
Rooms: approx £[X]–£[Y]/night | Singles: [Y/N/confirm]
Facilities: [comma list from Session Brief must-haves + nice-to-haves]
Group: [block-book assessment]
[2–3 sentence narrative in plain English, tailored to group profile]
[distance_notes line]
URL: [hotel website]
```

### Pubs CSV schema

```
WKT, name, description, pub_type, food, real_ale, continental_beer,
dartboard, pool_table, awards, ambience_score, nobby_score, distance_notes,
url, source_note
```

`description` field template:
```
[Pub type] | Ambience: [X]/5
Food: [summary] | Real ale: [Y/N] | Continental: [Y/N]
Games: Dartboard [Y/N/unconfirmed] | Pool [Y/N/unconfirmed]
Awards: [list or 'none found']
[Nobby's Nuts ★ if applicable]
[2–3 sentence narrative in plain English]
[distance_notes line]
URL: [pub website or CAMRA listing]
```

---

## Step 6 — Produce Interactive HTML Report

Generate a single self-contained HTML file (no external dependencies,
all CSS and JS inline). Structure:

- **Header**: event name, dates, group profile summary, geographic constraint
- **Tabs**: Hotels | Pubs | Map
- **Hotels tab**: sortable table (priority score / price / rating / name)
  - Row expands to full description + facilities icon row
  - Colour band: green (priority 4–5) / amber (3) / grey (1–2)
  - `[OUTSIDE RANGE]` venues shown with a warning banner
- **Pubs tab**: sortable table (ambience score / awards / name)
  - Row expands to full description + criteria checklist
    (✓/✗ for each Session Brief pub criterion)
  - Nobby's Nuts ★ shown as a gold star if awarded
- **Map tab**: show hotels and pubs by WKT coordinates
  - Preferred no-key implementation: a Leaflet map with OpenStreetMap tiles,
    styled markers for hotels and pubs, and a searchable/filterable venue list.
  - Display all hotel and pub pins by default. Use distinct marker colours or
    labels for hotel vs pub, and highlight top-rated venues with a subtle halo.
  - Clicking a venue in the side list must zoom/pan the map to that marker and
    open its popup.
  - Clicking a marker must open a neat popup card showing the most useful data
    values: venue type, score, rating/real ale, price/food, distance notes,
    short character summary, and an "Open in Google Maps" link.
  - Parse WKT as `POINT (lon lat)` and convert to Google Maps latitude/longitude
    query order for external links.
  - If the user provides a Google My Maps share URL, embed that map instead so
    all markers can be shown simultaneously.
  - Do not require a Google Maps API key unless the user specifically wants
    Google Maps JavaScript API styling and marker behaviour.
- **Shortlist toggle** on each row if Session Brief says report is shareable
  with the group (state kept in memory during the session; note that
  localStorage is not available in claude.ai artifacts)
- **Footer**: "Researched [date] | Verify all details direct with venue
  before booking | Prices are indicative"

### Visual style requirements

Use a modern, lightweight report style:

- Use a system sans-serif font stack such as `Inter, ui-sans-serif, system-ui,
  -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Arial, sans-serif`.
- Avoid serif body fonts, monospace narrative descriptions, heavy solid colour
  table headers, and full-row pastel fills.
- Prefer a warm neutral page background, white table surfaces, subtle borders,
  restrained shadows, and small status accents for priority/ambience.
- Keep tables dense enough for comparison, but use comfortable padding,
  readable line height, and sticky table headers.
- Keep badges, buttons, controls, and tabs at 8px border radius or less.
- Make expanded descriptions use the same readable sans-serif font as the rest
  of the report.

### HTML safety requirements

The report contains venue names, descriptions, titles, URLs, awards, and notes
that may include apostrophes, quotes, ampersands, currency symbols, and other
special characters. These strings must be serialized and rendered safely.

Mandatory rules:

- Serialize the hotel and pub data into JavaScript with `json.dumps(...)`; do
  not build JavaScript arrays or object literals by hand with f-strings.
- Do not insert raw venue strings into inline event handlers such as
  `onclick="toggleShortlist('p', 'The King's Arms')"`. Use `addEventListener`
  and pass values from the already-parsed data object instead.
- Do not put display metadata such as `title: "Nobby's Nuts awarded"` in CSS.
  `title` is an HTML attribute, not a stylesheet property. Put tooltips on the
  relevant HTML element with `element.title = ...` or `setAttribute('title', ...)`.
- Prefer `textContent` when inserting names, awards, descriptions, prices, or
  notes into the DOM. Use `innerHTML` only for small fixed markup that does not
  contain venue-provided text.
- If the report intentionally renders trusted rich text, escape all text first
  with an HTML escaping helper before adding links or line breaks.
- When selecting expandable rows, do not build CSS selectors directly from raw
  venue names. Use generated row IDs, array indexes, or `CSS.escape(name)`.
- Verify generated HTML by opening it in a browser console or running a quick
  script check that includes venue names with apostrophes, for example
  `Nobby's Nuts`, `The King's Arms`, and `Parr's Bank`.

Recommended Python pattern:

```python
import json

hotels_json = json.dumps(hotel_rows, ensure_ascii=False)
pubs_json = json.dumps(pub_rows, ensure_ascii=False)
```

Recommended JavaScript pattern:

```javascript
const HOTELS = /* json from Python */;

button.addEventListener("click", () => toggleShortlist("h", hotel.name));
nameCell.textContent = hotel.name;
badge.title = "Nobby's Nuts: bar snacks confirmed";
```

---

## Step 7 — Deliver Outputs

Present:
1. `hotels_enriched.csv` — reimport to Google My Maps
2. `pubs_enriched.csv` — reimport to Google My Maps
3. `reunion_report.html` — share with the group or use as planner reference

Verbal summary:
- Top 3 hotel recommendations with one-sentence rationale each
- Top 3 pub recommendations with one-sentence rationale each
- Any venues outside the geographic constraint
- Any venues where web research returned little or no data (flag for
  direct contact)
- Any venues appearing in both CSVs (e.g. hotel with attached pub)

---

## Writing Style for Narratives

Tailor all venue descriptions to the group profile from the Session Brief.
For the reference event (architects, 68–69, Manchester Polytechnic 1976):
- Write for people who have spent careers judging buildings and spaces
- Prefer plain, specific, honest language over marketing copy
- Note character details that an architect would notice (building type,
  materials, period, spatial quality)
- Be dry when appropriate — this group will appreciate it
- Avoid: "perfect for a relaxing getaway", "nestled in", "boasts",
  "stunning", "idyllic"

---

## Notes and Caveats

- September room rates may not yet be published; use current rates, flag
  that pricing must be confirmed direct with the venue
- CAMRA WhatPub and Good Beer Guide are the most reliable sources for
  pub quality; TripAdvisor pub reviews are unreliable for ale quality
- Some venues may appear in both CSVs — note this explicitly
- The WKT format is `POINT (longitude latitude)` — longitude first
- Deduplicate any venue appearing more than once in the same CSV by name
