# realm-mcp-memory

The [MCP memory server](https://github.com/modelcontextprotocol/servers/tree/main/src/memory)'s
knowledge graph under the shared `Memory` label. The reference `@modelcontextprotocol/server-memory`
is the memory most MCP clients are wired to — Claude Desktop, Cursor, Windsurf, anything that
speaks the protocol — and it keeps everything it knows in one JSON-lines file: an entity per line,
with its type and the observations recorded about it, and a relation per line. This realm reads
that file live, one memory per entity, so one Virtual Cypher query answers from the graph an MCP
client has been building alongside the assistant's own memory and every other system.

```cypher
MATCH (:Concept {value:'coffee'})-[r:RELEVANT_TO]->(m:Memory)      -- every memory system
RETURN m.source, m.text, r.score, r.mode ORDER BY m.source

MATCH (:Concept {value:'coffee'})-[r:RELEVANT_TO]->(m:McpMemory)   -- the MCP graph alone
RETURN m.name, m.entityType, m.observations, m.file
```

Every node carries `:Memory`, `:McpMemory` and the realm label `:Mcp`.

## The file

The server writes `memory.jsonl` next to itself, or wherever `MEMORY_FILE_PATH` points:

```jsonl
{"type":"entity","name":"Rod_Johnson","entityType":"person","observations":["Drinks a flat white every morning at Single O"]}
{"type":"relation","from":"Rod_Johnson","to":"Single_O","relationType":"regular_at"}
```

Put that file, or a symlink to it, under the world's `data/memory/mcp/`. The files walk is
confined to the world's `data/` sandbox, so a symlink is followed only when its target is declared
in the world's `world.yml`:

```yaml
filesTrustedRoots:
  - /Users/me/.local/share/mcp/memory.jsonl
```

Each entity line is one memory (`unit: line` on the `files` producer; `require: {type: entity}`
leaves the relation lines out). Nothing is ingested or indexed; the next query reads what the
server last wrote.

## The contract

| Memory property | From the entity |
|---|---|
| `id` | `name` (unique in the MCP graph) |
| `text` | `name (entityType): observation. observation. …` |
| `mentions` | `[name]` |
| `status` | `ACTIVE` — an entity in the file is a current fact |
| `created` / `revised` | the file's own times |
| `name`, `entityType`, `observations` | the entity's own fields, as the server keeps them |

Retrieval is lexical on the entity's own line — name, type and observations — which is what the
server's `search_nodes` tool matches on, so a memory this realm finds is one the server would
find. `via:'agentic-rag'` expands the topic into probe terms first.

## Not yet

Relation lines are not edges between memories. A `(:McpMemory)-[:RELATED_TO]->(:McpMemory)` join
over the relation lines is the natural next step.
