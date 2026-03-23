# Agent: Local Engine Path

## Mission
Add a local execution path for %%cypher that uses lance-graph's semiring
planner instead of sending queries to a remote database.

## Scope
- Detect connection config: if target is "local" or "lance-graph", route locally
- Call lance-graph's Cypher→Semiring planner (PR #29, blasgraph CSC engine)
- Return results as DataFrame + graph visualization
- Zero network hop, SIMD-native execution

## Dependencies
- lance-graph must be available as a Python extension (PyO3) or subprocess
- OR: route through evcxr kernel cell that calls lance-graph directly
