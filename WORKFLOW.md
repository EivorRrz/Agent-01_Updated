# Agent-01: Technical Workflow

**Document → Neo4j Graph.** File storage + Agentic AI (no MongoDB).

---

## Workflow Overview

```
User Document (PDF/DOCX/TXT)
        │
        ▼
┌─────────────────────────────────────────────────────────────────────┐
│  1. UPLOAD     →  2. PARSE   →  3. SCHEMA   →  4. CYPHER  →  5. INGEST  │
│  (File)          (LlamaParse/   (Azure       (Azure        (Neo4j)   │
│                   local)         OpenAI)      OpenAI)                  │
└─────────────────────────────────────────────────────────────────────┘
        │
        ▼
   Neo4j Graph (nodes + relationships)

Storage: ./data/documents/{docId}/ (meta.json, fullText.txt, schema.json, cypher-results.json)
```

---

## Step 1: Upload

**Trigger:** User uploads file via `POST /documents` (multipart/form-data)

| What happens | Where |
|--------------|-------|
| Multer receives file | `routes/documents.js` |
| File saved to `./uploads/temp/{timestamp}-{random}.{ext}` | Disk |
| Document record created in MongoDB | `models/Document.js` |
| Status set to `uploaded` | MongoDB |

**Output:** `docId` returned to client.

---

## Step 2: Process Start

**Trigger:** Client calls `POST /documents/:docId/process`

| What happens | Where |
|--------------|-------|
| `runPipeline(docId)` invoked | `services/orchestrator.js` |
| Document status → `parsing` | MongoDB |
| Pipeline runs asynchronously (202 response) | — |

---

## Step 3: Parse

**Service:** `services/parsing/index.js` → `parseDocument(docId)`

| What happens | Where |
|--------------|-------|
| Read file from `Document.filePath` | — |
| If `LLAMAPARSE_API_KEY` set → LlamaParse API | Cloud |
| Else → Local: `pdf-parse` (PDF), `mammoth` (DOCX), `fs` (TXT) | Local |
| Extract plain text | — |
| Save `fullText` to Document | MongoDB |
| Status → `parsed` | MongoDB |

**Output:** `Document.fullText` = full document text.

---

## Step 4: Chunk (Optional)

**Service:** `utils/chunking.js` → `chunkText(fullText)`

| What happens | Where |
|--------------|-------|
| Only runs if `useFullDocument: false` | `orchestrator.js` |
| Split by word count (default 1000 words) | — |
| Overlap between chunks (default 100 words) | — |
| Create DocumentChunk records | MongoDB |

**Default:** `useFullDocument: true` → **no chunking**, full document used as one unit.

---

## Step 5: Schema Extraction

**Service:** `services/schemaExtraction/index.js` → `extractSchema(docId)`

| What happens | Where |
|--------------|-------|
| Load full document text | MongoDB |
| Detect document type (business/financial/technical) | `utils/documentTypeDetector.js` |
| Build prompt with schema instructions + document text | — |
| Call Azure OpenAI | `utils/llm.js` → `callLLM()` |
| Parse JSON: `{ nodes, relationships }` | — |
| Validate and normalize | — |
| Save Schema record | MongoDB |

**Output:** Schema with `nodes` (labels + properties) and `relationships` (type, from, to).

---

## Step 6: Neo4j Constraints (Optional)

**Service:** `services/neo4jIngest/index.js` → `createConstraints(schema)`

| What happens | Where |
|--------------|-------|
| For each node label, create uniqueness constraint | Neo4j |
| Example: `CREATE CONSTRAINT ... FOR (n:Account) REQUIRE n.accountId IS UNIQUE` | — |

---

## Step 7: Cypher Generation

**Service:** `services/cypherGeneration/index.js`

| What happens | Where |
|--------------|-------|
| Load Schema + full text (or chunk text) | MongoDB |
| Build prompt: schema + text + MERGE rules | — |
| Call Azure OpenAI | `utils/llm.js` |
| Extract Cypher from response (strip markdown) | `utils/llm.js` → `extractCypher()` |
| Save ChunkCypherResult | MongoDB |
| Save Cypher to `.cypher` file | `utils/saveCypher.js` |

**Output:** MERGE statements for nodes and relationships.

---

## Step 8: Neo4j Ingestion

**Service:** `services/neo4jIngest/index.js` → `ingestAllChunks(docId)`

| What happens | Where |
|--------------|-------|
| Load ChunkCypherResult(s) | MongoDB |
| Validate Cypher with EXPLAIN | Neo4j |
| Split by `;` into statements | — |
| Execute schema statements first | Neo4j |
| Execute write statements in transaction | Neo4j |
| Update ChunkCypherResult status → `executed` | MongoDB |
| Update Document status → `completed` | MongoDB |

**Output:** Graph data in Neo4j.

---

## Step 9: Query (Optional)

**Trigger:** User sends `POST /query` with `{ docId, question }`

| What happens | Where |
|--------------|-------|
| Load Schema for docId | MongoDB |
| Build prompt: schema + question | `routes/query.js` |
| Call Azure OpenAI → read-only Cypher (MATCH/RETURN) | — |
| Execute Cypher in Neo4j | Neo4j |
| Return results | Client |

---

## Data Flow Summary

| Step | Input | Output | Storage |
|------|-------|--------|---------|
| 1 | File | docId | MongoDB (Document) |
| 2 | docId | — | — |
| 3 | File path | fullText | `data/documents/{docId}/fullText.txt` |
| 4 | fullText | chunks | — (full-doc mode) |
| 5 | fullText | schema | `data/documents/{docId}/schema.json` |
| 6 | schema | constraints | Neo4j |
| 7 | schema + text | Cypher | `data/documents/{docId}/cypher-results.json` |
| 8 | Cypher | graph | Neo4j |
| 9 | question + docId | results | — |

---

## File Watcher (Drop-in)

```bash
npm run watch
```

- **Watch folder:** `./watch` (or `WATCH_FOLDER`)
- **Mode:** Agentic by default (`WATCH_USE_AGENT=true`)
- **Flow:** Drop PDF/DOCX/TXT → auto process → move to `watch/processed/` or `watch/error/`
- **Set `WATCH_USE_AGENT=false`** for sequential pipeline mode

---

## File References

| Step | Primary file |
|------|--------------|
| 1 | `src/routes/documentsFile.js` |
| 2–8 | `src/pipeline/orchestrator.js` |
| 3 | `src/pipeline/parseDocument.js` |
| 5 | `src/pipeline/extractSchema.js` |
| 7 | `src/pipeline/generateCypher.js` |
| 8 | `src/pipeline/ingestToNeo4j.js` |
| 9 | `src/routes/queryFile.js` |
| Agent | `src/agent/documentAgent.js` |
| Watch | `src/scripts/file-watcher.js` |
