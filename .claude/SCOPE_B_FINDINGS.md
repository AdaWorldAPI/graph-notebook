# Scope B Findings: Query Protocols, Return Types, and vis.js Input Format

## 1. %%oc (OpenCypher) Protocol

OpenCypher has **three** protocol paths depending on the backend:

### 1a. Neptune DB — HTTP POST (`opencypher_http`)

- **Endpoint**: `https://{host}:{port}/openCypher`
- **Method**: HTTP POST
- **Content-Type**: `application/x-www-form-urlencoded`
- **Body**: form-encoded with key `query` containing the Cypher query string
- **Library**: Python `requests` (via `self._http_session.send()`)
- **Auth**: Optional SigV4 signing via `botocore.auth.SigV4Auth`

```python
# client.py line 568-571
headers['content-type'] = 'application/x-www-form-urlencoded'
url += f':{self.port}/openCypher'
data['query'] = query
```

### 1b. Neptune Analytics — HTTP POST (`opencypher_http`)

- **Endpoint**: `https://{host}/queries` (port optional)
- **Body**: includes `language: 'opencypher'` plus `query`
- Otherwise same as Neptune DB

```python
# client.py line 563-567
url += f'/queries'
data['language'] = 'opencypher'
data['query'] = query
```

### 1c. Neo4j — HTTP POST (non-Neptune path in `opencypher_http`)

- **Endpoint**: `{protocol}://{host}db/neo4j/tx/commit`
- **Content-Type**: `application/json`
- **Accept**: `application/vnd.neo4j.jolt+json-seq` (JOLT streaming format)
- **Body**: JSON with `statements` array:
  ```json
  {
    "statements": [
      { "statement": "<cypher query>" }
    ]
  }
  ```
- **Auth**: HTTP Basic Auth (Base64 of `username:password`)

```python
# client.py line 593-608
url += 'db/neo4j/tx/commit'
headers['content-type'] = 'application/json'
headers['Accept'] = 'application/vnd.neo4j.jolt+json-seq'
data_dict = {"statements": [{"statement": query}]}
data = json.dumps(data_dict)
if self.neo4j_auth:
    user_and_pass = self.neo4j_username + ":" + self.neo4j_password
    user_and_pass_base64 = b64encode(user_and_pass.encode())
    headers['authorization'] = user_and_pass_base64
```

### 1d. Neo4j — Bolt Binary Protocol (`opencyper_bolt`)

Used when `%%oc bolt` mode is specified.

- **Protocol**: Bolt binary protocol over TCP
- **URL**: `bolt://{host}:{port}` (with optional `/opencypher` path for Neptune IAM)
- **Library**: `neo4j` Python driver (`neo4j.GraphDatabase.driver`)
- **Auth**: Tuple `(username, password)` or `NeptuneBoltAuthToken` for IAM
- **Encryption**: Controlled by `self.ssl` flag passed as `encrypted` param
- **Database**: Configurable via `neo4j_database` (defaults to `neo4j.DEFAULT_DATABASE`)

```python
# client.py line 670-697
def get_opencypher_driver(self):
    url = f'bolt://{self.host}:{self.port}'
    # ... IAM or basic auth setup ...
    driver = GraphDatabase.driver(url, auth=auth_final, encrypted=self.ssl)
    return driver
```

Query execution uses `session.run()`:
```python
# client.py line 614-652
def opencyper_bolt(self, query: str, **kwargs):
    driver = self.get_opencypher_driver()
    with driver.session(database=self.neo4j_database) as session:
        res = session.run(query, kwargs)
        data = []
        for record in res:
            record_dict = {}
            for key in record.keys():
                value = record[key]
                if hasattr(value, 'labels'):  # Node
                    record_dict[key] = {
                        '~id': str(value.id),
                        '~entityType': 'node',
                        '~labels': list(value.labels),
                        '~properties': dict(value)
                    }
                elif hasattr(value, 'type'):  # Relationship
                    record_dict[key] = {
                        '~id': str(value.id),
                        '~entityType': 'relationship',
                        '~start': str(start_node.id),
                        '~end': str(end_node.id),
                        '~type': value.type,
                        '~properties': dict(value)
                    }
                else:
                    record_dict[key] = value
            data.append(record_dict)
    return data
```

---

## 2. %%gremlin Protocol

Gremlin has **two** protocol paths:

### 2a. WebSocket + GraphSON Serialization (default for Neptune DB and non-Neptune)

- **Protocol**: WebSocket (`wss://` or `ws://` depending on SSL)
- **Endpoint**: `wss://{host}:{port}/gremlin`
- **Library**: `gremlin_python.driver.client.Client` with `AiohttpTransport`
- **Serialization**: GraphSON v3 by default (`GraphSONSerializersV3d0`), also supports GraphSON v2 and GraphBinary v1
- **Query method**: String-based Gremlin query submitted via `client.submit(query, bindings)`
- **Auth**: Username/password passed to client constructor; for Neptune IAM, headers from SigV4-signed request
- **Traversal source**: `'g'` for Neptune, configurable for other servers

```python
# client.py line 439-477
def get_gremlin_connection(self, transport_kwargs) -> client.Client:
    nest_asyncio.apply()
    ws_url = f'{self.get_uri(use_websocket=True, use_proxy=False)}/gremlin'
    message_serializer = get_gremlin_serializer_driver_class(self.gremlin_serializer)
    return client.Client(ws_url, traversal_source, transport_factory=transport_factory_args,
                         username=self.gremlin_username, password=self.gremlin_password,
                         message_serializer=message_serializer,
                         headers=dict(request.headers), **transport_kwargs)

def gremlin_query(self, query, transport_args=None, bindings=None):
    c = self.get_gremlin_connection(transport_args)
    result = c.submit(query, bindings)
    future_results = result.all()
    results = future_results.result()
    c.close()
    return results
```

Serializer selection:
```python
# client.py line 195-201
def get_gremlin_serializer_driver_class(serializer_str: str):
    if serializer_str == GRAPHBINARYV1:
        return serializer.GraphBinarySerializersV1()
    elif serializer_str == GRAPHSONV2:
        return serializer.GraphSONSerializersV2d0()
    else:
        return serializer.GraphSONSerializersV3d0()
```

Available serializer MIME types:
```python
GREMLIN_SERIALIZERS_CLASS_TO_MIME_MAP = {
    'GraphSONMessageSerializerGremlinV1':    'application/vnd.gremlin-v1.0+json',
    'GraphSONMessageSerializerV2':           'application/vnd.gremlin-v2.0+json',
    'GraphSONMessageSerializerV3':           'application/vnd.gremlin-v3.0+json',
    'GraphSONMessageSerializerV4':           'application/vnd.gremlin-v4.0+json',
    'GraphSONUntypedMessageSerializerV1':    'application/vnd.gremlin-v1.0+json;types=false',
    'GraphSONUntypedMessageSerializerV2':    'application/vnd.gremlin-v2.0+json;types=false',
    'GraphSONUntypedMessageSerializerV3':    'application/vnd.gremlin-v3.0+json;types=false',
    'GraphSONUntypedMessageSerializerV4':    'application/vnd.gremlin-v4.0+json;types=false',
    'GraphBinaryMessageSerializerV1':        'application/vnd.graphbinary-v1.0'
}
```

### 2b. HTTP POST (`gremlin_http_query`)

Used when `--connection-protocol http` is specified (or forced for Neptune Analytics).

- **Neptune DB endpoint**: `POST https://{host}:{port}/gremlin` with body `{"gremlin": "<query>"}`
- **Neptune Analytics endpoint**: `POST https://{host}/queries` with body `{"query": "<query>", "language": "gremlin"}` and `Content-Type: application/json`
- **Response parsing**: JSON with `result.data` path:
  ```python
  # graph_magic.py line 1352-1353
  query_res_http_json = query_res_http.json()
  if 'result' in query_res_http_json:
      query_res = query_res_http_json['result']['data']
  ```

---

## 3. %%sparql Protocol

### HTTP POST via `requests`

- **Endpoint**: `https://{host}:{port}/{sparql_path}` (default path: `/sparql`)
- **Method**: HTTP POST
- **Content-Type**: `application/x-www-form-urlencoded` (default, set in `DEFAULT_SPARQL_CONTENT_TYPE`)
- **Body**: Form-encoded: `query=<SPARQL query>` for SELECT/CONSTRUCT/ASK/DESCRIBE, `update=<SPARQL update>` for INSERT/DELETE
- **Accept header**: `application/sparql-results+json` for SELECT queries (default for Neptune)
- **Library**: Python `requests` via `self._http_session.send()`
- **Auth**: Optional SigV4 signing via `botocore.auth.SigV4Auth`

```python
# client.py line 373-406
def sparql_query(self, query, headers=None, explain='', path=''):
    data = {'query': query}
    return self.do_sparql_request(data, headers, explain, path=path)

def do_sparql_request(self, data, headers=None, explain='', path=''):
    if 'content-type' not in headers:
        headers['content-type'] = DEFAULT_SPARQL_CONTENT_TYPE  # 'application/x-www-form-urlencoded'
    uri = f'{self._http_protocol}://{self.host}:{self.port}{sparql_path}'
    req = self._prepare_request('POST', uri, data=data, headers=headers)
    res = self._http_session.send(req, verify=self.ssl_verify)
    return res
```

Query type detection uses `SPARQLWrapper`:
```python
# client.py line 408-418
def sparql(self, query, headers=None, explain='', path=''):
    s = SPARQLWrapper('')
    s.setQuery(query)
    query_type = s.queryType.upper()
    if query_type in ['SELECT', 'CONSTRUCT', 'ASK', 'DESCRIBE']:
        return self.sparql_query(query, headers, explain, path=path)
    else:
        return self.sparql_update(query, headers, explain, path=path)
```

---

## 4. Return Types

### 4a. %%sparql returns

`client.sparql()` returns a `requests.Response`. The magic parses it:

```python
# graph_magic.py line 898-901
try:
    results = query_res.json()
except Exception:
    results = query_res.content.decode('utf-8')
```

For SELECT queries with `application/sparql-results+json`, the JSON structure is the W3C SPARQL Results JSON format:

```json
{
  "head": {
    "vars": ["s", "p", "o"]
  },
  "results": {
    "bindings": [
      {
        "s": {"type": "uri", "value": "http://example.org/resource1"},
        "p": {"type": "uri", "value": "http://example.org/property1"},
        "o": {"type": "literal", "value": "some value"}
      }
    ]
  }
}
```

This is then fed to `SPARQLNetwork.add_results(results)` for graph visualization.

### 4b. %%gremlin returns

**WebSocket path**: `client.gremlin_query()` returns a Python list of deserialized Gremlin objects. Elements can be:
- `gremlin_python.structure.graph.Path` objects (from `.path()` traversals)
- `gremlin_python.structure.graph.Vertex` objects
- `gremlin_python.structure.graph.Edge` objects
- Python dicts (from `.valueMap()`, `.elementMap()`, `.project()` etc.)
- Primitives (strings, numbers)

The `Path` object contains:
- `path.objects` — list of alternating Vertex/Edge/dict objects
- `path.labels` — list of step labels

```python
# gremlin_python types:
class Vertex:
    id: any       # vertex id
    label: str    # vertex label

class Edge:
    id: any       # edge id
    label: str    # edge label
    inV: Vertex   # target vertex
    outV: Vertex  # source vertex
```

**HTTP path**: Returns JSON dict with `result.data` as a list:
```python
query_res = query_res_http_json['result']['data']
```
For HTTP, key identifiers use string `'id'`/`'~id'` and `'label'` instead of `T.id`/`T.label`.

### 4c. %%oc returns

**HTTP path** (`opencypher_http`): Returns `requests.Response`. Parsed as JSON:
```python
res = oc_http.json()
```

The Neptune OpenCypher HTTP response format:
```json
{
  "results": [
    {
      "a": {
        "~id": "vertex-id-1",
        "~entityType": "node",
        "~labels": ["Person"],
        "~properties": {"name": "Alice", "age": 30}
      },
      "r": {
        "~id": "edge-id-1",
        "~entityType": "relationship",
        "~start": "vertex-id-1",
        "~end": "vertex-id-2",
        "~type": "KNOWS",
        "~properties": {"since": 2020}
      }
    }
  ]
}
```

Can also come in JOLT format (newline-separated JSON objects with `"data"` keys).

**Bolt path** (`opencyper_bolt`): Returns a Python list of dicts. Each dict maps column names to values. Node/Relationship values are converted to the same `~entityType`/`~id`/`~labels`/`~properties` dict structure shown above (see section 1d).

---

## 5. vis.js Input Format

### The pipeline: Network -> NetworkX -> JSON -> vis.js

The data flows through:
1. Query results are added to a language-specific `EventfulNetwork` subclass (GremlinNetwork, OCNetwork, SPARQLNetwork)
2. These extend `Network`, which wraps a NetworkX `MultiDiGraph`
3. `Network.to_json()` serializes via `networkx.readwrite.json_graph.node_link_data()`
4. The `Force` widget's `graph_to_json` serializer sends this to the browser
5. The TypeScript `ForceView.populateDatasets()` translates NetworkX links to vis.js edges

### JSON structure sent to browser

The `to_json()` method produces:

```python
# Network.py line 95-102
def to_json(self) -> dict:
    graph_data = json_graph.node_link_data(self.graph, edges="links")
    return {'graph': graph_data}
```

The resulting JSON structure (NetworkX `node_link_data` format):

```json
{
  "graph": {
    "directed": true,
    "multigraph": true,
    "graph": {},
    "nodes": [
      {
        "id": "vertex-id-1",
        "label": "Alice",
        "title": "Person: Alice",
        "group": "Person",
        "properties": {
          "name": "Alice",
          "age": 30
        }
      },
      {
        "id": "vertex-id-2",
        "label": "Bob",
        "title": "Person: Bob",
        "group": "Person",
        "properties": {
          "name": "Bob"
        }
      }
    ],
    "links": [
      {
        "source": "vertex-id-1",
        "target": "vertex-id-2",
        "key": "edge-id-1",
        "label": "KNOWS",
        "title": "KNOWS",
        "properties": {
          "since": 2020
        }
      }
    ]
  }
}
```

### TypeScript types (from `types.ts`)

```typescript
// The top-level network object
class ForceNetwork {
    graph: Graph;
}

// The graph contains nodes and links
class Graph {
    nodes: VisNode[];   // directly from NetworkX node_link_data
    links: Link[];      // from NetworkX, NOT called "edges"
}

// Nodes: id is required, everything else is dynamic
class VisNode implements Node {
    readonly id: string;
    // Plus any extra properties: label, title, group, properties, etc.
    [key: string]: any;
}

// Links come from NetworkX with source/target naming
class Link {
    key: string;        // edge id (NetworkX "key" for multigraph)
    label: string;
    source: string;     // from-node id
    target: string;     // to-node id
    [key: string]: any; // title, properties, etc.
}

// vis.js edges use from/to naming
class VisEdge implements Edge {
    readonly from: string;
    readonly to: string;
    readonly id: string;    // constructed as "{from}:{to}:{key}"
    label: string;
    [key: string]: any;
}
```

### Link-to-Edge translation

The `linksToEdges` method translates NetworkX links to vis.js edges:

```typescript
// force_widget.ts line 461-476
linksToEdges(links: Array<Link>): Array<VisEdge> {
    const edges = new Array<VisEdge>();
    const propsToSkip = ["source", "target", "key"];
    links.forEach(function (link) {
        const edge = new VisEdge(link.source, link.target, link.key, link.label);
        for (const propName in link) {
            if (!(propName in propsToSkip)) {
                edge[propName] = link[propName];
            }
        }
        edges.push(edge);
    });
    return edges;
}
```

The VisEdge constructor generates a composite id:
```typescript
// types.ts line 101
this.id = from + ":" + to + ":" + id;
```

### Node data fields used by vis.js

Each node added by the Network subclasses includes these fields that vis.js consumes:

| Field | Purpose | Set by |
|---|---|---|
| `id` | Unique node identifier | All networks: vertex id |
| `label` | Displayed text on node | Truncated to `label_max_length` chars |
| `title` | Tooltip on hover | Full label or custom tooltip property |
| `group` | Color grouping | Configurable: label, property, depth |
| `properties` | Flattened property dict | All key-value properties |

### Edge data fields used by vis.js

| Field | Purpose | Set by |
|---|---|---|
| `from` (via source) | Source node id | All networks |
| `to` (via target) | Target node id | All networks |
| `id` (via key) | Edge identifier | Edge id from graph |
| `label` | Displayed text on edge | Truncated to `edge_label_max_length` |
| `title` | Tooltip on hover | Edge label or custom property |
| `properties` | Flattened property dict | All key-value properties |

### JSON safety conversion

Before sending to the browser, the `convert_to_json_safe` function (in `force_widget.py`) ensures all dict keys are JSON-serializable by converting non-standard types (like Gremlin `T` enums) to strings:

```python
# force_widget.py line 14-34
def convert_to_json_safe(data):
    if isinstance(data, dict):
        return {
            str(k) if hasattr(k, '__class__') else k: convert_to_json_safe(v)
            for k, v in data.items()
        }
    elif isinstance(data, list):
        return [convert_to_json_safe(item) for item in data]
    elif hasattr(data, '__class__') and not isinstance(data, (str, int, float, bool)):
        return str(data)
    return data
```

---

## Summary Table

| Magic | Protocol | Library | Default Serialization | Return Type |
|---|---|---|---|---|
| `%%sparql` | HTTP POST (form-encoded) | `requests` | `application/sparql-results+json` | `requests.Response` -> JSON dict (W3C SPARQL Results) |
| `%%gremlin` (WS) | WebSocket | `gremlin_python` | GraphSON v3 untyped | Python list of Path/Vertex/Edge/dict |
| `%%gremlin` (HTTP) | HTTP POST (JSON) | `requests` | GraphSON v3 untyped MIME | JSON dict -> `result.data` list |
| `%%oc` (query) | HTTP POST (form/json) | `requests` | JSON | `requests.Response` -> JSON with `results` array |
| `%%oc` (bolt) | Bolt binary TCP | `neo4j` driver | Bolt native | Python list of dicts with `~entityType` structure |

### Key Files

- `/home/user/graph-notebook/src/graph_notebook/magics/graph_magic.py` — magic method definitions and result handling
- `/home/user/graph-notebook/src/graph_notebook/neptune/client.py` — Client class with all protocol implementations
- `/home/user/graph-notebook/src/graph_notebook/network/Network.py` — base Network with `to_json()` using NetworkX
- `/home/user/graph-notebook/src/graph_notebook/network/EventfulNetwork.py` — adds event callbacks for widget sync
- `/home/user/graph-notebook/src/graph_notebook/network/gremlin/GremlinNetwork.py` — Gremlin Path/elementMap -> graph
- `/home/user/graph-notebook/src/graph_notebook/network/opencypher/OCNetwork.py` — OpenCypher `~entityType` dict -> graph
- `/home/user/graph-notebook/src/graph_notebook/widgets/force/force_widget.py` — Python-side Force widget with JSON serializer
- `/home/user/graph-notebook/src/graph_notebook/widgets/src/force_widget.ts` — TypeScript vis.js rendering
- `/home/user/graph-notebook/src/graph_notebook/widgets/src/types.ts` — VisNode, VisEdge, Link, ForceNetwork types
