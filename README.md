# Document-to-Graph Pipeline

An end-to-end system that parses business documents, extracts graph schemas, and generates Neo4j Cypher queries automatically.

## 📋 Documentation

- **[SENIOR_MANAGER_BRIEF.md](./SENIOR_MANAGER_BRIEF.md)** - ⭐ Simple executive brief for senior managers
- **[PRESENTATION.md](./PRESENTATION.md)** - Complete presentation document for managers
- **[QUICK_REFERENCE.md](./QUICK_REFERENCE.md)** - Quick reference guide
- **[ARCHITECTURE.md](./ARCHITECTURE.md)** - Technical architecture details

## Architecture

```
DOC → Parse → SchemaOnce → Chunk → text2cypher → Cypher → Neo4j
```

### Pipeline Flow

1. **Upload & Parse**: Document is uploaded and parsed into clean text (LlamaParse or local parsers)
2. **Chunk**: Long documents are split into manageable chunks (default: 1000 words)
3. **Schema Extraction**: LLM analyzes the full document to extract graph schema (nodes, relationships, properties) - **done once per document**
4. **Cypher Generation**: text2cypher model generates Cypher MERGE statements for each chunk using the extracted schema
5. **Neo4j Ingestion**: Generated Cypher is executed in Neo4j with proper transactions and constraints

## Features

- **Document Parsing**: Supports PDF, DOCX, and TXT files via LlamaParse (cloud) or local tools (`pdf-parse`, `mammoth`)
- **Schema Extraction**: Uses LLM (DeepSeek-R1-Distill or similar) to extract graph schema once per document
- **Cypher Generation**: Uses fine-tuned text2cypher models (`tomasonjo/text2cypher-demo-16bit`) to generate Cypher per chunk
- **Neo4j Integration**: Executes generated Cypher with proper constraints, transactions, and idempotency (MERGE)
- **Scalable**: Handles documents with 100+ pages through intelligent chunking with overlap
- **Flexible LLM Backend**: Supports both Ollama (local) and Hugging Face (cloud) APIs
- **Audit Trail**: Full MongoDB staging layer tracks all steps, errors, and metadata

## Quick Start

1. **Install dependencies:**
   ```bash
   npm install
   ```

2. **Configure environment:**
   ```bash
   cp .env.example .env
   # Edit .env with your MongoDB, Neo4j, and LLM credentials
   ```

   See [ENV_SETUP.md](./ENV_SETUP.md) for detailed configuration.

3. **Setup Neo4j constraints:**
   ```bash
   npm run setup-neo4j
   ```

4. **Start the server:**
   ```bash
   npm start
   ```

5. **Test with a document:**
   ```bash
   npm test ./path/to/your/document.pdf
   ```

## Troubleshooting

### MongoDB connection failed (ECONNREFUSED / querySrv)

- **Atlas cluster paused**: Free tier clusters pause after 60 days. Resume at [cloud.mongodb.com](https://cloud.mongodb.com).
- **Network/firewall**: Corporate networks or VPNs may block MongoDB Atlas. Try from a different network or use local MongoDB.
- **Use local MongoDB**: Set `MONGODB_URI=mongodb://localhost:27017/document-graph-pipeline` and run MongoDB locally (e.g. via Docker).
- **Verify cluster hostname**: Ensure `MONGODB_URI` matches your Atlas cluster (e.g. `cluster0.xxxxx.mongodb.net`).

### Duplicate schema index warning

Remove redundant `index: true` from fields that already have `unique: true` or a compound index.

## API Endpoints

### Document Management

- `POST /documents` - Upload a document (multipart/form-data)
- `GET /documents` - List all documents
- `GET /documents/:id` - Get document details
- `GET /documents/:id/status` - Get processing status with detailed pipeline state
- `POST /documents/:id/process` - Run full pipeline on a document (async)

### Query

- `POST /query` - Query Neo4j with natural language
  ```json
  {
    "docId": "document-id",
    "question": "What are all the companies mentioned?"
  }
  ```

## Project Structure

```
src/
├── server.js                 # Express app entry point
├── config/
│   └── database.js           # MongoDB & Neo4j connections
├── models/
│   ├── Document.js           # Document metadata & status
│   ├── DocumentChunk.js      # Text chunks with status
│   ├── Schema.js             # Extracted graph schema
│   └── ChunkCypherResult.js  # Generated Cypher & execution results
├── services/
│   ├── parsing/
│   │   └── index.js          # Document parsing (LlamaParse/local)
│   ├── schemaExtraction/
│   │   └── index.js          # Schema extraction with LLM
│   ├── cypherGeneration/
│   │   └── index.js          # Cypher generation with text2cypher
│   ├── neo4jIngest/
│   │   └── index.js          # Neo4j ingestion with transactions
│   └── orchestrator.js       # Full pipeline coordinator
├── routes/
│   ├── documents.js           # Document upload/status routes
│   └── query.js              # Natural language query endpoint
├── utils/
│   ├── chunking.js           # Text chunking utilities
│   ├── llm.js                # LLM client (Ollama/HuggingFace)
│   └── logger.js             # Winston logger setup
└── scripts/
    ├── test-pipeline.js      # End-to-end test script
    └── setup-neo4j-constraints.js  # Neo4j constraint setup
```

## Design Decisions

### Why MongoDB for Staging?

- **Audit Trail**: Track every step of the pipeline
- **Resumability**: Can retry failed chunks without re-processing entire document
- **Metadata**: Store parsed text, schemas, generated Cypher for debugging
- **Scalability**: Handle large documents by storing chunks separately

### Why Schema Extraction Once?

- **Consistency**: All chunks use the same schema, ensuring consistent node/relationship types
- **Efficiency**: Only one LLM call for schema vs. per-chunk
- **Quality**: Schema extraction benefits from seeing the full document context

### Why MERGE Instead of CREATE?

- **Idempotency**: Safe to re-run pipeline without creating duplicates
- **Data Quality**: Prevents duplicate nodes/relationships
- **Production-Ready**: Standard practice for graph data ingestion

### Chunking Strategy

- **Overlap**: 100-word overlap prevents losing context at chunk boundaries
- **Size**: 1000 words balances context window with LLM token limits
- **Configurable**: Both size and overlap configurable via environment variables

## Testing

### End-to-End Test

```bash
npm test ./test-docs/sample.pdf
```

This will:
1. Upload the document
2. Run the full pipeline
3. Display results and status

### Manual API Testing

```bash
# Upload
curl -X POST http://localhost:3000/documents \
  -F "file=@document.pdf"

# Process (returns immediately, runs async)
curl -X POST http://localhost:3000/documents/{docId}/process

# Check status
curl http://localhost:3000/documents/{docId}/status

# Query
curl -X POST http://localhost:3000/query \
  -H "Content-Type: application/json" \
  -d '{"docId": "...", "question": "..."}'
```

## Configuration

See [ENV_SETUP.md](./ENV_SETUP.md) for detailed environment variable configuration.

Key settings:
- `CHUNK_SIZE_WORDS`: Words per chunk (default: 1000)
- `CHUNK_OVERLAP_WORDS`: Overlap between chunks (default: 100)
- `AZURE_OPENAI_API_KEY`: Azure OpenAI API key (required)
- `AZURE_OPENAI_ENDPOINT`: Azure OpenAI endpoint URL (required)
- `AZURE_OPENAI_DEPLOYMENT_NAME`: Deployment name (default: gpt-4o)

## Troubleshooting

### MongoDB Connection Issues
- Ensure MongoDB is running: `mongod` or MongoDB Atlas connection string is correct
- Check `MONGODB_URI` in `.env`

### Neo4j Connection Issues
- Ensure Neo4j is running (local) or Aura credentials are correct
- Check `NEO4J_URI`, `NEO4J_USER`, `NEO4J_PASSWORD` in `.env`
- Run `npm run setup-neo4j` to verify connection

### LLM API Issues (Azure OpenAI)
- Verify `AZURE_OPENAI_API_KEY` and `AZURE_OPENAI_ENDPOINT` are set in `.env`
- Ensure your Azure OpenAI deployment (e.g., gpt-4o) is active
- Check logs for rate limits (429) or authentication errors

### Cypher Generation Issues
- Check that schema was extracted successfully
- Verify Azure OpenAI is configured and accessible
- Review generated Cypher in MongoDB `ChunkCypherResult` collection

