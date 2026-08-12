---
name: graphify
description: Use when the user wants to turn a codebase in any language into a queryable knowledge graph — mentions "graphify", "knowledge graph of the codebase", "map this codebase", "graph the project", "trace connections between X and Y", or wants call-graph/architecture visualization without a vector store. Covers installing the `graphify` CLI (`pip install graphifyy && graphify install`), registering it as a Claude Code skill, running the first `/graphify .` pass on a project, tree-sitter language coverage, the cross-language call-graph limitation in polyglot/microservice repos, useful flags (--mode deep, --update, --watch, --wiki, export formats), and verifying the output before handing it over.
---

# Graphify: codebase → knowledge graph

[graphify](https://github.com/Graphify-Labs/graphify) parses a project with tree-sitter (deterministic
AST extraction, no LLM, no vector store, nothing leaves the machine) and produces a knowledge graph:
`graph.json`, an interactive `graph.html`, and a `GRAPH_REPORT.md` summary. Every edge is tagged
`EXTRACTED` (explicit in source, e.g. an import or function call) or `INFERRED` (resolved by graphify),
so the output states its own confidence rather than presenting guesses as fact.

**Cross-platform**: the CLI and tree-sitter parsing behave identically on Windows, macOS, and Linux.
Only the install command's fallback (pip vs. pipx) is platform-specific, noted below.

## When this is actually worth reaching for

A single-module question ("what does X depend on?", "what does this function call?") is usually
answered just as well, and more cheaply, by reading the file/manifest directly — no install, no
build-the-graph wait. Don't assume every dependency/relationship question means invoking this
skill; a generic ask without the tool named will often (correctly) fall back to plain code
reading instead, since that's the lighter-weight path. Reach for graphify specifically when the
question spans many files/modules at once (tracing a call chain across dozens of submodules,
finding every consumer of a symbol repo-wide) or needs a repeatable export (wiki, Neo4j, SVG) —
and if you want it invoked for a simpler question anyway, name the tool explicitly rather than
relying on the question alone to trigger it.

## Workflow: Spec → Plan → Init

1. **Spec** — confirm the target root(s) before running anything. For a polyglot repo (e.g. a
   JS/TS frontend alongside a backend in a different language), decide whether to graph each
   project separately or point graphify at a shared monorepo root — see **Cross-language
   limitation** below before assuming one graph will link them.
2. **Plan** — state the exact command about to run (`/graphify .` vs. a specific subfolder,
   `--mode deep` vs. default, whether `--watch` should stay running afterward) before executing.
3. **Init** — install, run the first pass, then verify the output (see Verification) before
   reporting the graph as ready.

## Install

```bash
pip install graphifyy && graphify install
```

- Requires Python 3.10+.
- The PyPI package is `graphifyy` (extra "y") but the CLI command is `graphify` — easy to
  mistype either direction.
- `graphify install` registers the skill into Claude Code automatically — that's the supported
  path. Don't manually `curl` a `SKILL.md` from a random fork/personal-account URL as a
  substitute; if graphify's own docs ever suggest that as a "manual alternative," prefer the
  built-in installer instead so you're not writing arbitrary fetched content into
  `~/.claude/CLAUDE.md`.

**Platform fallbacks if the plain `pip install` fails:**
- **Windows** — command not found after install: add Python's `Scripts` folder to `PATH`.
- **macOS** (`externally-managed-environment` error) — use `pipx install graphifyy` instead.
- **Linux**, same symptom — `pipx install graphifyy` also works if the system Python blocks
  global pip installs.

## First run

From the project root, in Claude Code:

```
/graphify .
```

Output lands in `graphify-out/`: `graph.html` (interactive, force-directed, open in a browser),
`graph.json` (the raw graph, used by `query`/`path`/exports), and `GRAPH_REPORT.md` (text
summary — god nodes, community breakdown, notable edges).

## Language coverage

Tree-sitter extraction covers 36+ languages, including `.py .ts .js .go .rs .java .c .cpp .rb .cs
.kt .scala .php` and more — check the target repo's actual file extensions against graphify's
supported list before a first run if unsure. Package/config manifests (e.g. `package.json`) are
also parsed as their own node type, not just the source files they reference.

## Cross-language limitation (read before graphing a polyglot repo)

The call-graph pass runs per-language. Pointing graphify at a root that mixes multiple languages
(e.g. a JS/TS frontend calling a backend written in a different language) produces one combined
graph file, but it will **not** auto-resolve edges that only exist through a network boundary —
e.g. a frontend `fetch('/api/orders')` call won't link to the backend controller method that
serves it, because that connection isn't visible in either file's AST. Expect well-formed
*per-language* subgraphs inside the one output, not a single call chain spanning languages. If the
whole point is tracing a request across services, say so up front (Spec step) so the plan accounts
for it rather than treating the first graph as already complete.

## Useful flags

```
/graphify ./path --mode deep     # more aggressive INFERRED edge extraction
/graphify ./path --update        # re-extract only changed files, merge into existing graph
/graphify ./path --watch         # auto-sync as files change (code: instant, docs: notifies)
/graphify ./path --wiki          # agent-crawlable wiki (index.md + article per community)
/graphify ./path --svg           # export graph.svg
/graphify ./path --graphml       # export graph.graphml (Gephi, yEd)
/graphify ./path --neo4j         # generate cypher.txt for Neo4j import
/graphify ./path --mcp           # start MCP stdio server over the graph
/graphify query "question"       # ask the knowledge base a question
/graphify path "NodeA" "NodeB"   # trace the connection between two named nodes
graphify hook install            # post-commit git hook, keeps the graph current automatically
```

For an existing graph, always prefer `--update` over re-running a full pass — it only
re-extracts changed files instead of rebuilding everything.

**Confirmed gotcha**: `GRAPH_REPORT.md`'s "Built from commit" freshness line reads
`git rev-parse HEAD` from the *process's current working directory*, not from the folder being
graphed. If graphify is invoked while the shell's cwd is a different repo than the target path
(e.g. running it from a plugin/tooling repo but pointing it at a project checked out elsewhere),
that line silently reports the wrong repo's commit — the graph content itself is unaffected, only
that one staleness stamp is misleading. Don't trust it to answer "is this graph stale" in that
setup; `cd` into the target repo before running graphify, or just re-run `--update` on a schedule
instead of relying on the commit-hash comparison.

## Verification

Don't report the graph as ready off a zero exit code alone — open `GRAPH_REPORT.md` and check:
- Node/edge counts are non-trivial for the project's actual size (a suspiciously low count
  usually means most files were skipped — check the extension list matched what's really in the
  repo, e.g. `.tsx`/`.jsx` variants if the frontend uses JSX).
- The god-node and community list roughly matches what you already know about the project's
  architecture — if it doesn't, something parsed wrong before trusting the graph for real
  analysis.
- For a polyglot repo, confirm each language actually appears as nodes (spot-check one file per
  language by name in `graph.json` or the report) rather than assuming everything parsed
  correctly just because the command didn't error.
