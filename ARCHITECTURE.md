# GitNexus Architecture

A deep-dive into how GitNexus indexes codebases into a knowledge graph, identifies connections between symbols, and delivers architectural intelligence to AI agents.

---

## Table of Contents

- [Overview](#overview)
- [High-Level Architecture](#high-level-architecture)
- [How Repository Parsing Works](#how-repository-parsing-works)
  - [File Discovery (Parallel I/O)](#1-file-discovery-parallel-io)
  - [Structure Mapping](#2-structure-mapping)
  - [AST Parsing with Tree-sitter](#3-ast-parsing-with-tree-sitter)
  - [Import Resolution](#4-import-resolution)
  - [Call Graph Construction](#5-call-graph-construction)
  - [Class Inheritance Detection](#6-class-inheritance-detection)
  - [Community Detection (Leiden Algorithm)](#7-community-detection-leiden-algorithm)
  - [Process Detection (Execution Flow Tracing)](#8-process-detection-execution-flow-tracing)
  - [Search Index Construction](#9-search-index-construction)
- [Why It's Fast](#why-its-fast)
- [The Knowledge Graph Data Model](#the-knowledge-graph-data-model)
  - [Node Types](#node-types)
  - [Relationship Types](#relationship-types)
  - [Confidence Scoring](#confidence-scoring)
- [Storage and Persistence](#storage-and-persistence)
- [How AI Agents Query the Graph](#how-ai-agents-query-the-graph)
- [Web UI Architecture](#web-ui-architecture)
- [Supported Languages](#supported-languages)

---

## Overview

GitNexus transforms a raw codebase into a **queryable knowledge graph**. Every function, class, method, import, call chain, inheritance relationship, and execution flow is extracted, scored, and stored in a graph database. AI agents (Cursor, Claude Code, Windsurf, etc.) then query this graph through MCP tools to get deep architectural understanding of any codebase—without reading every file.

The core insight: **precompute structure at index time, not query time**. By clustering code into communities, tracing execution flows, and scoring relationships with confidence values during indexing, tools can return complete, reliable context in a single call instead of requiring multi-query exploration chains.

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    AI Agents / Web UI                             │
│         (Cursor, Claude Code, Windsurf, Browser)                 │
│                           │                                      │
│                    MCP Tools / HTTP API                           │
│         (query, context, impact, rename, cypher, ...)            │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│                     Query Layer                                   │
│                                                                   │
│   ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐     │
│   │ Hybrid Search│  │ Cypher Engine│  │  Impact / Context  │     │
│   │ (BM25 + RRF)│  │   (KuzuDB)  │  │    Analyzers       │     │
│   └─────────────┘  └──────────────┘  └────────────────────┘     │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│                   KuzuDB Graph Database                           │
│                                                                   │
│   Nodes: File, Function, Class, Method, Interface, Community,     │
│          Process, Enum, Variable, ...                             │
│   Edges: CALLS, IMPORTS, DEFINES, EXTENDS, IMPLEMENTS,           │
│          MEMBER_OF, STEP_IN_PROCESS, CONTAINS, ...               │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│                  9-Stage Indexing Pipeline                         │
│                                                                   │
│   1. File Discovery ──► 2. Structure ──► 3. AST Parsing          │
│   4. Import Resolution ──► 5. Call Detection ──► 6. Inheritance  │
│   7. Community Detection ──► 8. Process Tracing ──► 9. Search    │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│                     Local Repository                              │
│              (cloned Git repo on disk)                            │
└──────────────────────────────────────────────────────────────────┘
```

---

## How Repository Parsing Works

When you run `npx gitnexus analyze`, the pipeline processes a local repository through **9 sequential stages**. Each stage builds on the output of previous stages to construct a complete knowledge graph.

The entry point is `runPipelineFromRepo()` in `gitnexus/src/core/ingestion/pipeline.ts`.

### 1. File Discovery (Parallel I/O)

**Source:** `gitnexus/src/core/ingestion/filesystem-walker.ts`

The pipeline starts by scanning the repository's file tree using glob pattern matching:

```
walkRepository(repoPath)
  └─► glob('**/*') to find all files
  └─► Filter out ignored paths (node_modules, .git, build artifacts, etc.)
  └─► Read file contents in parallel batches of 32 concurrent reads
  └─► Skip files larger than 512KB (usually generated/vendored code)
  └─► Return: Array of { path, content } pairs
```

**Key design choices:**
- Batch I/O (`READ_CONCURRENCY = 32`): reads 32 files simultaneously using `Promise.allSettled`, maximizing disk throughput
- Smart filtering: skips binary files, vendored code, and build artifacts before reading content
- Size limits: files over 512KB are skipped to avoid crashing the parser on generated code

### 2. Structure Mapping

**Source:** `gitnexus/src/core/ingestion/structure-processor.ts`

Builds the folder/file hierarchy as graph nodes:

```
For each file path (e.g., "src/auth/validate.ts"):
  ├─► Create Folder node: "src"
  ├─► Create Folder node: "src/auth"
  ├─► Create File node: "src/auth/validate.ts"
  └─► Create CONTAINS edges: src ──CONTAINS──► src/auth ──CONTAINS──► validate.ts
```

Every path segment becomes a `Folder` or `File` node, connected by `CONTAINS` relationships. This gives agents structural context—they can ask "what's in the auth directory?" and get a complete answer.

### 3. AST Parsing with Tree-sitter

**Source:** `gitnexus/src/core/ingestion/parsing-processor.ts`

This is the most computationally intensive stage. Each source file is parsed into an Abstract Syntax Tree (AST) using [Tree-sitter](https://tree-sitter.github.io/), and symbols are extracted:

```
For each source file:
  ├─► Detect language from file extension
  ├─► Parse file content into AST using Tree-sitter
  ├─► Run language-specific Tree-sitter queries to extract:
  │     ├─► Function declarations → Function nodes
  │     ├─► Class declarations → Class nodes
  │     ├─► Method definitions → Method nodes
  │     ├─► Interface declarations → Interface nodes
  │     ├─► Enum declarations → Enum nodes
  │     └─► Variable declarations → Variable nodes
  ├─► Create DEFINES edges: File ──DEFINES──► Function/Class/Method
  ├─► Register each symbol in the SymbolTable
  └─► Cache the AST for reuse in later stages
```

**Worker Pool Parallelism:** Parsing is distributed across multiple CPU cores using a worker thread pool:

```
Main Thread                    Worker 1          Worker 2          Worker 3
    │                              │                 │                 │
    ├──► Split files into chunks ──┤                 │                 │
    │                              ├── Parse files   │                 │
    │                              ├── Extract calls │                 │
    │                              ├── Extract imports                 │
    │                              │                 ├── Parse files   │
    │                              │                 ├── Extract calls │
    │                              │                 │                 ├── Parse files
    │                              │                 │                 ├── Extract calls
    ◄── Merge results ────────────◄┘                ◄┘                ◄┘
```

The worker pool uses `N-1` workers (where N = CPU cores) to leave one core free for the main thread. Each worker receives a chunk of files and returns extracted symbols, calls, imports, and heritage data—all in a single pass. This avoids re-parsing in later stages.

**Symbol Table:** A dual-index data structure for O(1) symbol lookups:

```
File-Specific Index (high confidence):
  "src/auth/validate.ts" → { "validateUser" → "Function:src/auth/validate.ts:validateUser" }
  "src/auth/session.ts"  → { "createSession" → "Function:src/auth/session.ts:createSession" }

Global Reverse Index (fuzzy fallback):
  "validateUser" → [{ nodeId: "Function:...", filePath: "src/auth/validate.ts" }]
  "createSession" → [{ nodeId: "Function:...", filePath: "src/auth/session.ts" }]
```

**AST Cache:** An LRU cache sized to fit all files in the repository. ASTs parsed during this stage are reused in the import, call, and heritage stages, avoiding redundant parsing.

### 4. Import Resolution

**Source:** `gitnexus/src/core/ingestion/import-processor.ts`

Resolves module imports across files to build the dependency graph:

```
For each source file:
  ├─► Extract import statements using language-specific AST queries:
  │     TypeScript:  import { foo } from './bar'
  │     Python:      from auth.validate import check_password
  │     Go:          import "github.com/user/repo/pkg"
  │     Java:        import com.example.service.UserService
  │     C/C++:       #include "auth/validate.h"
  │
  ├─► Resolve import paths to actual files:
  │     ├─► Apply TypeScript path aliases (from tsconfig.json)
  │     ├─► Apply Go module paths (from go.mod)
  │     ├─► Try file extensions: .ts, .tsx, .js, .jsx, /index.ts, etc.
  │     └─► Handle relative vs absolute imports
  │
  └─► Create IMPORTS edge: File A ──IMPORTS──► File B
       └─► Store in ImportMap: Map<filePath, Set<importedFilePaths>>
```

**Path alias resolution:** The processor reads `tsconfig.json` to resolve aliases like `@/components/Button` → `src/components/Button`. For Go, it reads `go.mod` to resolve module-prefixed imports.

**ImportMap:** A `Map<string, Set<string>>` that records which files import from which other files. This is used by the call processor in the next stage to resolve cross-file function calls with high confidence.

### 5. Call Graph Construction

**Source:** `gitnexus/src/core/ingestion/call-processor.ts`

This stage traces function calls across the entire codebase. For every function call expression found in the AST, it determines: **who is calling** and **what is being called**.

```
For each call expression in the AST:
  ├─► Extract the called function name (e.g., "validateUser")
  ├─► Filter out built-in/noise functions (console.log, Array.map, etc.)
  │
  ├─► Find the CALLER (walk up the AST to the enclosing function):
  │     validateUser() called inside handleLogin()
  │     └─► Caller = "Function:src/api/auth.ts:handleLogin"
  │
  ├─► Resolve the TARGET using priority strategy:
  │     Strategy B (cheapest): Same-file lookup via SymbolTable
  │       └─► confidence: 0.85, reason: "same-file"
  │     Strategy A (import-aware): Check if target is in an imported file
  │       └─► confidence: 0.9, reason: "import-resolved"
  │     Strategy C (fuzzy fallback): Global symbol search
  │       └─► confidence: 0.3-0.5, reason: "fuzzy-global"
  │
  └─► Create CALLS edge: handleLogin ──CALLS(0.9)──► validateUser
```

**Resolution priority:**
1. **Same-file** (cheapest—single map lookup): if `validateUser` is defined in the same file, it resolves immediately with 0.85 confidence
2. **Import-resolved** (cross-file): if `validateUser` is defined in a file that the current file imports, it resolves with 0.9 confidence
3. **Fuzzy-global** (last resort): search all definitions globally by name. If there's exactly one match, 0.5 confidence; if multiple, 0.3 confidence

**Built-in filtering:** Common standard library functions (`console.log`, `map`, `filter`, `useState`, `print`, `len`, etc.) are excluded from call tracking to reduce noise.

### 6. Class Inheritance Detection

**Source:** `gitnexus/src/core/ingestion/heritage-processor.ts`

Extracts class hierarchies across all supported languages:

```
For each class/interface declaration:
  ├─► TypeScript: class UserService extends BaseService implements Cacheable
  ├─► Python:     class UserService(BaseService)
  ├─► Java:       class UserService extends BaseService implements Serializable
  ├─► Go:         (interface embedding and struct composition)
  ├─► Rust:       impl Trait for Struct
  │
  └─► Create edges:
        UserService ──EXTENDS──► BaseService      (confidence: 1.0)
        UserService ──IMPLEMENTS──► Cacheable      (confidence: 1.0)
```

The heritage processor uses the SymbolTable to resolve parent class/interface names to their graph node IDs, even when they're defined in different files.

### 7. Community Detection (Leiden Algorithm)

**Source:** `gitnexus/src/core/ingestion/community-processor.ts`

After all relationships are built, the Leiden algorithm clusters symbols into **functional communities**—groups of code that work together frequently:

```
Step 1: Build an undirected graph from CALLS + EXTENDS + IMPLEMENTS edges
        (only symbols with at least one connection are included)

Step 2: Run Leiden algorithm (modularity optimization)
        ├─► Resolution: 1.0 (default—adjustable)
        ├─► Random walk: enabled
        └─► Finds groups where internal connections are denser than external

Step 3: Generate heuristic labels for each community
        ├─► Most common parent folder name → "Authentication", "Database"
        ├─► Common function name prefix → "User", "Payment"
        └─► Fallback: "Cluster_N"

Step 4: Calculate cohesion score (0-1) per community
        └─► internalEdges / totalEdges (sampled for large communities)

Step 5: Create Community nodes + MEMBER_OF edges
        ├─► Community node: { name, heuristicLabel, cohesion, symbolCount }
        └─► MEMBER_OF edge: Function ──MEMBER_OF──► Community
```

**What communities reveal:** Instead of navigating by folder structure, agents can think in terms of functional areas: "the Authentication cluster", "the Database layer", "the API routing group". This is especially powerful for repos where functional concerns span multiple directories.

**Singleton filtering:** Communities with only 1 member are skipped—they're just isolated nodes with no meaningful cluster.

### 8. Process Detection (Execution Flow Tracing)

**Source:** `gitnexus/src/core/ingestion/process-processor.ts`

Traces **execution flows** through the codebase by finding entry points and following call chains:

```
Step 1: Score all functions as potential entry points
        ├─► Call ratio: callees / (callers + 1) — higher = more likely entry
        ├─► Export status: exported functions rank higher
        ├─► Name patterns: handleLogin, onClick, processPayment, main
        ├─► Framework detection: Express routes, Next.js pages, Django views
        └─► Test exclusion: test files are deprioritized

Step 2: BFS from top-scored entry points via CALLS edges
        ├─► Max depth: 10 hops
        ├─► Max branching: 4 callees per node (take highest-confidence)
        └─► Record the trace: [entryPoint, step1, step2, ..., terminal]

Step 3: Classify each process
        ├─► intra_community: all steps in the same community
        └─► cross_community: steps touch 2+ communities (architectural flow)

Step 4: Deduplicate and filter
        ├─► Minimum 3 steps (2-step is just "A calls B"—trivial)
        └─► Remove duplicate traces (same nodes in same order)

Step 5: Create Process nodes + STEP_IN_PROCESS edges
        ├─► Process: { name, heuristicLabel, processType, stepCount }
        └─► STEP_IN_PROCESS: Function ──STEP(1)──► Process
                             Function ──STEP(2)──► Process
                             ...
```

**Dynamic sizing:** The maximum number of processes scales with codebase size: `max(20, min(300, symbolCount / 10))`. Small repos get at least 20 processes; large repos cap at 300.

**Entry point scoring** (`gitnexus/src/core/ingestion/entry-point-scoring.ts`) uses language-specific heuristics:
- Universal: `main`, `init`, `handleX`, `onX`, `XController`, `processX`
- Python: `app`, `get_*`, `post_*`, `view_*`
- Java: `doGet`, `doPost`, `createX`, `buildX`
- JavaScript/TypeScript: `useX` (React hooks)

### 9. Search Index Construction

**Source:** `gitnexus/src/core/search/` and `gitnexus/src/core/embeddings/`

Two search indexes are built for fast retrieval:

**BM25 (Full-Text Search):** A keyword-based index built directly in KuzuDB using its FTS extension. Indexes symbol names, file paths, and descriptions.

**Semantic Embeddings (Optional):** Uses `transformers.js` to generate vector embeddings for each symbol's textual description. Enables similarity-based search ("find functions related to authentication").

**Hybrid Search (Reciprocal Rank Fusion):** At query time, results from BM25 and semantic search are merged using RRF:

```
For each result appearing in either ranking:
  RRF_score = 1/(K + bm25_rank + 1) + 1/(K + semantic_rank + 1)
  where K = 60 (standard constant)

Final results are sorted by RRF_score (highest first)
```

This hybrid approach catches both exact keyword matches and semantically related results.

---

## Why It's Fast

GitNexus achieves high-speed indexing through several architectural choices:

| Technique | What It Does | Impact |
|-----------|-------------|--------|
| **Worker Thread Pool** | Distributes AST parsing across N-1 CPU cores | 3-8x faster parsing on multi-core machines |
| **Single-Pass Workers** | Workers extract symbols, calls, imports, and heritage in one pass per file | Avoids 3 additional full-repo passes |
| **AST Cache (LRU)** | Caches parsed ASTs sized to fit all files; reused across import/call/heritage stages | Eliminates redundant parsing |
| **Batch File I/O** | Reads 32 files concurrently using `Promise.allSettled` | Maximizes disk throughput |
| **Bulk CSV Load** | Generates CSV files, then uses KuzuDB's `COPY FROM` for bulk insert | 100x faster than row-by-row INSERT |
| **Symbol Table (Dual Index)** | O(1) lookup for symbol resolution (file-specific + global) | Fast call target resolution |
| **Smart Filtering** | Skips binary, vendored, >512KB files; filters 200+ built-in function names | Less noise, less work |
| **Yield-to-Event-Loop** | Yields every 20 files to keep Node.js responsive | Progress reporting without blocking |
| **Singleton Pruning** | Communities with 1 member are skipped; processes with <3 steps are pruned | Reduces graph size |

**Typical performance:** A 1,000-file TypeScript project indexes in ~10-20 seconds on a modern machine.

---

## The Knowledge Graph Data Model

### Node Types

| Node Type | What It Represents | Key Properties |
|-----------|--------------------|----------------|
| `File` | A source file | name, filePath |
| `Folder` | A directory | name, filePath |
| `Function` | A function/procedure | name, filePath, startLine, endLine, isExported |
| `Class` | A class definition | name, filePath, startLine, endLine, isExported |
| `Method` | A method inside a class | name, filePath, startLine, endLine |
| `Interface` | An interface/protocol | name, filePath, startLine, endLine |
| `Enum` | An enum type | name, filePath |
| `Variable` | A significant variable/constant | name, filePath |
| `Community` | A functional cluster of symbols | heuristicLabel, cohesion, symbolCount |
| `Process` | An execution flow trace | heuristicLabel, processType, stepCount, communities |

### Relationship Types

| Relationship | Meaning | Example |
|-------------|---------|---------|
| `CONTAINS` | Folder/file hierarchy | `src/` ──► `src/auth/` ──► `validate.ts` |
| `DEFINES` | File defines a symbol | `validate.ts` ──► `validateUser()` |
| `IMPORTS` | File imports from another | `auth.ts` ──► `validate.ts` |
| `CALLS` | Function calls another | `handleLogin()` ──► `validateUser()` |
| `EXTENDS` | Class inherits from another | `AdminUser` ──► `BaseUser` |
| `IMPLEMENTS` | Class implements interface | `UserService` ──► `IUserService` |
| `MEMBER_OF` | Symbol belongs to community | `validateUser()` ──► `Authentication` cluster |
| `STEP_IN_PROCESS` | Symbol is step N in a process | `validateUser()` ──STEP(2)──► `LoginFlow` |
| `USES` | General dependency | Variable/type usage |
| `DECORATES` | Decorator applied to symbol | `@Controller` ──► `UserController` |
| `OVERRIDES` | Method overrides parent | `AdminUser.validate()` ──► `BaseUser.validate()` |

### Confidence Scoring

Every relationship carries a **confidence score** (0–1) and a **reason string** so AI agents can gauge reliability:

| Confidence | Reason | Meaning |
|-----------|--------|---------|
| 1.0 | `""` (structure) | File hierarchy, inheritance—always correct |
| 0.9 | `"import-resolved"` | Target found in an imported file—high confidence |
| 0.85 | `"same-file"` | Target found in the same file—high confidence |
| 0.5 | `"fuzzy-global"` | Only one definition found globally—moderate |
| 0.3 | `"fuzzy-global"` | Multiple definitions found globally—low confidence |
| 1.0 | `"leiden-algorithm"` | Community membership—algorithmic assignment |
| 1.0 | `"trace-detection"` | Process step—deterministic BFS trace |

Agents can filter by `minConfidence` to exclude uncertain connections (e.g., `impact({minConfidence: 0.8})`).

---

## Storage and Persistence

**Graph Database:** [KuzuDB](https://kuzudb.com/), an embedded graph database supporting Cypher queries, full-text search, and vector indexes.

**On-Disk Layout:**

```
your-repo/
├── .gitnexus/                    ← Index directory (gitignored)
│   ├── kuzu                      ← KuzuDB database file
│   ├── csv/                      ← Intermediate CSVs for bulk loading
│   │   ├── File.csv
│   │   ├── Function.csv
│   │   ├── Class.csv
│   │   ├── Community.csv
│   │   ├── Process.csv
│   │   └── CodeRelation.csv      ← All relationships in one file
│   ├── meta.json                 ← Staleness metadata (commit hash, timestamp)
│   └── embeddings.json           ← Cached vector embeddings
└── ...

~/.gitnexus/
└── registry.json                 ← Global pointer to all indexed repos
```

**Bulk Loading Pipeline:**

```
In-Memory Graph
      │
      ▼
generateAllCSVs()  ──►  CSV files (one per node type + one for all relationships)
      │
      ▼
COPY FROM (KuzuDB)  ──►  Graph database (persistent, queryable)
      │
      ▼
CREATE FTS INDEX  ──►  BM25 full-text search on symbol names
      │
      ▼
runEmbeddingPipeline()  ──►  Vector embeddings (optional, for semantic search)
```

**Multi-Repo Registry:** Each `gitnexus analyze` stores the index inside `.gitnexus/` in the repo (portable, gitignored) and registers a pointer in `~/.gitnexus/registry.json`. The MCP server reads this registry to serve any indexed repo without per-project configuration.

---

## How AI Agents Query the Graph

GitNexus exposes **7 MCP tools** that AI agents call to get architectural intelligence:

| Tool | What It Returns | Example |
|------|----------------|---------|
| `list_repos` | All indexed repositories | Discover available repos |
| `query` | Hybrid search results grouped by execution process | "Find code related to authentication" |
| `context` | 360° view of a symbol: callers, callees, imports, processes | "What depends on UserService?" |
| `impact` | Blast radius at depth 1/2/3 with confidence scores | "What breaks if I change validateUser?" |
| `detect_changes` | Git-diff mapped to affected symbols and processes | "What did my last commit affect?" |
| `rename` | Multi-file rename plan with graph + text search | "Rename validateUser to verifyUser" |
| `cypher` | Raw Cypher queries against the graph | Custom graph exploration |

**Example: `impact` tool response:**

```
TARGET: Function validateUser (src/auth/validate.ts:15)

UPSTREAM (what depends on this):
  Depth 1 (WILL BREAK):
    handleLogin      [CALLS 90%] → src/api/auth.ts:45
    handleRegister   [CALLS 90%] → src/api/auth.ts:78
    UserController   [CALLS 85%] → src/controllers/user.ts:12
  Depth 2 (LIKELY AFFECTED):
    authRouter       [IMPORTS]   → src/routes/auth.ts
```

Each tool response includes "next step hints" that guide the agent toward the most useful follow-up query.

---

## Web UI Architecture

The web UI (`gitnexus-web/`) runs the **same indexing pipeline** entirely in the browser using WebAssembly:

| Component | CLI (Native) | Web (WASM) |
|-----------|-------------|------------|
| Tree-sitter | Native bindings | web-tree-sitter (WASM) |
| KuzuDB | Native embedded DB | kuzu-wasm (in-memory) |
| Embeddings | transformers.js (CPU/GPU) | transformers.js (WebGPU/WASM) |
| Workers | Node.js worker_threads | Web Workers + Comlink |
| Visualization | — | Sigma.js (WebGL graph rendering) |
| AI Chat | — | LangChain ReAct agent |

**Bridge Mode:** `gitnexus serve` starts a local HTTP server. The web UI auto-detects it and can browse all CLI-indexed repos without re-uploading or re-indexing.

---

## Supported Languages

GitNexus provides full AST parsing, import resolution, call detection, and inheritance tracking for:

| Language | File Extensions | Import Style | Special Features |
|----------|----------------|-------------|-----------------|
| TypeScript | `.ts`, `.tsx` | ES modules, require | Path alias resolution via tsconfig.json |
| JavaScript | `.js`, `.jsx`, `.mjs` | ES modules, require, dynamic imports | — |
| Python | `.py` | `import`, `from X import Y` | Relative imports |
| Java | `.java` | `import com.example.X` | Package-based resolution |
| Go | `.go` | `import "pkg/path"` | go.mod module path resolution |
| C | `.c`, `.h` | `#include "header.h"` | — |
| C++ | `.cpp`, `.hpp`, `.cc` | `#include`, `using namespace` | — |
| C# | `.cs` | `using Namespace` | — |
| Rust | `.rs` | `use crate::module` | Trait implementations |

Each language has dedicated Tree-sitter queries defined in `gitnexus/src/core/ingestion/tree-sitter-queries.ts` that extract the language-specific syntax for functions, classes, imports, calls, and inheritance.
