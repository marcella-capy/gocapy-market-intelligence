---
name: defense-monitor
description: >
  Weekly defense/aerospace/space contract intelligence monitor. Scrapes war.gov contract awards,
  USASpending API, SAM.gov solicitations, and defense industry news. Outputs a markdown report
  to Google Drive, a CSV data file, and a Gmail draft digest to mark@gocapy.com. Filters for
  manufacturing-relevant contracts useful for Go Capy's contract manufacturer clients.
  Triggers on "run defense monitor", "defense contracts", "weekly defense intel",
  "check defense contracts", "run defense intel", or any request to monitor defense/aerospace
  contract activity.
---

# Defense/Aerospace/Space Contract Monitor

Scrapes 14+ defense/aerospace sources weekly, filters for manufacturing-relevant signals, and
outputs a full intel package: MD report, CSV data, and Gmail draft digest.

---

## API Keys

Read from `~/.claude/global.env`:
- `FIRECRAWL_API_KEY` — web scraping
- `PERPLEXITY_API_KEY` — news summarization and contractor enrichment

No auth needed for USASpending API or SAM.gov public search API.

---

## Output Destinations

| Output | Destination |
|---|---|
| MD report | Google Drive folder ID `1mtnHuD58Xun-vf06BwT8m_bZYVypFyFB` |
| CSV data | Same Google Drive folder (auto-converts to Google Sheet) |
| Gmail draft | `mark@gocapy.com` |

---

## Manufacturing Relevance Filters

### NAICS Codes
- 332710 — Machine Shops
- 336412 — Aircraft Engine Parts
- 336413 — Other Aircraft Parts
- 334418 — Printed Circuit Assembly
- 332812 — Metal Coating/Engraving
- 336414–336419 — Guided Missile/Space Vehicle Parts
- 332919 — Other Metal Valve Manufacturing
- 332510 — Hardware Manufacturing
- 333511 — Industrial Mold Manufacturing
- 336411 — Aircraft Manufacturing

### Keywords (for filtering contract descriptions)
machining, fabrication, components, assemblies, parts, tooling, casting, forging, CNC,
additive, composite, subcontract, supplier, manufacturing, precision, machined, milling,
turning, sheet metal, welding, heat treat, plating, coating, stamping, extrusion,
injection mold, die cast, foundry, forge, titanium, aluminum, steel, alloy, fastener,
bearing, seal, gasket, valve, actuator, hydraulic, pneumatic, propulsion, airframe,
fuselage, landing gear, turbine, engine, missile, munition, ordnance, submarine,
vehicle, armor, howitzer, radar, sonar, satellite, spacecraft, rocket, launch

### Go Capy Client Verticals (flag when these appear)
- AGMI (Alpha Grainger) — precision CNC mill turn, screw machining, AS9100, ITAR
- PATRIOT — defense/aerospace manufacturing
- HARVEY VOGEL — metal fabrication
- VRC — manufacturing
- GENERAL FOUNDRY — foundry/casting
- MEGATECH — manufacturing
- FRANKLIN CASTING — casting
- SHELLCAST — die casting
- TMX — manufacturing

---

## Execution — follow these steps in order

### Step 1: Scrape war.gov Contract Index

Use PowerShell to call Firecrawl and get the contracts listing page:

```powershell
$apiKey = $env:FIRECRAWL_API_KEY  # loaded from ~/.claude/global.env at session start
$headers = @{ "Content-Type" = "application/json"; "Authorization" = "Bearer $apiKey" }
$body = @{
    url = "https://www.defense.gov/News/Contracts/"
    formats = @("markdown")
} | ConvertTo-Json -Depth 5
$response = Invoke-RestMethod -Uri "https://api.firecrawl.dev/v1/scrape" -Method Post -Headers $headers -Body $body
$indexMarkdown = $response.data.markdown
```

Parse the markdown to extract article URLs for the past 7 days. The links look like:
```
[Contracts for May 14, 2026](https://www.war.gov/News/Contracts/Contract/Article/XXXXXXX/contracts-for-may-14-2026/)
```

Extract ALL article URLs from the past 7 calendar days (typically 5 business days).

### Step 2: Scrape Each Daily Contract Article

For each article URL from Step 1, scrape the full contract text:

```powershell
$body = @{
    url = "<article-url>"
    formats = @("markdown")
} | ConvertTo-Json -Depth 5
$response = Invoke-RestMethod -Uri "https://api.firecrawl.dev/v1/scrape" -Method Post -Headers $headers -Body $body
```

Parse each article's markdown to extract individual contracts. Each contract entry contains:
- **Agency** (ARMY, NAVY, AIR FORCE, DEFENSE LOGISTICS AGENCY, MISSILE DEFENSE AGENCY, etc.)
- **Contractor name** and location
- **Dollar amount**
- **Description** of work
- **Contract number** (in parentheses at end)
- **Completion date**
- **Contracting activity**

The text is organized by agency with bold headers like `**ARMY**`, `**NAVY**`, etc.
Each paragraph is one contract. Parse each paragraph to extract the structured fields.

Store all contracts in a PowerShell array of objects:
```powershell
$contracts = @()
$contracts += [PSCustomObject]@{
    Date = "2026-05-14"
    Agency = "NAVY"
    Contractor = "Marine Polymer Inc."
    Location = "Stevensville, Maryland"
    Amount = 11475000
    AmountFormatted = "$11,475,000"
    Description = "spare tiles supporting auxiliary system for submarine hull treatment"
    ContractNumber = "N00104-26-C-YA28"
    ManufacturingRelevant = $true
    RelevanceReason = "submarine components, precision manufacturing"
}
```

### Step 3: Query USASpending API

Query for recent awards filtered by manufacturing NAICS codes:

```powershell
$usaBody = @{
    filters = @{
        time_period = @( @{
            start_date = "<7-days-ago-YYYY-MM-DD>"
            end_date = "<today-YYYY-MM-DD>"
        })
        award_type_codes = @("A","B","C","D")
        naics_codes = @("332710","336412","336413","334418","332812","336414","336415","336419")
    }
    fields = @("Award ID","Recipient Name","Award Amount","Description","Awarding Agency","Start Date")
    limit = 50
    page = 1
    sort = "Award Amount"
    order = "desc"
} | ConvertTo-Json -Depth 5
$usaResponse = Invoke-RestMethod -Uri "https://api.usaspending.gov/api/v2/search/spending_by_award/" -Method Post -Headers @{ "Content-Type" = "application/json" } -Body $usaBody
```

Add results to the contracts array with source = "USASpending".

### Step 4: Query SAM.gov for Active Solicitations

Query the SAM.gov public search API for manufacturing-related solicitations:

```powershell
$samKeywords = @("machining", "CNC", "fabrication", "casting", "forging", "precision parts", "aircraft components", "missile parts")
foreach ($keyword in $samKeywords) {
    $samUri = "https://sam.gov/api/prod/sgs/v1/search/?index=opp&q=$keyword&page=0&size=10&mode=search&sort=-modifiedDate"
    $samResponse = Invoke-RestMethod -Uri $samUri -Method Get -Headers @{ "Accept" = "application/hal+json" }
    # Process results...
}
```

Extract from each result:
- title
- type (Solicitation, Combined Synopsis/Solicitation, etc.)
- solicitationNumber
- responseDate (deadline)
- organizationHierarchy (agency chain)
- modifiedDate

Store active, non-canceled solicitations with approaching deadlines.

### Step 5: Scrape Industry News Sites

Use Firecrawl to scrape these sites for headlines (markdown mode):

1. **GovConWire** — `https://www.govconwire.com/category/contract-awards/`
2. **SpaceNews** — `https://spacenews.com/section/news/`
3. **Defense News** — `https://www.defensenews.com/industry/`
4. **Air & Space Forces Magazine** — `https://www.airandspaceforces.com/category/news/`

For each site:
```powershell
$body = @{ url = "<site-url>"; formats = @("markdown") } | ConvertTo-Json -Depth 5
$response = Invoke-RestMethod -Uri "https://api.firecrawl.dev/v1/scrape" -Method Post -Headers $headers -Body $body
```

Extract headlines and brief descriptions. Filter for manufacturing/supply chain relevance using
the keywords list above. Keep only the top 10-15 most relevant headlines across all sources.

### Step 6: Filter for Manufacturing Relevance

Score each contract/signal on manufacturing relevance (0-100):
- +30 if description contains manufacturing keywords
- +20 if NAICS code matches our list
- +20 if contractor is a known manufacturer (not a service/IT company)
- +15 if description mentions subcontracting or supply chain
- +15 if relevant to a Go Capy client vertical (defense, aerospace, casting, machining, etc.)

Keep contracts scoring >= 30 as "relevant". Flag contracts scoring >= 60 as "high priority".

### Step 7: Generate Markdown Report

Create a comprehensive markdown report with this structure:

```markdown
# Defense/Aerospace Weekly Intel — Week of [DATE]

> Generated [timestamp] | Sources: war.gov, USASpending, SAM.gov, GovConWire, SpaceNews, Defense News

## Executive Summary
[2-3 sentence overview of the week's most notable contracts and trends]

## High-Priority Contract Awards
[Top 5-10 manufacturing-relevant contracts from war.gov + USASpending]

| Contractor | Amount | Agency | Description | Relevance |
|---|---|---|---|---|
| ... | ... | ... | ... | ... |

### Contract Details
[For each high-priority contract, 2-3 sentences on what it means for contract manufacturers]

## Active Solicitations (SAM.gov)
[Manufacturing-relevant RFPs/RFIs with approaching deadlines]

| Title | Solicitation # | Agency | Deadline | Type |
|---|---|---|---|---|
| ... | ... | ... | ... | ... |

## Industry Signals
[Relevant headlines from defense/aerospace news sites]

- **[Headline]** (Source, Date) — [1-line summary of manufacturing relevance]

## Outreach Candidates
[Contractors from this week's awards that may need manufacturing partners]

| Company | Contract | Why They May Need Subs |
|---|---|---|
| ... | ... | ... |

## Week-Over-Week Trends
[Any notable patterns: spending up/down, new programs ramping, agencies active]

---
*Sources: war.gov/News/Contracts, USASpending.gov, SAM.gov, GovConWire, SpaceNews, DefenseNews, Air & Space Forces Magazine*
```

Save the report locally first:
```
${CLAUDE_PLUGIN_ROOT}/skills\defense-monitor\output\defense-intel-YYYY-MM-DD.md
```

### Step 8: Generate CSV Data

Create a CSV with ALL contracts (not just filtered) for the Google Sheet:

```csv
Date,Source,Agency,Contractor,Location,Amount,Description,Contract Number,Manufacturing Relevant,Relevance Score,Relevance Reason
2026-05-14,war.gov,NAVY,Marine Polymer Inc.,"Stevensville, Maryland",11475000,"spare tiles for submarine hull treatment",N00104-26-C-YA28,Yes,75,"submarine components, precision manufacturing"
```

Save locally:
```
${CLAUDE_PLUGIN_ROOT}/skills\defense-monitor\output\defense-data-YYYY-MM-DD.csv
```

### Step 9: Upload to Google Drive

Upload both files to Google Drive folder `1mtnHuD58Xun-vf06BwT8m_bZYVypFyFB`:

**MD Report:**
Use `mcp__9af8d6d1-f19d-493f-b9ce-5200fd00d2e4__create_file` with:
- `title`: `Defense Intel — Week of YYYY-MM-DD`
- `textContent`: the full markdown report
- `contentMimeType`: `text/plain`
- `parentId`: `1mtnHuD58Xun-vf06BwT8m_bZYVypFyFB`

**CSV Data:**
Use `mcp__9af8d6d1-f19d-493f-b9ce-5200fd00d2e4__create_file` with:
- `title`: `Defense Contracts Data — YYYY-MM-DD`
- `textContent`: the CSV content
- `contentMimeType`: `text/csv`
- `parentId`: `1mtnHuD58Xun-vf06BwT8m_bZYVypFyFB`

(text/csv auto-converts to Google Sheet in Drive)

### Step 10: Create Gmail Draft

Use `mcp__f4680a1f-d05d-42b9-9edc-6fe9d46d74b9__create_draft` with:
- `to`: `["mark@gocapy.com"]`
- `subject`: `Defense/Aerospace Weekly Intel — Week of [DATE]`
- `body`: concise plain-text digest (NOT the full report)

**Email body format:**
```
Defense/Aerospace Weekly Intel — Week of [DATE]

TOP CONTRACTS THIS WEEK
========================
1. [Contractor] — $[Amount] — [Brief description] ([Agency])
2. ...
(top 10 most relevant)

ACTIVE SOLICITATIONS (deadlines approaching)
=============================================
1. [Title] — Due [Date] — [Agency]
2. ...
(top 5)

INDUSTRY SIGNALS
================
- [Headline] — [1-line summary]
- ...
(top 5)

OUTREACH CANDIDATES
===================
- [Company] — [Why they may need manufacturing partners]
- ...

Full report: [link to Google Drive MD file]
Data sheet: [link to Google Drive CSV/Sheet]

--
Generated by Go Capy Defense Intel Monitor
```

### Step 11: Return Results to User

Show the user:
1. Summary of contracts found and filtered
2. Link to the Google Drive MD report
3. Link to the Google Drive data sheet
4. Confirmation that Gmail draft was created
5. List of outreach candidates flagged for the pipeline

---

## Scheduling

This skill is designed to run weekly via `/schedule`. Set up with:
- **Schedule**: Every Monday at 7:00 AM ET
- **Prompt**: `run defense monitor`

---

## Notes

- The war.gov contracts page lists ~10 articles per page. For a weekly run, the first page (most recent 10 days) is sufficient.
- Firecrawl credits used per run: ~8-12 (1 for index + 5 for daily articles + 2-4 for news sites)
- USASpending and SAM.gov APIs are free with no rate limits for this volume.
- For the Google Sheet integration: each week's data is uploaded as a new CSV (auto-converts to Sheet). The master sheet at `1poP1PenXAaaGk_NgP1dha5uqhxDUy64luga7HvimdAg` can reference these weekly sheets.
