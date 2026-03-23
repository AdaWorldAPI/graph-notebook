# graph-notebook — Graph Query Magics for Jupyter

## Role in Stack
Source of graph query magic implementations. We extract %%oc, %%gremlin, %%sparql
and port them into marimo. Also provides vis.js graph visualization.

## Current State (upstream AWS)
- Jupyter magics: %%oc (openCypher), %%gremlin, %%sparql
- Connects to: Neo4j (Bolt), Neptune, any Gremlin Server, any SPARQL endpoint
- vis.js rendering of graph results with node/edge visualization
- Python package, pip-installable

## Our Fork's Mission
1. Extract magic implementations as standalone Python modules (no Jupyter dependency)
2. Add %%nars magic for NARS truth-value queries
3. Add %%cypher local path that routes to lance-graph semiring instead of remote DB
4. Adapt vis.js rendering for marimo's output model

## Key Directories
- src/graph_notebook/magics/ — magic cell implementations
- src/graph_notebook/visualization/ — vis.js rendering
- src/graph_notebook/neptune/ — AWS Neptune specifics (may not need)
- src/graph_notebook/configuration/ — connection config

## Integration Points
- **marimo** → our magics become marimo cell types
- **kernel-protocol** → magics route through kernel wire format
- **lance-graph** → local %%cypher execution path
- **Neo4j** → remote Bolt connections (existing, keep)
- **FalkorDB** → remote Bolt connections (add)
