# Implementation Plan: Elasticsearch Chunking Strategies Demo (`demo.ipynb`)

This plan outlines the structure for a Jupyter Notebook (`demo.ipynb`) designed to practically demonstrate chunking strategies for providing context to agentic AI. It compares Elasticsearch's native `semantic_text` (convenient with chunk retrieval via semantic highlighting) with manual external chunking approaches that give AI agents the specific passages they need for grounded reasoning.

## Notebook Structure & Flow

### 1. Prerequisites and Environment Setup
*   **Infrastructure:** Use Terraform to provision an Elastic Cloud Serverless project (ES 9.5.0). The Terraform configuration outputs the API key and Cloud ID to `.env`.
*   **Dependencies:** Install required Python packages via `requirements.txt`: `elasticsearch`, `python-dotenv`, `ipykernel`, `semchunk`, `langchain-text-splitters`.
*   **Connection:** Establish a connection to the Elastic Serverless project using credentials from `.env`.
*   **Sample Data:** Load `assets/sample_doc.md` — a 2,427-word markdown article on "Building Resilient Distributed Systems" with H1/H2/H3 structure.
*   **Shared Query:** All scenarios use the same query: `"How do distributed systems handle failures and recovery?"`
*   **Embedding Model:** Scenarios 3–5 use the `.jina-embeddings-v5-text-small` built-in inference endpoint (1024 dims). Inference calls are batched in groups of 16 (API limit).

### 2. The Native Way: Using `semantic_text`

*   **Scenario 1: Semantic Text — Automatic Chunking**
    *   **Mapping:** Single `semantic_text` field (`content`) with `chunking_settings`: `strategy: "recursive"`, `max_chunk_size: 200`, `separator_group: "markdown"`.
    *   **Ingest:** Index the raw full-text document as a single string.
    *   **Chunk Inspection:** Use the `fields` API with `"format": "chunks"` to expose ES's internal auto-chunking (18 chunks).
    *   **Query:** `semantic` query on `content` field.
    *   **Chunk Retrieval:** Use `"type": "semantic"` highlighting with `"order": "score"` to retrieve the most relevant chunks ranked by semantic similarity.

*   **Scenario 2: Semantic Text — Pre-chunked Arrays**
    *   **Chunking:** Regex split on all markdown headings (H1/H2/H3) via `re.split(r'(?=^#{1,3} )', ...)`, producing one chunk per section.
    *   **Ingest:** Pass the chunk array to the `semantic_text` field.
    *   **Query:** Same `semantic` query with semantic highlighting. Retrieves the most relevant chunks from the pre-defined boundaries.

### 3. The Manual Way: External Chunking & `dense_vector`

All manual scenarios use `dense_vector` with `int8_hnsw` quantization and the `.jina-embeddings-v5-text-small` inference endpoint for embeddings.

> **Note on chunking library choice:** The chunking library and the indexing strategy are independent decisions. Each scenario pairs a specific chunker with a specific indexing pattern for demonstration purposes, but any chunker could be substituted.

*   **Scenario 3: External Chunking — One Document Per Chunk**
    *   **Concept:** Each chunk is an independent top-level document, enabling per-chunk metadata, filtering, and hybrid retrieval.
    *   **Chunking:** `semchunk` with `chunk_size=200` (word-count token counter).
    *   **Mapping:** `dense_vector` (1024 dims, `int8_hnsw`), `parent_id` (keyword), `chunk_index` (integer), `chunk_text` (text).
    *   **Ingest:** Each chunk indexed as an independent top-level document.
    *   **Query:** kNN vector search returning top-5 chunks with scores.
    *   **Dedup:** Field collapsing on `parent_id` deduplicates results when multiple chunks from the same document match.

*   **Scenario 4: Hybrid Search with Reranking**
    *   **Concept:** Query-level capability — queries the same Scenario 3 index, no reindexing needed.
    *   **Note:** In `article.md`, this is not a standalone section — it is presented as "Search - Query 2" within the "External Chunking — One Document Per Chunk" H2 section. The notebook has it as a separate `## Scenario 4` cell.
    *   **Query:** Elasticsearch retriever API with nested structure:
        *   `text_similarity_reranker` (`.jina-reranker-v3`) wrapping:
            *   `linear` retriever combining:
                *   `standard` BM25 `match` on `chunk_text` (weight 0.3)
                *   `knn` vector search on `embedding` (weight 0.7)
        *   Uses `request_timeout=120` for reranker model warmup on first call.

*   **Scenario 5: External Chunking — Nested Chunks**
    *   **Concept:** Chunks live as `nested` objects inside a single parent document, guaranteeing document cohesion and atomic indexing.
    *   **Chunking:** LangChain `RecursiveCharacterTextSplitter` with `chunk_size=800, chunk_overlap=100`.
    *   **Mapping:** `title` (text) + `chunks` (nested) containing `chunk_index`, `chunk_text`, and `dense_vector`.
    *   **Ingest:** Single parent document with array of nested chunk objects.
    *   **Query:** `nested` query wrapping kNN with `inner_hits` to extract matching chunks. Demonstrates the querying complexity vs. Scenario 3's flat kNN.

### 4. Cleanup
*   Terraform destroy to tear down the Elastic Cloud Serverless project.
*   Remove `.env` file.
