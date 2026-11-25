# Dataset Cleaner & Exporter (Remote MCP Server)

**Deduplicate, flatten and clean messy JSON rows in a single tool call.** Hand it a list of rows from a scraper, a CRM export or an API, and it hands back clean, spreadsheet-ready rows (or CSV text) plus an exact account of what was merged and why.

Built for AI agents. No install, no API key, no signup. Connect by URL and call it.

```
https://dataset-cleaner-exporter.nerolabs.workers.dev/mcp
```

Free to use while in early access.

## What it does

One call runs a fixed pipeline, in this order:

1. **Expand** (optional) one array field, such as `offers` or `variants`, into one row per entry, repeating the other fields.
2. **Flatten** nested objects into columns: `address.city` becomes `address_city`. Arrays stay as one JSON-text cell.
3. **Clean** every value: trim text, lowercase valid emails, reduce phone numbers to digits (keeping a leading `+`), lowercase URL hosts and drop the trailing slash, and optionally strip HTML and turn `"42"` and `"true"` into real numbers and booleans. Blank text becomes `null`.
4. **Keep, remove or rename** columns.
5. **Deduplicate** on the fields you name, or the whole row: exact, normalized (ignores case and whitespace, the default) or fuzzy (near-duplicates). The most complete row survives.

Deterministic, no AI in the loop, nothing guessed. A value it cannot confidently read, such as `"not-an-email"` in an email column, is left exactly as it was. The summary tells you how many rows came in, how many duplicates were removed, how many rows went out, how full every column is, and warns you when something needs a look, for example rows with an empty dedup key, which are kept as they are rather than merged with each other.

## Tools

| Tool | What it does |
|---|---|
| `list_capabilities` | Lists the exact cleaning rules, dedup modes, keep strategies, step order and limits. Processes no data. |
| `clean_rows` | Runs the pipeline on the rows you pass and returns the clean rows (or CSV text) plus a summary. |

## Connect

**Claude Code**

```bash
claude mcp add --transport http dataset-cleaner-exporter https://dataset-cleaner-exporter.nerolabs.workers.dev/mcp
```

**Claude Desktop / claude.ai:** Settings, Connectors, Add custom connector, paste the URL above.

**Cursor, Windsurf, VS Code and other MCP clients**

```json
{
  "mcpServers": {
    "dataset-cleaner-exporter": {
      "url": "https://dataset-cleaner-exporter.nerolabs.workers.dev/mcp"
    }
  }
}
```

## Example

Five messy lead rows go in, two of them duplicates with different casing and whitespace:

```json
{
  "rows": [
    {"company": "  Acme Corp  ", "email": "SALES@ACME.COM", "phone": "+1 (555) 123-4567", "website": "HTTPS://Acme.com/", "address": {"city": "Austin", "state": "TX"}},
    {"company": "Acme Corp", "email": "sales@acme.com ", "phone": "555.123.4567", "website": "https://acme.com", "address": {"city": "Austin", "state": ""}},
    {"company": "Globex Inc", "email": " Info@Globex.io", "phone": "+1 555 987 6543", "website": "https://globex.io/", "address": {"city": "Denver", "state": "CO"}, "notes": "<b>Hot</b> lead &amp; ready"},
    {"company": "globex inc", "email": "info@globex.io", "phone": "", "website": "", "address": {"city": "Denver", "state": "CO"}},
    {"company": "Initech", "email": "hello@initech.com", "phone": "(555) 010-0199", "website": "http://www.INITECH.com/about/", "address": {"city": "Dallas", "state": "TX"}}
  ],
  "dedupKeys": ["email"],
  "stripHtml": true
}
```

Three clean rows come out. Each duplicate pair kept its more complete row:

```json
{
  "rows": [
    {"company": "Acme Corp", "email": "sales@acme.com", "phone": "+15551234567", "website": "https://acme.com", "address_city": "Austin", "address_state": "TX"},
    {"company": "Globex Inc", "email": "info@globex.io", "phone": "+15559876543", "website": "https://globex.io", "address_city": "Denver", "address_state": "CO", "notes": "Hot lead & ready"},
    {"company": "Initech", "email": "hello@initech.com", "phone": "5550100199", "website": "http://www.initech.com/about", "address_city": "Dallas", "address_state": "TX"}
  ],
  "summary": {
    "inputRowCount": 5,
    "rowsAddedByExpansion": 0,
    "duplicatesRemoved": 2,
    "droppedByMaxItems": 0,
    "outputRowCount": 3,
    "dedupModeUsed": "normalized",
    "columns": ["company", "email", "phone", "website", "address_city", "address_state", "notes"],
    "fillRates": {"company": 1, "email": 1, "phone": 1, "website": 1, "address_city": 1, "address_state": 1, "notes": 0.333},
    "warnings": []
  }
}
```

Add `"outputFormat": "csv"` to get the same result back as CSV text, ready to save as a file.

## Skills

This server also publishes ready-made playbooks (Skills over MCP), so Claude and other agents know the steps for the whole job, not just each tool. Each is a `SKILL.md` at `skill://<name>/SKILL.md`, listed by `skills/list` and `resources/list`, read with `resources/read`.

- `clean-dataset-to-csv`: Make a scraped or exported JSON dataset spreadsheet-ready: flatten nested fields, clean emails, phones and URLs, remove duplicates (exact, normalized or fuzzy), pick and rename columns, and return CSV text in one clean_rows call.
- `dedupe-lead-list`: Remove duplicate leads or contacts from one or several merged lists, keeping the most complete record for each person or company, and say exactly how many were removed.

## Limits

- Up to **150 rows per call**, counted after any array expansion. For bigger lists, split them across several calls. Anything larger returns a clear message rather than failing silently.
- **Fuzzy** deduplication compares every pair of rows, so it runs on up to 100 rows with at most 1,000 characters of key text in total. Name a short field in `dedupKeys`, such as a company name. Beyond that it falls back to normalized matching and says so in the warnings.
- Rows are passed inline as JSON. Output is JSON rows or CSV text; there is no Excel file.

## Privacy

Your rows are processed in memory and never stored. To see which tools get used, each call records the tool name, row counts, whether it succeeded, the client name your app reports, the country and a one-way hashed caller ID. Your data, your arguments and your IP address are never kept in that log.

## Also available

The same engine runs on the Apify Store as [Dataset Cleaner & Exporter](https://apify.com/nerolabs/dataset-cleaner-exporter), which also reads Apify datasets, CSV, Excel and JSON files by URL and Google Sheets, handles up to 100,000 records per run, exports real downloadable CSV and Excel files, appends to a named dataset that accumulates across scheduled runs, and pushes the result to a webhook when a run finishes.

Built by **Nero Labs**.
