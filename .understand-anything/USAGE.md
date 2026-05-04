# Using the Boulder knowledge graph

`knowledge-graph.json` is a generated map of Boulder. Use it as a navigation index: to locate code, follow imports, and see how files cluster into layers. Verify anything you plan to act on against the live Boulder checkout first. The graph is an index, not source of truth.

## What's inside

- Scope: Boulder at the commit recorded in [`meta.json`](./meta.json).
- Corpus: 327 files. Tests, `vendor/`, generated `*.pb.go`, and static data lists are excluded. [`.understandignore`](./.understandignore) has the exact filter.
- Graph: 1,620 nodes, 3,340 edges.
  - Nodes by type: 253 `file`, 246 `class`, 1,005 `function`, 42 `table` (SQL DDL), 29 `config`, 22 `document`, 12 `schema`, 8 `pipeline`, 2 `service`.
  - Edges by type: 1,293 `contains`, 1,200 `imports`, 732 `exports`. Smaller counts of `documents`, `defines_schema`, `triggers`, `related`, `depends_on`.
- 10 layers: `layer:web-frontend`, `layer:orchestration`, `layer:validation`, `layer:issuance`, `layer:storage`, `layer:publication`, `layer:services`, `layer:entrypoints`, `layer:shared`, `layer:infrastructure-ops`.
- A 12-step tour that walks the ACME issuance lifecycle starting at `cmd/boulder/main.go`.

## Node ID conventions

- `file:<path>`
- `function:<path>:<name>`
- `class:<path>:<name>`
- `config:<path>`, `document:<path>`, `service:<path>`, `pipeline:<path>`, `schema:<path>`
- `table:<sql-path>:<table-name>`

## Queries

```bash
GRAPH=.understand-anything/knowledge-graph.json

# Which layer owns a file?
jq -r --arg id "file:ra/ra.go" \
  '.layers[] | select(.nodeIds | index($id)) | .name' $GRAPH

# Files in one layer
jq -r '.layers[] | select(.id == "layer:validation") | .nodeIds[]' $GRAPH

# What does ra/ra.go import?
jq -r --arg id "file:ra/ra.go" \
  '.edges[] | select(.source == $id and .type == "imports") | .target' $GRAPH

# Fan-in: who imports sa/sa.go?
jq -r --arg id "file:sa/sa.go" \
  '.edges[] | select(.target == $id and .type == "imports") | .source' $GRAPH

# Functions in a file, with summaries
jq -r --arg fp "wfe2/wfe.go" \
  '.nodes[] | select(.type == "function" and .filePath == $fp) | "\(.name) — \(.summary)"' $GRAPH

# ACME endpoint handlers
jq -r '.nodes[] | select(.type == "function" and (.tags | index("api-handler"))) | .id' $GRAPH

# Service entry points
jq -r '.nodes[] | select(.tags | index("entry-point")) | .filePath' $GRAPH

# Proto to Go implementation edges
jq '.edges[] | select(.type == "defines_schema")' $GRAPH

# Tour
jq '.tour[] | {order, title, description, nodeIds}' $GRAPH
```

## Where the graph helps

Use it to answer questions like: which layer does this file sit in, what imports it, and what does it import. That's enough for most "where should I look first?" decisions.

It's useful for scoping refactors (pull all files in a layer, or all files that import a package), for locating `main()` entry points via the `entry-point` tag, and for citing layers or tour steps when explaining Boulder's architecture.

## Where it doesn't

Summaries and tags were written by LLM subagents. The structural data — node IDs, imports, line ranges — comes from tree-sitter and is reliable. The prose around it is best effort. For issuance logic, validation semantics, or anything touching crypto, read the Go source.

The graph is a snapshot at `meta.gitCommitHash`. If Boulder has moved on, it's stale. `git log -- <path>` will tell you how much has changed.

Tests, mocks, generated gRPC stubs, and the integration harness aren't in the graph. That's a scope decision, not a claim that they don't exist.

`iana/data/` (static domain lists) is also excluded. The Go code that reads those files is present.

## When the graph is wrong

Compare `meta.json` to Boulder HEAD. If they diverge, regenerate. Don't patch the JSON by hand — it's generated output and your edits will be overwritten next run. [`PROVENANCE.md`](./PROVENANCE.md) has the regenerate steps.

## Visualize

Run `/understand-dashboard` from this repo root. Requires the understand-anything plugin installed at `~/.understand-anything-plugin`. It starts a local Vite server and prints a tokenized URL; open that URL in a browser.
