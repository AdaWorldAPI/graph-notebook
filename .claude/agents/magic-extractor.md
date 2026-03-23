# Agent: Magic Extractor

## Mission
Decouple graph magics from Jupyter so they work as standalone query executors.

## Scope
- %%oc: openCypher → Bolt → result DataFrame + graph viz
- %%gremlin: Gremlin → websocket → result
- %%sparql: SPARQL → HTTP → result
- Each magic should be: parse(cell_text) → execute(connection) → Result(df, graph)
- No IPython/Jupyter imports in the extracted code

## Output
- graph_notebook/core/cypher.py — standalone Cypher executor
- graph_notebook/core/gremlin.py — standalone Gremlin executor
- graph_notebook/core/sparql.py — standalone SPARQL executor
- graph_notebook/core/nars.py — NEW: NARS executor
