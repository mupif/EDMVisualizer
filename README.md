# EDM Interactive Browser

Browser-based tools for exploring entities stored in the **MuPIF DB Entity Data Model (EDM)**. They show how entities are composed, which entities they were derived from, and which workflow executions produced them.

The repository contains two independent single-page tools:

| Tool | File | Purpose |
|---|---|---|
| **Interactive browser** | [`DynamicVisualizer/index.html`](DynamicVisualizer/index.html) | Explore an entity graph step by step: expand and collapse nodes, follow executions, and inspect metadata and full data in a side panel. |
| **Static visualizer** | [`StaticVisualizer/index.html`](StaticVisualizer/index.html) | Show the context of a selected entity as a compact Graphviz graph: entity types, labels and IDs, and their links to other entities and executions. It is meant for linking from or embedding in other pages. |

Both tools talk directly to the MuPIF DB REST API (default: `https://test.mupif.org/api/`) and use the same authentication as the Python client [`mupifDB/api/client_mupif.py`](https://github.com/mupif/mupifDB/blob/master/mupifDB/api/client_mupif.py).

## Prerequisites

- **A modern web browser**: a current version of Chrome, Edge, Firefox or Safari.
- **A MuPIF DB account** on the target server. The username is your e-mail address.
- **Network access** to the MuPIF DB API and to the CDNs the pages load their libraries from:
  - Interactive browser: Cytoscape.js, dagre, cytoscape-dagre and JSONEditor (cdnjs.cloudflare.com, cdn.jsdelivr.net).
  - Static visualizer: Viz.js, the Graphviz build for the browser (cdn.jsdelivr.net).
- **Optional:** Python 3 or any other static file server, to serve the pages over `http://localhost` (recommended, see below).

There is nothing to build and no server-side component.

## Installation

```bash
git clone <repository-url> EDMInteractiveBrowser
cd EDMInteractiveBrowser
```

Serve the directory with any static web server, for example:

```bash
python3 -m http.server 8000
```

Then open:

- Interactive browser: <http://localhost:8000/DynamicVisualizer/index.html>
- Static visualizer: <http://localhost:8000/StaticVisualizer/index.html>

Opening the files directly from disk (`file://…`) may also work, but serving them from `localhost` is more reliable. Some browser features, such as the clipboard used by the **Copy** button, are only available on `http://localhost` or HTTPS.

## Configuration

Settings are constants at the top of the `<script>` block of each page.

**Interactive browser** (`DynamicVisualizer/index.html`):

| Constant | Default | Meaning |
|---|---|---|
| `API_ROOT_URL` | `https://test.mupif.org/api/` | Base URL of the MuPIF DB API |
| `EDM_DB_NAME` | `dms_sysbet01` | Default EDM database |
| `INITIAL_NODE_TYPE` | `Calibration` | Type of the entity shown after sign-in |
| `INITIAL_NODE_ID` | `6ab514418294832e099ab9d8` | ID of the entity shown after sign-in |
| `API_QUERY_PARAMS` | `?max_level=2&tracking=false&meta=true` | Query used when fetching an entity; `max_level` controls how many nested levels are loaded at once |

The database, initial entity and depth can also be set per link with [page parameters](#page-parameters-linking-and-embedding). These constants are the fallbacks.

**Static visualizer** (`StaticVisualizer/index.html`):

| Constant | Default | Meaning |
|---|---|---|
| `API_ROOT_URL` | `https://test.mupif.org/api/` | Base URL of the MuPIF DB API |
| `DEFAULT_ENTITY` | `dms_sysbet01` / `Calibration` / `6ab514418294832e099ab9d8` / depth `2` | Entity and depth used when the page parameters don't specify them |

## Usage

### Signing in

Both pages open with a **Sign in to MuPIF DB** dialog. After a successful sign-in:

- The page keeps your credentials **in memory only**; nothing is written to browser storage. Closing or reloading the page signs you out.
- The access token is refreshed automatically shortly before it expires. If refreshing fails, or the server rejects the token, the page signs in again with the stored credentials.
- If signing in again fails (for example, because the password changed), the sign-in dialog reappears. In the interactive browser, the graph you have built is kept.

### Interactive browser

After sign-in, the configured initial entity is loaded and laid out left to right.

**Mouse actions**

- **Left-click** a node or edge to show it in the **Inspector** on the right.
- **Right-click** a node or edge for actions:
  - **➕ Fetch & Expand**: loads an unloaded entity, or the details of an execution, and adds the related entities to the graph.
  - **➖ Collapse Data**: turns a loaded node back into a stub and removes its edges.
  - **👁️ Inspect in Sidebar**: same as left-click.
- Scroll to zoom and drag to pan.

**Graph legend**

| Element | Meaning |
|---|---|
| Filled blue node `[-] Type` | Loaded entity |
| White node with blue outline `[+] …` | Entity that is referenced but not loaded yet |
| Light-blue edge with diamond | Composition: the entity contains the linked entity (`contains: <attribute>`) |
| Purple dashed edge `belongs to` | Link to the parent entity |
| Orange edge `derived from` | Upstream entity with no recorded execution |
| White hexagon, orange dashed outline `⚙ Execution` | Execution that produced the entity; details not loaded yet |
| Orange hexagon `⚙ Workflow vN / Status` | Execution with loaded details; orange edges link its inputs and outputs |

**Inspector**

The header shows the selected element's type and full ID, with a **📋 Copy** button. Below it are two tabs:

- **Metadata**:
  - *Entity:* type, database, parent, upstream and execution IDs, and whether the data is loaded. IDs that are in the graph are links that select and centre the element. If the execution's details are not loaded yet, a **⬇ Load execution details** button is shown; once they are loaded, the workflow and a status badge appear next to the execution.
  - *Execution:* workflow and version, status, requester, created/submitted/started/ended times, duration and attempts, plus **Inputs** and **Outputs** tables (Name, EDMPath, Units, Value). Each EDMPath links to the entity it maps to.
  - *Other links:* relation type and its From/To entities.
- **Data**: the complete JSON of the selected element in a searchable tree. String values that match IDs of nodes in the graph are highlighted; clicking one jumps to that node.

The selected tab stays active as you click through the graph.

### Static visualizer

The static visualizer gives a compact picture of an entity's **context**, not its data. It is meant to be linked to or embedded in other pages. It has no controls of its own: the entity and the traversal depth come from the [page parameters](#page-parameters-linking-and-embedding).

After you sign in, the page fetches the entity up to the configured depth and the executions that produced the fetched entities. It then renders a Graphviz graph:

- **Entities** show only their type, their `label` attribute (if they have one) and a shortened ID (e.g. `6ab514…`). The selected entity is filled dark blue. Hover over an entity to see its full ID.
- **Entities beyond the traversal depth** (referenced but not fetched) are grey dashed boxes showing their type, if it is known, and their shortened ID.
- **Edges:**
  - composition, labelled with the attribute name and drawn with a diamond on the container;
  - `belongs to`, for a parent outside the fetched tree;
  - `derived from`, for an upstream entity without an execution;
  - executions, as hexagon nodes labelled `⚙ <workflow>`, version, status and the shortened execution ID, linked from their inputs and to their outputs. Hover over an execution node to see its full ID.

If the page was given a `link`, clicking the graph opens it.

### Page parameters (linking and embedding)

Other web pages can open either tool, or embed it in an `<iframe>`, and choose what it shows with URL query parameters. Parameter values must be URL-encoded. The user still signs in on the opened page.

| Parameter | Interactive browser | Static visualizer | Meaning |
|---|---|---|---|
| `db` | ✓ | ✓ | EDM database (default `dms_sysbet01`) |
| `type` | ✓ | ✓ | Entity type of the initial entity |
| `id` | ✓ | ✓ | Entity ID of the initial entity |
| `depth` | ✓ (default `2`) | ✓ (default `2`) | Traversal depth: `max_level` of the EDM query; `-1` = full depth. In the interactive browser it applies to every Fetch & Expand. |
| `link` |  | ✓ | Optional URL opened when the user clicks the graph. `{db}`, `{type}`, `{id}` and `{depth}` are replaced by the rendered entity's values. Relative URLs are resolved against the static visualizer page. Only `http(s)` URLs are accepted. |
| `target` |  | ✓ | Where `link` opens: `_blank` (default, new tab or window), `_self`, `_top` (whole page when embedded), or a window name to reuse the same tab |

The API URL is intentionally **not** a parameter. Otherwise, a crafted link could make the sign-in dialog send the user's password to a different server.

Examples:

```text
DynamicVisualizer/index.html?db=dms_sysbet01&type=BeamState&id=647718cbe054fab36036d16a&depth=-1

StaticVisualizer/index.html?type=Calibration&id=6ab514418294832e099ab9d8&depth=1&link=..%2FDynamicVisualizer%2Findex.html%3Ftype%3D%7Btype%7D%26id%3D%7Bid%7D
```

The second example shows the context of the Calibration entity one level deep. Clicking it opens the interactive browser for the same entity in a new tab.

Build such URLs with `URLSearchParams`, which takes care of the encoding:

```html
<iframe id="edm-graph" width="100%" height="500"></iframe>
<script>
  const params = new URLSearchParams({
    type: 'Calibration',
    id: '6ab514418294832e099ab9d8',
    depth: '1',
    link: '../DynamicVisualizer/index.html?type={type}&id={id}',  // open the interactive browser on click
    target: '_blank'
  });
  document.getElementById('edm-graph').src = `StaticVisualizer/index.html?${params}`;
</script>
```

## How executions are drawn

An entity whose `meta` contains `execution` was produced by a MuPIF workflow execution. Both tools fetch the execution from `GET /api/executions/{id}` and draw it as a single hexagonal **execution node** (shared by all entities with the same execution), so N inputs and M outputs need N+M edges instead of N×M:

- **Upstream entity → execution → entity** when the entity has `meta.upstream`. Without upstream, the execution node points to the entity.
- **Inputs → execution → outputs** for the entities listed in the execution's `EDMMapping` (edge labels are the mapping names):
  - A mapping is an **output** when an `OutputsData` item writes to it (the first segment of its `EDMPath`, e.g. `C` in `C.q1`, equals the mapping `Name`), or when `createNew` contains an ID.
  - All other mappings are **inputs**.
  - An entity that is both input and output (updated in place) is drawn only as an output: **execution → entity**.

Mapped entities that are not yet in the graph are added as unloaded nodes. In the static visualizer, an execution that cannot be fetched is drawn as a dashed `⚙ execution` node, and the rest of the graph is still rendered.

## API endpoints used

| Method & path | Used for |
|---|---|
| `POST /api/login` | Sign-in (form-encoded `grant_type=password`, `username`, `password`) |
| `POST /api/refresh_token` | Token refresh (Bearer token) |
| `GET /api/EDM/{db}/schema` | Entity type schema of a database |
| `GET /api/EDM/{db}/{type}/{id}?max_level=…&tracking=false&meta=true` | Entity data with metadata |
| `GET /api/executions/{id}` | Execution details |

All requests except login send `Authorization: Bearer <token>`.

## Troubleshooting

| Symptom | What to check |
|---|---|
| `Login failed: Incorrect username or password` | Use your MuPIF DB e-mail address as the username. |
| `Network Error!` or `Failed to fetch` | API reachable from your network? CDN scripts blocked by an extension or proxy? Check the browser console. |
| `HTTP error! status: 404` on Fetch & Expand | The entity type or ID does not exist in that database, e.g. a stale reference. |
| Execution node stays `⚙ Execution` | The execution could not be loaded (deleted, or not accessible to your account). The browser console has details. |
| **Copy** button reports "Clipboard access denied" | Serve the page from `http://localhost` or HTTPS instead of `file://`. |

## Repository layout

```
DynamicVisualizer/index.html   Interactive browser (Cytoscape.js + JSONEditor)
StaticVisualizer/index.html    Static entity-context graph (Graphviz via Viz.js)
README.md
```
