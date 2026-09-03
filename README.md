# data-tg-mcp

MCP server for any Huwise/Opendatasoft data portal.

## Installation

```bash
uv sync
```

## Usage

```bash
uv run main.py
```

## Debug
```bash
npx @modelcontextprotocol/inspector uv run main.py
```

### Install with uvx
```bash
uvx --from git+https://github.com/ogdtg/mcp-data-tg data-tg-mcp
```

## Selecting a catalog

The catalog is chosen by whoever deploys the server via the `.env` file next to
`main.py`. All Huwise/Opendatasoft portals share the same API
path, so you only set the domain:

```
# .env
DATA_PORTAL_DOMAIN=data.tg.ch
```

The full API base URL is built as
`https://<domain>/api/explore/v2.1`.

The `.env` file is committed, so a fork carries its
catalog choice through `uvx` installs as well. This fork is configured for
`data.tg.ch`, the Kanton Thurgau open data portal.

## Hosting on Posit Connect

Besides the stdio entry point used above, `main.py` also exposes a
Streamable HTTP ASGI app as `app`, so this repo can be published to Posit
Connect via **Git-backed publishing** and reached as a remote MCP server.

1. In Posit Connect, create new content and choose to publish from a Git
   repository, pointing at this repo/branch.
2. Connect auto-detects the included `manifest.json`
   (`appmode: python-fastapi`, entrypoint `main:app`) and
   `requirements.txt`, builds the environment, and starts the app itself —
   no extra configuration is needed.
3. Before publishing, edit `.env` so `DATA_PORTAL_DOMAIN` points at the
   catalog you want this deployment to serve (already set to `data.tg.ch`
   for this fork).
4. Once deployed, the MCP endpoint is available at the content's URL with
   the `/mcp` path appended, e.g. `https://<connect-server>/content/<guid>/mcp`.
   Configure that URL as a remote/HTTP MCP server in your client.

If you change dependencies, regenerate `requirements.txt` and
`manifest.json` before pushing:

```bash
uv sync
uv pip freeze | grep -v '^-e ' > requirements.txt
uvx rsconnect write-manifest fastapi --entrypoint main:app --overwrite .
```

## Configuration

### OpenCode

Add to your OpenCode config:

```json
{
  "mcpServers": {
    "data-tg": {
      "command": "uv",
      "args": [
        "--directory",
        "/ABSOLUTE/PATH/TO/data-tg-mcp",
        "run",
        "main.py"
      ]
    }
  }
}
```

### Cursor

Add to your Cursor config (`~/.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "data-tg": {
      "command": "uv",
      "args": [
        "--directory",
        "/ABSOLUTE/PATH/TO/data-tg-mcp",
        "run",
        "main.py"
      ]
    }
  }
}
```

## Tools

### `get_datasets`
Search and list available datasets.

Two search modes:
- `semantic` (default): ranks the catalog by meaning using the `vector_similarity` explore endpoint from Huwise. Best for natural-language / conceptual queries. Matches synonyms and other languages.
- `lexical`: classic full-text match on the exact terms.

```
# semantic (default) — natural language, ranked by relevance
get_datasets(search="air quality measurements")

# lexical — exact full-text match
get_datasets(search="luft", search_mode="lexical")

# combine with facet filters
get_datasets(search="bevölkerung", refine="publisher:Statistisches Amt")
```

### `get_dataset`
Get detailed metadata for a specific dataset.

```
get_dataset(dataset_id="100113")
```

### `get_records`
Query records from a dataset with ODSQL filtering.

```
get_records(dataset_id="100113", where="pm25 > 10", limit=100, order_by="time DESC")
```

### `get_facets`
Get available facet values for filtering.

```
get_facets(facet="publisher")  # Options: publisher, keyword, theme, features, modified, language
```

### `export_dataset_url`
Get download URL for dataset export.

```
export_dataset_url(dataset_id="100113", format="csv", where="sensornr=240")
```

Formats: `csv`, `json`, `geojson`, `xlsx`, `shp`, `parquet`, `gpx`, `kml`, `rdfxml`, `jsonld`, `turtle`
