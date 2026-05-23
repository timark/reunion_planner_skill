# Reunion Planner AI Agent Skill

An AI agent skill for planning a group reunion weekend from Google My Maps
venue exports. The skill researches hotels and pubs, enriches the original map
data with ratings and venue details, and produces CSVs plus a single-file HTML
report that the group can browse and shortlist from.

This repository currently contains the skill definition and sample/input data
for a 50th anniversary Manchester Polytechnic Architecture reunion.

## Repository Contents

```text
.
├── reunion-planner-SKILL.md   # AI agent skill definition and workflow
└── data/
    ├── 50- Hotels.csv         # Google My Maps hotel export, 45 rows
    └── 50- Pubs.csv           # Google My Maps pub export, 49 rows
```

## What The Skill Does

The skill guides an AI agent through a full venue research workflow:

- Parse hotel and pub CSVs exported from Google My Maps.
- Validate venue geography against the reunion's travel constraints.
- Research hotels for ratings, prices, room types, facilities, group booking
  suitability, and overall character.
- Research pubs for food, beer, pub character, games, awards, and amenities.
- Add a small number of extra hotel and pub suggestions where useful.
- Write enriched CSVs that can be imported back into Google My Maps.
- Generate a standalone interactive HTML report with sortable tables,
  expandable rows, shortlist toggles, and localStorage persistence.

## Expected Inputs

The skill expects two CSV files in Google My Maps format:

```csv
WKT,name,description
"POINT (-2.33916 53.2889441)",Dun Cow,
```

The `WKT` field must be a point in longitude/latitude order:

```text
POINT (lon lat)
```

The user should also provide an event brief covering:

- Group profile
- Dates
- Preferred area or travel constraints
- Hotel requirements
- Pub criteria
- Ambience and style preferences

## Current Event Context

The included skill file is tailored for a 50th anniversary reunion:

- Group: approximately 5-10 men, aged 68-69
- Background: started Manchester Polytechnic Architecture in 1976
- Dates: weekend of 19-20 September 2026
- Geography: roughly within 1 hour's drive of Manchester Airport, Chester, and Preston
- Hotel needs: single rooms preferred, restaurant and bar essential, block booking useful
- Pub interests: traditional character, continental or craft beer, dartboard, pool table,
  and a nostalgic nod to Nobby's Nuts

## Outputs

When run, the skill should produce:

```text
hotels_enriched.csv
pubs_enriched.csv
reunion_report.html
```

The enriched CSVs are designed to be re-imported into Google My Maps while also
including structured columns for ratings, facilities, prices, and source notes.

## CSV Writing Requirement

The skill requires enriched CSVs to be written with Python's `csv` module using
`quoting=csv.QUOTE_ALL`.

This is important because the description fields can contain commas, newlines,
star symbols, currency symbols, and URLs. Quoting all fields keeps the files
valid in Excel, pandas, and Google My Maps.

Example:

```python
import csv

with open("hotels_enriched.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=hotel_fields, quoting=csv.QUOTE_ALL)
    writer.writeheader()
    writer.writerows(hotel_rows)
```

## Using The Skill

Use `reunion-planner-SKILL.md` as the skill definition in an AI-agent-compatible
skills directory, or open it directly as the working instructions for an AI
agent.

At runtime, provide:

1. The hotel CSV.
2. The pub CSV.
3. The event brief.

For this repository's bundled data, the inputs are:

```text
data/50- Hotels.csv
data/50- Pubs.csv
```

## Research Caveats

- Venue details, prices, room availability, awards, and ratings change over time.
- September 2026 room prices may not be published yet, so current prices may need
  to be used as a proxy.
- All booking-critical details should be verified directly with each venue before
  committing the group.
- Some venues may appear in both the hotel and pub datasets.
- Duplicate rows should be handled during enrichment where identified.

## Development Notes

This repository is intentionally lightweight. There is no package manifest,
test runner, or application code yet; the main artifact is the skill markdown
file plus CSV data.

If this grows into a runnable tool, likely next steps would be:

- Add a Python script for parsing, validating, and writing enriched CSVs.
- Add a report generator for `reunion_report.html`.
- Add validation tests for CSV schemas and WKT parsing.
- Add sample generated outputs for comparison.
