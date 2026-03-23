# graph-notebook — Transcode to Rust

Read the Python source in this repo. Read `.claude/SCOPE_B_FINDINGS.md`.
Transcode graph-notebook's query executors to a standalone Rust crate.

Gremlin over WebSocket. Cypher over Bolt. SPARQL over HTTP.
vis.js graph rendering.

The crate must work on its own. No dependency on marimo,
kernel-protocol, quarto, lance-graph, ndarray, or rs-graph-llm.

Output: a Rust crate in this repo that executes graph queries.
Someone adds it to their Cargo.toml, they can query Neo4j from Rust.

Read first. Transcode. Test.
