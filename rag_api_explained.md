# Understanding `rag-api`: A Guided Architecture and Workflow

**Audience:** Engineers and technical stakeholders  
**Purpose:** Explain how the `rag-api` project handles tenants, document ingestion, extraction, indexing, storage, and querying.  
**Repository:** [BukeLy/rag-api](https://github.com/BukeLy/rag-api)  
**Related projects:** [LightRAG](https://github.com/HKUDS/LightRAG), [RAG-Anything](https://github.com/HKUDS/RAG-Anything), [MinerU](https://github.com/opendatalab/MinerU), [Docling](https://github.com/docling-project/docling), [DeepSeek-OCR](https://github.com/deepseek-ai/DeepSeek-OCR)

---

## 1. Executive summary

`rag-api` is an API layer that combines multi-tenant management, document parsing, multimodal processing, LightRAG indexing, and query endpoints.

The easiest way to understand it is to divide it into four stages:

1. **Tenant and engine management:** the API identifies a tenant and obtains that tenant's LightRAG instance.
2. **Document understanding:** an uploaded file is parsed into text and/or structured multimodal content.
3. **Knowledge indexing:** LightRAG chunks content, creates embeddings, extracts entities and relationships, and persists the resulting data.
4. **Question answering:** LightRAG retrieves relevant chunks and graph information, then gives the assembled context to an LLM.

A crucial distinction:

> The original uploaded PDF is not the same thing as the indexed knowledge. The upload is temporarily staged for processing; the extracted content and its derived representations are what LightRAG indexes.

## 2. Main components and responsibilities

| Component | Responsibility |
|---|---|
| FastAPI application | Exposes HTTP endpoints such as insertion, querying, and task-status polling. |
| `MultiTenantRAGManager` | Creates, caches, and returns a LightRAG instance for a tenant. |
| Tenant ID / workspace | Identifies the logical tenant data space used by LightRAG. |
| Parser selection | Chooses direct text insertion, MinerU, Docling, or DeepSeek-OCR according to the request and file. |
| MinerU | Document-parsing pipeline for layout-aware extraction from PDFs and other supported inputs. |
| Docling | Document conversion and understanding framework; combines format readers and optional/specialized models. |
| DeepSeek-OCR | Vision-language neural model used to recognize and represent document content. |
| RAG-Anything | Multimodal document-processing layer integrated with LightRAG for supported parser paths. |
| LightRAG | Performs chunking, embedding, entity/relation extraction, indexing, and retrieval. |
| DragonflyDB / Redis-compatible storage | Key-value records and document-status records in the configured external-storage setup. |
| Qdrant | Vector records used for semantic retrieval. |
| Memgraph | Knowledge-graph nodes and edges. |

### Component boundary

```text
rag-api (HTTP/API + tenant orchestration)
    |
    +-- parser selection and document task handling
    |      +-- direct text path
    |      +-- MinerU
    |      +-- Docling
    |      +-- DeepSeek-OCR
    |
    +-- RAG-Anything (for supported multimodal parsing paths)
    |
    +-- LightRAG (indexing and retrieval)
           |
           +-- DragonflyDB / Redis-compatible KV and status storage
           +-- Qdrant vector storage
           +-- Memgraph graph storage
```

## 3. Tenant lifecycle and LightRAG instances

### 3.1 Is creating a tenant the first step?

There is no requirement to call a dedicated tenant-creation endpoint before using the insertion or query endpoints. A request supplies a `tenant_id`; when that tenant first needs RAG functionality, the manager can create its LightRAG instance lazily.

The tenant ID is therefore used as a routing/workspace key. It should not be confused with authentication or authorization: an application must independently ensure that a caller is allowed to use the supplied tenant ID.

### 3.2 Lazy instance creation

At application startup, the manager and supporting services are initialized. Tenant-specific LightRAG instances are not all created at startup.

When a request for `company_A` arrives:

1. The manager checks its in-memory instance cache.
2. If `company_A` is already cached, it returns that instance.
3. Otherwise, it loads any tenant-specific configuration and combines it with global defaults.
4. It constructs a LightRAG object, using the tenant ID as its workspace.
5. It initializes the LightRAG storage components.
6. It caches and returns the instance.

```text
Request with tenant_id = company_A
                |
                v
      MultiTenantRAGManager
                |
         Is instance cached?
           /          \
         yes           no
          |             |
          v             v
     Return it     Load tenant/global config
                        |
                        v
                 Construct LightRAG
                 workspace=company_A
                        |
                        v
                Initialize storages
                        |
                        v
                   Cache it
                        |
                        v
                    Return it
```

The manager has a bounded in-memory cache and per-tenant creation coordination. The exact cache policy and defaults should be checked against the deployed repository version; implementation comments and documentation may not always use identical terminology.

### 3.3 What is a LightRAG instance?

A LightRAG instance is the configured RAG engine for a workspace. It is not a database server.

Conceptually, the project supplies settings and functions such as:

- `working_dir`: local working directory.
- `workspace`: tenant's logical data space.
- `llm_model_func`: callable used for LLM requests.
- `embedding_func`: callable used to create vectors.
- concurrency settings for LLM and embedding calls.
- storage backend selections for KV, vector, graph, and document status.

The project can also attach optional reranking and vision functions.

**Important:** Creating a LightRAG instance initializes storage components/connections. It does not mean that a new DragonflyDB, Qdrant, or Memgraph server is created for every tenant.

### 3.4 Tenant isolation: logical versus physical

A workspace is a logical namespace or data-space boundary. Multiple tenant instances can use the same database infrastructure while storing data under different workspaces, depending on the storage backend's implementation.

This is not, by itself, a complete security boundary. Production systems should also enforce tenant authorization at the API layer and verify that every storage operation is scoped correctly.

## 4. Document ingestion: from upload to indexing

The main insertion workflow starts at `POST /insert`.

### 4.1 Upload and task creation

At a high level:

1. The request supplies a tenant ID, a document ID, a file, and optional parser settings.
2. The API validates the upload (including size and non-empty checks).
3. The uploaded file is staged under a temporary path, typically in `/tmp`.
4. The API creates a task record with a pending status.
5. It schedules document processing using FastAPI background-task handling and returns an accepted response with a task ID.

This task path is not automatically equivalent to a separate durable message broker. In the current implementation, the background task is run by the API process; task storage may be configured separately (for example, in memory or Redis-compatible storage).

### 4.2 Four extraction paths

The API has four conceptual ways to get content ready for indexing:

| Path | What it is | Typical result |
|---|---|---|
| Direct insertion | Reads already-textual formats without a document-understanding model. | Text string passed to LightRAG. |
| Docling | Document conversion/understanding framework using format readers and optional/specialized ML models. | Structured `DoclingDocument`, exportable as Markdown/JSON, with layout and document elements. |
| MinerU | Layout-aware document parsing pipeline that orchestrates multiple models and algorithms. | Markdown and structured parsing artifacts, potentially including extracted images and layout information. |
| DeepSeek-OCR | Vision-language neural model for visual document understanding and OCR-like generation. | Generated text/Markdown (and, depending on inference interface, structured/grounding-related output). |

The parser selected depends on the requested parser setting, file type, configuration, and available services. Do not assume every file type or every deployment uses all four paths.

#### A. Direct insertion

For supported plain-text-like formats (such as `.txt`, `.md`, `.markdown`, `.json`, and `.csv` in the current project path), the API reads the file content and passes it directly to LightRAG.

```text
file -> read text -> LightRAG.ainsert(text)
```

No MinerU, Docling, or DeepSeek-OCR inference is required for this route.

#### B. Docling

Docling is a document conversion and understanding framework, not a single neural network. Depending on the input and configuration, it can use ordinary file/text extraction, layout analysis, OCR, and specialized models for document elements such as tables.

It produces a structured document representation (`DoclingDocument`). That representation can be exported to Markdown or JSON. Markdown is convenient for text-centric RAG; the structured representation can preserve richer document structure and metadata.

Docling can be used as a Python library inside a worker. It does not inherently require a separate model-serving API, although it can also be deployed behind a service for independent scaling.

#### C. MinerU

MinerU is a document-parsing pipeline, not one model. It coordinates multiple stages, which can include page/layout analysis, reading-order reconstruction, OCR, table recognition, formula handling, and extraction of figures.

Its output is not necessarily one plain-text string. Depending on version and output mode, it can include Markdown, structured JSON, page/block/layout information, and extracted image files.

MinerU can run locally or be called as a remote service. In `rag-api`, local and remote MinerU paths are supported. The remote path makes the file temporarily accessible to the remote service and then processes the returned parsing result. The current remote path should not be assumed to fall back automatically to local MinerU if the remote request fails.

#### D. DeepSeek-OCR

DeepSeek-OCR is a vision-language neural model. It receives document images/pages together with a task prompt and generates a textual representation of the visual content. It is different from a traditional OCR library that only recognizes character strings.

The `rag-api` integration uses an OCR client, receives Markdown/text output, and sends that content to LightRAG. Self-hosting it means operating model inference infrastructure (typically GPU-backed); alternatively, a compatible hosted inference endpoint can be used.

### 4.3 What is RAG-Anything doing?

RAG-Anything is an additional multimodal processing layer integrated with LightRAG.

In the local MinerU and Docling paths, the project uses RAG-Anything with the tenant's LightRAG instance to process the document. This is useful when parsed documents contain more than running text, such as tables, images, and equations.

The DeepSeek-OCR path in this project is more direct: the OCR client returns Markdown/text, which is passed to `LightRAG.ainsert()`.

The remote MinerU path has its own result-processing step before insertion.

Do not assume that every parser returns identical data or that every path uses RAG-Anything in exactly the same way.

### 4.4 Temporary files and parser artifacts

Keep three storage concepts separate:

- **Original upload:** staged temporarily (for example, under `/tmp`) and removed after processing in the task's cleanup path.
- **Parser artifacts:** intermediate outputs may be written under local output directories (for example, `./output`), depending on parser and configuration.
- **Indexed knowledge:** chunks, vectors, graph data, metadata, and statuses persisted by LightRAG's configured storage backends.

The original PDF is not automatically stored as a permanent binary object in Qdrant, Memgraph, or DragonflyDB by this ingestion flow.

## 5. What LightRAG does with extracted content

Once LightRAG receives text or content from the parser/RAG-Anything integration, it builds searchable representations.

A simplified conceptual pipeline is:

```text
Extracted content
       |
       v
Normalize / prepare content
       |
       v
Split into chunks
       |
       +--------------------------+
       |                          |
       v                          v
Store chunk text             Embed chunks
       |                          |
       v                          v
DragonflyDB / KV                Qdrant
                                  |
                                  |
Chunks -> LLM entity/relationship extraction
                  |
                  v
          Entities and relations
                  |
          +-------+--------+
          |                |
          v                v
      Graph records    Embeddings
          |                |
          v                v
       Memgraph           Qdrant
```

This is a mental model; actual indexing includes additional steps such as deduplication/merging, source tracking, document status updates, caching, and configurable extraction behavior.

### 5.1 Chunking

LightRAG splits the content into smaller passages (chunks). Each chunk has an identifier and metadata linking it to its source document.

Why chunk?

- Retrieval can select only relevant parts of a long document.
- Embedding and LLM context limits are easier to manage.
- Source passages can be associated with extracted graph information.

### 5.2 Chunk embeddings

The embedding function converts each chunk into a numeric vector. The vector is stored in Qdrant with identifying metadata so it can be retrieved by semantic similarity later.

A vector is not the original passage. It is a numerical representation used for similarity search.

### 5.3 Entity and relationship extraction

LightRAG calls the configured LLM to identify entities and relationships described in chunks.

Illustrative source text:

> Tesla acquired SolarCity in 2016.

Possible extraction:

- Entity: Tesla (company)
- Entity: SolarCity (company)
- Relationship: Tesla — acquired —> SolarCity

The LLM may also produce descriptions for entities and relationships. Extraction quality depends on the model, prompts, chunk content, and configuration.

### 5.4 Knowledge graph storage

Entities become graph nodes and relationships become graph edges in Memgraph. Nodes and edges can carry descriptions, properties, and source references.

LightRAG can merge information about the same entity found in multiple chunks/documents, rather than treating every mention as a completely unrelated entity. The precise merge behavior depends on the LightRAG implementation and configuration.

### 5.5 Entity and relationship vectors

LightRAG can also embed entity and relationship descriptions and store those vectors in Qdrant. Therefore Qdrant is not only a chunk-vector index: it can also support semantic retrieval of graph entities and relations.

## 6. What each database stores

In the external-storage configuration used by this project, the main storage roles are:

| Backend | LightRAG storage role | Conceptual contents |
|---|---|---|
| DragonflyDB (Redis-compatible) | KV storage | Document/chunk text and associated key-value records, caches, and other LightRAG state. |
| DragonflyDB (Redis-compatible) | Document-status storage | Document processing/indexing status and related metadata. |
| Qdrant | Vector storage | Embeddings for chunks and, where enabled by the indexing path, entities and relationships; IDs and metadata for retrieval. |
| Memgraph | Graph storage | Entity nodes and relationship edges, with descriptions/properties and source associations. |
| Local working/output directories | Local files/artifacts | Working files and parser outputs; not a substitute for the configured databases. |

The table describes roles, not a guarantee that every internal field is stored exactly as phrased for every LightRAG version. LightRAG's storage schema can evolve.

### 6.1 A concrete example

Suppose a document says:

> Elon Musk joined Tesla in 2004. Tesla acquired SolarCity in 2016.

The conceptual indexed state is:

**DragonflyDB / KV**
- Text of the relevant chunks.
- Chunk identifiers and source-document associations.
- Other key-value records and caches.

**Qdrant**
- Vector for the chunk containing the statements.
- Vectors for extracted entity descriptions (for example, Tesla, Elon Musk, SolarCity).
- Vectors for extracted relationship descriptions (where that indexing path is enabled).

**Memgraph**
- Nodes: Elon Musk, Tesla, SolarCity.
- Edges: Elon Musk — joined —> Tesla; Tesla — acquired —> SolarCity.

**Document status**
- A record for the document and its processing/indexing state.

## 7. Querying: how the three databases work together

The query endpoint receives a tenant ID and a question, obtains that tenant's LightRAG instance, builds a `QueryParam`, and calls LightRAG's query method.

The retrieval strategy depends on the selected mode.

### 7.1 Query modes

| Mode | Main retrieval idea |
|---|---|
| `naive` | Retrieve relevant text chunks using vector similarity. |
| `local` | Focus on specific entities and their nearby graph context. |
| `global` | Focus on broader relationships and graph-level context. |
| `hybrid` | Combine local and global graph retrieval. |
| `mix` | Combine graph-oriented retrieval with direct chunk-vector retrieval. |
| `bypass` | Skip RAG retrieval and send the question directly to the LLM. |

Exact retrieval details are version-dependent; these are the practical mental models for the modes.

### 7.2 Example question

Assume the indexed documents contain:

- Elon Musk joined Tesla in 2004.
- Tesla acquired SolarCity in 2016.

Question:

> How is Elon Musk connected to Tesla, and what did Tesla acquire?

A simplified `mix`-mode workflow:

1. LightRAG may use the LLM to identify low-level keywords (specific entities such as `Elon Musk` and `Tesla`) and high-level keywords (broader topics such as business relationships and acquisitions).
2. It searches Qdrant for relevant entity, relationship, and chunk vectors.
3. It uses the retrieved graph identifiers to obtain graph records and connections from Memgraph.
4. It obtains the original text passages and metadata from KV storage, using source/chunk references and/or chunk search results.
5. It assembles graph information and text passages into a context.
6. It sends the question and context to the query LLM to generate the answer.

```text
Question
   |
   v
Keyword extraction / query preparation
   |
   v
Qdrant: find relevant vectors
   |
   +--------------------------+
   |                          |
   v                          v
Entity / relation hits     Chunk hits
   |                          |
   v                          |
Memgraph: fetch graph         |
records and connections       |
   |                          |
   +-------------+------------+
                 |
                 v
       DragonflyDB KV: fetch
       original chunk text
                 |
                 v
        Assemble query context
                 |
                 v
             Query LLM
                 |
                 v
             Answer
```

The three stores have complementary roles:

- **Qdrant finds candidates by semantic similarity.**
- **Memgraph supplies structured graph records and connections.**
- **DragonflyDB supplies the original text and key-value records needed to ground the answer.**

### 7.3 Not every mode uses all three stores in the same way

For example, `naive` is primarily chunk-vector retrieval and does not need graph traversal as its central strategy. Graph-oriented modes use entity/relationship retrieval and graph data. `mix` combines graph-oriented retrieval and chunk retrieval.

Storage may also be accessed for caches or supporting metadata, so the table below describes the main retrieval path rather than every possible internal read.

| Mode | Qdrant | Memgraph | KV storage |
|---|---|---|---|
| `naive` | Chunk-vector search | Not central | Fetch chunk text |
| `local` | Entity-vector search | Fetch local graph context | Fetch source chunks |
| `global` | Relationship-vector search | Fetch graph context | Fetch source chunks |
| `hybrid` | Entity and relationship search | Fetch graph context | Fetch source chunks |
| `mix` | Graph-related and chunk-vector search | Fetch graph context | Fetch source chunks |

## 8. Task status and asynchronous processing

Insertion is accepted asynchronously. The API returns a task ID rather than waiting for every parsing and indexing operation to finish.

A client can poll the task-status endpoint. The task-status path checks the task record and, while processing is underway, can inspect LightRAG's document status to determine whether indexing has completed or failed.

Conceptually:

```text
POST /insert
     |
     v
Return task_id (accepted)
     |
     v
Background processing
     |
     +--> parse
     +--> index in LightRAG
     |
     v
GET /task/{task_id}
     |
     v
Check task + LightRAG document status
     |
     +--> pending / processing
     +--> completed
     +--> failed
```

This polling behavior is important: task completion is reconciled with LightRAG's document status; receiving an accepted response does not mean the document is already searchable.

## 9. End-to-end workflow summary

### 9.1 Ingestion

```text
Client
  |
  | POST /insert (tenant_id, doc_id, file, parser options)
  v
FastAPI insert endpoint
  |
  +--> validate and stage temporary upload
  |
  +--> create task
  |
  +--> schedule processing
          |
          v
    Get/create tenant LightRAG instance
          |
          v
    Select extraction route
          |
          +--> direct text
          +--> Docling -> RAG-Anything
          +--> local MinerU -> RAG-Anything
          +--> remote MinerU -> result processor
          +--> DeepSeek-OCR -> Markdown/text
                         |
                         v
                 LightRAG indexing
                         |
             +-----------+-----------+
             |           |           |
             v           v           v
          KV store     Qdrant     Memgraph
             |           |           |
             +-----------+-----------+
                         |
                         v
                  Document status
                         |
                         v
                Temporary file cleanup
```

### 9.2 Query

```text
Client
  |
  | POST /query (tenant_id, question, mode)
  v
FastAPI query endpoint
  |
  v
Get/create tenant LightRAG instance
  |
  v
LightRAG retrieval strategy
  |
  +--> Qdrant vector search
  +--> Memgraph graph retrieval (graph modes)
  +--> KV chunk/text retrieval
  |
  v
Assemble context
  |
  v
LLM generates answer
  |
  v
Return answer
```

## 10. Important design and operational considerations

1. **Tenant authorization:** A tenant ID is a routing/workspace identifier, not proof that the caller is authorized to access that tenant. Enforce authorization separately.
2. **Tenant isolation:** Confirm workspace scoping across every configured storage backend. For stronger isolation requirements, consider separate collections, databases, credentials, or deployments.
3. **Background task durability:** FastAPI background tasks run in the API process. For production workloads requiring retries, crash recovery, and independent scaling, evaluate a durable task queue.
4. **Original-file retention:** The insertion path stages uploads temporarily and cleans them up. If users must later download original files, add explicit object storage and retention policies.
5. **Parser deployment:** Docling and MinerU are processing frameworks/pipelines; DeepSeek-OCR is a model. They do not all require the same deployment strategy.
6. **GPU requirements:** OCR, layout, and vision models can require GPU resources depending on configuration. Simple text extraction and some Docling paths can run without a dedicated GPU.
7. **Parsing artifacts:** Decide whether Markdown, JSON, images, and intermediate outputs should be retained, and define cleanup/retention rules.
8. **Indexing cost and latency:** LLM extraction and embedding generation can be expensive. Concurrency limits, caching, batching, and model selection matter.
9. **Document updates/deletions:** Define how re-uploading or deleting a document removes or refreshes its chunks, vectors, graph contributions, and statuses.
10. **Version awareness:** LightRAG, RAG-Anything, MinerU, Docling, and model interfaces evolve. Verify configuration and storage behavior against the exact versions deployed.

## 11. Glossary

| Term | Plain-language definition |
|---|---|
| Tenant | A customer or organization using the platform. |
| Tenant ID | Identifier used to select a tenant's configuration and workspace. |
| Workspace | Logical data namespace used by LightRAG. |
| LightRAG instance | Configured RAG engine associated with a workspace. |
| Parser | Software that reads a source document and extracts its content. |
| OCR | Optical character recognition: extracting text from images or scanned pages. |
| Layout analysis | Identifying document regions such as titles, paragraphs, tables, and figures. |
| RAG-Anything | Multimodal document-processing layer integrated with LightRAG. |
| Chunk | A smaller passage created from a larger document for indexing and retrieval. |
| Embedding | A numeric vector representing text or other content for similarity search. |
| Vector database | Database designed to store and search embeddings. |
| Entity | A named thing or concept, such as a person, company, or place. |
| Relationship | A connection between two entities. |
| Knowledge graph | A graph whose nodes represent entities and edges represent relationships. |
| KV storage | Key-value storage for records such as text, metadata, and caches. |
| Reranking | Reordering retrieved candidates to put the most relevant results first. |
| Multimodal content | Content involving more than plain text, such as images, tables, and equations. |

## 12. Official project references

- `rag-api`: https://github.com/BukeLy/rag-api
- LightRAG: https://github.com/HKUDS/LightRAG
- RAG-Anything: https://github.com/HKUDS/RAG-Anything
- MinerU: https://github.com/opendatalab/MinerU
- MinerU output documentation: https://opendatalab.github.io/MinerU/reference/output_files/
- Docling: https://github.com/docling-project/docling
- Docling documentation: https://docling-project.github.io/docling/
- DeepSeek-OCR: https://github.com/deepseek-ai/DeepSeek-OCR

---

**Presentation tip:** Start with Sections 1–3 to explain the system and tenant model. Use Sections 4–6 to explain ingestion and storage. Then use Section 7 to explain answering questions. The Mermaid diagrams are also provided separately in `rag_api_workflows.mmd` so they can be rendered or edited independently.
