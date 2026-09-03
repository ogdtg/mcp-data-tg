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

## Hosting on Posit Connect (via GitHub)

Besides the stdio entry point used above, `main.py` also exposes a
Streamable HTTP ASGI app as `app`, so this repo can be published to Posit
Connect via **Git-backed publishing** and reached as a remote MCP server.
Everything Connect needs is committed: `manifest.json`
(`appmode: python-fastapi`, entrypoint `main:app`), `requirements.txt`,
and the `.env` with the catalog domain.

1. Before publishing, check `.env` — `DATA_PORTAL_DOMAIN` selects the
   catalog this deployment serves (already set to `data.tg.ch` for this
   fork). Commit and push any change to GitHub.
2. In Posit Connect, click **Publish → Import from Git**, enter this
   repository's GitHub URL, pick the branch, and select the root directory
   (where `manifest.json` lives).
3. Connect builds the environment from `manifest.json`/`requirements.txt`
   and starts the app itself — no extra configuration. It also polls the
   branch and automatically redeploys when you push new commits.
4. In the content's **Access** panel, set who can access it. For use as a
   Claude connector it must be reachable without a Connect login:
   set access to **Anyone — no login required**.
5. The MCP endpoint is the content URL with `/mcp` appended, e.g.
   `https://<connect-server>/content/<guid>/mcp`. Verify it responds:

   ```bash
   npx @modelcontextprotocol/inspector
   # transport: "Streamable HTTP", URL: https://<connect-server>/content/<guid>/mcp
   ```

If you change dependencies or files, regenerate `requirements.txt` and
`manifest.json` before pushing:

```bash
uv sync
uv pip freeze | grep -v '^-e ' > requirements.txt
uvx rsconnect write-manifest fastapi --entrypoint main:app --overwrite .
```

## Connecting to Claude (web)

Once deployed on Posit Connect, add the server to claude.ai as a custom
connector (requires a Pro, Max, Team, or Enterprise plan; on Team/Enterprise
an admin may need to do this):

1. In claude.ai go to **Settings → Connectors** and click
   **Add custom connector**.
2. Enter a name (e.g. `data.tg.ch`) and the MCP endpoint URL
   `https://<connect-server>/content/<guid>/mcp`, leave the OAuth fields
   empty (the server is unauthenticated), and save.
3. In a chat, open the **search & tools** menu, enable the connector, and
   ask e.g. *"Which datasets about air quality are available on data.tg.ch?"*.

Note: the content must be publicly reachable (step 4 above) — claude.ai
custom connectors support no authentication or OAuth, but cannot send Posit
Connect API keys.

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
