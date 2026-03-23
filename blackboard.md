# Blackboard — graph-notebook

> Single-binary architecture: graph-notebook's query engines transcoded to Rust.

## What Exists

graph-notebook is a Jupyter extension providing `%%sparql`, `%%gremlin`, and `%%oc` (OpenCypher) magics for querying graph databases. It's tightly coupled to IPython/Jupyter.

## Query Executors

### %%sparql (`graph_magic.py` lines 764-1080)
- Protocol: HTTP POST via `requests` + `SPARQLWrapper`
- Content-Type: `application/sparql-query`
- Returns: JSON (SELECT) or RDF (CONSTRUCT/DESCRIBE)
- Modes: `query`, `explain`

### %%gremlin (`graph_magic.py` lines 1114-1552)
- Protocol: WebSocket via `gremlin_python` OR HTTP POST
- Serialization: GraphSON v3/v4, GraphBinary
- Transport: `AiohttpTransport` for async WebSocket
- Returns: traversal results (vertices, edges, paths, values)
- Modes: `query`, `explain`, `profile`

### %%oc / %%opencypher (`graph_magic.py` lines 1589-3944)
- Protocol: HTTP POST to Neptune endpoint OR Bolt via `neo4j` Python driver
- Bolt: `neo4j>=5.0.0,<=5.23.1`
- Returns: JSON or JOLT format
- Modes: `query`, `bolt`, `explain`

## Visualization Pipeline

1. Raw results → Language-specific `Network` subclass (GremlinNetwork, SPARQLNetwork, OCNetwork)
2. All extend base `Network` wrapping `networkx.MultiDiGraph`
3. Network → `EventfulNetwork` → `Force` widget (vis.js)
4. Frontend: `vis-network@9.1.6` renders graph with physics simulation
5. Tables: `itables>=2.0.0` for DataTables rendering

## Jupyter Coupling Points

- `IPython.core.magic`: `Magics`, `@magics_class`, `@cell_magic`, `@line_cell_magic`
- `IPython.core.display`: `display()`, `HTML()`, `clear_output()`
- `ipywidgets`: `Tab`, `Output`, `Layout`, `DOMWidget`
- `@jupyter-widgets/base`: `DOMWidgetModel`, `DOMWidgetView`

## What Gets Transcoded to Rust

| Python Component | Rust Replacement |
|---|---|
| `Client.sparql()` | `reqwest` HTTP POST |
| `Client.gremlin_query()` | `tokio-tungstenite` WebSocket |
| `Client.opencypher_bolt()` | `bolt-proto` or hand-rolled Bolt |
| `Client.opencypher_http()` | `reqwest` HTTP POST |
| `GremlinNetwork` / `SPARQLNetwork` / `OCNetwork` | `QueryResult` struct (nodes + edges + rows) |
| `Force` widget (vis.js) | Static vis.js assets served by binary |

## Key Files

| File | Purpose |
|---|---|
| `src/graph_notebook/magics/graph_magic.py` | All magic implementations (62K lines) |
| `src/graph_notebook/neptune/client.py` | Query executor client (~1244 lines) |
| `src/graph_notebook/network/Network.py` | Base graph network class |
| `src/graph_notebook/network/gremlin/GremlinNetwork.py` | Gremlin result → graph |
| `src/graph_notebook/network/sparql/SPARQLNetwork.py` | SPARQL result → graph |
| `src/graph_notebook/network/opencypher/OCNetwork.py` | OpenCypher result → graph |
| `src/graph_notebook/widgets/force/force_widget.py` | vis.js Python widget |
| `src/graph_notebook/widgets/src/force_widget.ts` | vis.js TypeScript frontend |
